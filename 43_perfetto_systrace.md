# Perfetto & Systrace — System Tracing

## 1. Tổng quan

Perfetto là hệ thống tracing thế hệ mới của Android (từ Android 9), thay thế Systrace. Cung cấp system-wide profiling cho CPU, memory, GPU, I/O.

```
App / Framework / Kernel
  │ trace points (ATRACE_*, ftrace events)
  ▼
traced (Perfetto daemon)
  │ Ring buffer
  ▼
perfetto CLI / UI
  │
  ▼
Trace analysis (ui.perfetto.dev)
```

---

## 2. Perfetto Components

```
System:
  traced         ← Central service broker, manages tracing sessions
  traced_probes  ← Kernel data sources (ftrace, /proc, sysfs)
  heapprofd      ← Heap profiler (malloc sampling)
  
Config:
  TraceConfig protobuf → defines what to trace
  
Output:
  .perfetto-trace (protobuf binary) hoặc
  .pftrace (gzipped)
```

---

## 3. Capture Trace

```bash
# Quick trace (Android 10+)
adb shell perfetto \
    -c - --txt \
    -o /data/misc/perfetto-traces/trace.perfetto-trace \
<<EOF
buffers: { size_kb: 63488 fill_policy: RING_BUFFER }
data_sources: {
    config {
        name: "linux.ftrace"
        ftrace_config {
            ftrace_events: "sched/sched_switch"
            ftrace_events: "sched/sched_wakeup"
            ftrace_events: "power/cpu_frequency"
            ftrace_events: "power/suspend_resume"
            atrace_categories: "am"
            atrace_categories: "gfx"
            atrace_categories: "view"
            atrace_categories: "input"
        }
    }
}
data_sources: {
    config {
        name: "linux.process_stats"
        process_stats_config { proc_stats_poll_ms: 1000 }
    }
}
duration_ms: 5000
EOF

# Pull trace
adb pull /data/misc/perfetto-traces/trace.perfetto-trace
```

---

## 4. Atrace Categories

```bash
# List available categories
adb shell atrace --list_categories
# gfx         - Graphics
# input       - Input
# view        - View System
# webview     - WebView
# wm          - Window Manager
# am          - Activity Manager
# sm          - Sync Manager
# audio       - Audio
# video       - Video
# camera      - Camera
# hal         - Hardware Modules
# app         - Application
# res         - Resource Loading
# dalvik      - Dalvik VM
# rs          - RenderScript
# bionic      - Bionic C Library
# power       - Power Management
# pm          - Package Manager
# ss          - System Server
# database    - Database
# network     - Network
# adb         - ADB
# vibrator    - Vibrator
# aidl        - AIDL calls

# Quick atrace recording (older method)
adb shell atrace -t 5 -b 32768 gfx input view am -o /sdcard/trace.html
adb pull /sdcard/trace.html
```

---

## 5. Instrument App Code

```java
// Add trace points to app code
import android.os.Trace;

// Named trace section (visible in Perfetto)
Trace.beginSection("MyApp:processFrame");
try {
    processFrame();
} finally {
    Trace.endSection();
}

// Async trace (for cross-thread tracking)
Trace.beginAsyncSection("MyApp:networkRequest", cookie);
// ... in callback on different thread ...
Trace.endAsyncSection("MyApp:networkRequest", cookie);

// Counter trace (visible as value over time)
Trace.setCounter("MyApp:pendingRequests", pendingCount);
```

```cpp
// Native code tracing
#include <trace.h>

ATRACE_CALL();          // Trace this function
ATRACE_NAME("my_task"); // Named trace in this scope
ATRACE_BEGIN("step1");
// ...
ATRACE_END();
ATRACE_INT("my_value", count);
```

---

## 6. Perfetto UI Analysis

```
Upload trace lên: https://ui.perfetto.dev

Panels:
  CPU:
    ├── CPU frequency over time (power/cpu_frequency)
    ├── CPU idle states
    └── Per-CPU task scheduling

  Processes:
    ├── Thread slices (what each thread was doing)
    ├── Wakeup latency
    └── Async slices

  GPU (Pi4/vc4):
    ├── GPU frequency
    └── Rendering timeline

  Memory:
    ├── RSS per process
    ├── Heap profiler flamegraph
    └── LMK events

SQL queries in UI:
  SELECT ts, dur, name FROM slice
  WHERE name LIKE 'MyApp:%'
  ORDER BY dur DESC LIMIT 10;
```

---

## 7. Heap Profiler

```bash
# Profile heap allocation (Android 10+)
# Start recording heap allocations for a process
adb shell heapprofd --pid $(adb shell pidof com.myapp) \
    --output /data/misc/perfetto-traces/heap.perfetto-trace \
    --duration 10000

# Pull and analyze
adb pull /data/misc/perfetto-traces/heap.perfetto-trace
# Open in ui.perfetto.dev → Heap Profile → flamegraph
```

---

## 8. Perfetto Config Recipes

```bash
# Trace GPU rendering (Pi4 + vc4)
perfetto_config_gpu="
buffers: { size_kb: 32768 fill_policy: RING_BUFFER }
data_sources: {
    config {
        name: \"linux.ftrace\"
        ftrace_config {
            atrace_categories: \"gfx\"
            ftrace_events: \"drm/drm_vblank_event\"
            ftrace_events: \"gpu_scheduler/drm_sched_job\"
        }
    }
}
duration_ms: 3000
"

# Memory pressure trace
perfetto_config_mem="
data_sources: {
    config {
        name: \"linux.ftrace\"
        ftrace_config {
            ftrace_events: \"lowmemorykiller/lowmemory_kill\"
            ftrace_events: \"oom/oom_score_adj_update\"
        }
    }
}
data_sources: {
    config {
        name: \"linux.sys_stats\"
        sys_stats_config { meminfo_period_ms: 500 }
    }
}
duration_ms: 10000
"
```

---

## 9. Systrace (Legacy)

```bash
# Systrace wrapper (calls atrace + generates HTML)
# Requires Android SDK tools

cd $ANDROID_SDK/platform-tools/systrace

python systrace.py \
    --time=5 \
    -o trace.html \
    gfx input view am sched freq

# Hoặc dùng trực tiếp atrace
adb shell atrace -t 5 -b 16384 \
    gfx input view am \
    > /tmp/trace.txt

# Convert to HTML (requires catapult)
trace2html /tmp/trace.txt -o trace.html
```

---

## 10. Debug Performance Issues

```bash
# Frame rendering perf
adb shell dumpsys gfxinfo com.myapp
# Janky frames: 3 (0.24%)
# 50th percentile: 4ms
# 95th percentile: 16ms  ← near 16.67ms limit!
# 99th percentile: 32ms  ← jank!

# Identify slow frames with Perfetto:
# Look for slices > 16ms in com.myapp rendering thread
# → Check what caused the long frame

# CPU throttling check (Pi4)
adb shell cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq
# If lower than max → throttled → performance degraded
```

---

## 11. File quan trọng trong source

| File | Mô tả |
|------|-------|
| `external/perfetto/` | Perfetto source |
| `frameworks/native/cmds/atrace/atrace.cpp` | atrace tool |
| `frameworks/base/core/java/android/os/Trace.java` | Java trace API |
| `frameworks/base/core/jni/android_os_Trace.cpp` | JNI trace impl |
| `kernel/common/include/trace/` | Kernel trace points |
| `kernel/common/kernel/trace/` | ftrace framework |
