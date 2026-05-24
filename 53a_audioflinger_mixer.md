# AudioFlinger MixerThread — Deep Dive: Cycle, FastMixer, và Pipeline

Đây là phần đi sâu vào cơ chế mixing của AudioFlinger: từ threadLoop() timing, AudioMixer internals, FastMixer real-time thread, đến buffer model và latency analysis.

---

## 1. Kiến trúc tổng thể

```
App (AudioTrack)
  │  shared memory (ashmem / audio_track_cblk_t)
  ▼
AudioFlinger::PlaybackThread (MixerThread)
  │  mTracks[] — all tracks in session
  │  prepareTracks_l() + AudioMixer::process()
  ▼
FastMixer (SCHED_FIFO RT thread)
  │  StateQueue<FastMixerState>
  ▼
AudioStreamOut → HAL (ALSA/BluetoothAudio)
  │  ALSA write → kernel ALSA driver → DAC
  ▼
Speaker / DAC
```

---

## 2. MixerThread::threadLoop() — Main Rendering Loop

```cpp
// frameworks/av/services/audioflinger/Threads.cpp
bool AudioFlinger::MixerThread::threadLoop()
{
    // ...
    while (!exitPending())
    {
        // Bước 1: Lấy lock và chuẩn bị tracks
        { // AutoMutex scope
            Mutex::Autolock _l(mLock);
            prepareTracks_l(&tracksToRemove);
        }

        // Bước 2: Mix
        if (mMixerStatus == MIXER_TRACKS_READY) {
            mAudioMixer->process();   // ← Đây là core mixing
        }

        // Bước 3: Write to output (HAL)
        mBytesWritten += mixBufferSize;
        int bytesWritten = (int)mOutput->stream->write(
                mOutput->stream, mMixBuffer, mixBufferSize);

        // Bước 4: Sleep nếu không có track active
        if (sleepTime > 0) {
            usleep(sleepTime);
        }
    }
}
```

### 2.1 Sleep Time vs Active Wait Time

```
┌────────────────────────────────────────────────────────┐
│ MixerThread timing model                               │
│                                                        │
│ ┌──────────┐ prepareTracks ┌────────┐ write HAL       │
│ │  sleep   │──────────────►│  mix   │──────────►      │
│ │ (μs)     │               │process │                  │
│ └──────────┘               └────────┘                  │
│                                                        │
│ sleepTime = 0 → active: có track đang active           │
│ sleepTime > 0 → idle: không có track, tiết kiệm CPU    │
└────────────────────────────────────────────────────────┘
```

```cpp
// Timing calculation
uint32_t minFrames = 1;
if ((mixerStatus == MIXER_TRACKS_ENABLED) && framesWritten != 0) {
    minFrames = mNormalFrameCount;
}
// mNormalFrameCount = output buffer size / frame size
// Ví dụ: 48kHz stereo 16bit, buffer=4096 bytes → 1024 frames → 21.3ms period
```

---

## 3. prepareTracks_l() — Chuẩn Bị Track Trước Khi Mix

```cpp
AudioFlinger::PlaybackThread::mixer_state
AudioFlinger::MixerThread::prepareTracks_l(
        Vector<sp<Track>> *tracksToRemove)
{
    mixer_state mixerStatus = MIXER_IDLE;
    size_t count = mActiveTracks.size();

    for (size_t i = 0; i < count; i++) {
        const sp<Track> t = mActiveTracks[i];
        Track* const track = t.get();

        // Kiểm tra underrun: producer chưa cung cấp đủ data
        size_t framesReady = track->framesReady();
        if (framesReady == 0) {
            // Underrun: ghi lặp buffer cuối hoặc silence
            if (track->isStopping_1()) {
                track->mState = TrackBase::STOPPING_2;
            } else {
                track->mAudioTrackServerProxy->tallyUnderrunFrames(
                        desiredFrames);
            }
            // Track không mix lần này
            continue;
        }

        // Set mixing parameters cho AudioMixer
        mAudioMixer->setParameter(
                name,
                AudioMixer::TRACK,
                AudioMixer::MAIN_BUFFER,
                (void *)track->mainBuffer());

        mAudioMixer->setParameter(
                name,
                AudioMixer::TRACK,
                AudioMixer::AUX_BUFFER,
                (void *)track->auxBuffer());

        // Volume: linear gain với ramp
        mAudioMixer->setParameter(
                name,
                AudioMixer::VOLUME,
                AudioMixer::VOLUME0,
                &left);  // float 0.0 - 1.0

        // Resampling nếu cần (src rate ≠ output rate)
        mAudioMixer->setParameter(
                name,
                AudioMixer::RESAMPLE,
                AudioMixer::SAMPLE_RATE,
                (void *)(uintptr_t)reqSampleRate);

        mAudioMixer->enable(name);
        mixerStatus = MIXER_TRACKS_READY;
    }
    return mixerStatus;
}
```

---

## 4. AudioMixer Internals

### 4.1 Track State trong AudioMixer

```cpp
// frameworks/av/services/audioflinger/AudioMixer.h
struct Track {
    uint32_t    sampleRate;         // source sample rate
    uint32_t    channelMask;        // AUDIO_CHANNEL_OUT_STEREO, etc.
    audio_format_t format;          // AUDIO_FORMAT_PCM_16_BIT, etc.

    // Buffer
    void*       in;                 // pointer to source data
    AudioBufferProvider* bufferProvider;  // cung cấp chunks khi mix

    // Volume với ramp
    union {
        int16_t volume[MAX_NUM_CHANNELS];   // Q4.12 fixed point
        int32_t volumeRL;                   // packed left+right
    };
    int32_t     prevVolume[MAX_NUM_CHANNELS];  // previous volume (để ramp)
    int32_t     volumeInc[MAX_NUM_CHANNELS];   // per-sample increment

    // Resampler
    AudioResampler* resampler;      // NULL nếu không cần resample
    uint32_t    frameCount;         // frames to mix per call

    // Hook function pointer — core của dispatch
    hook_t      hook;               // process__genericResampling hoặc process__nop
    const void* mIn;                // current position in input
};
```

### 4.2 Hook Dispatch — Compile-Time Optimization

```cpp
// AudioMixer chọn hook dựa trên combination của:
// - format (16bit, float, ...)
// - channel count (mono, stereo, multi)
// - resampling (yes/no)
// - volume (unity=1.0 vs non-unity)

void AudioMixer::setParameterHook(int name, ...) {
    Track& track = mState.tracks[name];

    // Chọn hook tối ưu
    if (track.needsRamp()) {
        if (track.resampler != NULL) {
            track.hook = track_t::MIXTYPE_MULTI_SAVEONLY
                       ? process__TwoTracks16BitsStereoNoResampling
                       : process__genericResampling;
        } else {
            track.hook = process__genericNoResampling;
        }
    }
    // Unity volume + no resample: fastest path
    if (volumesAreUnity && track.resampler == NULL) {
        track.hook = process__nop;  // nếu disabled
    }
}

// process() gọi tất cả enabled hooks
void AudioMixer::process() {
    mState.hook(&mState);  // → process__genericResampling hoặc tương tự
}
```

### 4.3 Volume Ramp — Smooth Volume Change

```cpp
// Thay vì set volume ngay lập tức (gây pop/click):
// - Mỗi frame, volume được interpolate tuyến tính từ prevVolume → targetVolume

// Trong mixing loop:
int32_t vl = track.volume[0];        // target volume left
int32_t incL = track.volumeInc[0];   // per-frame increment
int32_t prevL = track.prevVolume[0]; // current volume

// Với mỗi frame:
int32_t l = (*in++) << 12;           // 16bit sample → Q4.12
l = mulAdd(l, prevL, auxLeft);       // l * vol + aux accumulator
prevL += incL;                        // ramp step

// Sau N frames (ramp complete):
if (rampSteps == 0) {
    track.prevVolume[0] = track.volume[0];  // ramp done
}
```

---

## 5. AudioResampler — Chất Lượng vs Latency

```cpp
// frameworks/av/media/libaudioprocessing/AudioResampler.cpp
AudioResampler* AudioResampler::create(
        audio_format_t format,
        int inChannelCount,
        int32_t sampleRate,
        src_quality quality)
{
    switch (quality) {
    case LOW_QUALITY:
        // Linear interpolation: nhanh, chất lượng thấp
        return new AudioResamplerOrder1(format, inChannelCount, sampleRate);

    case MED_QUALITY:
        // Cubic interpolation
        return new AudioResamplerCubic(format, inChannelCount, sampleRate);

    case HIGH_QUALITY:
        // Sinc filter: chậm nhất, chất lượng cao nhất
        return new AudioResamplerSinc(format, inChannelCount, sampleRate);

    case VERY_HIGH_QUALITY:
        // Sinc với filter taps nhiều hơn
        return new AudioResamplerSinc(format, inChannelCount, sampleRate,
                                      AudioResamplerSinc::MOST_TAPS);
    }
}
```

**Khi nào resampling xảy ra:**
- App record/play ở 44100 Hz, HAL output là 48000 Hz → cần upsample 44100→48000
- Ratio = 48000/44100 = 1.0884... → fractional stepping
- Sinc filter: convolution với windowed sinc function

```
Input:  [s0, s1, s2, ...] @44100 Hz
Output: [o0, o1, o2, ...] @48000 Hz

Với mỗi output sample:
  phase accumulator += inputRate/outputRate  (fixed point Q31)
  integer part → input index
  fractional part → interpolation weight
  o[n] = Σ s[i+k] * sinc_filter[k * frac]
```

---

## 6. FastMixer — Real-Time Audio Thread

### 6.1 Thread Priority và Setup

```cpp
// frameworks/av/services/audioflinger/FastMixer.cpp
void FastMixer::onStart() {
    // Set SCHED_FIFO với priority cao nhất cho audio
    struct sched_param param;
    param.sched_priority = kRTPriorityMin;  // thường = 2

    if (sched_setscheduler(0, SCHED_FIFO, &param) != 0) {
        ALOGE("sched_setscheduler SCHED_FIFO failed: %s", strerror(errno));
    }

    // CPU affinity: pin to specific core nếu được config
    // Giảm cache misses, tránh migration overhead
}
```

### 6.2 FastMixerState — Lock-Free State Passing

```
MixerThread (non-RT)              FastMixer (RT)
─────────────────────────────────────────────────
                  StateQueue<FastMixerState>
                  ┌─────────────────────┐
write new state → │  mMutatingIndex     │ ← read committed state
(atomic swap)     │  mReadIndex         │   (atomic load)
                  └─────────────────────┘

Không dùng mutex vì mutex có thể cause priority inversion:
  - MixerThread (low priority) holds mutex
  - FastMixer (RT) blocks on mutex
  - Priority inversion: RT thread bị delay bởi low priority thread
```

```cpp
// StateQueue implementation (lock-free single-producer, single-consumer)
template<typename T>
class StateQueue {
    T   mStates[kN];           // circular buffer of states
    volatile int32_t mNext;    // atomic index to next state
    volatile int32_t mAck;     // acknowledgment from consumer

public:
    // Producer (MixerThread): prepare new state
    T* begin() {
        return &mStates[mWritingIndex];
    }

    // Commit: atomic write mNext (memory_order_release)
    void end(bool didModify) {
        if (didModify) {
            android_atomic_release_store(mWritingIndex, &mNext);
        }
    }

    // Consumer (FastMixer): get latest state
    const T* poll() {
        int32_t next = android_atomic_acquire_load(&mNext);
        if (next != mReadIndex) {
            mReadIndex = next;
            return &mStates[mReadIndex];
        }
        return NULL;  // no new state
    }
};
```

### 6.3 FastMixer::onWork() — Hot Path

```cpp
void FastMixer::onWork()
{
    const FastMixerState * const current = mCurrentState;
    const FastMixerState::Command command = current->mCommand;

    // Check for new state
    const FastMixerState *newState = mSQ.poll();
    if (newState != NULL) {
        // State changed: update tracks, output, etc.
        onStateChange();
        mCurrentState = newState;
    }

    if (command == FastMixerState::MIX_WRITE) {
        // Mix tất cả FastTracks
        for (size_t i = 0; i < FastMixerState::kMaxFastTracks; i++) {
            const FastTrack* track = &current->mFastTracks[i];
            if (track->mBufferProvider == NULL) continue;

            // Lấy buffer từ track
            AudioBufferProvider::Buffer buffer;
            buffer.frameCount = mFrameCount;
            track->mBufferProvider->getNextBuffer(&buffer);

            // Mix vào output: 16-bit mixing assembly-optimized
            AudioMixer::process_l(mMixBuffer, buffer.raw,
                                   buffer.frameCount,
                                   track->mGain);

            track->mBufferProvider->releaseBuffer(&buffer);
        }

        // Write to HAL
        ssize_t written = mOutputSink->write(mMixBuffer, mFrameCount);
        if (written < 0) {
            // HAL error — handle underrun
            mNativeFramesWrittenButNotPresented += mFrameCount;
        }

        // Update presentation timestamp
        mNativeFramesWrittenButNotPresented -=
                nativeFramesWrittenButNotPresented;
    }
}
```

### 6.4 FastTrack vs Normal Track

```
FastTrack:
  - Direct path to FastMixer (bypasses MixerThread mixing)
  - SCHED_FIFO priority cho AudioTrack thread
  - Cần: AUDIO_OUTPUT_FLAG_FAST | AUDIO_OUTPUT_FLAG_RAW
  - Latency: ~10-20ms (output buffer size / sample rate)
  - Không support: effects, resampling (phải exact sample rate)
  - Dùng cho: game audio, keyboard feedback, voice call

Normal Track:
  - Đi qua MixerThread với full feature set
  - Latency: ~50-100ms (deep buffer)
  - Support: effects chain, resampling, volume ramp
  - Dùng cho: music playback
```

---

## 7. Track Buffer Model — Shared Memory

### 7.1 audio_track_cblk_t — Control Block

```c
// frameworks/av/include/private/media/AudioTrackShared.h
struct audio_track_cblk_t {
    // Atomic indices — không dùng mutex để tránh cross-process blocking
    volatile int32_t    mFront;     // read pointer (server side: AudioFlinger)
    volatile int32_t    mRear;      // write pointer (client side: AudioTrack)
    volatile int32_t    mFlush;     // flush count (atomically incremented)
    volatile int32_t    mStop;      // stop sequence number

    uint32_t            mSampleRate; // sample rate
    uint16_t            mVolumeLR;   // packed volume (for fast tracks)
    uint8_t             mSendLevel;  // aux send level
    uint8_t             mPad1;

    // Notification frames: trigger client when X frames consumed
    uint32_t            mNotificationFramesReq;  // requested
    uint32_t            mNotificationFramesAct;  // actual
};
```

### 7.2 Shared Memory Layout

```
ashmem region (shared between AudioTrack client và AudioFlinger server)
┌──────────────────────────────────────────────────────┐
│  audio_track_cblk_t  (control block, ~64 bytes)      │
│    mFront, mRear (atomic read/write pointers)        │
├──────────────────────────────────────────────────────┤
│  Audio data buffer                                   │
│  [frame0][frame1]...[frameN]  (circular)             │
│                                                      │
│  Client (AudioTrack):                                │
│    writes at mRear → advances mRear                  │
│  Server (AudioFlinger):                              │
│    reads at mFront → advances mFront                 │
└──────────────────────────────────────────────────────┘
```

### 7.3 AudioTrackServerProxy — Server-Side Access

```cpp
// AudioTrackServerProxy: AudioFlinger dùng để đọc data từ shared mem
class AudioTrackServerProxy : public ServerProxy {
public:
    // Số frames sẵn sàng để đọc
    size_t framesReady() {
        int32_t front = android_atomic_acquire_load(&mCblk->mFront);
        int32_t rear = android_atomic_acquire_load(&mCblk->mRear);
        // Circular buffer: rear - front (mod buffer size)
        ssize_t filled = rear - front;
        if (filled < 0) filled += mFrameCount;
        return (size_t) filled;
    }

    // Lấy buffer pointer để đọc
    status_t obtainBuffer(Buffer* buffer, ...) {
        // Returns pointer to next readable chunk
        // Advances mFront sau khi đọc
    }

    // Detect underrun
    void tallyUnderrunFrames(uint32_t underrunFrames) {
        mCblk->mServer += underrunFrames;  // notify client
        android_atomic_or(CBLK_UNDERRUN, &mCblk->mFlags);
    }
};
```

### 7.4 Underrun Detection Flow

```
AudioFlinger MixerThread                 Client AudioTrack
─────────────────────────────────────────────────────────
prepareTracks_l():
  framesReady() == 0?
  → YES: underrun!
     tallyUnderrunFrames(N)
     mCblk->mFlags |= CBLK_UNDERRUN
     [fill silence hoặc repeat last buffer]
                            ←────────────────────────────
                            AudioTrack::obtainBuffer():
                              checks CBLK_UNDERRUN flag
                              calls onUnderrunOccurred()
                              application callback: AudioTrack.ERROR_DEAD_OBJECT?
```

---

## 8. Mixing Pipeline Timing

### 8.1 Frame Count Calculation

```cpp
// MixerThread::readOutputParameters_l()
mNormalFrameCount = (mOutput->stream->get_buffer_size(mOutput->stream)
                     / mFrameSize);
// get_buffer_size() → HAL xác định based on hardware capability
// RPi4 ALSA: thường 4096 bytes → 1024 frames @stereo 16bit
// 1024 frames @ 48kHz = 21.3ms period

// Deep buffer (music): thường 4096+ frames = 85ms+
// Low latency: 256 frames = 5.3ms

mNormalFrameCount = max(mNormalFrameCount,
        roundup((uint32_t)(mSampleRate * kMinNormalMixBufferSizeMs / 1000)));
// kMinNormalMixBufferSizeMs = 20ms
```

### 8.2 Latency Stack

```
Audio latency = app buffer + mixer buffer + HAL buffer + kernel DMA buffer
                            + DSP processing + analog path

Typical deep buffer path (music):
  App AudioTrack buffer:    100ms  (large, avoid CPU wake)
  MixerThread buffer:        21ms
  HAL/ALSA kernel buffer:    10ms
  ────────────────────────────────
  Total input-to-speaker:  ~131ms  (acceptable for music)

Low latency path (game/voice):
  App FastTrack buffer:       5ms
  FastMixer buffer:           5ms
  HAL buffer:                 5ms
  ────────────────────────────────
  Total:                    ~15ms  (good for interactive)
```

### 8.3 ALSA Write Path

```cpp
// HAL write → ALSA pcm_write → kernel
// Trong audio HAL (hardware/libhardware/modules/audio/ hoặc vendor HAL):

static ssize_t out_write(struct audio_stream_out *stream,
                          const void* buffer, size_t bytes)
{
    struct stub_stream_out *out = (struct stub_stream_out *)stream;

    // ALSA write
    int ret = pcm_write(out->pcm, buffer, bytes);
    // pcm_write() → ioctl(SNDRV_PCM_IOCTL_WRITEI_FRAMES)
    // Kernel ALSA: DMA transfer → hardware FIFO → DAC
    if (ret != 0) {
        // Xử lý ALSA error: -EPIPE = xrun (buffer underrun trong kernel)
        pcm_prepare(out->pcm);  // recover
    }
    return bytes;
}
```

---

## 9. Effect Chains

### 9.1 EffectChain Structure

```cpp
// EffectChain: ordered list of audio effects applied to a session
class EffectChain : public RefBase {
    Vector<sp<EffectModule>> mEffects;  // ordered list of effects
    int     mSessionId;     // AUDIO_SESSION_OUTPUT_MIX = -1 (applies to all)
                            // hoặc app's AudioSession ID

    // Processing buffer
    int32_t *mInBuffer;     // input 32-bit buffer
    int32_t *mOutBuffer;    // output 32-bit buffer
};

// Effect processing order:
// 1. Track effects (per-track): EQ, reverb per-stream
// 2. Session effects: applied to all tracks in session
// 3. Output mix effects (sessionId = -1): applied to final mix
```

### 9.2 process_l() trong EffectChain

```cpp
void AudioFlinger::EffectChain::process_l()
{
    // Copy mixed audio vào effect input buffer
    memcpy(mInBuffer, mMixBuffer, mFrameCount * sizeof(int32_t) * mChannelCount);

    // Apply each effect in chain
    for (size_t i = 0; i < mEffects.size(); i++) {
        sp<EffectModule> effect = mEffects[i];
        if (effect->state() == EffectModule::ACTIVE) {
            effect->process();
            // effect->process() → HAL effect library
            // ioctl hoặc shared memory với effect DSP
        }
    }

    // Copy output back to mix buffer
    memcpy(mMixBuffer, mOutBuffer, mFrameCount * sizeof(int32_t) * mChannelCount);
}
```

### 9.3 Effect Timing: Pre-Mix vs Post-Mix

```
Pre-mix effects (IIR_EQUALIZER, BASS_BOOST):
  → Applied per-track trước khi mix
  → Chạy với track's sample rate

Post-mix effects (REVERB, VIRTUALIZER):
  → Applied sau khi mix tất cả tracks
  → Chạy với output sample rate

OutputMix effects (sessionId = AUDIO_SESSION_OUTPUT_MIX):
  → Applied ngay trước khi write to HAL
  → Global: ảnh hưởng tất cả audio
```

---

## 10. Bluetooth A2DP vs Wired ALSA

### 10.1 A2DP Output Thread

```cpp
// BluetoothAudio HAL (thay thế legacy A2dpAudioInterface)
// framework: packages/modules/Bluetooth/system/audio_bluetooth_hw/

// A2DP có latency cao hơn vì:
// 1. SBC/AAC encoding: ~10-20ms
// 2. Bluetooth radio latency: ~20-40ms
// 3. ACL packet scheduling: jitter 7.5ms intervals
// 4. Remote device buffer: ~50ms

// A2DP output thread dùng riêng, không dùng MixerThread:
// AudioFlinger tạo A2dpOffloadThread hoặc DirectOutputThread
// Không có FastMixer cho Bluetooth path
```

### 10.2 So sánh Paths

```
ALSA Wired:                          Bluetooth A2DP:
  MixerThread                          MixerThread
  → AudioMixer::process()              → AudioMixer::process()
  → FastMixer (optional)               → (no FastMixer)
  → ALSA HAL                           → BluetoothAudio HAL
  → pcm_write()                        → SBC/AAC encoder
  → DMA → DAC                          → BT stack (btif_a2dp)
  Latency: 15-50ms                     → ACL packets
                                       → Remote speaker
                                       Latency: 100-300ms
```

### 10.3 A2DP Sink Audio Synchronization

```cpp
// A2DP timing: AudioFlinger phải cấp data đúng pace của BT
// BluetoothAudioSession::GetPresentationPosition() →
//   trả về timestamp của sample hiện đang play trên remote device
// AudioFlinger dùng để sync presentation timestamp

// Tránh: gửi data quá nhanh → BT controller buffer overflow
//         gửi quá chậm → BT underrun → audible glitch
```

---

## 11. Systrace Analysis

### 11.1 Key Tracepoints

```
adb shell atrace --async_start -b 32768 -c audio

Trong systrace sẽ thấy:
  AudioOut_D: MixerThread main loop (40ms)
    ├─ mixer_track_prep: prepareTracks_l()
    ├─ mixer_process: AudioMixer::process() (~2-5ms)
    └─ write: HAL write (~0.5ms nếu non-blocking)

  FastMixer: 5ms period (nếu có FastTrack)
    ├─ FM_getNextBuffer: lấy track buffer
    ├─ FM_mix: actual mixing
    └─ FM_write: HAL write

  AudioTrack latency markers:
    AudioTrack::write() [app] → ... → speaker
    
  Underrun indicators:
    "underrun" tag trong AudioOut_D thread
    AudioTrack::onUnderrunOccurred
```

### 11.2 ALSA Xrun Detection

```bash
# Xem ALSA xrun stats
adb shell cat /proc/asound/card0/pcm0p/sub0/status
# state: RUNNING / XRUN
# hw_ptr, appl_ptr, avail_max

# Đo latency thực tế
adb shell oboe-latency-test
# Sends click, measures echo latency (round-trip)
# Typical: 30-70ms wired, 150-300ms Bluetooth
```

---

## 12. RPi4-Specific: bcm2835-audio HAL

```
RPi4 audio path:
  AudioFlinger MixerThread
  → ALSA HAL (alsa_audio.so)
  → pcm_write() → ALSA kernel
  → bcm2835-audio driver (vc4-hdmi hoặc headphone jack)
  → VCHIQ → VideoCore IV firmware
  → HDMI audio / analog out

Vấn đề RPi4:
  - bcm2835-audio có high jitter vì đi qua VCHIQ (VideoCore IPC)
  - Latency có thể lên 100ms+ cho HDMI audio
  - Giải pháp: USB audio adapter (direct ALSA, thấp latency hơn)
  - Hoặc: external audio HAL với ALSA direct (không qua bcm2835-audio)

adb shell cat /proc/asound/cards
# → bcm2835 Headphones hoặc vc4-hdmi
adb shell dumpsys media.audio_flinger | grep -A5 "Output thread"
```

---

## 13. Debug Commands

```bash
# Xem tất cả AudioFlinger threads và tracks
adb shell dumpsys media.audio_flinger

# Output example:
# Output thread 0x7f8b4a2000 type 0 (MIXER):
#   I/O handle: 4
#   TID: 3421
#   Standby: no
#   Sample rate: 48000 Hz
#   Frame count: 1024
#   Normal frame count: 1024
#   Channel count: 2
#   Format: 0x1 (16-bit PCM)
#   Mixer buffer: 0x7f8b4a4000
#   Effect buffer: 0x7f8b4a6000
#   Track 0:
#     id=10 client=1234 pid=5678
#     state=ACTIVE
#     frameCount=8192 frameCountToBeReady=4096
#     sampleRate=44100 speed=1.000 pitch=1.000
#     Underruns: server=5 user=0

# Xem FastMixer state
adb shell dumpsys media.audio_flinger | grep -A20 "FastMixer"

# Theo dõi underrun
adb shell logcat -s AudioFlinger:V | grep -i underrun

# Đo latency
adb shell arecord -D hw:0,0 -f S16_LE -r 48000 -c 2 -d 5 /data/test.wav &
adb shell aplay -D hw:0,0 /data/sine.wav
# → measure round-trip với click signal
```

---

## 14. Full Data Flow Diagram

```
App AudioTrack                          AudioFlinger MixerThread
──────────────────────────────────────────────────────────────────────
write(pcm_data)
  → mProxy->obtainBuffer()              prepareTracks_l()
  → write to shared memory                ← framesReady() check
  → advance mRear (atomic)                ← obtainBuffer() from shared mem
  → if mNotificationFrames reached:
       wake up AudioFlinger            AudioMixer::process()
                                         for each track:
                                           resample (if needed)
                                           volume ramp
                                           mix into output buffer
                                         
                           ┌─────────────────────────────────────┐
                           │          StateQueue                  │
                           │  MixerThread writes FastMixerState  │
                           │  FastMixer reads (lock-free)        │
                           └─────────────────────────────────────┘

                                         FastMixer (SCHED_FIFO)
                                           onWork():
                                             getNextBuffer per FastTrack
                                             mix into output
                                             write to HAL (ALSA)
                                           
                                         HAL → pcm_write()
                                           → ioctl WRITEI_FRAMES
                                           → kernel ALSA DMA
                                           → hardware FIFO → DAC
                                           → Sound output 🔊
```
