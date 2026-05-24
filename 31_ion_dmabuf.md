# ION / DMA-BUF — Shared Memory cho Camera/GPU/Codec

## 1. Tổng quan

ION/DMA-BUF giải quyết bài toán: Camera capture frame → GPU render → Video encoder encode, **không copy buffer** giữa các bước.

```
Camera ISP → [DMA-BUF fd] → GPU (texture) → [DMA-BUF fd] → Video Encoder
                ↑                                ↑
         Cùng physical memory, zero copy
```

**Config:**
```
CONFIG_DMABUF_HEAPS=y       ← DMA-BUF heaps framework (Android 12+)
CONFIG_DMABUF_HEAPS_SYSTEM=y ← System heap (vmalloc-based)
CONFIG_DMABUF_HEAPS_CMA=y   ← CMA heap (contiguous memory)
CONFIG_ION=y                 ← Legacy ION (Android < 12)
CONFIG_DMA_SHARED_BUFFER=y  ← DMA-BUF core
```

---

## 2. Kiến trúc

```
                    DMA-BUF fd (file descriptor)
                           │
          ┌────────────────┼────────────────┐
          │                │                │
   ┌──────▼──────┐  ┌──────▼──────┐  ┌─────▼───────┐
   │  Camera HAL  │  │  GPU (V3D)  │  │ Video codec  │
   │  (write)     │  │  (read/tex) │  │  (read)      │
   └──────────────┘  └──────────────┘  └──────────────┘
          │                │                │
          └────────────────┼────────────────┘
                    Physical memory (1 allocation)
                    Mapped vào address space của mỗi process/device
```

---

## 3. DMA-BUF Heaps (Android 12+)

```c
/* /dev/dma_heap/ — các heap có sẵn */
// /dev/dma_heap/system     ← General purpose (vmalloc)
// /dev/dma_heap/system-uncached ← Non-cached (DMA)
// /dev/dma_heap/linux,cma ← Contiguous memory

/* Cấp phát từ userspace */
#include <linux/dma-heap.h>
#include <sys/ioctl.h>

int heap_fd = open("/dev/dma_heap/system", O_RDONLY);

struct dma_heap_allocation_data alloc = {
    .len       = 4 * 1024 * 1024,  /* 4MB */
    .fd_flags  = O_RDWR | O_CLOEXEC,
};
ioctl(heap_fd, DMA_HEAP_IOCTL_ALLOC, &alloc);

int buf_fd = alloc.fd;  /* DMA-BUF fd */

/* Map để CPU access */
void *ptr = mmap(NULL, alloc.len, PROT_READ | PROT_WRITE,
                 MAP_SHARED, buf_fd, 0);

/* Chia sẻ với process khác qua fd passing (SCM_RIGHTS) */
```

---

## 4. ION (Legacy, Android < 12)

```c
/* /dev/ion — legacy interface */
#include <ion/ion.h>

int ion_fd = ion_open();

/* Alloc từ CMA heap */
ion_alloc(ion_fd,
          4 * 1024 * 1024,  /* size */
          0,                 /* align */
          ION_HEAP(ION_CMA_HEAP_ID),
          ION_FLAG_CACHED,
          &handle);

/* Lấy DMA-BUF fd từ handle */
int buf_fd;
ion_share(ion_fd, handle, &buf_fd);

/* buf_fd có thể pass sang process khác */
```

---

## 5. Camera Pipeline — Zero Copy

```
libcamera (userspace)
    │  Cấp phát DMA-BUF buffers từ /dev/dma_heap/linux,cma
    │  Pass fd sang kernel camera driver
    ▼
bcm2835-camera (kernel V4L2)
    │  DMA từ ISP → physical buffer
    │  Return fd khi frame done
    ▼
libcamera (userspace)
    │  Pass fd sang GPU
    ▼
Mesa/V3D (GPU)
    │  dma_buf_import() → import external buffer
    │  Tạo GL texture từ DMA-BUF
    ▼
Display / Video encoder
    │  Nhận cùng fd
    │  DMA read trực tiếp
```

---

## 6. DMA-BUF Fence — Synchronization

```c
/* Đồng bộ giữa camera và GPU */
#include <linux/sync_file.h>

/* Camera báo frame done qua fence fd */
int fence_fd = ...; /* từ V4L2 dequeue */

/* GPU chờ fence trước khi dùng buffer */
struct sync_merge_data data = {
    .name = "gpu-wait",
    .fd2 = fence_fd,
};
ioctl(fence_fd, SYNC_IOC_MERGE, &data);

/* Hoặc dùng EGL sync */
EGLSyncKHR egl_sync = eglCreateSyncKHR(display,
    EGL_SYNC_NATIVE_FENCE_ANDROID, attribs);
eglWaitSyncKHR(display, egl_sync, 0);
```

---

## 7. Gralloc — Android Buffer Allocator

Gralloc là HAL layer trên DMA-BUF cho Android graphics:

```cpp
/* Allocate buffer */
buffer_handle_t handle;
gralloc.allocate(
    1920,       /* width */
    1080,       /* height */
    HAL_PIXEL_FORMAT_YCBCR_420_888,  /* format */
    GRALLOC_USAGE_HW_CAMERA_WRITE | GRALLOC_USAGE_HW_TEXTURE,
    &handle);

/* Lock để CPU access */
void *data;
gralloc.lock(handle, GRALLOC_USAGE_SW_READ_OFTEN,
             0, 0, 1920, 1080, &data);
/* ... process data ... */
gralloc.unlock(handle);

/* Bên dưới: gralloc4 dùng DMA-BUF heaps */
```

---

## 8. Debug DMA-BUF

```bash
# Xem tất cả DMA-BUF allocations
cat /sys/kernel/debug/dma_buf/bufinfo
# name   size       flags  mode  count  exp_name
# camera 4194304   0x2    0660  3      camera_server
# gpu    16777216  0x2    0660  2      surfaceflinger

# DMA-BUF stats
cat /sys/kernel/debug/dma_buf/stats

# Xem heaps
ls /dev/dma_heap/
# linux,cma  system  system-uncached

# Theo dõi alloc/free
adb shell atrace gfx camera

# Xem buffer per process
cat /proc/$(pgrep cameraserver)/smaps | grep -A5 "dma-buf"
```

---

## 9. File quan trọng trong source

| File | Mô tả |
|------|-------|
| `kernel/common/drivers/dma-buf/dma-buf.c` | DMA-BUF core |
| `kernel/common/drivers/dma-buf/heaps/` | DMA-BUF heaps |
| `kernel/common/drivers/dma-buf/sync_file.c` | Sync fence |
| `hardware/interfaces/graphics/allocator/` | Gralloc HAL |
| `system/memory/libmemunreachable/` | Memory leak detection |
