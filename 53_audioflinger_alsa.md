# AudioFlinger & ALSA — Android Audio Stack

## 1. Tổng quan

```
App (MediaPlayer, AudioTrack, AudioRecord)
  │ AudioManager API
  ▼
AudioPolicyService  ← routing decisions (which device plays what)
  │
AudioFlinger        ← mixing, effects, latency management
  │ Audio HAL (AIDL)
  ▼
Audio HAL implementation (tinyalsa / alsa-lib)
  │ PCM read/write
  ▼
ALSA kernel driver (/dev/snd/pcmC0D0p)
  │
Hardware (I2S DAC, HDMI audio, USB audio)
```

---

## 2. AudioFlinger Architecture

```
AudioFlinger (system_server companion, runs in audioserver process):

  ├── MixerThread (for speakers/headphones)
  │     ├── Track 1 (App1 audio, 44100Hz, stereo)
  │     ├── Track 2 (App2 audio, 48000Hz, mono)
  │     └── AudioMixer → resample + mix → output buffer
  │
  ├── RecordThread (for microphone)
  │     ├── RecordTrack 1 (App recording)
  │     └── AudioMixer (effects chain)
  │
  ├── DuplicatingThread (for simultaneous output devices)
  │
  └── OffloadThread (hardware offload for MP3/AAC)

Period:
  MixerThread runs on timer: period = buffer_size / sample_rate
  Default: 48000 Hz, 256 frames → 5.3ms period
  Low latency: can go down to 96 frames → 2ms
```

---

## 3. Audio Track Types

```java
// Java API: AudioTrack
AudioTrack track = new AudioTrack.Builder()
    .setAudioFormat(new AudioFormat.Builder()
        .setEncoding(AudioFormat.ENCODING_PCM_16BIT)
        .setSampleRate(44100)
        .setChannelMask(AudioFormat.CHANNEL_OUT_STEREO)
        .build())
    .setTransferMode(AudioTrack.MODE_STREAM)
    .setBufferSizeInBytes(bufferSize)
    .build();

track.play();
track.write(pcmData, 0, pcmData.length);

// Audio Attributes (routing hint)
AudioAttributes attrs = new AudioAttributes.Builder()
    .setUsage(AudioAttributes.USAGE_MEDIA)           // music
    .setContentType(AudioAttributes.CONTENT_TYPE_MUSIC)
    .build();
```

---

## 4. AudioPolicy — Routing

```
AudioPolicyService quyết định:
  Which output device to use?
    → Speaker vs Headphone vs Bluetooth vs HDMI
  Audio focus arbitration:
    → Music ducks khi phone call
    → Navigation TTS ducks music
  Strategies:
    STRATEGY_MEDIA         → speakers/headphones
    STRATEGY_PHONE         → earpiece/headset
    STRATEGY_SONIFICATION  → notification sounds
    STRATEGY_ACCESSIBILITY → TTS, screen reader
```

```java
// Audio focus management
AudioManager am = getSystemService(AudioManager.class);
AudioFocusRequest request = new AudioFocusRequest.Builder(
    AudioManager.AUDIOFOCUS_GAIN)
    .setOnAudioFocusChangeListener(listener)
    .build();
int result = am.requestAudioFocus(request);
// AUDIOFOCUS_REQUEST_GRANTED or AUDIOFOCUS_REQUEST_DELAYED
```

---

## 5. ALSA — Kernel Audio

```
ALSA (Advanced Linux Sound Architecture):
  /dev/snd/controlC0     ← mixer controls (volume, mux)
  /dev/snd/pcmC0D0p      ← PCM playback (Card0, Device0, Playback)
  /dev/snd/pcmC0D0c      ← PCM capture (microphone)
  /dev/snd/pcmC1D0p      ← Second audio card (USB audio)

Hardware abstractions:
  Card → Device → Subdevice
  Card 0: bcm2835 (HDMI + headphone)
  Card 1: USB audio (if connected)
```

```bash
# List ALSA cards
adb shell cat /proc/asound/cards
#  0 [b1             ]: bcm2835_hdmi - bcm2835 HDMI 1
#                       bcm2835 HDMI 1
#  1 [Headphones     ]: bcm2835_headp - bcm2835 Headphones
#                       bcm2835 Headphones

# PCM devices
adb shell cat /proc/asound/devices
adb shell cat /proc/asound/pcm

# Mixer controls
adb shell tinymix --card 0     # list all controls

# Play audio (TinyALSA)
adb shell tinyplay /sdcard/test.wav --card 0 --device 0
adb shell tinyplay /sdcard/test.wav -D 0 -d 0 -r 44100 -c 2 -b 16

# Record audio
adb shell tinycap /sdcard/capture.wav -D 0 -d 0 -r 44100 -c 1 -b 16 -T 5

# Volume control
adb shell tinymix 'Master Playback Volume' 100
```

---

## 6. TinyALSA vs ALSA-lib

```
Full ALSA (alsa-lib):
  libasound.so → complex, dmix/dsnoop plugins
  Used on desktop Linux
  Too heavy for Android

TinyALSA:
  Minimal ALSA wrapper for Android
  system/media/audio_utils/tinymixer.cpp
  tinyplay, tinycap, tinymix tools
  
Android uses TinyALSA via Audio HAL:
  audio_hw.c → pcm_open() / pcm_write() / pcm_read()
  from external/tinyalsa/
```

---

## 7. Pi4 Audio Setup

```
Pi4 audio outputs:
  1. HDMI audio (bcm2835-hdmi)
     → 2ch/8ch PCM, compressed (AC3/DTS passthrough)
  2. 3.5mm analog headphone (bcm2835-headphones)
     → PWM-based DAC (low quality, for testing)
  3. USB audio (class-compliant)
     → Best quality for serious use
  4. I2S external DAC (e.g., HiFiBerry)
     → Connects to Pi GPIO pins 18-21

config.txt audio settings:
  dtparam=audio=on      ← Enable onboard audio
  dtoverlay=hifiberry-dacplus  ← Enable I2S DAC HAL
  hdmi_drive=2          ← Force HDMI audio output (CEA mode)
```

```bash
# Xem audio routing
adb shell dumpsys audio | head -50

# Test HDMI audio
adb shell "echo 1 > /sys/class/sound/card0/power/autosuspend_delay_ms"
adb shell tinyplay /sdcard/test.wav -D 0

# Check HDMI audio ELD (supported formats)
adb shell cat /proc/asound/card0/eld#0.0
```

---

## 8. Audio HAL (AIDL)

```
Android 12+: Audio HAL via AIDL
  android.hardware.audio.service → audioserver

Audio HAL flow:
  AudioFlinger opens device via HAL:
    IDevice::openOutputStream() → IStreamOut
    IDevice::openInputStream()  → IStreamIn
  
  Writes PCM: IStreamOut::write(buffer, bytes)
  Reads PCM:  IStreamIn::read(buffer, bytes)

Pi4 Audio HAL:
  device/brcm/rpi4/audio/ → audio HAL for Pi4
  Uses TinyALSA internally
  Routes to HDMI or headphones based on AudioPolicy
```

---

## 9. Audio Latency

```
Latency chain:
  App write() → AudioTrack buffer → AudioFlinger mix → HAL → ALSA → HW

Typical latencies:
  Normal:      ~100ms (large buffers, good for music)
  Low latency: ~20ms  (AUDIO_OUTPUT_FLAG_LOW_LATENCY)
  Fast mixer:  ~5ms   (AUDIO_OUTPUT_FLAG_FAST, for games)

Measure latency:
adb shell dumpsys audio | grep -i latency
adb shell dumpsys media.audio_flinger | grep "Latency"
```

---

## 10. Debug AudioFlinger

```bash
# AudioFlinger state
adb shell dumpsys media.audio_flinger

# AudioPolicy state
adb shell dumpsys media.audio_policy

# Audio device list
adb shell dumpsys audio | grep "Available"

# Audio effects
adb shell dumpsys media.audio_flinger | grep "Effect"

# Underrun counter (audio glitches)
adb shell dumpsys media.audio_flinger | grep underrun

# HAL log
adb logcat -s audio_hw audio_route AudioFlinger AudioPolicyService
```

---

## 11. File quan trọng trong source

| File | Mô tả |
|------|-------|
| `frameworks/av/services/audioflinger/AudioFlinger.cpp` | AudioFlinger |
| `frameworks/av/services/audioflinger/Threads.cpp` | MixerThread, RecordThread |
| `frameworks/av/services/audiopolicy/AudioPolicyService.cpp` | Policy routing |
| `hardware/interfaces/audio/aidl/` | Audio HAL AIDL interface |
| `device/brcm/rpi4/audio/` | Pi4 audio HAL |
| `external/tinyalsa/` | TinyALSA library |
| `kernel/common/sound/arm/bcm2835.c` | BCM2835 ALSA driver |
| `kernel/common/sound/core/pcm.c` | ALSA PCM core |
