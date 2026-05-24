# Kernel I2C — Raspberry Pi 4

## 1. Tổng quan

I2C (Inter-Integrated Circuit) là bus 2 dây (SDA + SCL) để giao tiếp với các sensor, EEPROM, display controller, v.v.

**Config đã enabled:**
```
CONFIG_I2C=y
CONFIG_I2C_BCM2835=y        ← Hardware I2C controller BCM2711
CONFIG_I2C_BCM2708=y        ← Bit-bang I2C (software)
CONFIG_I2C_BRCMSTB=y
CONFIG_I2C_GPIO=y            ← GPIO bit-bang I2C
CONFIG_I2C_CHARDEV=y         ← /dev/i2c-N userspace access
CONFIG_I2C_MUX=y             ← I2C multiplexer support
```

**Driver files:**
```
kernel/common/drivers/i2c/busses/i2c-bcm2835.c   ← Hardware I2C driver
kernel/common/drivers/i2c/busses/i2c-bcm2708.c   ← Bit-bang fallback
kernel/common/drivers/i2c/i2c-core-base.c         ← I2C core
kernel/common/drivers/i2c/i2c-dev.c               ← /dev/i2c-N
```

---

## 2. Kiến trúc I2C trên Pi4

```
BCM2711 SoC
├── I2C Controller 0 (BSC0) → GPIO 0/1   (camera, EEPROM HAT)
├── I2C Controller 1 (BSC1) → GPIO 2/3   ← User I2C bus (40-pin pin 3/5)
├── I2C Controller 3 → GPIO 4/5
├── I2C Controller 4 → GPIO 8/9
├── I2C Controller 5 → GPIO 12/13
└── I2C Controller 6 → GPIO 22/23

Userspace access:
/dev/i2c-0   ← BSC0
/dev/i2c-1   ← BSC1 (thường dùng)
/dev/i2c-N   ...
```

---

## 3. Userspace I2C — i2c-tools

```bash
# Detect devices trên bus 1
i2cdetect -y 1
#      0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
# 00:          -- -- -- -- -- -- -- -- -- -- -- -- --
# 10: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
# 20: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
# 30: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
# 40: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
# 50: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
# 60: -- -- -- -- -- -- -- -- 68 -- -- -- -- -- -- --
# 70: -- -- -- -- -- -- -- --
# 0x68 = MPU6050 gyroscope/accelerometer

# Đọc register 0x75 (WHO_AM_I) từ MPU6050 tại addr 0x68
i2cget -y 1 0x68 0x75
# 0x68

# Ghi vào register
i2cset -y 1 0x68 0x6B 0x00  # Wake up MPU6050 (power management)

# Dump toàn bộ register space
i2cdump -y 1 0x68
```

---

## 4. I2C từ Userspace (ioctl)

```c
/* i2c_userspace.c */
#include <linux/i2c-dev.h>
#include <sys/ioctl.h>
#include <fcntl.h>

#define MPU6050_ADDR  0x68
#define REG_ACCEL_X   0x3B

int main(void)
{
    int fd = open("/dev/i2c-1", O_RDWR);

    /* Set slave address */
    ioctl(fd, I2C_SLAVE, MPU6050_ADDR);

    /* Write register address, then read data */
    uint8_t reg = REG_ACCEL_X;
    write(fd, &reg, 1);

    uint8_t buf[6];
    read(fd, buf, 6);  /* 6 bytes = X, Y, Z (16-bit each) */

    int16_t ax = (buf[0] << 8) | buf[1];
    int16_t ay = (buf[2] << 8) | buf[3];
    int16_t az = (buf[4] << 8) | buf[5];

    printf("Accel: X=%d Y=%d Z=%d\n", ax, ay, az);
    close(fd);
    return 0;
}
```

---

## 5. Viết Kernel I2C Driver

```c
/* my_sensor.c — Driver cho sensor giả định tại I2C addr 0x48 */
#include <linux/i2c.h>
#include <linux/module.h>
#include <linux/iio/iio.h>

#define MY_SENSOR_TEMP_REG  0x00
#define MY_SENSOR_ADDR      0x48

struct my_sensor_data {
    struct i2c_client *client;
};

static int my_sensor_read_temp(struct i2c_client *client, int *temp)
{
    s32 val = i2c_smbus_read_word_swapped(client, MY_SENSOR_TEMP_REG);
    if (val < 0)
        return val;

    /* Convert raw value to millidegree Celsius */
    *temp = (val >> 4) * 625 / 10;
    return 0;
}

static int my_sensor_probe(struct i2c_client *client)
{
    struct my_sensor_data *data;

    data = devm_kzalloc(&client->dev, sizeof(*data), GFP_KERNEL);
    if (!data)
        return -ENOMEM;

    data->client = client;
    i2c_set_clientdata(client, data);

    dev_info(&client->dev, "my_sensor probed at 0x%02x\n",
             client->addr);
    return 0;
}

static const struct i2c_device_id my_sensor_id[] = {
    { "my_sensor", 0 },
    { }
};
MODULE_DEVICE_TABLE(i2c, my_sensor_id);

static const struct of_device_id my_sensor_dt_ids[] = {
    { .compatible = "myvendor,my-sensor" },
    { }
};

static struct i2c_driver my_sensor_driver = {
    .driver = {
        .name           = "my_sensor",
        .of_match_table = my_sensor_dt_ids,
    },
    .probe    = my_sensor_probe,
    .id_table = my_sensor_id,
};

module_i2c_driver(my_sensor_driver);
MODULE_LICENSE("GPL");
```

---

## 6. Các Sensor I2C phổ biến trên Pi4

| Sensor | Addr | Driver trong kernel | Đo |
|--------|------|--------------------|----|
| MPU-6050 | 0x68/0x69 | `drivers/iio/imu/inv_mpu6050/` | Gia tốc + Gyro |
| BMP280 | 0x76/0x77 | `drivers/iio/pressure/bmp280.c` | Áp suất + Nhiệt độ |
| BMP180 | 0x77 | `drivers/iio/pressure/bmp085.c` | Áp suất + Nhiệt độ |
| LSM6DSX | 0x6A/0x6B | `drivers/iio/imu/st_lsm6dsx/` | 6-axis IMU |
| SHT3x | 0x44/0x45 | `drivers/hwmon/sht3x.c` | Nhiệt độ + Độ ẩm |
| ADXL345 | 0x53/0x1D | `drivers/iio/accel/adxl345_i2c.c` | Gia tốc kế |
| HMC5883L | 0x1E | `drivers/iio/magnetometer/hmc5843.c` | Từ trường (compass) |
| ADS1115 | 0x48-0x4B | `drivers/iio/adc/ads1015.c` | ADC 16-bit |

---

## 7. Debug I2C

```bash
# Enable I2C debug logging
echo 7 > /sys/module/i2c_bcm2835/parameters/debug

# Trace I2C transfers với Ftrace
echo i2c_bcm2835_xfer > /sys/kernel/debug/tracing/set_ftrace_filter
echo function > /sys/kernel/debug/tracing/current_tracer

# Monitor tốc độ I2C bus (default 100kHz, có thể tăng lên 400kHz)
# Thêm vào config.txt: dtparam=i2c_arm_baudrate=400000
cat /sys/bus/i2c/devices/i2c-1/of_node/clock-frequency

# Scan tất cả I2C bus
for i in $(seq 0 9); do
    [ -e /dev/i2c-$i ] && echo "Bus $i:" && i2cdetect -y $i
done
```

---

## 8. I2C Performance Test

```bash
# Benchmark read throughput
i2ctransfer -y 1 w1@0x68 0x3B r14  # Đọc 14 bytes liên tiếp từ MPU6050

# Stress test với nhiều lần đọc
for i in $(seq 1 1000); do
    i2cget -y 1 0x68 0x75 > /dev/null
done
time ...
```

---

## 9. Device Tree cho I2C device

```dts
/* Thêm vào Device Tree Overlay */
&i2c1 {
    status = "okay";
    clock-frequency = <400000>;  /* Fast mode 400kHz */

    mpu6050: mpu6050@68 {
        compatible = "invensense,mpu6050";
        reg = <0x68>;
        interrupt-parent = <&gpio>;
        interrupts = <4 IRQ_TYPE_LEVEL_HIGH>;
    };

    bmp280: bmp280@76 {
        compatible = "bosch,bmp280";
        reg = <0x76>;
    };
};
```

---

## 10. File quan trọng trong source

| File | Mô tả |
|------|-------|
| `drivers/i2c/busses/i2c-bcm2835.c` | Hardware I2C driver cho BCM2711 |
| `drivers/i2c/i2c-core-base.c` | I2C core subsystem |
| `drivers/i2c/i2c-dev.c` | /dev/i2c-N interface |
| `drivers/iio/imu/inv_mpu6050/` | MPU-6050/9250 driver |
| `drivers/iio/pressure/bmp280-core.c` | BMP280 driver |
| `include/linux/i2c.h` | I2C kernel API |
