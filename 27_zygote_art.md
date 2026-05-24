# Zygote & Android Runtime (ART)

## 1. Tổng quan

Zygote là process cha của mọi Android app. ART (Android Runtime) là máy ảo thực thi Dalvik bytecode (.dex) với AOT + JIT compilation.

---

## 2. Zygote — Process Factory

```
init
 └─ zygote  (app_process /system/bin --zygote)
      │  Fork & specialize
      ├─ system_server   ← SystemServer.java
      ├─ com.android.phone
      ├─ com.android.systemui
      ├─ com.myapp.example
      └─ ...tất cả Android app đều là fork của zygote
```

### Tại sao fork từ Zygote?

```
fork() = copy-on-write clone của parent process
→ Zygote đã preload:
  • Android framework classes (android.*, java.*)
  • Common resources (themes, drawables)
  • ART runtime
→ App mới kế thừa tất cả qua COW mapping
→ Khởi động app chỉ mất ~50-100ms thay vì ~2-3 giây
```

---

## 3. Luồng App Launch

```
startActivity("com.myapp")
      │
      ▼
ActivityManagerService (system_server)
      │ Kiểm tra process đã tồn tại chưa
      │ Process chưa có → cần tạo mới
      ▼
Process.start("com.myapp", ...)
      │ Ghi lệnh vào socket /dev/socket/zygote
      ▼
Zygote (lắng nghe ZygoteServer socket)
      │ Nhận lệnh spawn
      ▼
Zygote.forkAndSpecialize()
      │ fork()               ← Linux system call
      │ Child process:
      │   dropCapabilities() ← Bỏ quyền root
      │   setuid/setgid()    ← Set UID/GID của app
      │   closeOpenFiles()   ← Đóng Zygote's sockets
      ▼
RuntimeInit.applicationInit()
      │ Tìm main class: ActivityThread
      ▼
ActivityThread.main()
      │ Looper.prepareMainLooper()
      │ attach() → AMS qua Binder
      ▼
App running!
```

---

## 4. Zygote Preloading

```java
// frameworks/base/core/java/com/android/internal/os/ZygoteInit.java

static void preload(TimingsTraceLog bootTimingsTraceLog) {
    // 1. Preload classes (~7000 classes)
    preloadClasses();    // từ /system/etc/preloaded-classes

    // 2. Preload resources (drawables, layouts)
    preloadResources();

    // 3. Preload OpenGL ES libs
    preloadOpenGL();

    // 4. Preload WebView
    maybePreloadWebView();

    // 5. Preload shared libraries (libc, libart, ...)
    preloadSharedLibraries();
}
```

---

## 5. ART — Android Runtime

### 5.1 DEX → Native Code pipeline

```
Source code (.java / .kt)
      │ javac / kotlinc
      ▼
Bytecode (.class)
      │ d8 (dexer)
      ▼
DEX bytecode (.dex)
      │  Lúc install: dexopt
      ▼
┌─────────────────────────────────┐
│         ART Compilation         │
│                                 │
│  AOT (Ahead-of-Time):           │
│    dex2oat → .oat (native)     │ ← Compile lúc install/idle
│                                 │
│  JIT (Just-in-Time):            │
│    Interpreter → profile data  │ ← Chạy lần đầu
│    JIT compiler → hot methods  │ ← Compile method "nóng"
│                                 │
│  Profile-guided AOT:            │
│    .prof → dex2oat (targeted)  │ ← Android 7+ (best of both)
└─────────────────────────────────┘
      │
      ▼
Native machine code (ARM64)
```

### 5.2 Compilation filters

```bash
# Xem compilation state của app
adb shell cmd package dump-profiles com.myapp
adb shell oatdump --oat-file=/data/app/com.myapp-.../oat/arm64/base.odex | head -20

# Compilation filters (từ thấp đến cao):
# verify         → Chỉ verify DEX, không compile
# quicken        → Optimize interpreter hints
# space-profile  → AOT chỉ method trong profile
# speed-profile  → AOT + JIT cho profile methods (default production)
# speed          → AOT toàn bộ
# everything     → AOT tất cả kể cả unused
```

---

## 6. ART Garbage Collector

ART dùng **Concurrent Copying (CC) GC** từ Android 8+:

```
Heap layout:
┌─────────────────────────────────────────────┐
│  Region Space (young + old gen)             │
│  ┌──────┬──────┬──────┬──────┬───────────┐  │
│  │ Region│ Region│ Region│ Region│ ...    │  │
│  │ 1MB  │  1MB  │  1MB  │  1MB  │        │  │
│  └──────┴──────┴──────┴──────┴───────────┘  │
│                                             │
│  Non-moving space (large objects, ART objs) │
└─────────────────────────────────────────────┘

GC Types:
- Minor GC: thu dọn young gen (frequent, fast)
- Major GC: thu dọn toàn heap (concurrent, rare)
- Concurrent Compacting: di chuyển objects, giảm fragmentation
```

```bash
# Xem GC logs
adb logcat -s art
# I/art: Background partial concurrent mark sweep GC freed 1234(56KB)
#        AllocSpace objects, 0(0B) LOS objects, 45% free,
#        123MB/223MB, paused 1.2ms total 45.6ms

# Force GC
adb shell am force-stop com.myapp  # Kill + restart
# Trong code: Runtime.getRuntime().gc()
```

---

## 7. Class Loading & Dex

```java
// Cách Android load class tại runtime
ClassLoader cl = new PathClassLoader(
    "/data/app/com.myapp.../base.apk",
    ClassLoader.getSystemClassLoader()
);
Class<?> cls = cl.loadClass("com.myapp.MainActivity");

// Dynamic DEX loading
DexClassLoader dcl = new DexClassLoader(
    "/sdcard/plugin.dex",
    context.getCacheDir().getAbsolutePath(),
    null,
    getClassLoader()
);
```

---

## 8. Dex2oat — AOT Compiler

```bash
# Xem OAT files được generate
ls /data/app/com.myapp-*/oat/arm64/
# base.odex   ← OAT file (native code)
# base.vdex   ← DEX data (verified)
# base.art    ← ART image (preloaded objects)

# Manual compile
dex2oat --dex-file=classes.dex \
        --oat-file=classes.oat \
        --compiler-filter=speed-profile \
        --profile-file=classes.prof

# Xem OAT contents
oatdump --oat-file=base.odex | grep "method_offset"
```

---

## 9. JIT Compilation

```
Lần đầu method được gọi → ART interpreter thực thi
Sau N lần gọi → JIT compiler compile thành native
JIT profile được lưu: /data/misc/profiles/ref/com.myapp/primary.prof

Background dexopt (khi device idle + charging):
  artd đọc profiles → dex2oat AOT compile hot methods
  → Lần sau app dùng native code trực tiếp
```

---

## 10. ART Internal — Debug

```bash
# Xem ART internal state
adb shell kill -SIGUSR1 <pid>   # Dump ART state to log

# Heap dump
adb shell am dumpheap com.myapp /data/local/tmp/heap.hprof
adb pull /data/local/tmp/heap.hprof
# Phân tích bằng Android Studio Memory Profiler

# Method tracing
adb shell am profile start com.myapp /data/local/tmp/trace.bin
# ... trigger actions ...
adb shell am profile stop com.myapp
adb pull /data/local/tmp/trace.bin
# Phân tích bằng Android Studio CPU Profiler

# ART stats
adb shell dumpsys meminfo com.myapp
# Total PSS: 45678 kB
# Java Heap: 12345 kB
# Native Heap: 6789 kB
```

---

## 11. Zygote64 vs Zygote32

Android chạy 2 Zygote để hỗ trợ cả 32-bit và 64-bit app:

```
init
├── zygote64   (/dev/socket/zygote)     ← 64-bit apps (default)
└── zygote32   (/dev/socket/zygote_secondary)  ← 32-bit apps
```

```bash
# Xem trên Pi4
ps -A | grep zygote
# root  ...  zygote64
# root  ...  zygote32
```

---

## 12. File quan trọng trong source

| File | Mô tả |
|------|-------|
| `frameworks/base/core/java/com/android/internal/os/ZygoteInit.java` | Zygote main, preload |
| `frameworks/base/core/java/com/android/internal/os/ZygoteConnection.java` | Handle spawn requests |
| `frameworks/base/core/java/android/app/ActivityThread.java` | App main loop |
| `art/runtime/runtime.cc` | ART runtime init |
| `art/runtime/gc/heap.cc` | ART heap management |
| `art/compiler/dex2oat.cc` | AOT compiler |
| `art/runtime/jit/jit.cc` | JIT compiler |
| `dalvik/vm/` | Legacy Dalvik (reference) |
