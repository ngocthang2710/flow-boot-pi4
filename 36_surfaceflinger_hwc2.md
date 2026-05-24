# SurfaceFlinger & HWC2 — Android Display Compositor

## 1. Tổng quan

SurfaceFlinger là compositor của Android — nhận buffers từ mọi app/process và tổng hợp thành 1 frame để hiển thị.

```
App1 buffer ──┐
App2 buffer ──┤ SurfaceFlinger ──► HWC2 HAL ──► Display
StatusBar  ──┘    (compositor)    (hardware)
```

---

## 2. Kiến trúc Display Pipeline

```
App (UI thread)
  │  Canvas/OpenGL draw vào Surface buffer
  ▼
BufferQueue (producer ↔ consumer)
  │  Double/Triple buffering
  ▼
SurfaceFlinger (system_server process)
  │  Composite tất cả layers
  │  Sync với VSync
  ▼
HWC2 (Hardware Composer HAL)
  │  Offload composition lên hardware
  │  Trên Pi4: drm_hwcomposer
  ▼
DRM/KMS (vc4 driver)
  │  Atomic page flip
  ▼
HDMI/DSI Display
```

---

## 3. BufferQueue — Producer/Consumer

```cpp
/* Producer (App): */
// Surface → acquire buffer
ANativeWindowBuffer *buffer;
ANativeWindow_dequeueBuffer(window, &buffer, &fence);
// ... draw vào buffer ...
ANativeWindow_queueBuffer(window, buffer, fence);

/* Consumer (SurfaceFlinger): */
// BufferItem item;
// mSurfaceFlingerConsumer->acquireBuffer(&item, presentWhen);
// ... composite ...
// mSurfaceFlingerConsumer->releaseBuffer(item.mSlot, item.mFrameNumber, ...);

/* BufferQueue internals:
   DEQUEUED: app đang draw
   QUEUED:   app done, chờ SF consume
   ACQUIRED: SF đang composite
   FREE:     available cho app */
```

---

## 4. Layer Types

| Layer | Mô tả |
|-------|-------|
| `BufferLayer` | App UI buffer (OpenGL/Vulkan rendered) |
| `EffectLayer` | Color fill, dimming effects |
| `ContainerLayer` | Group children layers |
| `DisplayColorLayer` | Color correction |

---

## 5. VSync — Synchronization

```
VSync signal (60Hz = 16.67ms, 120Hz = 8.33ms):

[VSync]──────────────────────[VSync]──────────────────[VSync]
   │                            │                        │
   │◄── App renders frame 1 ───►│◄── App renders frame 2►│
   │                            │                        │
   │         SF composites frame 1          SF composites frame 2
   │         Display shows frame 1          Display shows frame 2

Choreographer API:
  Choreographer.getInstance().postFrameCallback(callback)
  callback.doFrame(frameTimeNanos)  ← Gọi mỗi VSync
```

---

## 6. HWC2 — Hardware Composer

HWC2 offload composition lên hardware thay vì GPU:

```
SurfaceFlinger submit layers lên HWC2:
  Layer 1: StatusBar (opaque, small) → HW overlay
  Layer 2: App content (opaque)     → HW overlay
  Layer 3: Keyboard (transparent)   → GPU blend → HW overlay

HWC2 quyết định:
  - Layer nào dùng hardware overlay (zero GPU cost)
  - Layer nào cần GPU composite (fallback)
  - Combine result → display
```

```bash
# Pi4 dùng drm_hwcomposer
# device/brcm/rpi4/device.mk:
# com.android.hardware.graphics.composer.drm_hwcomposer

adb shell dumpsys SurfaceFlinger | head -100
# SurfaceFlinger global state:
# [A] HWC layers:
#   Layer 1: StatusBar  DEVICE (hardware overlay)
#   Layer 2: App        CLIENT (GPU composite)
```

---

## 7. Fence Synchronization

```
App render done → GPU fence → SF acquire buffer
SF composite done → retire fence → display shows frame

Timeline:
CPU: [draw cmd] → GPU → [fence signal] → SF → [display]
                              ↑ sync point
```

---

## 8. Debug SurfaceFlinger

```bash
# Dump SF state
adb shell dumpsys SurfaceFlinger

# Layer list
adb shell dumpsys SurfaceFlinger | grep "Layer\|Surface"

# Frame timing
adb shell dumpsys SurfaceFlinger --latency <layer_name>

# GPU composition rate (GPU được gọi bao nhiêu lần)
adb shell dumpsys SurfaceFlinger | grep "GPU Composition"

# Enable GPU overdraw visualization
adb shell setprop debug.hwui.overdraw show
adb shell setprop debug.hwui.overdraw false

# Gfxinfo (frame timing per app)
adb shell dumpsys gfxinfo com.myapp
# Total frames rendered: 1234
# Janky frames: 5 (0.40%)
# 50th percentile: 5ms
# 90th percentile: 8ms
# 95th percentile: 15ms
# 99th percentile: 32ms

# Real-time frame timing
adb shell dumpsys gfxinfo com.myapp framestats
```

---

## 9. Presentation Time

```
Frames có 3 timestamps:
1. App draws (lateWake)
2. SF composites (postCompositionWake)
3. Display shows (presentTime)

Jank = frame không kịp VSync deadline
     → frame bị hiển thị muộn → "dropped frame"

Android 12+: Jank classification
  DISPLAY_HAL    : HWC quá chậm
  APP             : App render quá chậm
  SF_SCHEDULING  : SF scheduling issue
```

---

## 10. Pi4 Display Stack

```
Pi4 Display:
  App (OpenGL ES via Mesa)
    │ Surface/EGL
    ▼
  SurfaceFlinger
    │ layers
    ▼
  drm_hwcomposer (AIDL HAL)
    │ DRM atomic commit
    ▼
  vc4 DRM driver (kernel)
    │ HVS (Hardware Video Scaler)
    ▼
  HDMI output (vc4_hdmi)
```

```bash
# Xem DRM planes
adb shell cat /sys/kernel/debug/dri/0/state

# VSync events
adb shell cat /sys/kernel/debug/dri/0/vblank_count

# Display mode
adb shell cat /sys/class/drm/card0-HDMI-A-1/modes
```

---

## 11. Gralloc & Display Buffers

```
Buffer formats trên Pi4:
  HAL_PIXEL_FORMAT_RGBA_8888    → 4 bytes/pixel (GPU rendering)
  HAL_PIXEL_FORMAT_RGBX_8888    → 4 bytes/pixel (no alpha)
  HAL_PIXEL_FORMAT_RGB_565      → 2 bytes/pixel (low memory)
  HAL_PIXEL_FORMAT_YCBCR_420_888 → Camera/Video (4:2:0)

Pi4 gralloc: minigbm_gbm_mesa
  → DRM/GBM buffers
  → Shared với GPU và display via DMA-BUF
```

---

## 12. File quan trọng trong source

| File | Mô tả |
|------|-------|
| `frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp` | SF compositor (12000+ lines) |
| `frameworks/native/services/surfaceflinger/Layer.cpp` | Layer management |
| `frameworks/native/libs/gui/BufferQueue.cpp` | Producer-consumer buffer queue |
| `frameworks/native/libs/gui/Surface.cpp` | ANativeWindow implementation |
| `hardware/interfaces/graphics/composer/aidl/` | HWC2 AIDL interface |
| `kernel/common/drivers/gpu/drm/vc4/` | VC4 display driver |
| `device/brcm/rpi4/` → `drm_hwcomposer` | Pi4 HWC2 HAL |
