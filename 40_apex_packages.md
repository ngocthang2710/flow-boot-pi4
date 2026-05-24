# APEX — Updatable Mainline Modules

## 1. Tổng quan

APEX (Android Pony EXpress) là định dạng package cho phép cập nhật các module hệ thống (Mainline) qua Play Store mà không cần OTA.

```
Trước APEX (Android 9):
  Cập nhật media codec → OTA toàn bộ
  
Từ Android 10+ (APEX):
  Cập nhật com.android.media → Play Store push → apply ngay
  Không cần reboot toàn bộ (chỉ restart services liên quan)
```

---

## 2. APEX File Format

```
com.android.media.apex
├── apex_manifest.json   ← Metadata (name, version)
├── apex_manifest.pb     ← Protobuf version
├── AndroidManifest.xml  ← (optional)
├── apex_payload.img     ← ext4 image với dm-verity
│   ├── lib/             ← Shared libraries (.so)
│   ├── lib64/
│   ├── javalib/         ← JAR files
│   ├── bin/             ← Executables
│   └── etc/             ← Config files
└── apex_pubkey          ← Public key cho verification
```

---

## 3. APEX Mount Flow

```
Boot time:
  /system/apex/com.android.media.apex  ← factory image
  /data/apex/active/com.android.media.apex  ← updated version

apexd (APEX daemon):
  1. Scan /data/apex/active/ + /system/apex/
  2. Select highest version
  3. Verify signature (apex_pubkey)
  4. dm-verity setup trên apex_payload.img
  5. Loop mount lên /apex/com.android.media/
  
Result:
  /apex/com.android.media/lib64/libmedia.so  ← accessible
  /apex/com.android.media/javalib/framework.jar
```

---

## 4. Mainline Modules

| Module | Package | Nội dung |
|--------|---------|---------|
| Media | `com.android.media` | libmedia, codecs |
| Security | `com.android.conscrypt` | TLS/crypto (Conscrypt) |
| Runtime | `com.android.art` | ART runtime (Android 12+) |
| Network Stack | `com.android.networkstack` | ConnectivityService |
| DNS Resolver | `com.android.resolv` | netd DNS resolver |
| WiFi | `com.android.wifi` | WifiService |
| Bluetooth | `com.android.btservices` | Bluetooth stack |
| DocumentsUI | `com.android.documentsui` | File picker UI |
| Permission Controller | `com.android.permissioncontroller` | Permission UI |
| ExtServices | `com.android.ext.services` | Autofill, text classifier |

---

## 5. Android.bp — APEX Build

```python
# Android.bp
apex {
    name: "com.android.media",
    manifest: "apex_manifest.json",
    
    native_shared_libs: [
        "libmedia",
        "libmediadrm",
    ],
    
    java_libs: [
        "framework-media",
    ],
    
    prebuilts: [
        "media_codec_config",
    ],
    
    key: "com.android.media.key",      // APK key alias
    certificate: ":com.android.media", // Signing cert
    
    min_sdk_version: "29",             // Min Android version
    updatable: true,                   // Mainline-updateable
}
```

---

## 6. APEX Key Management

```bash
# Generate APEX key pair
openssl genrsa -out com.android.media.pem 4096
openssl rsa -in com.android.media.pem -pubout \
    -out com.android.media.avbpubkey

# Build với custom keys
m PRODUCT_EXTRA_OTA_KEYS=path/to/keys apex

# Sign APEX manually
apexer --key com.android.media.pem \
       --pubkey com.android.media.avbpubkey \
       apex_dir/ com.android.media.apex
```

---

## 7. APEX Versioning

```json
// apex_manifest.json
{
    "name": "com.android.media",
    "version": 319999999,
    "requireNativeLibs": ["libz.so"]
}
```

```
Version scheme:
  YYYYMMDD format → 319999999 = version 3.19999999
  
Override in /data/apex/active/:
  Higher version than /system/apex/ → active version wins
  
Rollback:
  apexd tracks previous version
  Rollback via PackageManager
```

---

## 8. apexd — APEX Daemon

```cpp
// system/apexd/apexd.cpp

// Phase 1 (early boot, before /data):
//   Mount bootstrap APEXes from /system/apex/
//   Cần cho early services (ART, resolv)

// Phase 2 (after /data mount):
//   Scan /data/apex/active/
//   Select & activate updated versions
//   Replace bootstrap mounts

// Key operations:
//   activatePackage()  → dm-verity setup + loop mount
//   deactivatePackage() → unmount
//   stagePackages()    → copy to /data/apex/staging/
```

---

## 9. Debug APEX

```bash
# List active APEX packages
adb shell cmd apexservice list
# Active packages:
# com.android.media v319999999 [path: /apex/com.android.media]
# com.android.conscrypt v319999999 [...]

# Verify APEX integrity
adb shell cmd apexservice verify com.android.media

# Check mounted APEX
mount | grep apex
# /dev/loop0 on /apex/com.android.media type ext4 (ro,...)

# APEX active versions
ls -la /data/apex/active/
ls -la /system/apex/

# Dump apexd state
adb shell dumpsys apexservice
```

---

## 10. APEX trên Pi4

```
Pi4 có APEX nhưng:
  - Không có Play Store → không tự update
  - APEX từ /system/apex/ được mount
  - /data/apex/active/ trống (no updates)
  
Kiểm tra:
adb shell ls /system/apex/
# com.android.adbd.apex
# com.android.art.apex
# com.android.conscrypt.apex
# com.android.media.apex
# com.android.resolv.apex
# ...
```

---

## 11. File quan trọng trong source

| File | Mô tả |
|------|-------|
| `system/apexd/apexd.cpp` | APEX daemon |
| `system/apexd/apex_file.cpp` | APEX file parsing |
| `system/apexd/apexd_loop.cpp` | Loop device management |
| `build/soong/apex/apex.go` | Soong APEX module type |
| `system/core/fs_mgr/libapex/` | APEX library |
| `hardware/interfaces/apex/` | (APEX HAL interface) |
