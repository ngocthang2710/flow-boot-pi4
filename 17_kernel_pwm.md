# Kernel PWM — Raspberry Pi 4

## 1. Tổng quan

PWM (Pulse Width Modulation) tạo tín hiệu xung vuông với tần số và duty cycle có thể điều chỉnh — dùng để điều khiển tốc độ motor, độ sáng LED, servo, quạt.

**Config đã enabled:**
```
CONFIG_PWM=y
CONFIG_PWM_BCM2835=y    ← Hardware PWM BCM2711 (2 channel)
CONFIG_PWM_BRCMSTB=y    ← Broadcom STB PWM
CONFIG_PWM_RP1=y         ← RP1 PWM (Pi5 IO chip, forward-compatible)
CONFIG_PWM_SYSFS=y       ← /sys/class/pwm/ interface
```

**Driver files:**
```
kernel/common/drivers/pwm/pwm-bcm2835.c   ← BCM2711 hardware PWM
kernel/common/drivers/pwm/pwm-rp1.c       ← RP1 PWM controller
kernel/common/drivers/pwm/core.c           ← PWM core
```

---

## 2. PWM Channels trên Pi4

BCM2711 có **2 hardware PWM channel**:

```
PWM Channel 0:
  GPIO12 (Alt0) ← 40-pin pin 32
  GPIO18 (Alt5) ← 40-pin pin 12 (thường dùng)
  GPIO40 (Alt0) ← CM4 only

PWM Channel 1:
  GPIO13 (Alt0) ← 40-pin pin 33
  GPIO19 (Alt5) ← 40-pin pin 35
  GPIO41 (Alt0) ← CM4 only

Audio conflict: PWM0/1 cũng dùng cho analog audio output
→ Nếu enable audio: dtparam=audio=on → PWM bị chiếm
→ Phải dùng: dtoverlay=audremap để tránh conflict
```

---

## 3. Enable PWM trong config.txt

```ini
# /boot/config.txt

# Enable 1 channel PWM (GPIO18)
dtoverlay=pwm

# Enable 2 channel PWM (GPIO18 + GPIO19)
dtoverlay=pwm-2chan

# Enable với pin cụ thể
dtoverlay=pwm,pin=12,func=4        # GPIO12, Alt0
dtoverlay=pwm-2chan,pin=12,func=4,pin2=13,func2=4
```

---

## 4. Sysfs PWM Interface

```bash
# Kiểm tra PWM chips
ls /sys/class/pwm/
# pwmchip0

# Export channel 0
echo 0 > /sys/class/pwm/pwmchip0/export
# Tạo ra /sys/class/pwm/pwmchip0/pwm0/

# Cấu hình tần số 1kHz (period = 1,000,000 ns = 1ms)
echo 1000000 > /sys/class/pwm/pwmchip0/pwm0/period

# Duty cycle 50% (500,000 ns)
echo 500000 > /sys/class/pwm/pwmchip0/pwm0/duty_cycle

# Enable
echo 1 > /sys/class/pwm/pwmchip0/pwm0/enable

# Dim LED xuống 25%
echo 250000 > /sys/class/pwm/pwmchip0/pwm0/duty_cycle

# Servo control: 50Hz (20ms period), 1-2ms pulse
echo 20000000 > /sys/class/pwm/pwmchip0/pwm0/period   # 20ms
echo 1500000  > /sys/class/pwm/pwmchip0/pwm0/duty_cycle # 1.5ms = center

# Disable + unexport
echo 0 > /sys/class/pwm/pwmchip0/pwm0/enable
echo 0 > /sys/class/pwm/pwmchip0/unexport
```

---

## 5. PWM từ Kernel Module

```c
/* pwm_fan_driver.c — Điều khiển quạt theo nhiệt độ */
#include <linux/pwm.h>
#include <linux/thermal.h>
#include <linux/platform_device.h>

struct fan_data {
    struct pwm_device  *pwm;
    struct thermal_zone_device *tz;
    struct delayed_work work;
};

static void fan_update(struct work_struct *work)
{
    struct fan_data *fan = container_of(to_delayed_work(work),
                                        struct fan_data, work);
    int temp;
    thermal_zone_get_temp(fan->tz, &temp);

    /* Map temp → duty cycle */
    /* 40°C = 0%, 70°C = 100% */
    u64 duty_ns;
    struct pwm_state state;
    pwm_get_state(fan->pwm, &state);

    if (temp < 40000) {
        duty_ns = 0;
    } else if (temp > 70000) {
        duty_ns = state.period;
    } else {
        duty_ns = state.period * (temp - 40000) / 30000;
    }

    pwm_config(fan->pwm, duty_ns, state.period);
    pwm_enable(fan->pwm);

    schedule_delayed_work(&fan->work, msecs_to_jiffies(2000));
}

static int fan_probe(struct platform_device *pdev)
{
    struct fan_data *fan;
    fan = devm_kzalloc(&pdev->dev, sizeof(*fan), GFP_KERNEL);

    fan->pwm = devm_pwm_get(&pdev->dev, NULL);
    if (IS_ERR(fan->pwm))
        return PTR_ERR(fan->pwm);

    /* 25kHz PWM — tần số chuẩn cho PC fan */
    pwm_config(fan->pwm, 0, 40000);  /* period = 40µs */
    pwm_enable(fan->pwm);

    INIT_DELAYED_WORK(&fan->work, fan_update);
    schedule_delayed_work(&fan->work, 0);

    return 0;
}
```

---

## 6. PWM API trong Kernel

```c
#include <linux/pwm.h>

/* Lấy PWM device */
struct pwm_device *pwm = pwm_get(&pdev->dev, "fan");
/* hoặc */
struct pwm_device *pwm = pwm_request(0, "my-pwm");

/* Cấu hình */
pwm_config(pwm, duty_ns, period_ns);

/* Dùng API mới (atomic) */
struct pwm_state state = {
    .period   = 1000000,   /* 1ms = 1kHz */
    .duty_cycle = 500000,  /* 50% */
    .polarity = PWM_POLARITY_NORMAL,
    .enabled  = true,
};
pwm_apply_state(pwm, &state);

/* Đọc state */
pwm_get_state(pwm, &state);

/* Enable/disable */
pwm_enable(pwm);
pwm_disable(pwm);

/* Trả về */
pwm_put(pwm);
```

---

## 7. Ứng dụng thực tế trên Pi4

### Servo motor
```bash
# Servo chuẩn: 50Hz, pulse 1-2ms
echo 20000000 > period        # 20ms
echo 1000000  > duty_cycle    # 1ms = -90°
echo 1500000  > duty_cycle    # 1.5ms = 0°
echo 2000000  > duty_cycle    # 2ms = +90°
```

### DC Motor (via L298N H-Bridge)
```bash
# 1kHz PWM cho motor
echo 1000000 > period
# Tốc độ 75%
echo 750000 > duty_cycle
```

### LED Dimming
```bash
# Breathing effect bằng shell script
period=1000000
for i in $(seq 0 100 1000000) $(seq 1000000 -100 0); do
    echo $i > duty_cycle
    sleep 0.001
done
```

### Buzzer
```bash
# Tone 440Hz (note A)
echo 2272727 > period      # 1/440Hz = 2.27ms
echo 1136363 > duty_cycle  # 50%
echo 1 > enable

# Tone 880Hz (note A5)
echo 1136363 > period
echo 568181  > duty_cycle
```

---

## 8. Device Tree cho PWM consumer

```dts
/ {
    fan: pwm-fan {
        compatible = "pwm-fan";
        pwms = <&pwm 0 40000 0>;  /* channel 0, period 40µs, normal polarity */
        cooling-levels = <0 64 128 192 255>;

        thermal-zone = "cpu-thermal";
    };

    leds {
        compatible = "pwm-leds";
        led-blue {
            label = "blue";
            pwms = <&pwm 0 1000000 0>;  /* 1kHz */
            max-brightness = <255>;
        };
    };
};
```

---

## 9. Debug PWM

```bash
# Xem PWM state
cat /sys/kernel/debug/pwm
# platform/fe20c000.pwm, 2 PWM devices
#  pwm-0  (sysfs): requested enabled period: 20000000 ns duty: 1500000 ns polarity: normal

# Trace PWM calls
echo pwm_config > /sys/kernel/debug/tracing/set_ftrace_filter
echo function > /sys/kernel/debug/tracing/current_tracer

# Đo actual frequency bằng oscilloscope hoặc logic analyzer
# Kết nối GPIO18 → logic analyzer → đo period
```

---

## 10. File quan trọng trong source

| File | Mô tả |
|------|-------|
| `drivers/pwm/pwm-bcm2835.c` | BCM2711 hardware PWM, 2 channels |
| `drivers/pwm/pwm-rp1.c` | RP1 PWM (Pi5 IO chip) |
| `drivers/pwm/core.c` | PWM core, sysfs export |
| `drivers/hwmon/pwm-fan.c` | PWM fan driver |
| `drivers/leds/leds-pwm.c` | PWM LED driver |
| `include/linux/pwm.h` | PWM kernel API |
