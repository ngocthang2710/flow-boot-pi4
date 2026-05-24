# AIDL / HIDL / HAL — Android Treble Architecture

## 1. Tổng quan — Project Treble

Trước Android 8: Framework và HAL cùng process → Update OS phải kèm HAL driver mới.  
Android 8+ (Treble): Framework và HAL chạy process riêng, giao tiếp qua versioned interface.

```
Trước Treble:                    Sau Treble (Treble):
┌──────────────────┐             ┌──────────────┐  ┌──────────────┐
│ Android Framework│             │   Framework  │  │   Vendor HAL │
│ + HAL code       │             │  (system/)   │  │  (vendor/)   │
│ (1 process)      │             │              │  │              │
└──────────────────┘             └──────┬───────┘  └──────┬───────┘
OS update = rebuild all                 │  AIDL/HIDL       │
                                        └──────────────────┘
                                  OS update độc lập với vendor
```

---

## 2. HAL Types

| Loại | Mô tả | Giao tiếp | Thời điểm |
|------|-------|-----------|-----------|
| **Legacy HAL** | .so lib load trực tiếp | Function call | Trước Android 8 |
| **Binderized HAL** | Process riêng (hwbinder) | HIDL/AIDL | Android 8-10 |
| **AIDL HAL** | Process riêng (binder/hwbinder) | AIDL stable | Android 11+ |
| **Passthrough HAL** | .so load trong framework | HIDL local | Thiết bị upgrade |

---

## 3. HIDL (HAL Interface Definition Language) — Android 8-10

```hidl
// hardware/interfaces/audio/6.0/IDevice.hal
package android.hardware.audio@6.0;

interface IDevice {
    openOutputStream(
        AudioIoHandle ioHandle,
        DeviceAddress device,
        AudioConfig config,
        AudioOutputFlags flags
    ) generates (Result retval, IStreamOut outStream, AudioConfig suggestedConfig);

    openInputStream(...) generates (...);
};
```

**Đặc điểm HIDL:**
- Versioned: `@6.0`, `@7.0` — không backward compat tự động
- Transport: hwbinder (hardware binder, riêng với app binder)
- Generated code: C++ server/client stubs từ hidl-gen
- Passthrough mode: load .so trực tiếp (không qua IPC)

```bash
# Xem HIDL services đang chạy
adb shell lshal
# Interface                    Transport  Server  PID
# android.hardware.audio@7.0  hwbinder   vendor  1234
# android.hardware.camera@3.6 hwbinder   vendor  1235
```

---

## 4. AIDL HAL (Android 11+) — Stable AIDL

Từ Android 11, Android chuyển sang AIDL cho HAL mới:

```aidl
// hardware/interfaces/audio/aidl/android/hardware/audio/core/IModule.aidl
package android.hardware.audio.core;

@VintfStability
interface IModule {
    IStreamOut openOutputStream(
        in OpenOutputStreamArguments args,
        out OpenOutputStreamReturn ret
    );

    IStreamIn openInputStream(in OpenInputStreamArguments args);

    void setAudioPatch(in AudioPatch patch);
}
```

**Tại sao AIDL tốt hơn HIDL:**
- `@VintfStability` → stable interface giữa các Android version
- Dùng cùng AIDL toolchain với app/framework
- Backward compat tự động (optional methods, parcelables)
- Transport: binder thay vì hwbinder riêng

---

## 5. Ví dụ Pi4 — Audio HAL

```
Pi4 Audio Stack:

App → AudioTrack → AudioFlinger (system_server)
         │  AIDL HAL
         ▼
com.android.hardware.audio.rpi (vendor process)
         │  ALSA (Advanced Linux Sound Architecture)
         ▼
bcm2711 audio hardware (HDMI / analog jack)
```

```java
// device/brcm/rpi4/device.mk
PRODUCT_PACKAGES += \
    com.android.hardware.audio.rpi    ← APEX-packaged AIDL HAL
```

---

## 6. Viết AIDL HAL mới

### Bước 1: Định nghĩa interface

```aidl
// hardware/interfaces/myhw/aidl/android/hardware/myhw/IMyHardware.aidl
package android.hardware.myhw;

@VintfStability
interface IMyHardware {
    int readSensor();
    void setSetting(in String key, in String value);
    ParcelFileDescriptor openStream();
}
```

### Bước 2: Implement HAL service

```cpp
// vendor/myvendor/myhw/MyHardware.cpp
#include <android/hardware/myhw/BnMyHardware.h>

class MyHardware : public BnMyHardware {
public:
    ndk::ScopedAStatus readSensor(int32_t* out) override {
        // Đọc từ hardware thực tế
        *out = read_hw_register(SENSOR_REG);
        return ndk::ScopedAStatus::ok();
    }

    ndk::ScopedAStatus setSetting(
            const std::string& key,
            const std::string& value) override {
        hw_set_config(key.c_str(), value.c_str());
        return ndk::ScopedAStatus::ok();
    }
};

int main() {
    ABinderProcess_setThreadPoolMaxThreadCount(4);
    auto hw = ndk::SharedRefBase::make<MyHardware>();
    const std::string name = IMyHardware::descriptor + std::string("/default");
    binder_status_t status = AServiceManager_addService(
        hw->asBinder().get(), name.c_str());
    ABinderProcess_joinThreadPool();
}
```

### Bước 3: Client (framework)

```cpp
// Trong framework service
#include <android/hardware/myhw/IMyHardware.h>

std::shared_ptr<IMyHardware> hw = IMyHardware::fromBinder(
    ndk::SpAIBinder(AServiceManager_waitForService(
        "android.hardware.myhw.IMyHardware/default")));

int32_t value;
hw->readSensor(&value);
```

---

## 7. Vendor Interface (VNDK)

```
VNDK (Vendor NDK): tập hợp shared libs framework cho phép vendor dùng

/system/lib64/vndk/          ← Libs chỉ vendor được dùng
/system/lib64/vndk-sp/       ← Same-process HAL libs

Vendor partition:
/vendor/lib64/               ← HAL libs, không được dùng libs ngoài VNDK
```

```bash
# Kiểm tra VNDK compliance
adb shell ldd /vendor/bin/hw/android.hardware.audio@7.0-service | grep "not found"
# → Nếu có → HAL dùng lib không thuộc VNDK → lỗi Treble
```

---

## 8. Hardware Manifest & Compatibility Matrix

```xml
<!-- device/brcm/rpi4/manifest.xml — Device HAL manifest -->
<manifest version="2.0" type="device">
    <hal format="aidl">
        <name>android.hardware.audio.core</name>
        <version>1</version>
        <interface>
            <name>IModule</name>
            <instance>default</instance>
        </interface>
    </hal>
    <hal format="aidl">
        <name>android.hardware.camera.provider</name>
        <version>1</version>
        <interface>
            <name>ICameraProvider</name>
            <instance>internal/0</instance>
        </interface>
    </hal>
</manifest>
```

```xml
<!-- framework_compatibility_matrix.xml — Framework yêu cầu gì -->
<compatibility-matrix version="1.0" type="framework">
    <hal format="aidl" optional="false">
        <name>android.hardware.audio.core</name>
        <version>1</version>
        <interface>
            <name>IModule</name>
            <instance>default</instance>
        </interface>
    </hal>
</compatibility-matrix>
```

```bash
# Kiểm tra compatibility
vintf-check
# COMPATIBLE → OK
# INCOMPATIBLE → device thiếu HAL mà framework cần
```

---

## 9. Xem HAL đang chạy trên Pi4

```bash
# List tất cả HAL services
adb shell lshal --neat

# List theo format
adb shell lshal -it
# Interface                          Transport  Server  PID   Clients
# android.hardware.audio.core/default aidl      1234   ... (audioserver:...)
# android.hardware.camera.provider   aidl      1235   ...

# Ping HAL service
adb shell lshal --ping android.hardware.audio.core/default
# OK (3.1 ms)

# Dump HAL
adb shell dumpsys media.audio_flinger
```

---

## 10. APEX — Updatable HAL

Từ Android 10, HAL có thể được đóng gói trong APEX để update độc lập:

```
com.android.hardware.audio.rpi.apex
├── apex_manifest.json
├── lib64/
│   └── android.hardware.audio.core-V1-ndk.so
└── bin/
    └── hw/android.hardware.audio.core-service.rpi
```

```bash
# Xem APEX trên Pi4
adb shell ls /apex/
# com.android.hardware.audio.rpi
# com.android.hardware.bluetooth.rpi
# ...

adb shell apexd-ctl status
```

---

## 11. File quan trọng trong source

| File | Mô tả |
|------|-------|
| `hardware/interfaces/` | Tất cả HAL interface definitions |
| `hardware/interfaces/audio/aidl/` | Audio AIDL HAL |
| `hardware/interfaces/boot/aidl/` | Boot Control AIDL |
| `hardware/interfaces/camera/provider/aidl/` | Camera AIDL HAL |
| `device/brcm/rpi4/manifest.xml` | Pi4 HAL manifest |
| `device/brcm/rpi4/framework_compatibility_matrix.xml` | Pi4 compat matrix |
| `system/libhidl/` | HIDL transport library |
| `system/tools/aidl/` | AIDL compiler |
