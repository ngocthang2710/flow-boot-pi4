# Kernel GPIO — Raspberry Pi 4

## 1. Tổng quan

GPIO (General Purpose Input/Output) là cơ chế cho phép kernel/userspace điều khiển các chân vật lý trên 40-pin header của Pi4.

**Config đã enabled:**
```
CONFIG_GPIOLIB=y
CONFIG_GPIO_SYSFS=y
CONFIG_GPIO_CDEV=y
CONFIG_GPIO_RASPBERRYPI_EXP=y   ← Pi firmware expansion GPIO
CONFIG_GPIO_BCM_VIRT=y           ← Virtual GPIO qua mailbox
CONFIG_GPIO_BRCMSTB=y
CONFIG_PINCTRL_RP1=y             ← RP1 I/O controller (Pi5 compatible)
```

**Driver files:**
```
kernel/common/drivers/gpio/gpiolib.c                  ← Core GPIO library (5148 lines)
kernel/common/drivers/pinctrl/bcm/pinctrl-bcm2835.c   ← BCM2835/2711 pin control
kernel/common/drivers/gpio/gpio-raspberrypi-exp.c     ← Firmware GPIO expansion
```

---

## 2. Kiến trúc GPIO trên Pi4

```
Userspace (app/script)
        │
        ├── /sys/class/gpio/          (sysfs - legacy)
        ├── /dev/gpiochipN            (chardev - modern)
        │
        ▼
    GPIO core (gpiolib.c)
        │
        ├── gpiochip0 → BCM2711 GPIO controller (54 GPIO)
        │     └── pinctrl-bcm2835.c (mux, pull, drive)
        │
        └── gpiochip1 → Raspberry Pi firmware GPIO expansion
              └── gpio-raspberrypi-exp.c (qua mailbox VCHIQ)
```

---

## 3. Sysfs Interface (legacy)

```bash
# Export GPIO 17 (physical pin 11)
echo 17 > /sys/class/gpio/export

# Set direction: out
echo out > /sys/class/gpio/gpio17/direction

# Set value: HIGH
echo 1 > /sys/class/gpio/gpio17/value

# Set direction: in + detect edge
echo in > /sys/class/gpio/gpio17/direction
echo rising > /sys/class/gpio/gpio17/edge

# Read value
cat /sys/class/gpio/gpio17/value

# Cleanup
echo 17 > /sys/class/gpio/unexport
```

---

## 4. Character Device Interface (hiện đại — Linux 4.8+)

```bash
# List GPIO chips
gpiodetect

# List lines trên chip0
gpioinfo gpiochip0

# Read GPIO line 17
gpioget gpiochip0 17

# Set GPIO line 17 = 1
gpioset gpiochip0 17=1

# Monitor event (edge detection)
gpiomon gpiochip0 17
```

---

## 5. GPIO từ Kernel Module

```c
#include <linux/gpio/consumer.h>
#include <linux/platform_device.h>

static struct gpio_desc *led_gpio;

static int my_probe(struct platform_device *pdev)
{
    /* Lấy GPIO từ device tree */
    led_gpio = devm_gpiod_get(&pdev->dev, "led", GPIOD_OUT_LOW);
    if (IS_ERR(led_gpio))
        return PTR_ERR(led_gpio);

    /* Set HIGH */
    gpiod_set_value(led_gpio, 1);
    return 0;
}

/* Dùng interrupt */
static irqreturn_t button_irq_handler(int irq, void *data)
{
    pr_info("Button pressed!\n");
    return IRQ_HANDLED;
}

static int setup_irq(struct gpio_desc *btn_gpio)
{
    int irq = gpiod_to_irq(btn_gpio);
    return request_irq(irq, button_irq_handler,
                       IRQF_TRIGGER_RISING, "button", NULL);
}
```

---

## 6. Device Tree Overlay cho custom GPIO

```dts
/* overlays/my-led.dts */
/dts-v1/;
/plugin/;

/ {
    compatible = "brcm,bcm2711";

    fragment@0 {
        target = <&gpio>;
        __overlay__ {
            my_led_pin: my_led_pin {
                brcm,pins = <17>;
                brcm,function = <1>;  /* output */
                brcm,pull = <0>;      /* no pull */
            };
        };
    };

    fragment@1 {
        target-path = "/";
        __overlay__ {
            my_led {
                compatible = "gpio-leds";
                pinctrl-names = "default";
                pinctrl-0 = <&my_led_pin>;
                led0 {
                    label = "my-led";
                    gpios = <&gpio 17 0>;
                    default-state = "off";
                };
            };
        };
    };
};
```

```bash
# Build và load overlay
dtc -@ -I dts -O dtb -o my-led.dtbo my-led.dts
cp my-led.dtbo /boot/overlays/
# Thêm vào config.txt:
# dtoverlay=my-led
```

---

## 7. Đo GPIO Latency bằng Ftrace

```bash
# Trace gpio_set_value function
echo gpio_set_value > /sys/kernel/debug/tracing/set_ftrace_filter
echo function > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on

# Trigger GPIO toggle
echo 1 > /sys/class/gpio/gpio17/value
echo 0 > /sys/class/gpio/gpio17/value

# Đọc kết quả
cat /sys/kernel/debug/tracing/trace
```

---

## 8. 40-pin Header Map (BCM2711)

```
3.3V  [ 1] [ 2] 5V
GPIO2 [ 3] [ 4] 5V       ← I2C SDA1
GPIO3 [ 5] [ 6] GND      ← I2C SCL1
GPIO4 [ 7] [ 8] GPIO14   ← UART TX
 GND  [ 9] [10] GPIO15   ← UART RX
GPIO17[11] [12] GPIO18   ← PWM0
GPIO27[13] [14] GND
GPIO22[15] [16] GPIO23
3.3V  [17] [18] GPIO24
GPIO10[19] [20] GND      ← SPI0 MOSI
GPIO9 [21] [22] GPIO25   ← SPI0 MISO
GPIO11[23] [24] GPIO8    ← SPI0 CLK / CE0
 GND  [25] [26] GPIO7    ← SPI0 CE1
...
```

---

## 9. Vọc vạch thực tế

```bash
# 1. Blink LED không cần code — chỉ dùng shell
while true; do
    echo 1 > /sys/class/gpio/gpio17/value; sleep 0.5
    echo 0 > /sys/class/gpio/gpio17/value; sleep 0.5
done

# 2. Đo thời gian toggle GPIO (bằng Python)
# gpio_benchmark.py
import time, subprocess
t0 = time.monotonic_ns()
for _ in range(1000):
    subprocess.run(['gpioset', 'gpiochip0', '17=1'], check=True)
    subprocess.run(['gpioset', 'gpiochip0', '17=0'], check=True)
print(f"Avg toggle: {(time.monotonic_ns()-t0)//2000} ns")

# 3. Monitor IRQ count của GPIO
watch -n1 cat /proc/interrupts | grep gpio
```

---

## 10. File quan trọng trong source

| File | Mô tả |
|------|-------|
| `drivers/gpio/gpiolib.c` | GPIO core library |
| `drivers/gpio/gpiolib-sysfs.c` | Sysfs interface |
| `drivers/gpio/gpiolib-cdev.c` | Character device interface |
| `drivers/pinctrl/bcm/pinctrl-bcm2835.c` | BCM2711 pin mux/config |
| `drivers/gpio/gpio-raspberrypi-exp.c` | Firmware GPIO expansion |
| `include/linux/gpio/consumer.h` | Kernel GPIO API |
