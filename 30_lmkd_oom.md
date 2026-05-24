# LMKD — Android Low Memory Killer

## 1. Tổng quan

LMKD (Low Memory Killer Daemon) là component quản lý memory pressure trong Android — quyết định process nào bị kill khi RAM thiếu.

```
Android Memory Hierarchy:
┌─────────────────────────────────────────────────────┐
│ Foreground app          → adj=0   (never kill)      │
│ Visible app             → adj=100                   │
│ Service                 → adj=200                   │
│ Cached app (recent)     → adj=900                   │
│ Cached app (background) → adj=906                   │
│ Empty process           → adj=1000 (kill first)     │
└─────────────────────────────────────────────────────┘
```

**Source:** `system/memory/lmkd/lmkd.cpp`  
**Kernel hook:** `/proc/<pid>/oom_score_adj`

---

## 2. OOM Score Adjustment

```bash
# Xem OOM score của mọi process
for pid in $(ls /proc | grep '^[0-9]'); do
    adj=$(cat /proc/$pid/oom_score_adj 2>/dev/null)
    comm=$(cat /proc/$pid/comm 2>/dev/null)
    echo "$adj $pid $comm"
done | sort -n | head -30

# Foreground app = 0
cat /proc/$(pgrep com.myapp)/oom_score_adj
# 0

# Background app = 900+
cat /proc/$(pgrep com.old.app)/oom_score_adj
# 900

# System server = luôn -900 (không bao giờ kill)
cat /proc/$(pgrep system_server)/oom_score_adj
# -900
```

---

## 3. Process States & ADJ Values

```java
// frameworks/base/services/core/java/com/android/server/am/ProcessList.java

// ADJ values quan trọng:
NATIVE_ADJ                = -1000; // native process, không quản lý
SYSTEM_ADJ                = -900;  // system server
PERSISTENT_PROC_ADJ       = -800;  // persistent service (phone)
PERSISTENT_SERVICE_ADJ    = -700;  // service của persistent app

FOREGROUND_APP_ADJ        = 0;     // app đang foreground
VISIBLE_APP_ADJ           = 100;   // visible nhưng không focus
PERCEPTIBLE_APP_ADJ       = 200;   // audio/notification
BACKUP_APP_ADJ            = 300;   // backup đang chạy
SERVICE_ADJ               = 500;   // service chạy lâu

HEAVY_WEIGHT_APP_ADJ      = 700;
HOME_APP_ADJ              = 800;   // launcher
PREVIOUS_APP_ADJ          = 850;   // app dùng trước

CACHED_APP_MIN_ADJ        = 900;   // cached app
CACHED_APP_MAX_ADJ        = 999;   // cached app (oldest)
```

---

## 4. LMKD Operation Modes

### 4.1 Legacy mode (kernel-side LMK)

```bash
# Kernel driver tại /sys/module/lowmemorykiller/
# Android < 10, kernel có built-in LMK
echo "0,1024,100,4096,200,16384" > \
    /sys/module/lowmemorykiller/parameters/minfree
echo "0,100,200,300,900,906" > \
    /sys/module/lowmemorykiller/parameters/adj
```

### 4.2 PSI mode (Android 10+, Pi4 dùng cái này)

```bash
# PSI = Pressure Stall Information
# CONFIG_PSI=y (đã enabled trong android_rpi4_defconfig)

# LMKD đọc memory pressure từ:
cat /proc/pressure/memory
# some avg10=0.00 avg60=0.00 avg300=0.00 total=0
# full avg10=2.34 avg60=1.12 avg300=0.89 total=12345678

# Khi "full" > threshold → LMKD kill process
```

---

## 5. Memory Kill Levels

```
Pressure level → Kill target:

LOW (some > 70%/s):
  Kill empty/cached processes với adj >= 900

MEDIUM (some > 100%/s):
  Kill background services với adj >= 600

HIGH (full > 70%/s):
  Kill visible processes với adj >= 200

CRITICAL (full > 90%/s):
  Kill everything kể cả foreground → last resort
```

---

## 6. LMKD Config

```bash
# System properties cấu hình LMKD
# ro.lmk.use_psi=true     ← Dùng PSI (default Android 10+)
# ro.lmk.low=1001         ← Threshold cho LOW pressure (MB)
# ro.lmk.medium=700
# ro.lmk.critical=0
# ro.lmk.kill_heaviest_task=true ← Kill process dùng RAM nhiều nhất
# ro.lmk.kill_timeout_ms=100     ← Timeout giữa các lần kill

getprop ro.lmk.use_psi
getprop ro.lmk.kill_heaviest_task
```

---

## 7. Monitor Memory Kills

```bash
# Xem kill log
adb logcat -s lmkd
# I/lmkd: Kill 'com.example.app' (1234), adj 900,
#         to free 45678kB; reason: pressure level 2

# Xem trong kernel log
adb shell dmesg | grep "lowmemory\|lmkd\|oom"

# Xem memory stats
adb shell cat /proc/meminfo | head -20
# MemTotal:       3906172 kB
# MemFree:          45678 kB
# MemAvailable:    234567 kB
# Buffers:          12345 kB
# Cached:          234567 kB

# App memory breakdown
adb shell dumpsys meminfo com.myapp
# Total PSS:  45678 kB
# Java Heap:  12345 kB
# Native Heap: 6789 kB
# Code:        9012 kB
# Stack:        345 kB
# Graphics:   15678 kB
```

---

## 8. cgroups Memory Control

```bash
# Android dùng cgroups để limit memory per app group
cat /dev/memcg/apps/memory.limit_in_bytes  # Giới hạn tổng cho apps
cat /dev/memcg/apps/memory.usage_in_bytes  # Usage hiện tại

# Per-app cgroup (tạo khi spawn app)
ls /dev/memcg/apps/
# uid_1000/  uid_10123/  uid_10234/

cat /dev/memcg/apps/uid_10123/memory.limit_in_bytes
cat /dev/memcg/apps/uid_10123/memory.oom_control
```

---

## 9. Memory Pressure Timeline

```
Bình thường:        [app1][app2][app3][cached1][cached2][cached3][free]
                                                          ↑
                                              LMKD không làm gì

Memory pressure:    [app1][app2][app3][cached1][cached2][     free    ]
                                                          ↑
                                            Kill cached3 (adj=906)

Cao hơn:            [app1][app2][app3][        free                   ]
                                         ↑
                                 Kill cached1, cached2

Critical:           [app1][app2][     free                            ]
                                 ↑
                         Kill app3 (adj=200, visible)

→ LMK kill theo thứ tự từ cao đến thấp ADJ score
```

---

## 10. File quan trọng trong source

| File | Mô tả |
|------|-------|
| `system/memory/lmkd/lmkd.cpp` | LMKD main |
| `system/memory/lmkd/statslog.cpp` | Kill event logging |
| `frameworks/base/services/core/java/com/android/server/am/ProcessList.java` | ADJ values, kill logic |
| `frameworks/base/services/core/java/com/android/server/am/OomAdjuster.java` | OOM adj calculation |
| `kernel/common/mm/oom_kill.c` | Kernel OOM killer |
| `kernel/common/kernel/psi/` | PSI implementation |
