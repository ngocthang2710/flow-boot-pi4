# Soong Build System — Android.bp & Blueprint

## 1. Tổng quan

Soong thay thế Make system trong Android (từ Android 7). Android.bp là build definition language.

```
Android Build System Evolution:
  Android < 7:   Android.mk (GNU Make)
  Android 7-9:   Android.bp (Soong) + Android.mk (mixed)
  Android 10+:   Soong only (Android.mk → .bp migration complete)
  Android 16:    Pure Soong
```

---

## 2. Build System Stack

```
Source tree (Android.bp files)
  │
  ▼
Blueprint (Go-based meta-build)
  │  Parses Android.bp
  ▼
Soong (Go-based build system)
  │  Generates Ninja build rules
  ▼
Ninja (fast, parallel build executor)
  │  Runs compilers/linkers
  ▼
Build artifacts (out/)
```

---

## 3. Android.bp Syntax

```python
# Module types
cc_library {
    name: "libmylib",
    srcs: ["src/foo.cpp", "src/bar.cpp"],
    hdrs: ["include/foo.h"],
    shared_libs: ["liblog", "libcutils"],
    static_libs: ["libbase"],
    cflags: ["-Wall", "-Werror"],
    include_dirs: ["include"],
    export_include_dirs: ["include"],
}

cc_binary {
    name: "myservice",
    srcs: ["main.cpp"],
    shared_libs: ["libmylib", "libbinder"],
    init_rc: ["myservice.rc"],
    vintf_fragments: ["myservice.xml"],
}

cc_test {
    name: "libmylib_test",
    srcs: ["test/foo_test.cpp"],
    static_libs: ["libmylib", "libgtest"],
    test_suites: ["general-tests"],
}
```

---

## 4. Java Module Types

```python
java_library {
    name: "framework-myservice",
    srcs: ["**/*.java"],
    sdk_version: "current",
    libs: ["framework"],
    static_libs: ["protobuf-java-lite"],
}

android_app {
    name: "MyApp",
    srcs: ["src/**/*.java", "src/**/*.kt"],
    resource_dirs: ["res"],
    manifest: "AndroidManifest.xml",
    certificate: "platform",   // platform signing key
    sdk_version: "current",
    optimize: {
        enabled: true,
        proguard_flags_files: ["proguard.flags"],
    },
}

android_library {
    name: "MyLib",
    srcs: ["*.java"],
    resource_dirs: ["res"],
    sdk_version: "current",
}
```

---

## 5. Common Module Properties

```python
cc_library_shared {
    name: "libexample",
    
    // Source files
    srcs: ["foo.cpp"],
    arch: {
        arm: { srcs: ["foo_arm.S"] },
        arm64: { srcs: ["foo_arm64.S"] },
        x86: { srcs: ["foo_x86.cpp"] },
    },
    
    // Visibility
    visibility: ["//frameworks/native/libs:__subpackages__"],
    
    // APEX/VNDK membership
    apex_available: ["com.android.media"],
    vndk: {
        enabled: true,
    },
    
    // Target/Host
    host_supported: true,
    device_supported: true,
    
    // Min SDK
    min_sdk_version: "29",
    
    // Strip debug symbols
    strip: {
        all: true,
    },
}
```

---

## 6. Build Variants

```
Build types:
  eng:    Developer build, full debug, no optimizations
  userdebug: User build + debug features (adb root, etc.)
  user:   Production build, minimal debug

Product variants:
  PRODUCT_NAME=rpi4
  PRODUCT_DEVICE=rpi4
  PRODUCT_BRAND=Android
  TARGET_BUILD_VARIANT=userdebug
```

```bash
# Setup build environment
source build/envsetup.sh

# Choose lunch target
lunch rpi4-userdebug   # Pi4 userdebug build

# Build
m                      # Full build
m -j$(nproc)          # Parallel with all CPU cores
m framework            # Build framework only
m services             # Build system_server
m kernel               # Build kernel

# Build specific module
m libmedia
m MyApp

# Install built APK
m MyApp && adb install $OUT/system/app/MyApp/MyApp.apk
```

---

## 7. Out Directory Structure

```
out/
├── soong/                  ← Soong generated files
│   ├── build.ninja         ← Generated Ninja rules
│   └── .bootstrap/
├── target/
│   └── product/rpi4/
│       ├── system/         ← /system partition content
│       ├── vendor/         ← /vendor partition content
│       ├── system.img      ← system partition image
│       ├── vendor.img      ← vendor partition image
│       ├── boot.img        ← boot image (kernel+ramdisk)
│       └── userdata.img    ← userdata image
└── host/
    └── linux-x86/
        ├── bin/            ← Host tools (aapt, dex2oat, adb, etc.)
        └── lib64/
```

---

## 8. PRODUCT_PACKAGES & Device Config

```makefile
# device/brcm/rpi4/device.mk

PRODUCT_PACKAGES += \
    SoundRecorder \
    Camera2 \
    Launcher3 \
    SystemUI \
    Settings

# APEX packages
PRODUCT_PACKAGES += \
    com.android.media \
    com.android.conscrypt

# HAL implementations
PRODUCT_PACKAGES += \
    android.hardware.audio.service \
    android.hardware.graphics.composer@2.4-service

# Overlays (customize resources)
DEVICE_PACKAGE_OVERLAYS += device/brcm/rpi4/overlay
```

---

## 9. Soong module query

```bash
# List all modules
m nothing 2>&1 | grep "^.*:" | head -50

# Query module info
./build/soong/bin/soong_ui --dumpvar-mode PRODUCT_PACKAGES

# Print module graph
./build/soong/bin/soong_ui --dumpvar-mode SOONG_MODULES

# Check what a module builds
grep -r "name: \"libmedia\"" --include="Android.bp" .

# Verify Android.bp syntax
bpfmt -l Android.bp   # list files with format issues
bpfmt -w Android.bp   # fix format
```

---

## 10. Make → Soong Migration

```makefile
# Old Android.mk
LOCAL_PATH := $(call my-dir)
include $(CLEAR_VARS)
LOCAL_MODULE := libmylib
LOCAL_SRC_FILES := foo.cpp bar.cpp
LOCAL_SHARED_LIBRARIES := liblog libcutils
include $(BUILD_SHARED_LIBRARY)
```

```python
# New Android.bp equivalent
cc_library_shared {
    name: "libmylib",
    srcs: ["foo.cpp", "bar.cpp"],
    shared_libs: ["liblog", "libcutils"],
}
```

---

## 11. File quan trọng trong source

| File | Mô tả |
|------|-------|
| `build/soong/` | Soong build system source |
| `build/soong/cc/cc.go` | C/C++ module types |
| `build/soong/java/java.go` | Java module types |
| `build/soong/apex/apex.go` | APEX module type |
| `build/make/core/main.mk` | Top-level Makefile |
| `build/make/envsetup.sh` | Build environment setup |
| `Android.bp` (root) | Root build file |
| `device/brcm/rpi4/Android.bp` | Pi4 device modules |
