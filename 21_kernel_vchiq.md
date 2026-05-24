# Kernel VCHIQ — VideoCore/ARM Interface — Raspberry Pi 4

## 1. Tổng quan

VCHIQ (VideoCore Host Interface Queue) là cơ chế IPC đặc biệt của Raspberry Pi để ARM cores giao tiếp với VideoCore firmware đang chạy trên GPU.

**Config đã enabled:**
```
CONFIG_BCM2835_VCHIQ=y      ← VCHIQ core driver
CONFIG_BCM2835_MBOX=y       ← Mailbox (kênh giao tiếp firmware)
CONFIG_RASPBERRYPI_FIRMWARE=y ← Raspberry Pi firmware interface
CONFIG_BCM2835_POWER=y      ← Power management qua firmware
CONFIG_VIDEO_BCM2835=y      ← Camera V4L2 driver qua VCHIQ
CONFIG_SND_BCM2835=y        ← Audio qua VCHIQ
```

**Driver files:**
```
kernel/common/drivers/staging/vc04_services/
├── interface/vchiq_arm/
│   ├── vchiq_core.c        ← VCHIQ protocol core (2000+ lines)
│   ├── vchiq_arm.c         ← ARM side implementation
│   ├── vchiq_dev.c         ← /dev/vchiq userspace device
│   └── vchiq_debugfs.c     ← DebugFS statistics
├── bcm2835-camera/         ← Camera qua VCHIQ
│   ├── mmal-vchiq.c        ← MMAL protocol over VCHIQ
│   └── bcm2835-camera.c    ← V4L2 driver
└── bcm2835-audio/          ← Audio qua VCHIQ
    └── bcm2835-pcm.c       ← ALSA PCM driver
```

---

## 2. Kiến trúc VCHIQ

```
┌─────────────────────────────────┐
│   ARM Cortex-A72 (Linux)        │
│                                 │
│  /dev/vchiq                     │
│  Camera (V4L2 → MMAL → VCHIQ)  │
│  Audio (ALSA → VCHIQ)           │
│                                 │
│  vchiq_arm.c                    │
│  vchiq_core.c (queue protocol)  │
└──────────┬──────────────────────┘
           │ Shared Memory (DMA coherent)
           │ Doorbell register (notify)
┌──────────▼──────────────────────┐
│   VideoCore VI Firmware         │
│   (runs on GPU, always on)      │
│                                 │
│   VCHIQ service dispatcher      │
│   ├── MMAL (multimedia)         │
│   ├── VCSM (shared memory)      │
│   ├── Mailbox services          │
│   └── Camera ISP pipeline       │
└─────────────────────────────────┘
```

---

## 3. VCHIQ Protocol — Cơ chế hoạt động

### Shared Memory Slots
```
Shared Memory Layout:
┌──────────────────────────────┐
│ Master slot (ARM → VC)       │ ← ARM ghi, VC đọc
├──────────────────────────────┤
│ Slave slot (VC → ARM)        │ ← VC ghi, ARM đọc
├──────────────────────────────┤
│ Message data                 │ ← Payload
└──────────────────────────────┘

Message types:
VCHIQ_MSG_CONNECT     - Kết nối khởi tạo
VCHIQ_MSG_OPEN        - Mở service
VCHIQ_MSG_OPENACK     - Xác nhận mở
VCHIQ_MSG_CLOSE       - Đóng service
VCHIQ_MSG_DATA        - Gửi data
VCHIQ_MSG_BULK_RX/TX  - Bulk transfer (DMA)
```

---

## 4. /dev/vchiq — Userspace Interface

```bash
# Kiểm tra VCHIQ device
ls -la /dev/vchiq
# crw-rw---- root  video  ...

# VCHIQ debug stats
cat /sys/kernel/debug/vchiq/stats

# Service stats
cat /sys/kernel/debug/vchiq/services
# service 0: MMAL
# service 1: ...
```

---

## 5. Mailbox — Firmware Property Interface

Mailbox là cơ chế đơn giản hơn VCHIQ để query firmware properties:

```bash
# Đọc properties qua vcgencmd (dùng mailbox internally)
vcgencmd measure_temp      # Nhiệt độ
vcgencmd measure_clock arm # CPU frequency
vcgencmd measure_clock core # GPU frequency
vcgencmd measure_volts core # Core voltage
vcgencmd get_mem arm       # ARM memory
vcgencmd get_mem gpu       # GPU memory
vcgencmd codec_enabled H264 # Codec support
vcgencmd get_config int    # config.txt values
vcgencmd display_power 0   # Turn off display
vcgencmd display_power 1   # Turn on display
```

---

## 6. Mailbox từ Kernel

```c
/* Dùng firmware mailbox để query property */
#include <linux/firmware/raspberrypi.h>

struct rpi_firmware *fw = rpi_firmware_get(NULL);

/* Query ARM memory */
struct {
    u32 base;
    u32 size;
} arm_mem;

rpi_firmware_property(fw,
    RPI_FIRMWARE_GET_ARM_MEMORY,
    &arm_mem, sizeof(arm_mem));

pr_info("ARM memory: base=0x%x size=%u MB\n",
        arm_mem.base, arm_mem.size >> 20);

/* Set CPU frequency */
u32 rate = 1500000000;  /* 1.5 GHz */
rpi_firmware_property(fw,
    RPI_FIRMWARE_SET_CLOCK_RATE,
    &(struct { u32 id; u32 rate; }){ 3, rate },
    sizeof(u32) * 2);
```

---

## 7. Camera qua VCHIQ/MMAL

MMAL (Multi-Media Abstraction Layer) là protocol chạy trên VCHIQ để control camera ISP trong firmware:

```bash
# Chụp ảnh qua MMAL (libcamera sử dụng VCHIQ)
libcamera-still -o photo.jpg

# Video capture
libcamera-vid -o video.h264 --width 1920 --height 1080

# Preview
libcamera-hello --timeout 5000

# Debug MMAL/VCHIQ camera
LIBCAMERA_LOG_LEVELS=*:DEBUG libcamera-still -o test.jpg 2>&1 | head -50
```

```bash
# V4L2 devices từ bcm2835-camera
ls /dev/video*
# /dev/video0   ← Camera

# Xem camera info
v4l2-ctl --list-devices
v4l2-ctl -d /dev/video0 --list-formats-ext

# Capture qua V4L2
v4l2-ctl --stream-mmap --stream-count=1 \
    --set-fmt-video=width=1920,height=1080,pixelformat=H264 \
    --stream-to=output.h264
```

---

## 8. Audio qua VCHIQ

```bash
# BCM2835 audio (analog 3.5mm + HDMI) qua VCHIQ
aplay -l
# card 0: bcm2835 HDMI 1
# card 1: bcm2835 Headphones

# Chọn output
amixer cset numid=3 2    # HDMI
amixer cset numid=3 1    # Analog jack

# Test audio
speaker-test -t sine -f 440

# Debug audio driver
cat /proc/asound/cards
cat /proc/asound/pcm
```

---

## 9. VCHIQ Debug

```bash
# Xem VCHIQ internal state
cat /sys/kernel/debug/vchiq/state
# State: connected
# tx_pos: 12345
# rx_pos: 12340

# Slot statistics
cat /sys/kernel/debug/vchiq/slots
# Slot 0: ARM→VC, used: 45, free: 83
# Slot 1: VC→ARM, used: 12, free: 116

# Trace VCHIQ messages
echo vchiq_arm_message_received >> /sys/kernel/debug/tracing/set_ftrace_filter
echo function >> /sys/kernel/debug/tracing/current_tracer

# Xem MMAL component list (nếu camera active)
cat /sys/kernel/debug/vchiq/services
```

---

## 10. GPU Memory (VCSM)

VCSM (VideoCore Shared Memory) cho phép ARM và VC dùng chung bộ nhớ:

```bash
# GPU memory split (ARM vs GPU)
vcgencmd get_mem arm
# arm=948M

vcgencmd get_mem gpu
# gpu=76M

# Thay đổi split trong config.txt:
# gpu_mem=128   ← GPU nhận 128MB
# Còn lại cho ARM
```

---

## 11. Khám phá VCHIQ từ Userspace

```c
/* vchiq_test.c — Mở VCHIQ và query version */
#include <fcntl.h>
#include <sys/ioctl.h>
#include <linux/ioctl.h>

#define VCHIQ_IOC_MAGIC  0xc4
#define VCHIQ_IOCGETCONFIG _IOR(VCHIQ_IOC_MAGIC, 4, int)

int main(void)
{
    int fd = open("/dev/vchiq", O_RDWR);
    if (fd < 0) { perror("open"); return 1; }

    int version = 0;
    ioctl(fd, VCHIQ_IOCGETCONFIG, &version);
    printf("VCHIQ version: %d\n", version);

    close(fd);
    return 0;
}
```

---

## 12. File quan trọng trong source

| File | Mô tả |
|------|-------|
| `drivers/staging/vc04_services/interface/vchiq_arm/vchiq_core.c` | VCHIQ protocol core |
| `drivers/staging/vc04_services/interface/vchiq_arm/vchiq_arm.c` | ARM side VCHIQ |
| `drivers/staging/vc04_services/interface/vchiq_arm/vchiq_dev.c` | /dev/vchiq |
| `drivers/staging/vc04_services/bcm2835-camera/mmal-vchiq.c` | Camera MMAL over VCHIQ |
| `drivers/staging/vc04_services/bcm2835-audio/bcm2835-pcm.c` | Audio ALSA driver |
| `drivers/firmware/raspberrypi.c` | Mailbox firmware interface |
| `drivers/mailbox/bcm2835-mailbox.c` | Hardware mailbox |
| `include/linux/firmware/raspberrypi.h` | Firmware API |
