# WakeLock, Doze & Android Power Management

## 1. Tổng quan

Android aggressively giảm power consumption khi màn hình tắt qua nhiều tầng cơ chế.

```
Screen ON:  App chạy tự do
Screen OFF: 
  ├─ WakeLock held → CPU stays ON
  ├─ Light Doze → restrict background (sau vài phút idle)
  ├─ Deep Doze → severe restriction (sau ~1 giờ stationary)
  └─ App Standby → restrict rarely-used apps
```

---

## 2. WakeLock

### 2.1 Kernel WakeLock

```c
/* Kernel giữ CPU thức qua wakeup sources */
#include <linux/pm_wakeup.h>

/* Declare wakeup source */
struct wakeup_source *ws = wakeup_source_register(dev, "my_driver");

/* Giữ CPU thức */
__pm_stay_awake(ws);
/* ... handle interrupt, process data ... */
__pm_relax(ws);

/* Xem active wakeup sources */
// cat /sys/kernel/debug/wakeup_sources
// cat /proc/wakelocks (legacy)
```

### 2.2 Android WakeLock (PowerManager)

```java
// Acquire wakelock từ app
PowerManager pm = getSystemService(PowerManager.class);
PowerManager.WakeLock wl = pm.newWakeLock(
    PowerManager.PARTIAL_WAKE_LOCK,
    "MyApp::MySyncTask"
);
wl.acquire(10 * 60 * 1000L); // max 10 phút
try {
    doWork();
} finally {
    wl.release();
}
```

### 2.3 WakeLock Types

| Type | CPU | Screen | Keyboard |
|------|-----|--------|---------|
| `PARTIAL_WAKE_LOCK` | ON | OFF | OFF |
| `SCREEN_DIM_WAKE_LOCK` | ON | DIM | OFF |
| `SCREEN_BRIGHT_WAKE_LOCK` | ON | BRIGHT | OFF |
| `FULL_WAKE_LOCK` | ON | BRIGHT | BRIGHT |

---

## 3. Debug WakeLock

```bash
# Xem wakelocks đang active
adb shell cat /sys/kernel/debug/wakeup_sources
# name          active_count  event_count  wakeup_count  expire_count  active_since  total_time  max_time
# radio-interface  12345      12345        0             0             0             45678ms     1234ms
# PowerManagerService.WakeLocks  3  3  0  0  12345  67890ms  5678ms

# Xem PowerManager wakelocks
adb shell dumpsys power | grep "Wake Locks"
# Wake Locks: size=2
#   PARTIAL_WAKE_LOCK 'MyApp::MySyncTask' (uid=10123, pid=1234)
#   PARTIAL_WAKE_LOCK 'NetworkStats' (uid=1000, pid=456)

# Xem battery stats (wakelocks over time)
adb shell dumpsys batterystats | grep "Wakelock"

# Reset stats
adb shell dumpsys batterystats --reset
```

---

## 4. Doze Mode

### 4.1 Doze States

```
ACTIVE: Screen ON, device in use
  ↓ Screen OFF + stationary
IDLE_PENDING: Preparing for Doze
  ↓ ~30 minutes idle
SENSING: Check if device moved (accelerometer)
  ↓ No motion
LOCATING: GPS/WiFi fix (optional)
  ↓
IDLE: Deep Doze
  │  Maintenance window mỗi vài giờ → IDLE_MAINTENANCE
  └─ Repeat

Light Doze (screen off, device may be moving):
  → Restrict network, jobs, alarms
  → Maintenance windows mỗi vài phút
```

### 4.2 Doze Restrictions

```
Khi trong IDLE (Deep Doze):
  - Network access: BLOCKED
  - Wake locks: IGNORED (dù app acquire vẫn không giữ CPU)
  - JobScheduler jobs: DEFERRED
  - Alarms: DEFERRED (trừ setAndAllowWhileIdle)
  - Sync adapters: BLOCKED
  - GPS: BLOCKED
```

```java
// App test Doze compliance
// Force Doze mode:
adb shell cmd deviceidle force-idle
// Check state:
adb shell cmd deviceidle get-state
// Exit:
adb shell cmd deviceidle unforce

// AlarmManager: set alarm hoạt động kể cả khi Doze
alarmManager.setAndAllowWhileIdle(
    AlarmManager.ELAPSED_REALTIME_WAKEUP,
    triggerTime,
    pendingIntent
);
// setExactAndAllowWhileIdle: exact time + Doze-proof
```

---

## 5. App Standby

```
App Standby buckets (dựa trên tần suất dùng):

ACTIVE (bucket=10):  Đang dùng, không restrict
WORKING_SET (20):    Dùng thường xuyên, ít restrict
FREQUENT (30):       Dùng hàng tuần, restrict vừa
RARE (40):           Hiếm dùng, restrict nhiều
RESTRICTED (45):     Không dùng, restrict tối đa
NEVER (50):          Không bao giờ dùng
```

```bash
# Xem bucket của app
adb shell am get-standby-bucket com.myapp
# FREQUENT

# Set bucket thủ công (test)
adb shell am set-standby-bucket com.myapp rare

# Force app standby
adb shell am set-standby-bucket com.myapp restricted
```

---

## 6. Suspend Flow

```
Android suspend flow:
  PowerManager.goToSleep()
    → DisplayManagerService (screen off)
    → SuspendBlocker release
    → PowerManagerService notifies all
    → Wakelocks checked: nếu có wakelock → abort suspend
    → /sys/power/wakeup_count check
    → /sys/power/state = "mem"
    → Linux PM suspend_enter()
    → Freeze processes (thpread_freeze)
    → Syscore suspend
    → CPU enters WFI/suspend state
    
Resume:
    → Wakeup interrupt (RTC, GPIO, etc.)
    → Resume từ WFI
    → Syscore resume
    → Thaw processes
    → Android components notified
    → Display on
```

```bash
# Trigger suspend
echo mem > /sys/power/state

# Xem wakeup count (để sync với kernel)
cat /sys/power/wakeup_count
echo 123 > /sys/power/wakeup_count && echo mem > /sys/power/state

# Xem wakeup sources (what woke us up)
cat /sys/power/pm_wakeup_irq
cat /proc/interrupts | grep wakeup
```

---

## 7. Battery Stats

```bash
# Full battery stats
adb shell dumpsys batterystats > battery.txt

# Tóm tắt
adb shell dumpsys batterystats --charged | head -100

# Battery historian (web tool)
adb bugreport bugreport.zip
# Upload lên https://bathist.ef.lc/

# Realtime battery
adb shell cat /sys/class/power_supply/battery/capacity
adb shell cat /sys/class/power_supply/battery/status  # Charging/Discharging
adb shell cat /sys/class/power_supply/battery/current_now  # mA
```

---

## 8. Pi4-specific Power

```bash
# Pi4 không có battery, nhưng có power management
# CPU frequency scaling (DVFS) → xem 19_kernel_cpufreq_dvfs.md
# Thermal throttling → xem 18_kernel_thermal.md

# Pi4 power states
vcgencmd get_throttled
# 0x0 = no throttling
# 0x50000 = throttled (over-temp + undervoltage occurred)

# CPU idle states
cat /sys/devices/system/cpu/cpu0/cpuidle/state*/name
# WFI   ← Wait For Interrupt (C1-like)

# Xem power consumption estimate
vcgencmd measure_volts core
vcgencmd measure_clock arm
```

---

## 9. File quan trọng trong source

| File | Mô tả |
|------|-------|
| `frameworks/base/services/core/java/com/android/server/power/PowerManagerService.java` | Android power manager |
| `frameworks/base/services/core/java/com/android/server/DeviceIdleController.java` | Doze controller |
| `kernel/common/drivers/base/power/wakeup.c` | Kernel wakeup sources |
| `kernel/common/kernel/power/suspend.c` | Kernel suspend/resume |
| `kernel/common/drivers/power/supply/` | Battery class |
