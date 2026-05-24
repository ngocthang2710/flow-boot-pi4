# Kernel SPI — Raspberry Pi 4

## 1. Tổng quan

SPI (Serial Peripheral Interface) là bus 4 dây tốc độ cao (MOSI, MISO, CLK, CS) dùng cho ADC, flash memory, màn hình, sensor tốc độ cao.

**Config đã enabled:**
```
CONFIG_SPI=y
CONFIG_SPI_BCM2835=y       ← Hardware SPI0/SPI3/SPI4/SPI5/SPI6
CONFIG_SPI_BCM2835AUX=y    ← Auxiliary SPI1/SPI2 (mini SPI)
CONFIG_SPI_SPIDEV=y        ← /dev/spidevX.Y userspace access
CONFIG_SPI_GPIO=y           ← Bit-bang SPI qua GPIO
CONFIG_SPI_MEM=y            ← SPI memory interface (flash)
```

**Driver files:**
```
kernel/common/drivers/spi/spi-bcm2835.c      ← Hardware SPI driver (1466 lines)
kernel/common/drivers/spi/spi-bcm2835aux.c   ← Auxiliary SPI
kernel/common/drivers/spi/spidev.c           ← Userspace /dev/spidevX.Y
kernel/common/drivers/spi/spi.c              ← SPI core
```

---

## 2. SPI Controllers trên Pi4

```
BCM2711 SoC:
├── SPI0 → GPIO 9/10/11 + CE0(8) CE1(7)    ← 40-pin header (main)
├── SPI1 → GPIO 19/20/21 + CE0(18) (AUX)
├── SPI2 → GPIO 40/41/42 (CM4 only)
├── SPI3 → GPIO 1/2/3
├── SPI4 → GPIO 4/5/6
├── SPI5 → GPIO 12/13/14
└── SPI6 → GPIO 19/20/21

Userspace devices:
/dev/spidev0.0  ← SPI0, Chip Select 0
/dev/spidev0.1  ← SPI0, Chip Select 1
/dev/spidev1.0  ← SPI1, Chip Select 0
```

---

## 3. Enable SPI trong config.txt

```ini
# /boot/config.txt
dtparam=spi=on         # Enable SPI0 → /dev/spidev0.0 và /dev/spidev0.1

# Hoặc dùng overlay cụ thể:
dtoverlay=spi0-2cs     # SPI0 với 2 chip select
dtoverlay=spi1-1cs     # SPI1 với 1 chip select
```

---

## 4. Userspace SPI — spidev

```c
/* spi_loopback_test.c */
#include <linux/spi/spidev.h>
#include <sys/ioctl.h>
#include <fcntl.h>

int main(void)
{
    int fd = open("/dev/spidev0.0", O_RDWR);

    /* Cấu hình SPI mode */
    uint8_t mode = SPI_MODE_0;
    ioctl(fd, SPI_IOC_WR_MODE, &mode);

    /* Cấu hình bits per word */
    uint8_t bits = 8;
    ioctl(fd, SPI_IOC_WR_BITS_PER_WORD, &bits);

    /* Cấu hình tốc độ (Hz) */
    uint32_t speed = 1000000;  /* 1 MHz */
    ioctl(fd, SPI_IOC_WR_MAX_SPEED_HZ, &speed);

    /* Full-duplex transfer */
    uint8_t tx[] = { 0xAA, 0x55, 0xFF };
    uint8_t rx[sizeof(tx)];

    struct spi_ioc_transfer tr = {
        .tx_buf        = (unsigned long)tx,
        .rx_buf        = (unsigned long)rx,
        .len           = sizeof(tx),
        .delay_usecs   = 0,
        .speed_hz      = speed,
        .bits_per_word = bits,
    };

    ioctl(fd, SPI_IOC_MESSAGE(1), &tr);
    printf("RX: %02x %02x %02x\n", rx[0], rx[1], rx[2]);

    close(fd);
    return 0;
}
```

---

## 5. Viết Kernel SPI Driver

```c
/* mcp3204_driver.c — ADC 12-bit 4-channel qua SPI */
#include <linux/spi/spi.h>
#include <linux/module.h>
#include <linux/iio/iio.h>

struct mcp3204_data {
    struct spi_device *spi;
};

static int mcp3204_read_channel(struct spi_device *spi, int ch, int *val)
{
    uint8_t tx[3] = {
        0x06 | (ch >> 2),     /* Start bit + SGL/DIFF + D2 */
        (ch & 0x03) << 6,     /* D1 + D0 */
        0x00
    };
    uint8_t rx[3];

    struct spi_transfer t = {
        .tx_buf = tx,
        .rx_buf = rx,
        .len    = 3,
    };
    struct spi_message m;
    spi_message_init(&m);
    spi_message_add_tail(&t, &m);

    int ret = spi_sync(spi, &m);
    if (ret)
        return ret;

    *val = ((rx[1] & 0x0F) << 8) | rx[2];  /* 12-bit result */
    return 0;
}

static int mcp3204_probe(struct spi_device *spi)
{
    spi->max_speed_hz = 1000000;
    spi->mode         = SPI_MODE_0;
    spi->bits_per_word = 8;
    spi_setup(spi);

    dev_info(&spi->dev, "MCP3204 ADC ready\n");
    return 0;
}

static const struct spi_device_id mcp3204_id[] = {
    { "mcp3204", 0 },
    { }
};
MODULE_DEVICE_TABLE(spi, mcp3204_id);

static struct spi_driver mcp3204_driver = {
    .driver   = { .name = "mcp3204" },
    .probe    = mcp3204_probe,
    .id_table = mcp3204_id,
};
module_spi_driver(mcp3204_driver);
MODULE_LICENSE("GPL");
```

---

## 6. SPI DMA Transfer

BCM2835 SPI driver hỗ trợ DMA để giảm CPU load với transfers lớn:

```c
/* DMA được tự động dùng khi transfer >= 96 bytes */

/* Manually force DMA trong driver */
struct spi_transfer t = {
    .tx_buf = tx_buf,
    .rx_buf = rx_buf,
    .len    = 4096,       /* Lớn → tự dùng DMA */
};

/* Kiểm tra DMA usage */
// cat /sys/kernel/debug/spi0/statistics
```

---

## 7. Benchmark SPI Throughput

```bash
# Tốc độ tối đa SPI0 trên Pi4: ~32 MHz
# Test với spidev_test (từ kernel tools/spi/)
spidev_test -D /dev/spidev0.0 -s 32000000 -p "HELLO" -v

# Đo throughput thực tế
dd if=/dev/zero bs=4096 count=1000 | \
    pv | \
    spidev_test -D /dev/spidev0.0 -s 8000000 2>/dev/null

# Trace SPI transfers
echo bcm2835_spi_transfer_one_dma >> /sys/kernel/debug/tracing/set_ftrace_filter
echo function > /sys/kernel/debug/tracing/current_tracer
```

---

## 8. SPI NOR Flash (QSPI)

```bash
# Pi4 CM4 có eMMC qua SPI — debug bằng mtd tools
cat /proc/mtd

# Flash driver: drivers/mtd/spi-nor/
# Dùng flashrom để đọc/ghi SPI flash ngoài
flashrom -p linux_spi:dev=/dev/spidev0.0,spispeed=1000 -r backup.bin
```

---

## 9. Device Tree cho SPI device

```dts
&spi0 {
    status = "okay";

    /* MCP3204 ADC tại CS0 */
    mcp3204@0 {
        compatible = "microchip,mcp3204";
        reg = <0>;              /* Chip Select 0 */
        spi-max-frequency = <1000000>;
        spi-cpol;               /* CPOL=1 nếu cần */
        spi-cpha;               /* CPHA=1 nếu cần */
    };

    /* Màn hình SPI ILI9341 tại CS1 */
    ili9341@1 {
        compatible = "ilitek,ili9341";
        reg = <1>;
        spi-max-frequency = <32000000>;
        dc-gpios = <&gpio 24 GPIO_ACTIVE_HIGH>;
        reset-gpios = <&gpio 25 GPIO_ACTIVE_LOW>;
    };
};
```

---

## 10. SPI Devices phổ biến trên Pi4

| Device | Loại | Driver | Tốc độ |
|--------|------|--------|--------|
| MCP3204 | ADC 12-bit | `drivers/iio/adc/mcp320x.c` | 1 MHz |
| MCP2515 | CAN controller | `drivers/net/can/spi/mcp251x.c` | 10 MHz |
| W25Q32 | SPI NOR Flash | `drivers/mtd/spi-nor/` | 80 MHz |
| ILI9341 | TFT LCD | `drivers/staging/fbtft/` | 32 MHz |
| MAX31865 | RTD sensor | `drivers/iio/temperature/` | 1 MHz |
| NRF24L01 | RF transceiver | (external driver) | 8 MHz |
| BME280 | Env sensor (SPI) | `drivers/iio/pressure/bmp280.c` | 1 MHz |

---

## 11. File quan trọng trong source

| File | Mô tả |
|------|-------|
| `drivers/spi/spi-bcm2835.c` | BCM2711 hardware SPI driver, DMA support |
| `drivers/spi/spi-bcm2835aux.c` | Auxiliary mini-SPI (SPI1/SPI2) |
| `drivers/spi/spidev.c` | Userspace /dev/spidevX.Y |
| `drivers/spi/spi.c` | SPI core, message queue |
| `drivers/iio/adc/mcp320x.c` | MCP3204/3208 ADC driver |
| `drivers/net/can/spi/mcp251x.c` | MCP2515 CAN driver |
| `include/linux/spi/spi.h` | SPI kernel API |
