# zRAM & Kernel Memory Management

## 1. Tổng quan

Pi4 không có swap partition trên disk — thay vào đó dùng **zRAM**: compressed RAM-based swap. Kernel memory management quyết định ai sống, ai chết khi RAM cạn.

```
Physical RAM (4GB/8GB trên Pi4)
  │
  ├── Kernel memory (buddy allocator, slab)
  ├── App processes (pages in active use)
  ├── Page cache (file cache, VFS layer)
  └── zRAM swap (compressed cold pages)
        ↑ Pages evicted here when RAM pressure
        
OOM:
  RAM full + zRAM full → LMKD kills cached apps
  Last resort: Linux OOM killer kills biggest process
```

---

## 2. zRAM — Compressed Swap

```
zRAM = virtual block device backed by RAM (compressed):
  /dev/zram0 → swap device
  
  When page evicted:
    LZ4/ZSTD compress page → smaller than 4KB
    Store in zRAM pool
    
  When page accessed again:
    Decompress from zRAM → restore to RAM
    
  Savings:
    LZ4:   ~50% compression → 4GB RAM ≈ 6GB effective
    ZSTD:  ~60% compression → 4GB RAM ≈ 7GB+ effective
    But: CPU cost for compress/decompress
```

```bash
# Xem zRAM stats
adb shell cat /proc/swaps
# Filename       Type    Size       Used    Priority
# /dev/zram0     partition 2097148  45678   5

adb shell cat /sys/block/zram0/mm_stat
# orig_data_size compr_data_size mem_used_total mem_limit mem_used_max
# 123456789      45678901        50000000       0         60000000

# Compression ratio
adb shell cat /sys/block/zram0/stat

# zRAM config
adb shell cat /sys/block/zram0/comp_algorithm
# [lz4] lzo lzo-rle zstd  ← [current]

adb shell cat /sys/block/zram0/disksize
# 2147483648  (2GB = half of 4GB RAM, typical)
```

---

## 3. Buddy Allocator — Physical Page Allocation

```
Buddy system:
  Manages physical pages in power-of-2 blocks
  
  Free lists: order 0 (4KB), 1 (8KB), ... 10 (4MB)
  
  Allocate 3 pages:
    Round up to order 2 (16KB = 4 pages)
    Split order 3 block if needed
    Return 16KB block, 1 page internal fragmentation
    
  Free block:
    Check if "buddy" is also free
    If yes → coalesce into larger order block
```

```bash
# Buddy allocator state
adb shell cat /proc/buddyinfo
# Node 0, zone DMA     3213  1432  876  543  321  210  ...
# Node 0, zone Normal  5432  3210  1234  ...
# (number of free blocks at each order)

# Fragmentation index
adb shell cat /proc/pagetypeinfo

# Memory zones
adb shell cat /proc/zoneinfo | head -40
```

---

## 4. Slab Allocator — Kernel Object Cache

```
Slab (SLUB in modern Linux):
  Efficient allocator for kernel objects of fixed size
  Avoids fragmentation for frequent alloc/free
  
  Caches:
    kmalloc-64, kmalloc-128, kmalloc-256, ...
    task_struct    ← process descriptors
    mm_struct      ← memory maps
    file           ← open file structs
    inode          ← filesystem inode cache
    binder_proc    ← Binder process state
    sk_buff        ← network socket buffers
    
  Memory: struct kmem_cache → per-CPU free list → partial slabs
```

```bash
# Slab usage
adb shell cat /proc/slabinfo | head -20
# name    <active_objs> <num_objs> <objsize> <objperslab> <pagesperslab>
# task_struct  2345  2400  9216  3  9
# mm_struct    456   480   1216  26  8
# inode_cache  12345 12800 720   22  4

# Top slab consumers
adb shell cat /proc/slabinfo | awk '{print $3*$4, $0}' | sort -rn | head -10
```

---

## 5. Page Cache — VFS File Cache

```
Page cache = RAM used to cache file contents:
  read() → data cached in page cache
  write() → written to page cache first (write-back)
  
  Benefits:
    Second read of same file → served from RAM (fast)
    Multiple processes reading same file → share pages
    
  On Pi4 (SD card):
    Page cache critical! SD I/O is slow (50-100 MB/s)
    Pages stay in cache until eviction pressure
    
  Eviction:
    LRU (Least Recently Used) policy
    Dirty pages flushed to disk before eviction
    Clean pages evicted immediately
```

```bash
# Memory breakdown
adb shell cat /proc/meminfo
# MemTotal:       3985232 kB   ← Total physical RAM
# MemFree:         234567 kB   ← Truly free
# MemAvailable:   2134567 kB   ← Available (free + reclaimable cache)
# Buffers:          45678 kB   ← Block device buffers
# Cached:         1234567 kB   ← Page cache (VFS)
# SwapTotal:      2097148 kB   ← zRAM size
# SwapFree:       1987654 kB   ← zRAM free
# Dirty:            12345 kB   ← Pages waiting to be written
# AnonPages:      1456789 kB   ← Anonymous pages (heap, stack)
# Mapped:          234567 kB   ← Files mapped into process space
# Slab:            345678 kB   ← Kernel slab
# SReclaimable:   234567 kB   ← Reclaimable slab (inode cache, etc.)
# SUnreclaim:     111111 kB   ← Non-reclaimable slab
```

---

## 6. Memory Pressure & PSI

```
PSI (Pressure Stall Information) — Android 10+:
  Measures time processes stall waiting for resources
  
  /proc/pressure/memory:
    some avg10=5.23 avg60=2.01 avg300=0.50 total=12345678
    full avg10=0.12 avg60=0.04 avg300=0.01 total=456789
    
    some = some tasks stalled (pressure)
    full = ALL tasks stalled (severe pressure)
    avg10/60/300 = moving average over 10s/60s/300s window
    
  LMKD monitors PSI:
    >50ms stall in 500ms window → kill low-priority apps
    (see 30_lmkd_oom.md for kill policy)
```

```bash
# Real-time PSI
adb shell watch -n 1 cat /proc/pressure/memory

# Trigger pressure (test)
adb shell stress-ng --vm 1 --vm-bytes 80% --timeout 30s
```

---

## 7. vm.* Tunables

```bash
# Key kernel memory tunables
adb shell sysctl vm.swappiness
# vm.swappiness = 100  (Android default, aggressive swap)
# 0 = avoid swap, 100 = use swap aggressively

adb shell sysctl vm.dirty_ratio
# vm.dirty_ratio = 20  (max % RAM for dirty pages before sync)

adb shell sysctl vm.dirty_background_ratio
# vm.dirty_background_ratio = 5  (start background writeback at 5%)

adb shell sysctl vm.min_free_kbytes
# vm.min_free_kbytes = 12288  (keep 12MB free always)

adb shell sysctl vm.extra_free_kbytes
# Android-specific: extra buffer above min_free_kbytes

# OOM score (per process)
adb shell cat /proc/$(pidof com.myapp)/oom_score
# 120  (higher = more likely to be killed)

adb shell cat /proc/$(pidof system_server)/oom_score_adj
# -900  (protected)
```

---

## 8. CMA — Contiguous Memory Allocator

```
CMA = reserves contiguous physical pages for DMA devices
  (see also 32_cgroups_cma.md)

Pi4 CMA uses:
  GPU (vc4): needs contiguous buffer for scanout
  Camera (Unicam): DMA buffer
  V3D: GPU render targets
  
Config:
  CMA_SIZE_MBYTES=256  → 256MB reserved for CMA
  config.txt:
    gpu_mem=128   ← VideoCore firmware uses this much
    
Check:
adb shell cat /proc/meminfo | grep CmaTotal
# CmaTotal:        262144 kB   (256MB)
adb shell cat /proc/meminfo | grep CmaFree
# CmaFree:         234567 kB
```

---

## 9. Memory Stats Per App

```bash
# App memory usage (detailed)
adb shell dumpsys meminfo com.myapp
# App Summary:
#   Java Heap:     45678 kB
#   Native Heap:   23456 kB
#   Code:          12345 kB
#   Stack:          2345 kB
#   Graphics:       8901 kB  ← GPU buffers
#   Private Other:  4567 kB
#   System:        23456 kB
#   Total:        120742 kB
# 
# TOTAL PSS:       120742
# TOTAL RSS:       234567

# PSS (Proportional Set Size) = shared pages / number_of_processes
# RSS (Resident Set Size) = total pages in RAM (shared counted fully)
# VSZ (Virtual Size) = all virtual mappings

# Memory per process (quick)
adb shell cat /proc/meminfo
adb shell dumpsys meminfo | grep "Total PSS" | sort -rn | head -20
```

---

## 10. OOM Killer vs LMKD

```
Two levels of memory reclaim:

Level 1 — LMKD (Android userspace):
  Monitors PSI, kills cached/background apps
  Graceful: apps can save state before die
  Prefer: lower OOM adj → safer
  
Level 2 — Linux OOM Killer (kernel):
  Last resort after all reclaim failed
  Kills highest oom_score process
  No graceful shutdown
  
  log: "Out of memory: Killed process 1234 (com.myapp) ..."
  
adb shell dmesg | grep "Out of memory"
adb shell dmesg | grep "oom_kill"
```

---

## 11. File quan trọng trong source

| File | Mô tả |
|------|-------|
| `kernel/common/mm/page_alloc.c` | Buddy allocator |
| `kernel/common/mm/slub.c` | SLUB allocator |
| `kernel/common/mm/vmscan.c` | Page reclaim, LRU eviction |
| `kernel/common/mm/swap.c` | Swap framework |
| `kernel/common/drivers/block/zram/zram_drv.c` | zRAM driver |
| `kernel/common/mm/oom_kill.c` | OOM killer |
| `kernel/common/mm/psi.c` | Pressure stall info |
| `system/memory/lmkd/lmkd.cpp` | Android LMKD daemon |
| `device/brcm/rpi4/` → fstab / init | Pi4 zRAM setup |
