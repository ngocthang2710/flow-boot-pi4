# cgroups v2 & CMA — Process Grouping & Contiguous Memory

## PHẦN 1: cgroups v2

### 1. Tổng quan

cgroups (control groups) là Linux mechanism để giới hạn tài nguyên (CPU, memory, I/O) cho nhóm process.

**Config:**
```
CONFIG_CGROUPS=y
CONFIG_CGROUP_CPUACCT=y
CONFIG_MEMCG=y              ← Memory cgroup
CONFIG_BLK_CGROUP=y         ← Block I/O cgroup
CONFIG_CGROUP_SCHED=y       ← CPU scheduler integration
CONFIG_CPUSETS=y            ← CPU set assignment
CONFIG_CGROUP_BPF=y         ← BPF programs cho cgroup
```

---

### 2. Android cgroup Hierarchy

```
/dev/cgroup (v1) hoặc /sys/fs/cgroup (v2)
├── cpuset/
│   ├── background/         ← Background processes: CPU 0-1
│   ├── foreground/         ← Foreground: CPU 0-3 (all)
│   ├── system-background/  ← System background: CPU 0
│   └── top-app/            ← Top app: CPU 0-3, boost
│
├── memory/  (memcg)
│   └── apps/
│       ├── uid_1000/       ← System UID
│       ├── uid_10123/      ← App UID
│       └── uid_10234/
│
└── blkio/
    ├── background/         ← Giảm I/O priority cho background
    └── foreground/         ← Full I/O priority
```

---

### 3. Task Assignment

```bash
# Xem cpuset của foreground app
cat /dev/cpuset/foreground/cpus
# 0-3  (all 4 cores)

cat /dev/cpuset/background/cpus
# 0-1  (only 2 cores)

cat /dev/cpuset/top-app/cpus
# 0-3

# Xem processes trong top-app group
cat /dev/cpuset/top-app/tasks
# 1234  (com.foreground.app)
# 1235

# Xem memory limit
cat /dev/memcg/apps/memory.limit_in_bytes
# 3145728000  (3GB)

# Per-app limit
cat /dev/memcg/apps/uid_10123/memory.limit_in_bytes
# 536870912  (512MB)
```

---

### 4. Android assigns cgroups

```java
// frameworks/base/services/core/java/com/android/server/am/ProcessList.java

// Khi app vào foreground
private void setProcessGroup(int pid, int group) {
    // group = THREAD_GROUP_TOP_APP, THREAD_GROUP_BG, etc.
    Process.setProcessGroup(pid, group);
}

// Native: libprocessgroup
// system/core/libprocessgroup/processgroup.cpp
bool SetTaskProfiles(int tid, const std::vector<std::string>& profiles) {
    // Áp dụng TaskProfiles từ /etc/task_profiles.json
    // → Move tid vào đúng cgroup
}
```

---

### 5. Task Profiles (Android 12+)

```json
// /etc/task_profiles.json
{
  "Attributes": [
    {"Name": "HighPerformance", "Controller": "cpuset", "File": "cpus", "Value": "0-3"},
    {"Name": "LowPower", "Controller": "cpuset", "File": "cpus", "Value": "0-1"}
  ],
  "Profiles": [
    {
      "Name": "SCHED_SP_FOREGROUND",
      "Actions": [
        {"Name": "JoinCgroup", "Params": {"Controller": "cpuset", "Path": "top-app"}},
        {"Name": "SetAttribute", "Params": {"Name": "HighPerformance"}}
      ]
    },
    {
      "Name": "SCHED_SP_BACKGROUND",
      "Actions": [
        {"Name": "JoinCgroup", "Params": {"Controller": "cpuset", "Path": "background"}}
      ]
    }
  ]
}
```

---

### 6. cgroup BPF

```c
/* Attach BPF program vào cgroup để control network */
// Ví dụ: Block network cho background processes

SEC("cgroup/skb")
int restrict_background_net(struct __sk_buff *skb)
{
    /* Chỉ cho phép DNS (53) và HTTPS (443) */
    if (skb->remote_port == 53 || skb->remote_port == 443)
        return 1;  /* allow */
    return 0;  /* drop */
}

/* Load vào cgroup */
bpf_prog_attach(prog_fd, cgroup_fd, BPF_CGROUP_INET_EGRESS, 0);
```

---

## PHẦN 2: CMA (Contiguous Memory Allocator)

### 7. Tổng quan CMA

CMA cấp phát vùng nhớ **liên tục vật lý** (contiguous physical) cho DMA devices như Camera, Video codec — những thiết bị không có IOMMU.

**Config:**
```
CONFIG_CMA=y
CONFIG_CMA_SIZE_MBYTES=32    ← Reserve 32MB cho CMA (default)
CONFIG_DMA_CMA=y
CONFIG_DMABUF_HEAPS_CMA=y    ← CMA heap cho DMA-BUF
```

---

### 8. Tại sao cần Contiguous Memory?

```
Virtual memory: có thể ghép nhiều vùng rời
Physical memory: CMA cần liền kề

Không CMA:         CMA:
┌──┬──┬──┬──┐     ┌──────────────┐
│A │  │B │  │     │ Camera buffer│
│  │xx│  │xx│     │ (liên tục)   │
└──┴──┴──┴──┘     └──────────────┘
Camera DMA cần     DMA hoạt động
physical liền kề!  bình thường
```

---

### 9. CMA Region trong Device Tree

```dts
/* arch/arm64/boot/dts/broadcom/bcm2711-rpi-4-b.dts */
/ {
    reserved-memory {
        #address-cells = <2>;
        #size-cells = <1>;
        ranges;

        /* CMA region cho camera/codec */
        linux,cma {
            compatible = "shared-dma-pool";
            reusable;
            size = <0x4000000>;    /* 64MB */
            alignment = <0x2000>;
            linux,cma-default;
        };
    };
};
```

---

### 10. CMA Allocation

```bash
# Xem CMA stats
cat /proc/meminfo | grep CMA
# CmaTotal:          65536 kB  (64MB total)
# CmaFree:           45678 kB  (45MB available)

# Debug CMA
cat /sys/kernel/debug/cma/cma-0/count
cat /sys/kernel/debug/cma/cma-0/used

# DMA-BUF heap dùng CMA
ls /dev/dma_heap/
# linux,cma    ← CMA-backed heap
```

```c
/* Kernel: cấp phát CMA */
#include <linux/cma.h>

struct page *pages = cma_alloc(dev_get_cma_area(dev),
                                nr_pages,
                                order,
                                GFP_KERNEL);
phys_addr_t phys = page_to_phys(pages);
void *virt = page_address(pages);

/* Free */
cma_release(dev_get_cma_area(dev), pages, nr_pages);
```

---

### 11. CMA + Camera trên Pi4

```
Pi4 Camera Pipeline + CMA:

libcamera → alloc từ /dev/dma_heap/linux,cma
                │  64MB CMA region
                │  Physical contiguous
                ▼
V4L2 (bcm2835-camera)
                │  DMA từ Camera ISP → CMA buffer
                │  Không cần copy
                ▼
GPU (V3D)
                │  Import DMA-BUF fd
                │  Tạo texture trực tiếp
                ▼
Display / Encode
```

---

### 12. File quan trọng trong source

| File | Mô tả |
|------|-------|
| `kernel/common/mm/cma.c` | CMA core |
| `kernel/common/drivers/dma-buf/heaps/cma_heap.c` | CMA DMA-BUF heap |
| `system/core/libprocessgroup/` | Android cgroup management |
| `system/core/libprocessgroup/profiles/task_profiles.json` | Task profile definitions |
| `frameworks/base/core/jni/android_os_Process.cpp` | setProcessGroup native |
