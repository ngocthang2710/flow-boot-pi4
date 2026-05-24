# Kernel Thermal Management — Raspberry Pi 4

## 1. Tổng quan

Thermal subsystem kernel quản lý nhiệt độ SoC và thực hiện throttling CPU khi quá nhiệt để bảo vệ phần cứng.

**Config đã enabled:**
```
CONFIG_THERMAL=y
CONFIG_THERMAL_EMULATION=y           ← Giả lập nhiệt độ để test
CONFIG_THERMAL_STATISTICS=y          ← Thống kê thermal trips
CONFIG_BCM2711_THERMAL=y             ← BCM2711 on-die sensor
CONFIG_CPU_THERMAL=y                 ← CPU cooling device
CONFIG_DEVFREQ_THERMAL=y             ← GPU/memory thermal
CONFIG_THERMAL_GOV_STEP_WISE=y       ← Bộ điều khiển step-wise
CONFIG_THERMAL_GOV_BANG_BANG=y       ← Bộ điều khiển bang-bang
CONFIG_THERMAL_GOV_USER_SPACE=y      ← Điều khiển từ userspace
CONFIG_THERMAL_GOV_POWER_ALLOCATOR=y ← Power-based allocation
```

**Driver files:**
```
kernel/common/drivers/thermal/broadcom/bcm2711_thermal.c  ← BCM2711 sensor (522 lines)
kernel/common/drivers/thermal/broadcom/bcm2835_thermal.c  ← BCM2835 sensor
kernel/common/drivers/hwmon/raspberrypi-hwmon.c           ← Pi hwmon
kernel/common/drivers/thermal/cpu_cooling.c               ← CPU cooling
kernel/common/drivers/thermal/thermal_core.c              ← Thermal core
```

---

## 2. Kiến trúc Thermal trên Pi4

```
BCM2711 SoC
├── AVS RO Temperature Sensor (on-die)
│     └── bcm2711_thermal.c đọc qua regmap
│
├── Thermal Zone 0: "cpu-thermal"
│     ├── Sensor: bcm2711_thermal
│     ├── Trip Points:
│     │     ├── trip0: 80°C → throttle (passive)
│     │     └── trip1: 85°C → shutdown (critical)
│     └── Cooling Devices:
│           └── cpufreq-cpu0 (giảm tần số)
│
└── /sys/class/thermal/thermal_zone0/
      ├── temp        ← nhiệt độ hiện tại (millidegree)
      ├── type        ← "cpu-thermal"
      └── trip_point_*/
```

---

## 3. Đọc nhiệt độ

```bash
# Đọc nhiệt độ CPU (millidegree Celsius)
cat /sys/class/thermal/thermal_zone0/temp
# 45000  →  45°C

# Đọc tất cả thermal zones
for zone in /sys/class/thermal/thermal_zone*/; do
    name=$(cat $zone/type)
    temp=$(cat $zone/temp)
    echo "$name: $((temp/1000))°C"
done

# Dùng vcgencmd (Pi firmware tool)
vcgencmd measure_temp
# temp=47.2'C

# Xem throttle status (bit flags)
vcgencmd get_throttled
# throttled=0x0
# Bit 0: currently throttled (Under-voltage)
# Bit 1: ARM frequency capped
# Bit 2: Currently throttled (over-temp)
# Bit 3: Soft temperature limit active
# Bit 16-19: same but "has occurred since last check"
```

---

## 4. Thermal Governors

```bash
# Xem governor hiện tại
cat /sys/class/thermal/thermal_zone0/policy
# step_wise

# Đổi sang user_space (manual control)
echo user_space > /sys/class/thermal/thermal_zone0/policy

# Đổi sang power_allocator (budget-based)
echo power_allocator > /sys/class/thermal/thermal_zone0/policy
```

### So sánh governors

| Governor | Cơ chế | Dùng khi |
|----------|--------|---------|
| `step_wise` | Tăng/giảm cooling 1 bước mỗi lần | Default, đơn giản |
| `bang_bang` | Bật/tắt cooling theo ngưỡng | Hệ thống 2 trạng thái |
| `user_space` | App userspace quyết định | Nghiên cứu, custom policy |
| `power_allocator` | Phân bổ power budget giữa CPU/GPU | Thiết bị có power constraint |

---

## 5. Trip Points

```bash
# Xem trip points
ls /sys/class/thermal/thermal_zone0/trip_point_*/
# trip_point_0_temp, trip_point_0_type, trip_point_0_hyst

cat /sys/class/thermal/thermal_zone0/trip_point_0_temp   # 80000 (80°C)
cat /sys/class/thermal/thermal_zone0/trip_point_0_type   # passive
cat /sys/class/thermal/thermal_zone0/trip_point_1_temp   # 85000 (85°C)
cat /sys/class/thermal/thermal_zone0/trip_point_1_type   # critical

# Thay đổi trip point (cần root)
echo 75000 > /sys/class/thermal/thermal_zone0/trip_point_0_temp
```

---

## 6. Thermal Emulation — Test không cần heating

```bash
# Enable thermal emulation (CONFIG_THERMAL_EMULATION=y)
# Giả lập nhiệt độ 85°C để test throttling
echo 85000 > /sys/class/thermal/thermal_zone0/emul_temp

# Xem CPU frequency bị throttle chưa
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq
# 600000  ← bị giảm từ 1500000 xuống còn 600MHz

# Quan sát cooling state
cat /sys/class/thermal/cooling_device0/cur_state
# 3  ← cooling level 3 đang active

# Tắt emulation
echo 0 > /sys/class/thermal/thermal_zone0/emul_temp
```

---

## 7. BCM2711 Thermal Driver — Code Analysis

```c
/* drivers/thermal/broadcom/bcm2711_thermal.c */
struct bcm2711_thermal_priv {
    struct thermal_zone_device *thermal;
    struct regmap              *regmap;
};

static int bcm2711_get_temp(struct thermal_zone_device *tz, int *temp)
{
    struct bcm2711_thermal_priv *priv = tz->devdata;
    int val;

    /* Đọc từ AVS RO register qua regmap */
    regmap_read(priv->regmap, AVS_RO_TEMP_STATUS, &val);

    /* Convert raw → millidegree Celsius */
    /* Formula: T = (val * 4807 / 1000) - 279580 */
    *temp = (val & AVS_RO_TEMP_STATUS_VALID_MSK) ?
            (int)(((val & 0x3FF) * 4807) / 1000 - 279580) : 0;

    return 0;
}

static struct thermal_zone_device_ops bcm2711_thermal_ops = {
    .get_temp = bcm2711_get_temp,
};
```

---

## 8. Monitor Thermal Realtime

```bash
# Script theo dõi nhiệt độ và tần số CPU
#!/bin/bash
while true; do
    temp=$(cat /sys/class/thermal/thermal_zone0/temp)
    freq=$(cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq)
    throttle=$(vcgencmd get_throttled | cut -d= -f2)
    printf "Temp: %d°C  Freq: %dMHz  Throttle: %s\n" \
           $((temp/1000)) $((freq/1000)) $throttle
    sleep 1
done

# Stress test để trigger throttling
stress --cpu 4 --timeout 60 &
# Quan sát throttling xảy ra
```

---

## 9. Cooling Devices

```bash
# Liệt kê cooling devices
ls /sys/class/thermal/cooling_device*/
cat /sys/class/thermal/cooling_device0/type    # cpufreq
cat /sys/class/thermal/cooling_device0/max_state  # 4 (5 levels: 0-4)
cat /sys/class/thermal/cooling_device0/cur_state  # 0 (không throttle)

# Manual throttle CPU (với user_space governor)
echo user_space > /sys/class/thermal/thermal_zone0/policy
echo 3 > /sys/class/thermal/cooling_device0/cur_state
# Kiểm tra freq
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq
```

---

## 10. Thermal Statistics

```bash
# Xem thống kê trip events
cat /sys/class/thermal/thermal_zone0/trip_point_0_temp
# Thống kê trong debugfs
cat /sys/kernel/debug/thermal/thermal_zone0 2>/dev/null
```

---

## 11. Device Tree Thermal

```dts
/* arch/arm64/boot/dts/broadcom/bcm2711.dtsi */
thermal-zones {
    cpu_thermal: cpu-thermal {
        polling-delay-passive = <2000>;  /* poll 2s khi passive */
        polling-delay = <10000>;         /* poll 10s bình thường */
        thermal-sensors = <&thermal>;

        trips {
            cpu_passive: cpu-passive {
                temperature = <80000>;  /* 80°C */
                hysteresis  = <2000>;   /* 2°C hysteresis */
                type = "passive";
            };
            cpu_critical: cpu-critical {
                temperature = <90000>;  /* 90°C */
                hysteresis  = <0>;
                type = "critical";
            };
        };

        cooling-maps {
            map0 {
                trip = <&cpu_passive>;
                cooling-device = <&cpu0 THERMAL_NO_LIMIT THERMAL_NO_LIMIT>;
            };
        };
    };
};
```

---

## 12. File quan trọng trong source

| File | Mô tả |
|------|-------|
| `drivers/thermal/broadcom/bcm2711_thermal.c` | BCM2711 temperature sensor |
| `drivers/thermal/thermal_core.c` | Thermal framework core |
| `drivers/thermal/cpu_cooling.c` | CPU frequency cooling |
| `drivers/thermal/gov_step_wise.c` | Step-wise governor |
| `drivers/thermal/gov_power_allocator.c` | Power allocator governor |
| `drivers/hwmon/raspberrypi-hwmon.c` | Pi firmware hwmon |
| `include/linux/thermal.h` | Thermal kernel API |
