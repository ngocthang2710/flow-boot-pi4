# Kernel DRM/GPU — V3D & VC4 — Raspberry Pi 4

## 1. Tổng quan

Pi4 có 2 GPU subsystem trong kernel:
- **V3D** — Broadcom VideoCore VI 3D engine (OpenGL ES, Vulkan)
- **VC4** — VideoCore IV legacy (display pipeline, video encoder, ISP)

**Config đã enabled:**
```
CONFIG_DRM=y
CONFIG_DRM_V3D=y           ← 3D rendering engine (OpenGL ES / Vulkan)
CONFIG_DRM_VC4=y           ← Display pipeline, HDMI, camera ISP
CONFIG_DRM_KMS_HELPER=y    ← KMS (Kernel Mode Setting) framework
CONFIG_DRM_GEM_DMA_HELPER=y ← GEM DMA memory allocator
CONFIG_DRM_SCHED=y         ← GPU job scheduler
CONFIG_DRM_DISPLAY_HDMI_HELPER=y
CONFIG_GPU_TRACEPOINTS=y   ← GPU tracepoints cho ftrace/perf
```

**Driver files:**
```
kernel/common/drivers/gpu/drm/v3d/
  ├── v3d_drv.c       ← Driver entry, init, device probing
  ├── v3d_gem.c       ← GEM (Graphics Execution Manager) - memory
  ├── v3d_sched.c     ← Job scheduler - submit/complete GPU jobs
  ├── v3d_mmu.c       ← GPU MMU - virtual → physical address
  ├── v3d_irq.c       ← Interrupt handling
  ├── v3d_debugfs.c   ← DebugFS interface
  └── v3d_perf.c      ← Performance counters

kernel/common/drivers/gpu/drm/vc4/
  ├── vc4_drv.c       ← VC4 driver main
  ├── vc4_hdmi.c      ← HDMI output
  ├── vc4_crtc.c      ← CRTC (display controller)
  ├── vc4_plane.c     ← Display planes
  └── vc4_vec.c       ← Composite video output
```

---

## 2. Kiến trúc GPU Pi4

```
BCM2711 SoC
├── V3D (VideoCore VI 3D)
│   ├── 4x QPU cores (Quad Processor Units)
│   ├── TMU (Texture Memory Unit)
│   ├── TLB (Tile Load/Store)
│   └── GPU MMU (IOMMU)
│
└── VC4 (VideoCore IV - Display)
    ├── HVS (Hardware Video Scaler) - composite display
    ├── HDMI0 + HDMI1 outputs
    ├── DSI (Display Serial Interface)
    └── Camera ISP pipeline

Stack userspace:
  App (OpenGL ES) → Mesa (libGL) → DRM ioctl → V3D kernel driver → GPU HW
  App (Vulkan)    → Mesa (Vulkan) → DRM ioctl → V3D kernel driver → GPU HW
  Compositor      → KMS/DRM      → VC4 driver → Display HW
```

---

## 3. DRM/KMS — Display Control

```bash
# Xem DRM devices
ls /dev/dri/
# card0   renderD128

# Xem DRM info
modetest -M vc4
# Encoders:
#   35      TMDS      CRTC: 36
# Connectors:
#   36      HDMI-A-1  1920x1080
# CRTCs:
#   37      active: 1920x1080@60

# Liệt kê display modes
modetest -M vc4 -c
```

---

## 4. V3D GPU Performance Counters

```bash
# V3D cung cấp performance counters qua debugfs
ls /sys/kernel/debug/dri/0/

# V3D-specific:
cat /sys/kernel/debug/dri/renderD128/v3d_gpu_reset_count
cat /sys/kernel/debug/dri/renderD128/v3d_bo_stats

# GPU utilization qua perf (CONFIG_GPU_TRACEPOINTS=y)
perf stat -e 'gpu:gpu_job_enqueue,gpu:gpu_job_complete' sleep 5

# Trace GPU jobs
echo drm_sched_job_enqueue >> /sys/kernel/debug/tracing/set_ftrace_filter
echo function >> /sys/kernel/debug/tracing/current_tracer
```

---

## 5. V3D Job Scheduler — Code Analysis

```c
/* drivers/gpu/drm/v3d/v3d_sched.c */

/* GPU job gồm nhiều loại (BIN, RENDER, TFU, CSD, CACHE_CLEAN) */
enum v3d_queue {
    V3D_BIN,         /* Binning pass — phân chia tile */
    V3D_RENDER,      /* Rendering pass — vẽ từng tile */
    V3D_TFU,         /* Texture Formatting Unit */
    V3D_CSD,         /* Compute Shader Dispatch */
    V3D_CACHE_CLEAN, /* Cache cleanup */
};

/* Submit job vào scheduler */
static int v3d_job_init(struct v3d_dev *v3d, struct drm_file *file_priv,
                         struct v3d_job *job, size_t size,
                         void (*free)(struct kref *ref),
                         u32 in_sync, u64 *out_sync, int ring)
{
    /* Tạo DRM scheduled job */
    drm_sched_job_init(&job->base, &v3d->queue[ring].sched_entity,
                       1, v3d->dev);
    /* ... */
}
```

---

## 6. GEM (Graphics Execution Manager) — Memory

```bash
# Xem GPU memory usage
cat /sys/kernel/debug/dri/renderD128/gem_objects 2>/dev/null

# V3D BOs (Buffer Objects)
cat /sys/kernel/debug/dri/renderD128/v3d_bo_stats
# allocated bos: 234
# v3d memory used: 45678912 bytes (43MB)
```

```c
/* Tạo GPU buffer từ kernel module */
#include <drm/drm_gem_dma_helper.h>

struct drm_gem_dma_object *bo;
bo = drm_gem_dma_create(drm_dev, size);
if (IS_ERR(bo))
    return PTR_ERR(bo);

/* Map vào GPU address space */
drm_gem_create_mmap_offset(&bo->base);
```

---

## 7. V3D MMU — GPU Virtual Memory

```bash
# V3D có MMU riêng (khác với ARM MMU)
# Cho phép sandbox GPU memory giữa các process

# Debug MMU
cat /sys/kernel/debug/dri/renderD128/v3d_mmu 2>/dev/null

# IOMMU trace
echo v3d_mmu_set_page >> /sys/kernel/debug/tracing/set_ftrace_filter
```

---

## 8. OpenGL ES Test trên Pi4

```bash
# Cài mesa tools (nếu có trong Android build)
# Test OpenGL ES
glmark2-es2 --benchmark

# Chạy glxgears (X11) hoặc weston demo
weston-simple-egl

# Debug GL calls
MESA_DEBUG=1 LIBGL_DEBUG=verbose glmark2-es2

# Xem Mesa driver info
LIBGL_DEBUG=verbose es2gears_x11 2>&1 | head -20
```

---

## 9. Vulkan trên Pi4

Pi4 hỗ trợ Vulkan 1.2 qua Mesa (v3dv driver):

```bash
# Kiểm tra Vulkan support
vulkaninfo | grep -A5 "GPU id"
# GPU id = 0 (V3D 7.1)
# driverName = V3D open-source driver

# Chạy Vulkan test
vkcube

# Xem Vulkan capabilities
vulkaninfo --summary
```

---

## 10. HDMI Output via VC4

```bash
# Xem HDMI status
cat /sys/class/drm/card0-HDMI-A-1/status  # connected

# Force resolution
cat /sys/class/drm/card0-HDMI-A-1/modes
# 1920x1080
# 1280x720
# 640x480

# Audio qua HDMI
cat /sys/class/drm/card0-HDMI-A-1/audioformat

# Brightness control (display pipeline)
cat /sys/class/backlight/*/brightness
```

---

## 11. GPU Tracing với Ftrace

```bash
# Trace V3D interrupt handler
echo v3d_irq >> /sys/kernel/debug/tracing/set_ftrace_filter
echo function >> /sys/kernel/debug/tracing/current_tracer
echo 1 >> /sys/kernel/debug/tracing/tracing_on

# Run GL benchmark
glmark2-es2 --benchmark

# Stop và xem trace
echo 0 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace | head -100

# Trace GPU job completion latency
echo drm_sched_process_job >> /sys/kernel/debug/tracing/set_ftrace_filter
```

---

## 12. DebugFS V3D

```bash
# Toàn bộ V3D debug info
ls /sys/kernel/debug/dri/
ls /sys/kernel/debug/dri/renderD128/

# GPU reset count (detect hung GPU)
cat /sys/kernel/debug/dri/renderD128/v3d_gpu_reset_count

# Performance counters header
cat /sys/kernel/debug/dri/renderD128/v3d_perf_counters
```

---

## 13. File quan trọng trong source

| File | Mô tả |
|------|-------|
| `drivers/gpu/drm/v3d/v3d_drv.c` | V3D driver init, probe |
| `drivers/gpu/drm/v3d/v3d_gem.c` | GPU buffer management |
| `drivers/gpu/drm/v3d/v3d_sched.c` | Job scheduler (BIN/RENDER/CSD) |
| `drivers/gpu/drm/v3d/v3d_mmu.c` | GPU MMU, page table |
| `drivers/gpu/drm/v3d/v3d_perf.c` | Performance counters |
| `drivers/gpu/drm/vc4/vc4_hdmi.c` | HDMI output driver |
| `drivers/gpu/drm/vc4/vc4_crtc.c` | Display controller |
| `include/uapi/drm/v3d_drm.h` | V3D ioctl interface |
