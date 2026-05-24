# vold & Android Storage — F2FS, Encryption, Volumes

## 1. Tổng quan

vold (Volume Daemon) quản lý mọi storage trong Android: mount, unmount, format, encryption, adopted storage.

```
vold ↔ StorageManagerService ↔ App
  │      (system_server)
  │ Netlink từ kernel (uevent)
  ├─ /data (internal)
  ├─ /mnt/expand/<id> (adopted external)
  └─ /mnt/media_rw/<id> (portable external)
```

---

## 2. vold Architecture

```
Kernel (block device events)
  │ uevent
  ▼
vold (native daemon, /system/bin/vold)
  ├── VolumeManager → quản lý volumes
  ├── NetlinkManager → nhận uevent từ kernel
  ├── KryptFS/FBE → encryption
  ├── Disk → nhận diện disk mới
  └── PublicVolume/PrivateVolume → mount logic

StorageManagerService (system_server)
  │  Binder → vold AIDL
  ▼
vold thực thi: mount, format, encrypt
```

---

## 3. Volume Types

| Type | Mount point | Ví dụ |
|------|------------|-------|
| `EmulatedVolume` | `/sdcard`, `/storage/emulated/0` | Internal storage via FUSE |
| `PublicVolume` | `/mnt/media_rw/<id>` | USB drive, SD card (portable) |
| `PrivateVolume` | `/mnt/expand/<id>` | SD card formatted as adopted storage |

---

## 4. Android File System Encryption

### 4.1 FBE (File-Based Encryption) — Android 7+

```
/data/
├── system/          ← Encrypted với Device Encrypted (DE) key
│   └── (boot up without user credentials)
├── user/0/          ← Encrypted với Credential Encrypted (CE) key
│   └── (requires unlock credentials)
└── media/           ← External storage FUSE layer
```

```bash
# Xem encryption state
adb shell getprop ro.crypto.type
# file  (FBE)

adb shell getprop ro.crypto.state
# encrypted

# Xem per-directory encryption policy
adb shell tune2fs -l /dev/block/by-name/userdata | grep encrypt
```

### 4.2 Encryption Keys

```
DE key (Device Encrypted):
  → Derived from TPM/TEE secret + hardware attestation
  → Unlock ở stage 2 init (trước user credentials)
  → /data/system, /data/misc, ... accessible

CE key (Credential Encrypted):
  → Derived from DE key + user PIN/password
  → Unlock sau user enter credentials
  → /data/user/0/ accessible (user data, apps)

Keystore lưu keys trong Trusty (TEE):
  Nếu wipe device → keys mất → data inaccessible instantly
```

---

## 5. F2FS (Flash-Friendly File System)

F2FS được optimize cho NAND flash (eMMC, UFS, SD card):

```
CONFIG_F2FS_FS=y
CONFIG_F2FS_FS_ENCRYPTION=y  ← FBE support
CONFIG_F2FS_FS_COMPRESSION=y ← LZ4/ZSTD compression
```

### F2FS vs ext4

| Đặc điểm | F2FS | ext4 |
|----------|------|------|
| Phù hợp | Flash (NAND) | HDD, eMMC |
| Write pattern | Log-structured | Overwrite |
| Wear leveling | Tự nhiên qua LFS | Không |
| Random write | Tốt hơn | Kém hơn |
| Mount time | Nhanh hơn | Chậm hơn |
| Compression | Có (LZ4/ZSTD) | Không |
| Android dùng | /data (Android 12+) | /data (cũ), /system |

---

## 6. FUSE — Scoped Storage

```
Android 10+: Scoped Storage qua FUSE
  App chỉ thấy file của mình (MediaStore)
  /sdcard → FUSE → /data/media/0

Stack:
  App read /sdcard/DCIM/photo.jpg
    │
    ▼
  FUSE kernel module
    │
    ▼
  vold's FuseService (userspace FUSE)
    │ Permission check, path translation
    ▼
  /data/media/0/DCIM/photo.jpg (real path)
```

```bash
# Xem FUSE mounts
mount | grep fuse
# /data/media on /mnt/pass_through/0/self/primary type fuse
# /data/media on /storage/emulated/0 type fuse

# Xem FUSE stats
adb shell dumpsys mount | grep fuse
```

---

## 7. Adopted Storage — SD Card as Internal

```bash
# Format SD card làm adopted storage
adb shell sm partition disk:179,0 private

# Xem volumes
adb shell sm list-volumes
# private mounted /mnt/expand/<uuid>
# emulated:<id> mounted /storage/emulated/0

# Move app sang SD card
adb shell pm move-package com.myapp private:disk:179,0
```

---

## 8. vold Debug

```bash
# Xem vold log
adb logcat -s vold

# Kết nối tới vold CLI
adb shell vdc volume list
# 0 693 VolumeList:
# 0 693 VolumeList: private type=2 disk= state=mounted uuid=... path=/mnt/expand/...

# Volume info
adb shell vdc volume info private

# Trigger rescan
adb shell vdc volume reset

# Benchmark storage
adb shell vdc volume benchmark private

# StorageManager API
adb shell dumpsys mount
```

---

## 9. Pi4 Storage Config

```
fstab.rpi4:
  /dev/block/mmcblk0p3   /data  ext4   nodev,noatime,nosuid  wait,check,formattable,quota

Pi4 dùng ext4 cho /data (không phải F2FS)
  → Vì SD card driver (mmcblk) được test tốt với ext4
  → F2FS tốt hơn nhưng cần config thêm

Không có adopted storage (không có slot SD card riêng)
```

---

## 10. File quan trọng trong source

| File | Mô tả |
|------|-------|
| `system/vold/VolumeManager.cpp` | Volume lifecycle |
| `system/vold/Disk.cpp` | Disk detection |
| `system/vold/PublicVolume.cpp` | Portable storage |
| `system/vold/PrivateVolume.cpp` | Adopted storage |
| `system/vold/Ext4Crypt.cpp` | FBE implementation |
| `kernel/common/fs/f2fs/` | F2FS kernel driver |
| `kernel/common/fs/fuse/` | FUSE kernel driver |
