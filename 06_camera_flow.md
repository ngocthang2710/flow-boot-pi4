# Camera Flow — Android 16 on Raspberry Pi 4

## Hai loại camera trên RPi4

| | External Camera (USB) | CSI Camera (ribbon cable) |
|---|---|---|
| Thiết bị | USB webcam, UVC | IMX219, OV5647, IMX477, IMX708… |
| Bus | USB → `/dev/video*` (UVC) | MIPI CSI-2 → Unicam → BCM2835 ISP |
| HAL | `ExternalCameraProvider` (AOSP) | `libcamera` + IPA |
| Config | `external_camera_config.xml` | `camera_hal.yaml` |

---

## Camera2 API (chung cho cả 2 loại)

```
App
 └── CameraManager.openCamera()
      └── CameraCaptureSession.setRepeatingRequest(captureRequest)
           └── Binder IPC (ICameraDeviceUser)
                └── CameraService (cameraserver process)
                     └── Camera3Device::processCaptureRequest()
                          └── CameraProviderManager → ICameraProvider AIDL
```

---

## Luồng 1 — USB Camera (External HAL)

```
┌──────────────────────────────────────────────────────────────────┐
│  CAMERASERVER                                                    │
│  CameraService → CameraDeviceClient → Camera3Device             │
│    → CameraProviderManager → ICameraProvider/external/0 AIDL    │
└──────────────────────────────┬───────────────────────────────────┘
                               │ AIDL IPC
┌──────────────────────────────▼───────────────────────────────────┐
│  EXTERNAL CAMERA HAL (vendor APEX)                               │
│  hardware/interfaces/camera/device/default/                      │
│                                                                  │
│  ExternalCameraDeviceSession::processCaptureRequest()            │
│                                                                  │
│  [1] configureV4l2StreamLocked():                                │
│       ioctl(fd, VIDIOC_S_FMT, ...)    ← set format/resolution   │
│       ioctl(fd, VIDIOC_REQBUFS, ...)  ← allocate buffers        │
│       ioctl(fd, VIDIOC_QBUF, ...)     ← queue buffers           │
│       ioctl(fd, VIDIOC_STREAMON, ...) ← start streaming         │
│                                                                  │
│  [2] dequeueV4l2FrameLocked():                                   │
│       ioctl(fd, VIDIOC_DQBUF, ...)    ← get filled buffer       │
│                                                                  │
│  [3] OutputThread::processRequest():                             │
│       MJPEG → libyuv::MJPGToNV12()   ← decode nếu cần          │
│       YUV scale/crop → output streams                           │
│       JPEG encode: libyuv + libjpeg_turbo                        │
│                                                                  │
│  [4] ioctl(fd, VIDIOC_QBUF) ← re-queue buffer                   │
└──────────────────────────────┬───────────────────────────────────┘
                               │ /dev/video0 (V4L2 UVC)
┌──────────────────────────────▼───────────────────────────────────┐
│  LINUX KERNEL — uvcvideo driver → USB bus → USB Webcam           │
└──────────────────────────────────────────────────────────────────┘
```

### Output formats hỗ trợ

```xml
<!-- external_camera_config.xml -->
<FpsList>
  <Limit width="640"  height="480"  fpsBound="30.0"/>
  <Limit width="1280" height="720"  fpsBound="15.0"/>
  <Limit width="1920" height="1080" fpsBound="10.0"/>
</FpsList>
<MaxJpegBufferSize bytes="3145728"/>  <!-- 3MB -->
<NumVideoBuffers count="4"/>
```

---

## Luồng 2 — CSI Camera (libcamera + BCM2835 ISP)

```
┌──────────────────────────────────────────────────────────────────┐
│  CAMERASERVER → ICameraProvider/internal/0 AIDL                  │
└──────────────────────────────┬───────────────────────────────────┘
                               │ AIDL IPC
┌──────────────────────────────▼───────────────────────────────────┐
│  LIBCAMERA HAL PROCESS                                           │
│  device/brcm/rpi4/camera/libcamera/                              │
│                                                                  │
│  libcamera CameraHal → libcamera::Camera::queueRequest()         │
│    ↓                                                             │
│  PipelineHandlerVc4 (vc4.cpp)                                    │
│                                                                  │
│  ┌── UNICAM (Image receiver) ──────────────────────────────┐    │
│  │  V4L2VideoDevice: /dev/video0 (unicam)                  │    │
│  │  → nhận raw Bayer data từ sensor qua MIPI CSI-2         │    │
│  │  → DMA buffer → memory (zero-copy)                      │    │
│  └──────────────────────┬───────────────────────────────────┘    │
│                          │ DMA buffer                             │
│  ┌── BCM2835 ISP ────────▼───────────────────────────────────┐   │
│  │  V4L2VideoDevice: /dev/video13-16 (bcm2835-isp)           │   │
│  │  Input:   Bayer RAW (RGGB/GRBG/BGGR)                     │   │
│  │  Output0: YUV420/RGB (full res → video/preview)           │   │
│  │  Output1: YUV420 (scaled → thumbnail)                    │   │
│  │  Stats:   histogram + AWB stats → IPA feedback            │   │
│  │                                                           │   │
│  │  ISP pipeline:                                            │   │
│  │   Bayer → Demosaic → Black level → Lens shading          │   │
│  │   → CCM (3x3 matrix) → Gamma → Denoise → Sharpen         │   │
│  │   → YUV conversion                                        │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌── IPA (Image Processing Algorithm) ─────────────────────┐    │
│  │  ipa/rpi/vc4/ ← isolated process (security)              │    │
│  │                                                          │    │
│  │  Mỗi frame:                                             │    │
│  │   controller/rpi/agc.cpp      ← AGC/AEC (auto exposure) │    │
│  │   controller/rpi/awb_bayes.cpp ← AWB (Bayesian)         │    │
│  │   controller/rpi/ccm.cpp      ← Color Correction Matrix │    │
│  │   controller/rpi/contrast.cpp ← gamma/tone mapping      │    │
│  │   controller/rpi/sdn.cpp      ← spatial denoise         │    │
│  │   controller/rpi/black_level.cpp ← black level          │    │
│  │                                                          │    │
│  │  → đọc Stats → tính exposure/gain/CCM/AWB gains          │    │
│  │  → gửi V4L2 controls lại ISP + sensor qua I2C           │    │
│  └──────────────────────────────────────────────────────────┘   │
└──────────────────────────────┬───────────────────────────────────┘
                               │ V4L2 + DMA buf
┌──────────────────────────────▼───────────────────────────────────┐
│  LINUX KERNEL                                                    │
│  bcm2835-isp (ISP) + unicam (MIPI CSI-2) + I2C (sensor control) │
└──────────────────────────────┬───────────────────────────────────┘
                               │ MIPI CSI-2 (data) + I2C (control)
┌──────────────────────────────▼───────────────────────────────────┐
│  SENSOR MODULE (ribbon cable)                                    │
│  camera_hal.yaml — supported sensors:                            │
│   IMX219  (8MP,  Pi Camera V2)  ← I2C addr 0x10                │
│   IMX477  (12MP, Pi HQ Camera)  ← I2C addr 0x1a                │
│   IMX708  (12MP, Pi Camera V3)  ← I2C addr 0x1a                │
│   OV5647  (5MP,  Pi Camera V1)  ← I2C addr 0x36                │
│   IMX500  (12MP, AI Camera)     ← I2C addr 0x1a                │
│   IMX296/IMX519/OV64A40...                                      │
└──────────────────────────────────────────────────────────────────┘
```

---

## 3A Feedback Loop (IPA)

```
Sensor output (RAW)
    ↓ Unicam → DMA → BCM2835 ISP
ISP Stats output (histogram, AWB stats)
    ↓ V4L2 → IPA process
IPA tính toán:
  AGC: lux meter + histogram → shutter time, analog gain
  AWB: Bayesian inference → R/B channel gains
  CCM: temperature → 3x3 color matrix
    ↓ IPC → PipelineHandlerVc4
V4L2 controls → sensor I2C (shutter, gain)
              → ISP registers (CCM, gamma, denoise)
    ↓ áp dụng tại frame tiếp theo (pipeline latency ~2 frames)
```

---

## Ignored video devices (external_camera_config.xml)

```xml
<!-- Internal V4L2 devices bị ignore bởi External Camera HAL -->
<ignore>
  <id>10</id>  <!-- bcm2835-isp output -->
  <id>11</id>  <!-- bcm2835-isp output -->
  <id>12</id>  <!-- bcm2835-isp output -->
  <id>13</id>  <!-- bcm2835-isp input -->
  <!-- ... etc -->
</ignore>
```
