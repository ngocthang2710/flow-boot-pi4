# Android Init System & Property System

## 1. Android Init — PID 1

`init` là process đầu tiên userspace khởi động (PID 1), đọc các file `.rc` và quản lý toàn bộ vòng đời service.

```
Kernel → /init (ramdisk) → /system/bin/init → đọc *.rc → spawn services
```

**Source:** `system/core/init/`

---

## 2. Init RC Language

### 2.1 Cấu trúc file .rc

```rc
# 4 loại section chính:

# 1. on <trigger> — Chạy actions khi trigger xảy ra
on early-init
    mount tmpfs tmpfs /dev mode=0755
    symlink /proc/self/fd /dev/fd

on init
    export PATH /system/bin:/vendor/bin
    mkdir /data 0771 system system

on boot
    write /proc/sys/vm/swappiness 100
    setprop ro.boot.revision 1

# 2. service — Định nghĩa daemon
service logd /system/bin/logd
    class core
    user logd
    group logd system
    socket logd stream 0666 logd logd

# 3. import — Include file rc khác
import /vendor/etc/init/hw/init.${ro.hardware}.rc

# 4. on property:<prop>=<value> — Trigger theo property
on property:sys.boot_completed=1
    start my_late_service
```

### 2.2 Triggers

| Trigger | Khi nào chạy |
|---------|-------------|
| `early-init` | Ngay sau kernel mount devtmpfs |
| `init` | Sau early-init |
| `early-fs` | Trước khi mount filesystem |
| `fs` | Mount filesystems |
| `late-fs` | Sau mount, trước post-fs |
| `post-fs-data` | Sau mount /data |
| `boot` | Sau post-fs-data |
| `property:<name>=<value>` | Khi property thay đổi |

### 2.3 Service directives

```rc
service my_daemon /vendor/bin/my_daemon arg1 arg2
    class main              # class để group start/stop
    user system             # chạy dưới user "system"
    group system inet       # groups
    capabilities NET_ADMIN  # Linux capabilities
    socket my_sock stream 0660 root system  # Unix socket
    file /dev/kmsg w        # File permission
    seclabel u:r:my_daemon:s0  # SELinux label
    onrestart restart other_service  # Khi restart, restart service khác
    disabled                # Không tự start, phải start thủ công
    oneshot                 # Không restart khi die
    critical                # Nếu crash liên tục → reboot to recovery
    writepid /dev/cpuset/tasks  # Ghi PID vào file
```

---

## 3. Init Boot Sequence

```
/init (first stage, từ ramdisk)
│  Mount: /sys, /proc, /dev (devtmpfs)
│  Mount /system, /vendor, /odm (early mount)
│  Verify AVB
│  Exec /system/bin/init second_stage
│
/system/bin/init (second stage)
│  LoadBootScripts() → đọc /init.rc, /vendor/etc/init/
│  ActionManager: queue các actions
│  ServiceList: đăng ký services
│
│  Loop chính:
│  ┌─────────────────────────────────────┐
│  │  am.ExecuteOneCommand()             │ ← Execute action
│  │  RestartProcesses()                 │ ← Restart crashed services
│  │  epoll_wait()                       │ ← Wait events
│  └─────────────────────────────────────┘
│
│  Triggers theo thứ tự:
│  early-init → init → early-fs → fs → post-fs → 
│  post-fs-data → boot → property:sys.boot_completed=1
```

---

## 4. Property System

Android Property System là key-value store toàn hệ thống, được share qua shared memory.

### 4.1 Kiến trúc

```
property_service (chạy trong init process)
      │  Unix socket /dev/socket/property_service
      │  Chỉ init mới có quyền SET property từ socket
      ▼
Shared memory region (/dev/__properties__)
      │  Tất cả process đều mmap và READ được
      │  Không cần IPC để đọc → siêu nhanh
      ▼
App/Service đọc: SystemProperties.get("ro.build.version.sdk")
```

### 4.2 Property types

| Prefix | Đặc điểm | Ví dụ |
|--------|---------|-------|
| `ro.*` | Read-only, set 1 lần lúc boot | `ro.build.version.sdk=36` |
| `persist.*` | Persistent qua reboot (lưu vào /data/property/) | `persist.sys.locale=en_US` |
| `sys.*` | Runtime, thay đổi được | `sys.boot_completed=1` |
| `ctl.*` | Control service start/stop | `ctl.start=logd` |
| `debug.*` | Debug properties | `debug.hwui.overdraw=show` |
| `vendor.*` | Vendor-specific | `vendor.hwc.drm.ctm=DRM_OR_IGNORE` |

### 4.3 Đọc/ghi property

```bash
# Shell
getprop ro.product.model                    # Đọc
setprop persist.debug.enable_debug_log 1   # Ghi (cần permission)
getprop | grep "ro.build"                   # Filter

# Watch property change
adb shell watchprop sys.boot_completed

# Dump tất cả
adb shell getprop > all_props.txt
```

```java
// Java
String model = SystemProperties.get("ro.product.model", "unknown");
int sdk = SystemProperties.getInt("ro.build.version.sdk", 0);
SystemProperties.set("debug.my_app.enable", "1");  // cần permission
```

```cpp
// C++
#include <cutils/properties.h>
char value[PROP_VALUE_MAX];
property_get("ro.product.model", value, "unknown");
property_set("debug.myservice.enable", "1");
```

### 4.4 Khởi động service qua property

```rc
# Trong .rc file:
on property:sys.boot_completed=1
    start my_late_service

# Start/stop service qua ctl.*
# setprop ctl.start logd   → init nhận lệnh start logd
# setprop ctl.stop logd    → init nhận lệnh stop logd
```

---

## 5. Property Security (SELinux)

```
# property_contexts — gán SELinux type cho property
ro.build.      u:object_r:build_prop:s0
persist.sys.   u:object_r:system_prop:s0
vendor.        u:object_r:vendor_default_prop:s0

# Policy rule: chỉ init mới được set ro.* 
allow init property_type:property_service set;

# App không thể set ro.*:
# adb shell setprop ro.test 1 → Failed (SELinux denied)
```

---

## 6. Init Control Flow — Restart Policy

```rc
service crashy_daemon /system/bin/crashy
    class main
    # Mặc định: restart khi crash
    # Sau 4 lần crash trong 4 phút → disable

service critical_daemon /system/bin/critical
    critical   # Sau 4 crash → reboot to recovery

service oneshot_job /system/bin/one_time_script
    oneshot    # Không restart khi exit
    disabled   # Phải trigger thủ công

# Trigger oneshot job
on property:my.trigger=run
    start oneshot_job
```

---

## 7. Ueventd — Device Node Creation

```rc
# ueventd.rc — tạo /dev nodes
/dev/binder        0666   root   root
/dev/gpiochip*     0660   root   gpio
/dev/i2c-*         0660   root   i2c
/dev/spidev*       0660   root   spi
/dev/video*        0660   root   video

# Kernel gửi uevent → ueventd tạo /dev node với đúng permissions
```

---

## 8. Watchdog trong Init

```cpp
// system/core/init/init.cpp

// Init có watchdog: nếu action/service block quá lâu
// init sẽ log + reboot

// Timeout thường gặp:
// - fs trigger: 60 giây để mount filesystem
// - Nếu vold chết giữa chừng → init timeout → panic
```

---

## 9. Debug Init

```bash
# Xem init log
adb logcat -s init

# Xem service state
adb shell getprop init.svc.logd    # running/stopped/restarting
adb shell getprop init.svc.vold

# Xem tất cả init services
adb shell getprop | grep init.svc

# Trigger action (test)
adb shell setprop ctl.restart surfaceflinger

# Xem property change log
adb logcat -s "init" | grep "property"

# Đọc init.rc trực tiếp
adb shell cat /init.rc
adb shell ls /vendor/etc/init/
```

---

## 10. File quan trọng trong source

| File | Mô tả |
|------|-------|
| `system/core/init/init.cpp` | Init main loop |
| `system/core/init/service.cpp` | Service management |
| `system/core/init/action.cpp` | Action trigger/execute |
| `system/core/init/property_service.cpp` | Property server |
| `system/core/init/ueventd.cpp` | Device node creation |
| `system/core/rootdir/init.rc` | Main init script |
| `device/brcm/rpi4/ramdisk/init.rpi4.rc` | Pi4 init script |
| `bionic/libc/include/sys/system_properties.h` | Property C API |
