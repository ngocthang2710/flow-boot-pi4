# Raspberry Pi 4 — OTA & Bootloader

## 1. Pi4 có cơ chế OTA không?

**Không.** Pi4 trong Android-16 (port của KonstaKANG) **không có OTA mechanism**.

### Bằng chứng từ source

**`device/brcm/rpi4/BoardConfig.mk`** — Không có bất kỳ config A/B nào:

```makefile
TARGET_NO_BOOTLOADER := true   # Không có bootloader hỗ trợ slot switching
TARGET_NO_RECOVERY   := true   # Không có recovery partition

# Static partitions — kích thước cố định
BOARD_BOOTIMAGE_PARTITION_SIZE   := 134217728  # 128MB
BOARD_SYSTEMIMAGE_PARTITION_SIZE := 3221225472 # 3072MB
BOARD_VENDORIMAGE_PARTITION_SIZE := 402653184  # 384MB
BOARD_USERDATAIMAGE_PARTITION_SIZE := 134217728 # 128MB

# KHÔNG CÓ:
# BOARD_SUPER_PARTITION_SIZE
# AB_OTA_PARTITIONS
# PRODUCT_VIRTUAL_AB_OTA
```

**`device/brcm/rpi4/ramdisk/fstab.rpi4`** — Block device cứng, không có slot suffix:

```
/dev/block/mmcblk0p5   /system   ext4  ro,barrier=1  wait,first_stage_mount
/dev/block/mmcblk0p6   /vendor   ext4  ro,barrier=1  wait,first_stage_mount
/dev/block/mmcblk0p7   /metadata ext4  nodev,...      wait,check,formattable
/dev/block/mmcblk0p3   /data     ext4  nodev,...      wait,check,formattable,quota
```

So sánh với device có A/B (Cuttlefish): `system /system erofs ro logical,slotselect,avb=vbmeta_system`

### Lý do không có OTA

1. **RPi firmware** không hỗ trợ A/B slot switching (khác Qualcomm/MediaTek ABL)
2. **SD card** thay được dễ dàng — flash lại nhanh hơn làm OTA
3. Không có **Bootloader Control Block (BCB)** để bootloader đọc slot metadata
4. Đây là board development/hobbyist, không phải thiết bị thương mại

### Cách update Pi4

| Phương án | Cách làm |
|-----------|---------|
| **Full flash** | `dd if=RaspberryVanillaAOSP16-*.img of=/dev/sdX` |
| **Từng partition** | `dd if=system.img of=/dev/mmcblk0p5` (cần tắt máy) |
| **APEX update** | Vẫn hoạt động — `device.mk` include `updatable_apex.mk` |
| **ADB push** | Push file thủ công qua ADB |

---

## 2. Bootloader Pi4 — Kiến trúc đặc biệt

Pi4 không dùng bootloader theo nghĩa Android. Thay vào đó là hệ thống **GPU-first boot** của Broadcom.

### 4 giai đoạn boot

```
Power ON
    │
    ▼ Stage 1
┌─────────────────────────────────────┐
│  ROM Bootloader (BCM2711 on-chip)   │
│  Chạy trên GPU VideoCore VI         │
│  • Cố định, không thể thay đổi      │
│  • Đọc EEPROM để load stage 2       │
└──────────────────┬──────────────────┘
                   │
    ▼ Stage 2
┌─────────────────────────────────────┐
│  EEPROM Bootloader                  │
│  (cập nhật được qua rpi-eeprom-update)│
│  • Chọn boot source theo thứ tự:    │
│    SD card → USB → PXE network     │
│  • Load GPU firmware từ FAT32      │
└──────────────────┬──────────────────┘
                   │ Đọc từ SD card partition p1 (FAT32, 128MB)
    ▼ Stage 3
┌─────────────────────────────────────┐
│  GPU Firmware (VideoCore VI)        │
│  start4.elf + fixup4.dat           │
│  (proprietary Broadcom binary)      │
│  • Đọc config.txt → cấu hình HW   │
│  • Setup ARM Cortex-A72 cores      │
│  • Load kernel Image               │
│  • Load DTB (bcm2711-rpi-4-b.dtb) │
│  • Đọc cmdline.txt → kernel args  │
└──────────────────┬──────────────────┘
                   │
    ▼ Stage 4
┌─────────────────────────────────────┐
│  Linux Kernel (ARM64)               │
│  • Parse DTB — nhận diện phần cứng │
│  • Mount ramdisk.img               │
│  • Khởi động Android first-stage init│
└─────────────────────────────────────┘
```

> **Điểm đặc biệt:** ARM Cortex-A72 **không phải core khởi động đầu tiên**. GPU VideoCore VI mới là core chạy đầu tiên, setup mọi thứ rồi mới đánh thức ARM. Không có `start4.elf` thì Pi4 không boot được dù kernel đúng.

---

## 3. Nội dung FAT32 boot partition (p1, 128MB)

Được tạo bởi `device/brcm/rpi4/mkbootimg.mk`:

```
boot/ (FAT32)
├── config.txt              ← GPU firmware đọc đầu tiên
├── cmdline.txt             ← Kernel command line (sinh lúc build)
├── Image                   ← Linux kernel ARM64
├── ramdisk.img             ← Android ramdisk (init + first-stage)
│
├── bcm2711-rpi-4-b.dtb     ← Device Tree cho Pi4 Model B
├── bcm2711-rpi-400.dtb     ← Pi 400
├── bcm2711-rpi-cm4.dtb     ← Compute Module 4
├── bcm2711-rpi-cm4s.dtb
├── bcm2711-rpi-cm4-io.dtb
│
├── overlays/               ← 370 Device Tree Overlay (.dtbo)
│   ├── vc4-kms-v3d.dtbo   ← Mesa/DRM graphics
│   ├── dwc2.dtbo          ← USB device mode
│   ├── disable-bt.dtbo    ← Disable Bluetooth
│   └── ...
│
├── start4.elf              ← GPU firmware (proprietary)
└── fixup4.dat              ← Relocation data cho start4.elf
```

---

## 4. `config.txt` — Thay thế cho bootloader config

```ini
# device/brcm/rpi4/boot/config.txt

# Kernel
arm_64bit=1
kernel=Image

# Ramdisk (load ngay sau kernel trong RAM)
initramfs ramdisk.img followkernel

# Hardware
dtparam=audio=on       # Enable audio qua GPIO
camera_auto_detect=1   # Tự detect camera module
arm_boost=1            # CPU boost

# Graphics — dùng KMS/DRM thay GPU firmware
disable_fw_kms_setup=1
dtoverlay=vc4-kms-v3d

# Debug
enable_uart=1          # Serial console ttyS0

# USB OTG (khác nhau giữa Pi4 và CM4)
dtoverlay=dwc2,dr_mode=peripheral
[cm4]
dtoverlay=dwc2,dr_mode=otg
[all]
```

---

## 5. `cmdline.txt` — Kernel arguments

Sinh lúc build từ `BOARD_KERNEL_CMDLINE`:

```
console=ttyS0,115200 androidboot.hardware=rpi4 androidboot.selinux=permissive
```

| Argument | Ý nghĩa |
|----------|---------|
| `console=ttyS0,115200` | Serial console debug qua UART |
| `androidboot.hardware=rpi4` | Android set `ro.hardware=rpi4` |
| `androidboot.selinux=permissive` | SELinux permissive (không enforce) |

---

## 6. Device Tree (DTB) và Overlay

DTB mô tả toàn bộ phần cứng cho kernel (Pi4 không có BIOS/UEFI):

```
bcm2711-rpi-4-b.dtb mô tả:
├── CPU: 4x Cortex-A72 @ 1.5GHz (BCM2711)
├── RAM: LPDDR4
├── GPIO controller (40-pin header)
├── I2C, SPI, UART buses
├── USB 3.0 controller (xhci)
├── PCIe bridge (→ USB 3.0, Ethernet)
├── VideoCore VI GPU
└── Camera interface (CSI-2)
```

Overlay (`.dtbo`) là các patch runtime cho DTB:
- `vc4-kms-v3d.dtbo` — Enable DRM/KMS display pipeline
- `dwc2.dtbo` — USB device mode (Gadget)
- `ramoops.dtbo` — Kernel crash logging

---

## 7. Layout SD card

```
# Tạo bởi rpi4-mkimg.sh
SD Card (15GB image, MBR partition table):

┌──────────────────────────────────────────────────────────────┐
│ p1: FAT32, 128MB  │ p2: Extended                            │
│ boot partition    │  ┌───────────┬──────────┬────────────┐  │
│ (GPU firmware,    │  │p5: ext4   │p6: ext4  │p7: ext4    │  │
│  kernel, DTB)     │  │ /system   │ /vendor  │ /metadata  │  │
│                   │  │ 3072MB    │ 384MB    │ 16MB       │  │
│                   │  └───────────┴──────────┴────────────┘  │
│                   │                                          │
│ p3: ext4, ~11GB (userdata /data)                            │
└──────────────────────────────────────────────────────────────┘

Block mapping:
  mmcblk0p1 → /boot      (FAT32, GPU firmware + kernel)
  mmcblk0p3 → /data      (ext4, userdata)
  mmcblk0p5 → /system    (ext4, Android framework)
  mmcblk0p6 → /vendor    (ext4, HAL + drivers)
  mmcblk0p7 → /metadata  (ext4, dynamic partition metadata)
```

---

## 8. So sánh với Android bootloader chuẩn

| Đặc điểm | Pi4 (RPi Firmware) | Android Phone (ABL/LK) |
|-----------|-------------------|----------------------|
| Boot đầu tiên | GPU VideoCore VI | ARM primary core |
| Cấu hình | `config.txt` (plain text) | BCB (binary block) |
| Fastboot | Không có | Có |
| A/B slot | Không có | Có |
| Secure Boot / AVB | Không | Có |
| Recovery | Không có | Có (partition riêng) |
| Boot source | SD → USB → PXE | eMMC cố định |
| Cập nhật | Flash lại SD card | OTA / fastboot |
