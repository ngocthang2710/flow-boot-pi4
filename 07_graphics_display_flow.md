# Graphics / Display Flow — Android 16 on Raspberry Pi 4

## Hardware

- **SoC**: BCM2711
- **GPU**: **V3D (VideoCore VI 3D)** — xử lý OpenGL ES + Vulkan
- **Display controller**: VC4 + HVS (Hardware Video Scaler) — blend layers
- **Output**: 2x HDMI (vc4hdmi0, vc4hdmi1)
- **Bus**: `/dev/dri/card0` (display), `/dev/dri/renderD128` (GPU render)

## Software Stack

```
App (OpenGL ES / Vulkan / Canvas)
 └── EGL (libEGL_mesa.so)
      └── Mesa V3D Gallium (OpenGL ES) / v3dv Vulkan driver (mesa3d-rpi)
           └── DRM_IOCTL_V3D_SUBMIT_CL → /dev/dri/renderD128
                └── V3D kernel driver

Gralloc (buffer allocator)
 └── minigbm vc4 backend (external/minigbm/vc4.c)
      └── DRM_IOCTL_VC4_CREATE_BO → GEM buffer
           └── dma-buf fd (shared across processes)

SurfaceFlinger
 └── Scheduler (VSYNC-driven, 60fps)
      └── CompositionEngine
           └── HWC (drm_hwcomposer) AIDL HAL
                └── drmModeAtomicCommit() → /dev/dri/card0
                     └── vc4 kernel driver → HVS → HDMI
```

---

## Luồng mỗi frame chi tiết

```
┌──────────────────────────────────────────────────────────────────┐
│  APP PROCESS                                                     │
│                                                                  │
│  ┌── Canvas / CPU ─────────────────────────────────────────┐    │
│  │  View.onDraw() → Skia → software rasterize (CPU)        │    │
│  └─────────────────────────────────────────────────────────┘    │
│  ┌── OpenGL ES / Vulkan ───────────────────────────────────┐    │
│  │  glDrawArrays() / vkQueueSubmit()                       │    │
│  │  → EGL → Mesa V3D / v3dv                                │    │
│  │  → QPU shader compilation (Broadcom QPU ISA)            │    │
│  │  → Tile-Based Deferred Rendering:                       │    │
│  │     Pass 1 BINNING: vertex → phân loại vào tiles 32x32  │    │
│  │     Pass 2 RENDER: mỗi tile → fragment shader → tile    │    │
│  │     buffer (on-chip SRAM) → write DRAM                  │    │
│  │  → DRM_IOCTL_V3D_SUBMIT_CL → /dev/dri/renderD128        │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  Surface::dequeueBuffer()  ← lấy buffer trống (GEM bo)          │
│  [vẽ vào buffer]                                                 │
│  Surface::queueBuffer()    ← trả buffer vào BufferQueue         │
└──────────────────────────────────────────────────────────────────┘
                    │ BufferQueue (IGraphicBufferProducer Binder IPC)
┌───────────────────▼──────────────────────────────────────────────┐
│  SURFACEFLINGER PROCESS                                          │
│  frameworks/native/services/surfaceflinger/                      │
│                                                                  │
│  [1] VSYNC signal từ HWC (60fps = 16.7ms/frame)                 │
│       Scheduler::onVsync() → schedule commit + composite         │
│                                                                  │
│  [2] BufferQueue::acquireBuffer()                                │
│       → lấy frame mới nhất từ mỗi app layer                     │
│                                                                  │
│  [3] HWC::validateDisplay()          ← AIDL IPC                 │
│       drm_hwcomposer phân loại mỗi Layer:                        │
│       DEVICE: đủ điều kiện DRM hardware overlay plane            │
│       CLIENT: cần GPU composite (GLES fallback)                  │
│                                                                  │
│  [4] Client composition (nếu có CLIENT layers)                  │
│       CompositionEngine dùng GLES composite → client target buf  │
│       → EGL → Mesa V3D → DRM_IOCTL_V3D_SUBMIT_CL               │
│                                                                  │
│  [5] HWC::presentDisplay()           ← AIDL IPC                 │
│       truyền buffer handles xuống drm_hwcomposer                 │
└───────────────────┬──────────────────────────────────────────────┘
                    │ AIDL IPC
                    │ android.hardware.graphics.composer3.IComposer
┌───────────────────▼──────────────────────────────────────────────┐
│  DRM_HWCOMPOSER HAL (vendor APEX)                                │
│  external/drm_hwcomposer/                                        │
│                                                                  │
│  HwcDisplay::presentDisplay()                                    │
│   → DrmAtomicStateManager::TryCommit()                           │
│                                                                  │
│  Build DRM atomic property set:                                  │
│   CRTC:      mode=1920x1080@60 (vendor.hwc.drm.force_mode)      │
│   Plane[0]:  fb_id=GEM handle, src/dst rect, zpos               │
│   Plane[1]:  fb_id=GEM handle, ...  (nếu DEVICE layer)          │
│   Connector: crtc_id → kết nối với vc4hdmi0                     │
│                                                                  │
│  DrmAtomicCommitSink::Commit():                                  │
│   drmModeAtomicCommit(fd, pset,                                  │
│       DRM_MODE_ATOMIC_NONBLOCK | DRM_MODE_ATOMIC_ALLOW_MODESET)  │
│   Retry on EBUSY (up to 9375 lần, mỗi lần 200-3200µs)          │
└───────────────────┬──────────────────────────────────────────────┘
                    │ ioctl /dev/dri/card0
┌───────────────────▼──────────────────────────────────────────────┐
│  LINUX KERNEL — DRM/KMS                                          │
│                                                                  │
│  vc4 DRM driver (drivers/gpu/drm/vc4/)                           │
│   → validate atomic state                                        │
│   → wait for vblank (VSYNC)                                      │
│   → program HVS (Hardware Video Scaler): scale + blend layers    │
│   → flip framebuffer tại vblank                                  │
│   → signal out-fence → thông báo HWC commit hoàn tất            │
│                                                                  │
│  vc4-hdmi driver → TMDS encoding → HDMI output                  │
└───────────────────┬──────────────────────────────────────────────┘
                    │ HDMI TMDS signal
┌───────────────────▼──────────────────────────────────────────────┐
│  HDMI Monitor / TV — 1920×1080 @ 60fps                           │
└──────────────────────────────────────────────────────────────────┘
```

---

## Buffer Lifecycle (Gralloc / minigbm vc4)

```
Gralloc::allocate(width, height, RGBA8888)
  → minigbm vc4 backend (external/minigbm/vc4.c)
  → DRM_IOCTL_VC4_CREATE_BO(size)    ← GEM buffer object
  → stride = ALIGN(width * 4, 64)   ← căn 64 bytes (ARM L1 cache line)
  → trả về dma-buf fd

Buffer shared qua processes (zero-copy):
  DRM_IOCTL_PRIME_HANDLE_TO_FD → dma-buf fd
    App process:          GPU render target
    SurfaceFlinger:       texture để composite
    drm_hwcomposer:       scanout framebuffer

Formats hỗ trợ (vc4.c):
  Render target: ARGB8888, RGB565, XRGB8888
  Texture only:  NV12, YVU420 (video playback)
```

---

## Hai chế độ Composition

### DEVICE Composition (fast path)
```
Layer A (GEM bo) + Layer B (GEM bo)
    ↓ drmModeAtomicCommit (2 DRM planes)
BCM2711 HVS hardware blending → HDMI
→ Không tốn GPU, không CPU copy
```

### CLIENT Composition (fallback)
```
Layer A + Layer B + Layer C
    ↓ GLES composite bởi SurfaceFlinger (V3D GPU)
client target buffer (1 GEM bo)
    ↓ drmModeAtomicCommit (1 DRM plane)
BCM2711 HVS → HDMI
```

---

## V3D GPU Architecture (VideoCore VI)

```
OpenGL ES call → Mesa V3D Gallium driver
  ↓
Shader compile → QPU ISA (Quad Processing Unit)
  ↓
Tile-Based Deferred Rendering:
  Binning pass: vertex shader, tile binning (32×32 px tiles)
  Rendering pass: per-tile fragment shader on QPU
    → tile buffer (on-chip SRAM) → blend → DRAM writeback
  ↓
DRM_IOCTL_V3D_SUBMIT_CL → /dev/dri/renderD128
  ↓
Kernel v3d driver → V3D hardware registers → GPU execute
  → syncobj fence → signal completion
```

---

## Cấu hình RPi4

```properties
# vendor.prop
vendor.hwc.drm.ctm=DRM_OR_IGNORE      # Color Transform Matrix: dùng DRM nếu có
vendor.hwc.drm.force_mode=1920x1080   # Force 1080p (không auto-detect EDID)
```

```ini
# config.txt
dtoverlay=vc4-kms-v3d        # Bật DRM/KMS + V3D GPU
disable_fw_kms_setup=1       # Kernel quản lý display, không phải GPU firmware
```

## Mesa3d-rpi vs Mesa3d

RPi4 dùng `external/mesa3d-rpi` (fork của Mesa) thay vì `external/mesa3d` để:
- Có patches Broadcom-specific cho V3D
- Support `v3dv` Vulkan driver (`src/broadcom/vulkan/`)
- V3D Gallium driver (`src/gallium/drivers/v3d/`)
- Tương thích với kernel vc4/v3d driver version cụ thể của RPi
