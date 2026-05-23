# Audio Flow — Android 16 on Raspberry Pi 4

## Ngõ ra âm thanh trên RPi4

| Output | Hardware | Driver | Config |
|--------|----------|--------|--------|
| **3.5mm jack** (default) | BCM2835 built-in DAC (PWM) | `bcm2835-audio` | `persist.vendor.audio.device=jack` |
| **HDMI** | VC4 GPU HDMI audio | `vc4-hdmi` + IEC958 | `persist.vendor.audio.device=hdmi0` |
| **USB DAC** | USB Audio Class | `snd-usb-audio` | `persist.vendor.audio.device=dac` |
| **Bluetooth A2DP** | CYW43455 | BT stack | Xem `05_a2dp_audio_flow.md` |

---

## Software Stack

```
App
 └── AudioTrack / MediaPlayer
      └── Shared memory (mmap) + Binder IPC
           └── AudioFlinger (mediaserver)
                └── MixerThread → mix PCM từ nhiều app
                     └── AIDL IPC (android.hardware.audio.core)
                          └── Audio HAL (vendor APEX)
                               └── StreamPrimary → StreamAlsa
                                    └── tinyalsa (jack/USB)
                                        hoặc alsa-lib (HDMI)
                                         └── /dev/snd/pcm*
                                              └── ALSA kernel
                                                   └── bcm2835-audio / vc4-hdmi
```

---

## Luồng phát audio chi tiết (output)

```
┌──────────────────────────────────────────────────────────────┐
│  APP PROCESS                                                 │
│  AudioTrack.write(pcmBuffer)                                 │
│  native AudioTrack (C++) via JNI                             │
└──────────────────────────────┬───────────────────────────────┘
                               │ Shared memory (mmap)
                               │ Control: Binder IPC
┌──────────────────────────────▼───────────────────────────────┐
│  MEDIASERVER (frameworks/av/)                                │
│                                                              │
│  AudioPolicyService                                          │
│   → route decision dựa trên audio_policy_configuration.xml  │
│   → primary module: Speaker (3.5mm) hoặc HDMI               │
│                                                              │
│  AudioFlinger::MixerThread                                   │
│   → lấy PCM từ các AudioTrack                               │
│   → resample nếu cần (44100→48000Hz)                        │
│   → mix thành 1 buffer PCM stereo 16-bit 48kHz              │
│                                                              │
│  StreamHalInterface (AIDL client wrapper)                    │
└──────────────────────────────┬───────────────────────────────┘
                               │ AIDL IPC
                               │ android.hardware.audio.core.IModule
┌──────────────────────────────▼───────────────────────────────┐
│  AUDIO HAL PROCESS (vendor APEX: com.android.hardware.audio.rpi) │
│  device/brcm/rpi4/audio/                                     │
│                                                              │
│  StreamOutPrimary::transfer()                                │
│    ↓ kế thừa StreamAlsa                                     │
│  StreamAlsa::start()                                         │
│    → alsa::openProxyForAttachedDevice()                      │
│    → alsa_device_proxy.c                                     │
│                                                              │
│  ┌── Jack (3.5mm) ──────────────────────────────────────┐   │
│  │  tìm card: /proc/asound/card*/id → "Headphones"      │   │
│  │  pcm_open(card, device=0, PCM_OUT, config)  [tinyalsa]│   │
│  │  pcm_write(proxy->pcm, buffer, size)                 │   │
│  └──────────────────────────────────────────────────────┘   │
│  ┌── HDMI ──────────────────────────────────────────────┐   │
│  │  persist.vendor.audio.device=hdmi0                   │   │
│  │  snd_pcm_open("default:CARD=vc4hdmi0") [alsa-lib]   │   │
│  │  → IEC958 subframe packing (S/PDIF format)           │   │
│  │  snd_pcm_writei(pcm_alsa, buffer, frames)            │   │
│  └──────────────────────────────────────────────────────┘   │
└──────────────────────────────┬───────────────────────────────┘
                               │ /dev/snd/pcmC0D0p
┌──────────────────────────────▼───────────────────────────────┐
│  LINUX KERNEL — ALSA subsystem                               │
│  ┌── Jack → bcm2835-audio → BCM2835 PWM DAC → 3.5mm      ┐  │
│  └── HDMI → vc4-hdmi driver → HDMI audio channel          ┘  │
└──────────────────────────────────────────────────────────────┘
```

---

## Audio Policy Configuration (audio_policy_configuration.xml)

```xml
<module name="primary" halVersion="3.0">
  <mixPort name="primary output" role="source" flags="AUDIO_OUTPUT_FLAG_PRIMARY">
    <profile format="AUDIO_FORMAT_PCM_16_BIT" samplingRates="48000"
             channelMasks="AUDIO_CHANNEL_OUT_STEREO"/>
  </mixPort>
  <devicePort tagName="Speaker" type="AUDIO_DEVICE_OUT_SPEAKER"/>
  <route type="mix" sink="Speaker" sources="primary output"/>
</module>
<!-- + USB, r_submix, bluetooth modules -->
```

## Card Detection (StreamPrimary::getCardId)

```cpp
// Đọc /proc/asound/card*/id để tìm card phù hợp:
// jack  → card có id "Headphones"
// dac   → card không phải "Headphones" / "vc4hdmi"
// hdmi0 → dùng snd_pcm_open("default:CARD=vc4hdmi0") qua alsa-lib
// Fallback: card 0 nếu không tìm thấy
```

## Stub Driver

Khi không có hardware audio: `useStubStream()` = true
→ HAL giả lập bằng `usleep(bufferDurationUs)` thay vì ghi vào hardware.

Enabled khi `ro.boot.audio.tinyalsa.simulate_input=true` (vendor.prop).

## Luồng ghi âm (input)

```
Microphone → bcm2835-audio → /dev/snd/pcmC0D0c
  → pcm_read() → StreamAlsa → Audio HAL
  → AudioFlinger RecordThread
  → Shared memory → AudioRecord (Java) → App
```

> **Lưu ý**: `ro.boot.audio.tinyalsa.simulate_input=true` trong vendor.prop → input thực tế bị simulate (RPi4 không có built-in mic thực sự).
