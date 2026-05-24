# DM-verity & Android Verified Boot (AVB)

## 1. Tổng quan

AVB đảm bảo mọi partition được boot là chưa bị tamper. DM-verity enforce integrity ở runtime.

```
Boot sequence với AVB:
Bootloader → verify vbmeta → verify partition hash → mount với dm-verity
                │
                ▼ Nếu hash sai → boot fail / orange state
```

---

## 2. AVB Chain of Trust

```
OEM private key (signing key)
      │ ký
      ▼
vbmeta partition (root of trust)
      │  chứa hash/signature của:
      ├── boot partition hash
      ├── vbmeta_system → system, product, system_ext
      └── vbmeta_vendor → vendor, odm

Lúc boot:
  Bootloader → đọc vbmeta → verify signature (OEM public key trong ROM)
             → verify boot, vendor, system...
             → Mount với dm-verity
```

---

## 3. DM-verity — Runtime Integrity

```
/dev/block/mmcblk0p5 (raw system partition)
        │
        ▼ Device Mapper
/dev/block/dm-0  (verified system)
        │  Mỗi lần đọc block → kernel verify hash
        │  Hash tree lưu ở cuối partition
        │  Root hash trong vbmeta (đã verify lúc boot)
        ▼
/system mount point
```

```
Hash tree structure:
                 Root Hash (trong vbmeta)
                      │
              ┌───────┴───────┐
           Level 1        Level 1
          /       \       /       \
        L2         L2   L2         L2
       / \        ...   ...       / \
    Data Data              Data Data Data
    Block Block             Block Block Block
    (4KB) (4KB)             (4KB) (4KB) (4KB)
```

---

## 4. Config

```
# Kernel
CONFIG_DM_VERITY=y              ← DM-verity target
CONFIG_DM_VERITY_VERIFY_ROOTHASH_SIG=y  ← Verify root hash signature
CONFIG_BLOCK=y                  ← Block layer

# BoardConfig.mk (Cuttlefish, Pi4 không có)
BOARD_AVB_ENABLE := true
BOARD_AVB_ALGORITHM := SHA256_RSA4096
BOARD_AVB_KEY_PATH := external/avb/test/data/testkey_rsa4096.pem
```

---

## 5. vbmeta Structure

```
vbmeta partition:
┌─────────────────────────────────────────┐
│ AvbVBMetaImageHeader                    │
│   magic: AVB0                           │
│   algorithm: SHA256_RSA4096             │
│   hash: [hash of descriptors]           │
│   signature: [RSA signature]            │
├─────────────────────────────────────────┤
│ Descriptors:                            │
│   AvbHashDescriptor (boot):             │
│     partition: "boot"                  │
│     hash: sha256(boot.img)             │
│   AvbChainPartitionDescriptor:          │
│     partition: "vbmeta_system"         │
│     → chứa public key để verify next  │
│   AvbPropertyDescriptor:               │
│     key: "com.android.build.id"        │
│     value: "SP1A.210812.016"           │
└─────────────────────────────────────────┘
```

---

## 6. AVB States

| State | Ý nghĩa | Hiển thị |
|-------|---------|---------|
| GREEN | Verified boot, locked | Không thông báo |
| YELLOW | Custom key, locked | Cảnh báo vàng |
| ORANGE | Unlocked bootloader | Cảnh báo cam |
| RED | Verification failed | Dừng boot |

---

## 7. avbtool — Làm việc với AVB

```bash
# Tạo vbmeta
avbtool make_vbmeta_image \
    --output vbmeta.img \
    --key testkey_rsa4096.pem \
    --algorithm SHA256_RSA4096 \
    --include_descriptors_from_image boot.img \
    --include_descriptors_from_image system.img

# Add hash footer vào partition image
avbtool add_hash_footer \
    --image boot.img \
    --partition_name boot \
    --partition_size $((64*1024*1024)) \
    --key testkey_rsa4096.pem \
    --algorithm SHA256_RSA4096

# Add hashtree footer (cho large partitions)
avbtool add_hashtree_footer \
    --image system.img \
    --partition_name system \
    --partition_size $((3*1024*1024*1024)) \
    --key testkey_rsa4096.pem

# Verify image
avbtool verify_image --image system.img --key testkey_rsa4096_pub.pem

# Info
avbtool info_image --image vbmeta.img
# Minimum libavb version: 1.0
# Header Block: 256 bytes
# Authentication Block: 576 bytes
# Aux Data Block: 4864 bytes
# Algorithm: SHA256_RSA4096
# Rollback Index: 0
# Descriptors:
#   Hash descriptor:
#     Image Name: boot
#     Image Size: 67108864 bytes
```

---

## 8. Pi4 — Không có AVB

Pi4 trong repo này **không enable AVB** (xem `13_rpi4_ota_bootloader.md`):

```makefile
# BoardConfig.mk (rpi4) — KHÔNG có:
# BOARD_AVB_ENABLE := true

# Vì:
# - TARGET_NO_BOOTLOADER := true
# - RPi firmware không verify AVB
# - Development board, không cần production security
```

Nhưng có thể học AVB trên Cuttlefish (Google's virtual device):
```bash
# Cuttlefish có full AVB
source build/envsetup.sh
lunch aosp_cf_x86_64_phone-userdebug
adb shell avbctl get-state
# Current state: LOCKED
```

---

## 9. DM-verity Runtime

```bash
# Xem dm-verity devices
adb shell ls /dev/block/mapper/
# system         ← dm-verity protected
# vendor
# product

# Xem dm device info
adb shell dmsetup table system
# 0 6291456 verity 1 /dev/block/.../system /dev/block/.../system
#   4096 4096 sha256 [root_hash] [salt]

# Verify error → kernel log
adb shell dmesg | grep "dm-verity"
# device-mapper: verity: ... corrupted data buffer
# → System bị tamper → reboot

# Disable verity (phải unlock bootloader)
adb disable-verity
adb reboot
```

---

## 10. Rollback Protection

```
vbmeta chứa Rollback Index (RBI):
  RBI = 5 → OS version 5

Bootloader lưu minimum RBI trong RPMB (Replay Protected Memory Block):
  Nếu flash OS với RBI < minimum → từ chối boot
  → Ngăn downgrade về version cũ có lỗi bảo mật
```

---

## 11. File quan trọng trong source

| File | Mô tả |
|------|-------|
| `external/avb/` | AVB tools và library |
| `external/avb/avbtool.py` | avbtool command |
| `external/avb/libavb/avb_vbmeta_image.c` | vbmeta parsing |
| `kernel/common/drivers/md/dm-verity-target.c` | DM-verity kernel driver |
| `kernel/common/drivers/md/dm-verity-fec.c` | Forward Error Correction |
| `system/core/fs_mgr/fs_mgr_avb.cpp` | AVB integration |
