# libcamera & V4L2 — Pi4 Camera Pipeline

## 1. Tổng quan

Pi4 camera stack dùng libcamera (modern) thay V4L2 trực tiếp. libcamera abstract hardware ISP pipeline.

```
Camera sensor (IMX219, OV5647, ...)
  │ MIPI CSI-2
  ▼
CSI-2 receiver (Unicam)
  │ Raw Bayer data (DMA to memory)
  ▼
ISP (Image Signal Processor) — Pi4: ISP via VideoCore firmware
  │ Demosaicing, AWB, AE, AF, noise reduction
  ▼
libcamera (IPA — Image Processing Algorithm)
  │ Camera HAL3 interface
  ▼
Android Camera2 API / CameraX
```

---

## 2. V4L2 (Video4Linux2)

### 2.1 V4L2 Overview

```
V4L2 = standard Linux camera/video interface
  /dev/video0, /dev/video1, ...
  
Device types:
  V4L2_CAP_VIDEO_CAPTURE  ← Camera
  V4L2_CAP_VIDEO_OUTPUT   ← Display/encoder
  V4L2_CAP_VIDEO_M2M      ← Memory-to-memory (encoder/decoder)
  V4L2_CAP_META_CAPTURE   ← Statistics (ISP hist, AF stats)
```

### 2.2 V4L2 Buffer Flow

```
                 ┌─────────────────────┐
  VIDIOC_REQBUFS │  Request N buffers  │
                 └─────────────────────┘
                         │
                 ┌─────────────────────┐
  VIDIOC_QBUF   │  Queue buffer       │ ← DMA target
                 └─────────────────────┘
                         │
                 ┌─────────────────────┐
  VIDIOC_STREAMON│  Start capture      │
                 └─────────────────────┘
                         │
                 ┌─────────────────────┐
  VIDIOC_DQBUF  │  Dequeue filled buf │ ← Ready frame
                 └─────────────────────┘
                         │
                         │ Process frame
                         │
                 Re-queue: VIDIOC_QBUF
```

---

## 3. Pi4 Camera Hardware

```
Cameras compatible với Pi4:
  Camera Module v1:  OV5647 (5MP, MIPI CSI-2)
  Camera Module v2:  IMX219 (8MP, MIPI CSI-2)
  Camera Module v3:  IMX708 (12MP, autofocus)
  HQ Camera:         IMX477 (12MP, interchangeable lens)
  
Connection:
  CSI-0: /dev/video0, /dev/video1 (Unicam0)
  CSI-1: /dev/video2, /dev/video3 (Unicam1)
  
Pi4 ISP:
  Hardware ISP trong VideoCore VI firmware
  Accessed via VCHIQ mailbox
  Not exposed as V4L2 M2M directly
```

---

## 4. libcamera Architecture

```
libcamera (userspace library):
  
  Application
    │ libcamera::CameraManager
    ▼
  Pipeline Handler (rpi/vc4)
    │
    ├── Unicam (sensor input, DMA)
    │     /dev/video0 (Bayer raw)
    │     /dev/video1 (embedded data)
    │
    └── ISP (VideoCore via VCHIQ)
          /dev/video12 (output0 - main)
          /dev/video13 (output1 - low-res)
          /dev/video14 (stats output)
    
  IPA (Image Processing Algorithm) in sandboxed process:
    libipa_rpi.so → Algorithms: AE, AWB, AGC, AF, denoise, ...
    
  Controls:
    AnalogueGain, ExposureTime, Brightness, Contrast, ...
```

---

## 5. Android Camera HAL3

```
Camera2 API (app)
  │
  ▼
CameraService (system_server)
  │ HIDL/AIDL
  ▼
Camera HAL3 (libcamera-hal.so)
  │ libcamera C++ API
  ▼
libcamera pipeline handler
  │
  ▼
V4L2 (Unicam + ISP)
  │
  ▼
Hardware (sensor + ISP)
```

```
Pi4 Camera HAL:
  device/brcm/rpi4/camera → libcamera-based HAL
  android.hardware.camera.provider@2.7 AIDL service
  Implements ICameraDevice, ICameraDeviceSession
```

---

## 6. V4L2 Code Example

```c
#include <linux/videodev2.h>
#include <sys/ioctl.h>
#include <fcntl.h>

int fd = open("/dev/video0", O_RDWR);

/* Query capabilities */
struct v4l2_capability cap;
ioctl(fd, VIDIOC_QUERYCAP, &cap);
// cap.capabilities & V4L2_CAP_VIDEO_CAPTURE

/* Set format */
struct v4l2_format fmt = {
    .type = V4L2_BUF_TYPE_VIDEO_CAPTURE,
    .fmt.pix = {
        .width  = 1920,
        .height = 1080,
        .pixelformat = V4L2_PIX_FMT_SRGGB10P, /* Bayer raw 10-bit */
        .field  = V4L2_FIELD_NONE,
    }
};
ioctl(fd, VIDIOC_S_FMT, &fmt);

/* Request buffers (MMAP) */
struct v4l2_requestbuffers reqbuf = {
    .count = 4,
    .type  = V4L2_BUF_TYPE_VIDEO_CAPTURE,
    .memory = V4L2_MEMORY_MMAP,
};
ioctl(fd, VIDIOC_REQBUFS, &reqbuf);

/* Map buffers */
for (int i = 0; i < 4; i++) {
    struct v4l2_buffer buf = { .type = ..., .memory = ..., .index = i };
    ioctl(fd, VIDIOC_QUERYBUF, &buf);
    void *ptr = mmap(NULL, buf.length, PROT_READ|PROT_WRITE,
                     MAP_SHARED, fd, buf.m.offset);
    /* queue buffer */
    ioctl(fd, VIDIOC_QBUF, &buf);
}

/* Start streaming */
int type = V4L2_BUF_TYPE_VIDEO_CAPTURE;
ioctl(fd, VIDIOC_STREAMON, &type);

/* Capture loop */
struct v4l2_buffer buf = { .type = ..., .memory = V4L2_MEMORY_MMAP };
ioctl(fd, VIDIOC_DQBUF, &buf);  /* wait for frame */
// process frame at buffers[buf.index]
ioctl(fd, VIDIOC_QBUF, &buf);   /* requeue */
```

---

## 7. Debug Camera

```bash
# List video devices
adb shell ls -la /dev/video*
# crw-rw---- 1 root video 81, 0 /dev/video0  (Unicam)
# crw-rw---- 1 root video 81,12 /dev/video12 (ISP output)

# Camera capabilities
adb shell v4l2-ctl --device=/dev/video0 --all

# List formats
adb shell v4l2-ctl -d /dev/video0 --list-formats-ext

# Capture frame (on device with camera attached)
adb shell v4l2-ctl -d /dev/video12 \
    --set-fmt-video=width=1920,height=1080,pixelformat=RGBP \
    --stream-mmap --stream-to=/sdcard/frame.raw \
    --stream-count=1

# Camera HAL test
adb shell cameraserver
adb shell dumpsys media.camera

# libcamera test (if shell accessible)
adb shell libcamera-vid -t 3000 --width 1920 --height 1080 \
    -o /sdcard/test.h264
```

---

## 8. Sensor Control (V4L2 Controls)

```bash
# List available controls
adb shell v4l2-ctl -d /dev/video0 --list-ctrls

# Set exposure
adb shell v4l2-ctl -d /dev/video0 \
    --set-ctrl=exposure=5000      # 5000 µs

# Set gain
adb shell v4l2-ctl -d /dev/video0 \
    --set-ctrl=analogue_gain=200  # 2.0x

# Set white balance
adb shell v4l2-ctl -d /dev/video0 \
    --set-ctrl=red_balance=1500 \
    --set-ctrl=blue_balance=1200

# Horizontal flip
adb shell v4l2-ctl -d /dev/video0 \
    --set-ctrl=horizontal_flip=1
```

---

## 9. ISP Pipeline on Pi4

```
Raw Bayer (from sensor) → ISP stages:
  
  1. Black Level Correction (BLC)
  2. Lens Shading Correction (LSC)
  3. Auto White Balance (AWB) — gain per channel
  4. Colour Filter Array Interpolation (Demosaic)
  5. Colour Correction Matrix (CCM)
  6. Gamma Curve
  7. Colour Space Conversion (CSC) → YUV
  8. Noise Reduction (denoise)
  9. Sharpening
  10. Output scaling (to final resolution)
  
Pi4: ISP runs in VideoCore VI firmware
libcamera IPA algorithms feed parameters to ISP
```

---

## 10. File quan trọng trong source

| File | Mô tả |
|------|-------|
| `kernel/common/drivers/media/platform/bcm2835/` | Unicam driver |
| `kernel/common/drivers/media/v4l2-core/` | V4L2 core |
| `kernel/common/include/uapi/linux/videodev2.h` | V4L2 API |
| `kernel/common/drivers/media/i2c/imx219.c` | IMX219 sensor driver |
| `hardware/interfaces/camera/` | Camera HAL AIDL |
| `device/brcm/rpi4/camera/` | Pi4 Camera HAL |
