# Tombstone & ANR — Crash Analysis

## 1. Tổng quan

```
Native crash → Tombstone (signal handler → debuggerd)
Java crash   → JVM exception → logcat + /data/tombstones/

ANR (App Not Responding):
  Input event not handled in 5s
  Broadcast not handled in 10s (fg) / 60s (bg)
  Service not started/bound in 20s
  → /data/anr/anr_*.txt (thread dump)
```

---

## 2. Tombstone — Native Crash

### 2.1 Crash Flow

```
SIGSEGV / SIGABRT / SIGFPE / SIGBUS
  │
  ▼
signal handler (bionic libc)
  │  ptrace attach
  ▼
debuggerd (/system/bin/debuggerd)
  │  Collect crash info
  ├── /data/tombstones/tombstone_XX
  └── logcat output (crash tag)

Tombstone content:
  ├── Process info (pid, name, timestamp)
  ├── Signal info (SIGSEGV, fault addr)
  ├── CPU registers (x0-x30, sp, lr, pc)
  ├── Backtrace (with symbols if available)
  ├── Memory map (all loaded .so files)
  └── Memory near crash address
```

### 2.2 Tombstone File

```
/data/tombstones/tombstone_00 (newest, wraps around 01-09)

Sample tombstone:
  *** *** *** *** *** *** *** *** *** *** *** *** *** *** *** ***
  Build fingerprint: 'Android/rpi4/rpi4:16/....'
  Revision: '0'
  ABI: 'arm64'
  Timestamp: 2026-05-24 10:15:30+0000
  
  pid: 1234, tid: 1234, name: com.myapp  >>> com.myapp <<<
  uid: 10456
  
  signal 11 (SIGSEGV), code 1 (SEGV_MAPERR), fault addr 0x0
  
  x0  0000000000000000  x1  0000000000000001
  x2  0000007fc1234567  ...
  sp  0000007fc1234500  lr  0000007b12345678  pc  0000007b12345670
  
  backtrace:
    #00 pc 00005670  /system/lib64/libmylib.so (my_function+32)
    #01 pc 00006789  /system/lib64/libmylib.so (caller+16)
    #02 pc 00123456  /data/app/com.myapp-xxx/lib/arm64/libapp.so
```

---

## 3. Symbolicate Tombstone

```bash
# Pull tombstone
adb pull /data/tombstones/tombstone_00 /tmp/

# Symbolicate với ndk-stack (Android NDK)
ndk-stack -sym $OUT/symbols/system/lib64/ \
    -dump /tmp/tombstone_00

# Hoặc dùng addr2line trực tiếp
addr2line -C -f -e \
    $OUT/symbols/system/lib64/libmylib.so \
    0x00005670
# my_function
# /src/system/lib64/libmylib.cpp:123

# Symbolicate với llvm-symbolizer
llvm-symbolizer \
    --obj=$OUT/symbols/system/lib64/libmylib.so \
    0x00005670
```

---

## 4. Java Crash

```
java.lang.NullPointerException: ...
  at com.myapp.MainActivity.onCreate(MainActivity.java:45)
  at android.app.Activity.performCreate(Activity.java:8000)
  ...
```

```bash
# Java crash tự log vào logcat
adb logcat -s AndroidRuntime
# FATAL EXCEPTION: main
# Process: com.myapp, PID: 1234
# java.lang.NullPointerException: Attempt to invoke virtual method...

# Tất cả crash (kể cả native)
adb logcat | grep -E "FATAL|DEBUG|backtrace"

# Tombstone cho Java crash (nếu có JNI)
adb pull /data/tombstones/
```

---

## 5. ANR — Application Not Responding

### 5.1 ANR Types

```
Input ANR:
  InputDispatcher gửi event
  App không consume trong 5 giây
  → System dialog: "App isn't responding"

Broadcast ANR:
  BroadcastReceiver.onReceive() chạy > 10s (foreground)
  BroadcastReceiver.onReceive() chạy > 60s (background)

Service ANR:
  Service.onCreate/onStartCommand không return trong 20s
  Service.onBind không return trong 20s

ContentProvider ANR:
  ContentProvider không respond trong 30s
```

### 5.2 ANR Trace File

```bash
# Pull ANR traces
adb pull /data/anr/
ls /data/anr/
# anr_2026-05-24-10-15-30-123.txt

# Format:
#   ----- pid 1234 at 2026-05-24 10:15:30 -----
#   Cmd line: com.myapp
#   Build fingerprint: ...
#   
#   DALVIK THREADS (11):
#   "main" prio=5 tid=1 Waiting
#     | group="main" sCount=1 dsCount=0 obj=0x...
#     | sysTid=1234 nice=0 cgrp=top-app
#     at android.database.sqlite.SQLiteDatabase.nativeExecute(SQLiteDatabase.java)
#     at com.myapp.MainActivity.onResume(MainActivity.java:200)
#     ...  ← main thread BLOCKED on SQLite!
```

### 5.3 Phân Tích ANR

```
Common ANR causes (từ stack trace):

1. Main thread doing DB query:
   at android.database.sqlite.SQLiteDatabase.nativeExecute
   → Fix: move DB to background thread

2. Main thread doing network:
   at java.net.Socket.connect
   → Fix: use Retrofit/OkHttp with callbacks

3. Main thread waiting for Binder:
   at android.os.BinderProxy.transactNative  ← waiting for IPC
   → Fix: async Binder or move to background

4. Deadlock:
   Thread A waiting for lock X (held by Thread B)
   Thread B waiting for lock Y (held by Thread A)
   → Fix: lock ordering, use tryLock with timeout

5. CPU overloaded:
   GC pause: "GCDaemon" doing full GC
   → Fix: reduce allocation, pool objects
```

---

## 6. bugreport — Comprehensive Debug Info

```bash
# Full bugreport (ZIP file)
adb bugreport /tmp/bugreport.zip

# Contents:
# ├── main_entry.txt        ← Main log
# ├── bugreport-*.txt       ← Full bugreport text
# ├── FS/data/tombstones/   ← All tombstones
# ├── FS/data/anr/          ← All ANR traces
# ├── proto/                ← Protobuf dumps (battery, etc.)
# └── version.txt

# Android Bug Report Analyzer:
# Upload ZIP to https://developer.android.com/studio/debug/bug-report
```

---

## 7. WTF (What a Terrible Failure)

```java
// Soft assertion — logs + sends to crash reporting
// Does NOT crash the process (unlike assert)
Log.wtf(TAG, "Unexpected state: " + state);

// Causes:
//   Binder service died unexpectedly
//   Illegal state in system services
//   → Look in logcat: grep "WTF"
```

---

## 8. Crash Reporting Integration

```java
// Custom uncaught exception handler
Thread.setDefaultUncaughtExceptionHandler((thread, throwable) -> {
    // Log to crash reporting service
    CrashReporter.log(throwable);
    // Chain to default handler (shows crash dialog)
    defaultHandler.uncaughtException(thread, throwable);
});

// Firebase Crashlytics (common)
FirebaseCrashlytics.getInstance().recordException(e);
```

---

## 9. Debug Tools

```bash
# Real-time logcat với crash filter
adb logcat -s "DEBUG" "AndroidRuntime" "ActivityManager"

# Xem pending ANR
adb shell dumpsys activity anr

# Trigger ANR manually (test)
adb shell am broadcast \
    -a android.intent.action.ANR_TEST \
    com.myapp

# Simulate OOM (test)
adb shell am send-trim-memory com.myapp COMPLETE

# Dump thread state (like ANR trace but manual)
adb shell kill -3 $(adb shell pidof com.myapp)
# Output in logcat: "Wrote stack traces to '/data/anr/anr_...'"
```

---

## 10. File quan trọng trong source

| File | Mô tả |
|------|-------|
| `debuggerd/crash_dump.cpp` | Native crash dump |
| `debuggerd/tombstoned/tombstoned.cpp` | Tombstone daemon |
| `frameworks/base/services/core/java/com/android/server/am/AnrController.java` | ANR logic |
| `frameworks/native/services/inputflinger/dispatcher/InputDispatcher.cpp` | Input ANR watchdog |
| `bionic/linker/linker_debug.cpp` | Signal handler (linker) |
| `dalvik/vm/Thread.cpp` (ART) | Thread dump for ANR |
