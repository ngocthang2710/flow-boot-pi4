# Kernel Kfence — Memory Safety — Raspberry Pi 4

## 1. Tổng quan

KFENCE (Kernel Electric Fence) là công cụ phát hiện lỗi bộ nhớ kernel theo xác suất — detect heap out-of-bounds, use-after-free, invalid free — với overhead gần như bằng 0 trong production.

**Config đã enabled:**
```
CONFIG_KFENCE=y
CONFIG_KFENCE_SAMPLE_INTERVAL=500   ← Sample 1/500 allocation (mỗi 500ms)
CONFIG_KFENCE_NUM_OBJECTS=255       ← Pool 255 objects
CONFIG_KFENCE_STRESS_TEST_INTERVAL=0
```

**Files:**
```
kernel/common/mm/kfence/
├── core.c     ← Core implementation (chính)
├── report.c   ← Bug report formatting
└── kfence.h   ← Internal definitions

kernel/common/include/linux/kfence.h  ← Public API
```

---

## 2. Cơ chế hoạt động

```
Normal allocation (SLUB/SLAB):
  kmalloc(64) → slab allocator → pointer

Kfence allocation (mỗi ~500ms):
  kmalloc(64) → Kfence pool → guarded pointer
                     │
                     ▼
  ┌──────────────────────────────────┐
  │ Guard page (PROT_NONE)           │ ← Access = immediate fault
  ├──────────────────────────────────┤
  │ Object (64 bytes được cấp)       │ ← Vùng hợp lệ
  ├──────────────────────────────────┤
  │ Guard page (PROT_NONE)           │ ← Access = immediate fault
  └──────────────────────────────────┘

→ Overflow qua trái hoặc phải → page fault → Kfence báo lỗi
→ Use-after-free → page freed → page fault → Kfence báo lỗi
```

---

## 3. Các lỗi Kfence phát hiện

| Loại lỗi | Mô tả |
|----------|-------|
| Out-of-bounds (OOB) | Đọc/ghi vượt ngoài vùng allocate |
| Use-after-free (UAF) | Truy cập sau khi đã kfree() |
| Invalid free | kfree() một pointer không hợp lệ |
| Memory corruption | Guard page bị ghi đè |

---

## 4. Đọc Kfence Report

Khi có lỗi, Kfence in ra `dmesg`:

```
==================================================================
BUG: KFENCE: out-of-bounds read in my_driver_read+0x4c/0x80

Out-of-bounds read at 0xffff888... (64B right of kfence-#12):
 my_driver_read+0x4c/0x80
 seq_read+0x112/0x3a0
 proc_reg_read+0x44/0x70

kfence-#12: 0xffff888....-0xffff888.... (64 bytes)
 allocated by task 1234 on cpu 2 at 12.345678s:
  kmalloc+0x50/0x80
  my_driver_probe+0x34/0x60
  platform_probe+0x44/0x90

CPU: 2 PID: 1234 Comm: my_daemon
==================================================================
```

---

## 5. Đọc Kfence Stats

```bash
# Xem Kfence statistics
cat /sys/kernel/debug/kfence/stats
# total allocs:        45231
# total frees:         45228
# allocation failures: 0
# bugs detected:       3

# Chi tiết objects
cat /sys/kernel/debug/kfence/objects
# Object 0: allocated, size=64, [0xffff888...0xffff888...]
# Object 1: freed
# Object 2: allocated, size=128
# ...
```

---

## 6. Trigger Kfence Bug — Test

### 6.1 Viết kernel module tạo OOB có chủ ý

```c
/* kfence_test_oob.c — Module tạo out-of-bounds để test Kfence */
#include <linux/module.h>
#include <linux/slab.h>

static int __init kfence_test_init(void)
{
    char *buf;
    int i;

    /* Kfence sẽ chọn allocation này theo xác suất */
    /* Có thể cần chờ vài giây để Kfence "sample" */
    for (i = 0; i < 1000; i++) {
        buf = kmalloc(64, GFP_KERNEL);
        if (!buf)
            continue;

        /* OOB WRITE: ghi vượt 64 bytes */
        buf[64] = 0xFF;  /* ← OOB! */
        /* Nếu Kfence đã allocate object này → crash + report */

        kfree(buf);
    }

    pr_info("kfence_test: done (check dmesg for bugs)\n");
    return 0;
}

static void __exit kfence_test_exit(void) { }

module_init(kfence_test_init);
module_exit(kfence_test_exit);
MODULE_LICENSE("GPL");
```

### 6.2 Trigger Use-after-Free

```c
static int __init kfence_uaf_test(void)
{
    char *buf = kmalloc(64, GFP_KERNEL);
    if (!buf)
        return -ENOMEM;

    kfree(buf);  /* Free */

    /* UAF: đọc sau khi free */
    char c = buf[0];  /* ← UAF! */
    pr_info("UAF read: %c\n", c);

    return 0;
}
```

---

## 7. Điều chỉnh Sample Rate

```bash
# Xem sample interval hiện tại (ms)
cat /sys/module/kfence/parameters/sample_interval
# 500

# Tăng sample rate để dễ trigger hơn (mỗi 100ms)
echo 100 > /sys/module/kfence/parameters/sample_interval

# Disable (0 = off)
echo 0 > /sys/module/kfence/parameters/sample_interval

# Bật lại
echo 500 > /sys/module/kfence/parameters/sample_interval
```

---

## 8. Kfence Pool

```bash
# Xem Kfence memory pool
cat /sys/kernel/debug/kfence/objects | head -20
# Object  0x... [in-use, 64b] allocated by PID 123:
#   kmalloc+...
#   bcm2835_spi_transfer_one+...
#
# Object  0x... [freed, 128b]
#   kmalloc+...

# Số lượng objects trong pool
grep "num_objects" /boot/config-$(uname -r)
# CONFIG_KFENCE_NUM_OBJECTS=255
```

---

## 9. Kfence vs KASAN

| Đặc điểm | Kfence | KASAN |
|----------|--------|-------|
| Overhead | ~0% (production-safe) | 2x memory, 50% slower |
| Detection | Xác suất (1/N alloc) | Deterministic (100%) |
| Coverage | Cơ bản (OOB, UAF) | Đầy đủ hơn |
| Config | `CONFIG_KFENCE=y` | `CONFIG_KASAN=y` |
| Mục đích | Production monitoring | Development/CI testing |
| Pi4 | Enabled (sample=500) | Thường disabled |

---

## 10. Kfence với Android Drivers trên Pi4

```bash
# Theo dõi Kfence liên quan đến driver cụ thể
dmesg | grep -A 20 "KFENCE"

# Filter bugs từ SPI driver
dmesg | grep -A 30 "KFENCE.*spi"

# Stress test SPI để trigger potential bugs
while true; do
    spidev_test -D /dev/spidev0.0 -s 1000000 -p "HELLO"
done &

# Monitor dmesg
dmesg -w | grep -i "kfence\|BUG\|WARN"
```

---

## 11. Kfence Report Analysis

```bash
# Tự động parse Kfence reports
dmesg | awk '/BUG: KFENCE/,/===/' | tee kfence_bugs.txt

# Count bugs by type
grep "out-of-bounds\|use-after-free\|invalid free" kfence_bugs.txt | sort | uniq -c

# Find most buggy drivers
grep "allocated by" -A 3 kfence_bugs.txt | grep "+" | awk -F'+' '{print $1}' | sort | uniq -c | sort -rn
```

---

## 12. Kfence trong Android Build

```makefile
# Kfence enabled theo defconfig
# android_rpi4_defconfig:
# CONFIG_KFENCE=y
# CONFIG_KFENCE_SAMPLE_INTERVAL=500
# CONFIG_KFENCE_NUM_OBJECTS=255

# Để enable KFENCE stress test trong CI:
# CONFIG_KFENCE_STRESS_TEST_INTERVAL=100
```

---

## 13. File quan trọng trong source

| File | Mô tả |
|------|-------|
| `mm/kfence/core.c` | Kfence core, allocation hooks |
| `mm/kfence/report.c` | Bug report formatting |
| `include/linux/kfence.h` | Public API |
| `arch/arm64/mm/kfence_init.c` | ARM64 page protection setup |
| `lib/test_kfence.c` | Kfence self-test |
