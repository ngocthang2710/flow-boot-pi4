# Kernel CPU Frequency Scaling (DVFS) — Raspberry Pi 4

## 1. Tổng quan

DVFS (Dynamic Voltage and Frequency Scaling) tự động điều chỉnh tần số và điện áp CPU theo tải, cân bằng giữa hiệu năng và tiêu thụ điện.

**Config đã enabled:**
```
CONFIG_CPU_FREQ=y
CONFIG_CPU_FREQ_STAT=y                  ← Thống kê thời gian ở mỗi freq
CONFIG_CPU_FREQ_DEFAULT_GOV_ONDEMAND=y  ← Governor mặc định: ondemand
CONFIG_CPU_FREQ_GOV_PERFORMANCE=y       ← Luôn chạy max freq
CONFIG_CPU_FREQ_GOV_POWERSAVE=y         ← Luôn chạy min freq
CONFIG_CPU_FREQ_GOV_USERSPACE=y         ← Manual control
CONFIG_CPU_FREQ_GOV_ONDEMAND=y          ← Tăng/giảm theo CPU load
CONFIG_CPU_FREQ_GOV_CONSERVATIVE=y      ← Tăng chậm hơn ondemand
CONFIG_CPU_FREQ_GOV_SCHEDUTIL=y         ← Tích hợp với scheduler
CONFIG_CPUFREQ_DT=y                     ← Device Tree based driver
CONFIG_ARM_RASPBERRYPI_CPUFREQ=y        ← Pi firmware cpufreq driver
CONFIG_ARM_SCMI_CPUFREQ=y              ← SCMI protocol
```

**Driver files:**
```
kernel/common/drivers/cpufreq/raspberrypi-cpufreq.c   ← Pi4 driver
kernel/common/drivers/cpufreq/cpufreq-dt.c             ← DT-based generic
kernel/common/drivers/cpufreq/cpufreq.c                ← Core
kernel/common/drivers/opp/core.c                        ← OPP (Operating Points)
```

---

## 2. Kiến trúc DVFS trên Pi4

```
Pi Firmware (VideoCore)
    │  Mailbox interface
    ▼
raspberrypi-cpufreq.c
    │  Gửi yêu cầu thay đổi freq lên firmware
    │  Firmware điều chỉnh PLL + điện áp
    ▼
CPUFreq Core
    ├── Governor (quyết định target freq)
    │     ├── schedutil ← tích hợp CFS scheduler
    │     ├── ondemand  ← dựa trên CPU idle time
    │     └── ...
    ├── OPP Table (các mức freq hợp lệ)
    │     ├── 600 MHz @ 0.825V
    │     ├── 1000 MHz @ 0.900V
    │     ├── 1500 MHz @ 0.950V  ← Max thường
    │     └── 1800 MHz @ 1.000V  ← Turbo (arm_boost=1)
    └── /sys/devices/system/cpu/cpu*/cpufreq/
```

---

## 3. Sysfs Interface

```bash
# Xem thông tin cpufreq
CPUFREQ=/sys/devices/system/cpu/cpu0/cpufreq

cat $CPUFREQ/scaling_driver        # raspberrypi-cpufreq
cat $CPUFREQ/scaling_governor      # schedutil
cat $CPUFREQ/scaling_available_governors
# performance powersave userspace ondemand conservative schedutil

cat $CPUFREQ/scaling_min_freq      # 600000  (600 MHz)
cat $CPUFREQ/scaling_max_freq      # 1800000 (1800 MHz)
cat $CPUFREQ/scaling_cur_freq      # 1500000 (tần số hiện tại)

cat $CPUFREQ/cpuinfo_min_freq      # hardware min
cat $CPUFREQ/cpuinfo_max_freq      # hardware max
cat $CPUFREQ/cpuinfo_transition_latency  # thời gian chuyển đổi (ns)

# Xem tất cả freq hợp lệ
cat $CPUFREQ/scaling_available_frequencies
# 600000 700000 800000 900000 1000000 1100000 1200000 1300000
# 1400000 1500000 1600000 1700000 1800000
```

---

## 4. Thay đổi Governor

```bash
# Set performance (luôn max freq)
echo performance > /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor

# Verify
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq
# 1800000

# Set powersave (luôn min freq)
echo powersave > /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor

# Set ondemand
echo ondemand > /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor

# Tuning ondemand governor
ODEM=/sys/devices/system/cpu/cpufreq/ondemand
echo 80  > $ODEM/up_threshold      # Tăng freq khi CPU load > 80%
echo 200000 > $ODEM/sampling_rate  # Sample mỗi 200ms
echo 5   > $ODEM/sampling_down_factor # Giảm freq chậm hơn

# Set schedutil (khuyến nghị cho Android)
echo schedutil > /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
# Tuning schedutil
echo 10000 > /sys/devices/system/cpu/cpufreq/schedutil/rate_limit_us
```

---

## 5. Lock tần số cụ thể

```bash
# Lock ở 1000MHz (userspace governor)
echo userspace > /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
echo 1000000 > /sys/devices/system/cpu/cpu0/cpufreq/scaling_setspeed

# Giới hạn max freq
echo 1200000 > /sys/devices/system/cpu/cpu0/cpufreq/scaling_max_freq

# Apply cho tất cả 4 core (cpu0-cpu3)
for cpu in 0 1 2 3; do
    echo performance > /sys/devices/system/cpu/cpu$cpu/cpufreq/scaling_governor
done
```

---

## 6. Thống kê thời gian ở từng tần số

```bash
# Xem time_in_state (CONFIG_CPU_FREQ_STAT=y)
cat /sys/devices/system/cpu/cpu0/cpufreq/stats/time_in_state
# 600000  15234    ← ở 600MHz trong 15234 * 10ms = 152s
# 800000  3421
# 1000000 8923
# 1500000 45123    ← hay dùng nhất
# 1800000 1234

# Xem số lần transition giữa các freq
cat /sys/devices/system/cpu/cpu0/cpufreq/stats/total_trans
# 28941
```

---

## 7. OPP (Operating Performance Points)

```bash
# Xem OPP table qua debugfs
cat /sys/kernel/debug/opp/*/opp_summary
# device         rate(Hz)    u_volt(uV)  u_volt_min  u_volt_max  available  shared
# platform/...   600000000   825000      825000      825000      true       false
# platform/...   1000000000  900000      900000      900000      true       false
# platform/...   1500000000  950000      950000      950000      true       false
# platform/...   1800000000  1000000     1000000     1000000     true       false
```

---

## 8. Benchmark theo Governor

```bash
#!/bin/bash
# benchmark_governors.sh
BENCH="sysbench cpu --cpu-max-prime=20000 run"

for gov in performance ondemand schedutil powersave; do
    echo $gov > /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
    sleep 1
    echo -n "$gov: "
    $BENCH | grep "total time:" | awk '{print $3}'
done
```

---

## 9. Thermal + DVFS Integration

```bash
# Khi nhiệt độ vượt trip point, thermal subsystem
# gọi cpufreq cooling device để giảm tần số

# Xem cooling device state
cat /sys/class/thermal/cooling_device0/type    # cpufreq
cat /sys/class/thermal/cooling_device0/max_state  # 13 (13 freq levels)
cat /sys/class/thermal/cooling_device0/cur_state  # 0 = no throttle

# Theo dõi throttling
watch -n1 'echo "Freq: $(cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq) Hz  Temp: $(($(cat /sys/class/thermal/thermal_zone0/temp)/1000))°C"'
```

---

## 10. Viết Custom Governor (Kernel Module)

```c
/* my_governor.c — Governor tùy chỉnh */
#include <linux/cpufreq.h>
#include <linux/module.h>

static int my_gov_start(struct cpufreq_policy *policy)
{
    pr_info("my_governor: started on CPU%d\n", policy->cpu);
    return 0;
}

static void my_gov_limits(struct cpufreq_policy *policy)
{
    /* Gọi khi giới hạn freq thay đổi */
    pr_info("my_governor: limits changed %u-%u kHz\n",
            policy->min, policy->max);
}

static struct cpufreq_governor my_governor = {
    .name   = "my_governor",
    .start  = my_gov_start,
    .limits = my_gov_limits,
    .owner  = THIS_MODULE,
};

static int __init my_gov_init(void)
{
    return cpufreq_register_governor(&my_governor);
}

static void __exit my_gov_exit(void)
{
    cpufreq_unregister_governor(&my_governor);
}

module_init(my_gov_init);
module_exit(my_gov_exit);
MODULE_LICENSE("GPL");
```

---

## 11. config.txt — Overclocking

```ini
# /boot/config.txt

# Overclock (void warranty!)
arm_boost=1          # Enable turbo 1800MHz (official)
over_voltage=6       # Tăng điện áp (cần tản nhiệt)
arm_freq=2000        # OC lên 2GHz

# Underclock để tiết kiệm điện
arm_freq=1000
over_voltage=-4      # Giảm điện áp (undervolt)

# Disable Turbo (lock freq cố định)
force_turbo=1
arm_freq=1500
```

---

## 12. File quan trọng trong source

| File | Mô tả |
|------|-------|
| `drivers/cpufreq/raspberrypi-cpufreq.c` | Pi4 driver, giao tiếp mailbox |
| `drivers/cpufreq/cpufreq.c` | CPUFreq core |
| `drivers/cpufreq/cpufreq_ondemand.c` | Ondemand governor |
| `drivers/cpufreq/cpufreq_schedutil.c` | Schedutil governor |
| `drivers/opp/core.c` | OPP framework |
| `include/linux/cpufreq.h` | CPUFreq kernel API |
