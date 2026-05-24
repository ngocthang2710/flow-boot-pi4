# Android Partition Layout

## 1. Tổng quan

Android sử dụng hệ thống phân vùng phân tầng, được định nghĩa ở nhiều cấp:

```
BoardConfig.mk  →  Kích thước và cấu hình partition
fstab.*         →  Ánh xạ partition tới mount point
liblp           →  Thư viện quản lý logical partition (runtime)
```

---

## 2. Nơi định nghĩa partition

### Cấp 1: `BoardConfig.mk` — Khai báo kích thước

```makefile
# source/device/google/cuttlefish/shared/BoardConfig.mk

# Super partition chứa tất cả logical partitions
BOARD_SUPER_PARTITION_SIZE := 8589934592   # 8 GiB

# Chia thành 2 group
BOARD_SUPER_PARTITION_GROUPS := google_system_dynamic_partitions \
                                 google_vendor_dynamic_partitions

# Group System (6.375 GiB)
BOARD_GOOGLE_SYSTEM_DYNAMIC_PARTITIONS_SIZE := 6845104128
BOARD_GOOGLE_SYSTEM_DYNAMIC_PARTITIONS_PARTITION_LIST := \
    product system system_ext system_dlkm

# Group Vendor (1.4 GiB)
BOARD_GOOGLE_VENDOR_DYNAMIC_PARTITIONS_SIZE := 1472200704
BOARD_GOOGLE_VENDOR_DYNAMIC_PARTITIONS_PARTITION_LIST := \
    odm vendor vendor_dlkm odm_dlkm
```

### Cấp 2: `fstab.*` — Mount table

```
# source/device/google/cuttlefish/shared/config/fstab.in
# <block device>               <mount>    <fs>   <flags>
/dev/block/by-name/boot        /boot      emmc   defaults  recoveryonly,slotselect,avb=boot
/dev/block/by-name/init_boot   /init_boot emmc   defaults  recoveryonly,slotselect
system                         /system    erofs  ro        logical,first_stage_mount,slotselect,avb=vbmeta_system
vendor                         /vendor    erofs  ro        logical,first_stage_mount,slotselect,avb=vbmeta
userdata                       /data      ext4   nodev     latemount,wait,check,quota
```

### Cấp 3: `liblp` — Thư viện quản lý logical partition

```
source/system/core/fs_mgr/liblp/
├── metadata_format.h   ← Cấu trúc metadata trong super partition
├── builder.h           ← Tạo partition table lúc build
├── reader.h            ← Đọc metadata lúc runtime
└── writer.h            ← Ghi metadata (OTA update)
```

---

## 3. Tất cả các partition và vai trò

| Partition | Loại | Mount | Filesystem | Vai trò |
|-----------|------|-------|-----------|---------|
| `boot` | Physical | `/boot` | emmc | Kernel + ramdisk |
| `init_boot` | Physical | `/init_boot` | emmc | Init ramdisk riêng (Android 13+) |
| `vendor_boot` | Physical | `/vendor_boot` | emmc | Vendor ramdisk |
| `super` | Physical | — | — | Container cho logical partitions |
| `system` | Logical (super) | `/system` | erofs/ext4 | Framework Android, read-only |
| `vendor` | Logical (super) | `/vendor` | erofs/ext4 | HAL, driver, read-only |
| `product` | Logical (super) | `/product` | erofs/ext4 | App OEM, read-only |
| `system_ext` | Logical (super) | `/system_ext` | erofs/ext4 | System extension |
| `odm` | Logical (super) | `/odm` | erofs/ext4 | OEM device manifest |
| `system_dlkm` | Logical (super) | `/system_dlkm` | erofs | Kernel modules (system) |
| `vendor_dlkm` | Logical (super) | `/vendor_dlkm` | erofs | Kernel modules (vendor) |
| `odm_dlkm` | Logical (super) | `/odm_dlkm` | erofs | Kernel modules (ODM) |
| `userdata` | Physical | `/data` | f2fs/ext4 | Data user, writable |
| `metadata` | Physical | `/metadata` | ext4 | Metadata cho dynamic partition |
| `vbmeta` | Physical | — | — | AVB verification data |

---

## 4. Layout vật lý của super partition

```
super partition:
┌──────────────────────────────┐
│ Disk Geometry (4096 bytes)   │ ← Magic: 0x616c4447
│ Geometry Backup (4096 bytes) │
│ Metadata (partition table)   │ ← Magic: 0x414C5030
│ Metadata Backup              │
├──────────────────────────────┤
│  system  (logical, 3-5 GB)   │
│  product (logical)           │
│  system_ext (logical)        │
│  system_dlkm (logical)       │
├──────────────────────────────┤
│  vendor  (logical)           │
│  odm     (logical)           │
│  vendor_dlkm (logical)       │
│  odm_dlkm  (logical)         │
└──────────────────────────────┘
```

---

## 5. Partition Attributes

```c
// source/system/core/fs_mgr/liblp/include/liblp/metadata_format.h
#define LP_PARTITION_ATTR_NONE          0x0
#define LP_PARTITION_ATTR_READONLY      (1 << 0)  // Read-only
#define LP_PARTITION_ATTR_SLOT_SUFFIXED (1 << 1)  // Cần slot suffix (_a/_b)
#define LP_PARTITION_ATTR_UPDATED       (1 << 2)  // Được update bởi OTA
#define LP_PARTITION_ATTR_DISABLED      (1 << 3)  // Partition bị disabled
```

---

## 6. Filesystem types được hỗ trợ

| Filesystem | Dùng cho | Đặc điểm |
|-----------|---------|---------|
| `ext4` | system, vendor, userdata | Traditional, stable |
| `erofs` | system, vendor (read-only) | Compressed, read-only, nhanh hơn |
| `f2fs` | userdata | Flash-friendly, tốt cho NAND |

```makefile
BOARD_SYSTEMIMAGE_FILE_SYSTEM_TYPE := erofs   # hoặc ext4
BOARD_VENDORIMAGE_FILE_SYSTEM_TYPE := erofs
BOARD_USERDATAIMAGE_FILE_SYSTEM_TYPE := f2fs  # hoặc ext4
```

---

## 7. Luồng mount lúc boot

```
Kernel boot
    │
    ▼
First-stage init (ramdisk)
    │  Đọc metadata từ /dev/block/by-name/super
    │  Tạo logical block devices qua Device Mapper:
    │    /dev/block/mapper/system
    │    /dev/block/mapper/vendor
    │    /dev/block/mapper/product ...
    │  Mount theo fstab (first_stage_mount entries)
    ▼
Second-stage init (/system/bin/init)
    │  Mount /data (latemount)
    │  Khởi động native services
    ▼
system_server / zygote
```

---

## 8. Android Verified Boot (AVB)

Mỗi partition được verify bởi AVB trước khi mount:

```makefile
# source/device/google/cuttlefish/shared/BoardConfig.mk
BOARD_AVB_ENABLE := true
BOARD_AVB_ALGORITHM := SHA256_RSA4096

# Chain vbmeta theo nhóm
BOARD_AVB_VBMETA_SYSTEM := product system system_ext
BOARD_AVB_VBMETA_VENDOR := vendor odm
```

```
vbmeta (root)
├── vbmeta_system → system, product, system_ext
└── vbmeta_vendor → vendor, odm
```

---

## 9. Kích thước thực tế theo device

| Device | Super | System Group | Vendor Group |
|--------|-------|-------------|-------------|
| Cuttlefish (virtual) | 8 GiB | 6.375 GiB | 1.4 GiB |
| Raspberry Pi 4 | N/A (static) | 3 GB | 384 MB |
| Generic ARM64 | — | 1 GB | — |

> **Pi4 không dùng dynamic partition** — mỗi partition là block device cố định (`/dev/block/mmcblk0p5`, `p6`...). Xem thêm `13_rpi4_ota_bootloader.md`.
