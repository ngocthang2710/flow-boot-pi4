# Simpleperf — CPU Profiling & Flame Graphs

## 1. Tổng quan

Simpleperf là CPU profiler của Android — based on Linux `perf` với Android-specific features: Java/Kotlin stack unwinding, symbolication.

```
App running
  │ PMU (Performance Monitoring Unit) interrupts
  ▼
Simpleperf (uses perf_event_open syscall)
  │ Sample stack traces at N Hz
  ▼
perf.data (raw samples)
  │ Post-process
  ▼
report / flamegraph.html
```

---

## 2. Basic Profiling

```bash
# Profile app for 10 seconds
adb push $ANDROID_NDK/prebuilt/android-arm64/simpleperf/simpleperf \
    /data/local/tmp/
adb shell chmod +x /data/local/tmp/simpleperf

# Record (from device)
adb shell /data/local/tmp/simpleperf record \
    -p $(adb shell pidof com.myapp) \
    -g \                              # Call graph (stack)
    -f 1000 \                         # 1000 samples/sec
    -o /data/local/tmp/perf.data \
    --duration 10

# Pull data
adb pull /data/local/tmp/perf.data /tmp/
```

---

## 3. Simpleperf from Host (Recommended)

```bash
# Android NDK includes simpleperf scripts
cd $ANDROID_NDK/simpleperf/

# app_profiler.py — simplified profiling
python app_profiler.py \
    -p com.myapp \
    -r "-g -f 1000 --duration 10" \
    -lib /path/to/app/libs \
    -o perf.data

# Hoặc profile specific activity
python app_profiler.py \
    -p com.myapp \
    -a .MainActivity \     # Launch and profile
    --duration 10
```

---

## 4. Generate Flame Graph

```bash
# Method 1: Simpleperf HTML report (recommended)
python report_html.py \
    -i perf.data \
    -o report.html \
    --add_source_code \
    --source_dirs /path/to/source/

# Open report.html in browser
# → Interactive flame graph, call tree, hot functions

# Method 2: FlameGraph (Brendan Gregg's tool)
python report_sample.py \
    -i perf.data \
    --show-art-frames \
    | flamegraph.pl > flame.svg

# Method 3: Perfetto integration
python pprof_proto_generator.py \
    -i perf.data \
    -o perf.pb \
    && cat perf.pb | gzip > perf.pb.gz
# Upload perf.pb.gz to ui.perfetto.dev
```

---

## 5. Flame Graph Reading

```
Flame Graph:
  
  ┌─────────────────────────────────────────┐
  │          main()                         │  Bottom = entry point
  ├─────────────────┬───────────────────────┤
  │  networkCall()  │  processData() 60%    │  Width = time spent
  ├─────────────────┤──────────┬────────────┤
  │                 │ parseJSON│  compress  │
  │                 ├──────────┤            │
  │                 │  Gson    │            │  Top = leaf (hot code)
  └─────────────────┴──────────┴────────────┘
  
  Interpretation:
  - Wide bar = hot function (lots of CPU time)
  - Tall stack = deep call chain
  - Flat top = most CPU in leaf function
```

---

## 6. Hardware Performance Counters

```bash
# List available PMU events on Pi4 (ARM Cortex-A72)
adb shell /data/local/tmp/simpleperf list

# Hardware events:
# cpu-cycles                    ← CPU cycles
# instructions                  ← Instructions executed
# cache-references              ← Cache refs
# cache-misses                  ← Cache misses
# branch-instructions           ← Branches
# branch-misses                 ← Branch mispredictions

# Profile cache misses (memory bottleneck analysis)
adb shell /data/local/tmp/simpleperf stat \
    -p $(pidof com.myapp) \
    -e cache-misses,cache-references,cpu-cycles \
    --duration 5

# Sample on cache misses (not time-based)
adb shell /data/local/tmp/simpleperf record \
    -p $(pidof com.myapp) \
    -e cache-misses:u \  # user-space cache misses
    -c 100000 \          # sample every 100k cache misses
    -g \
    --duration 10
```

---

## 7. Java/Kotlin Profiling

```bash
# Simpleperf unwraps ART JIT frames automatically
# Need --show-art-frames to include ART interpreter frames

python app_profiler.py \
    -p com.myapp \
    -r "-g -f 1000 --duration 10 --show-art-frames" \
    -o perf.data

# Report với Java frames
python report_html.py \
    -i perf.data \
    --show-art-frames \
    -o report.html

# Java frame identifiers in flame graph:
# com.myapp.MainActivity.processData  ← Java method
# art::interpreter::Execute            ← ART interpreter
# art_quick_invoke_stub                ← JNI glue
```

---

## 8. Tracing Specific Thread

```bash
# Profile specific thread
adb shell /data/local/tmp/simpleperf record \
    -t $(adb shell ps -T -p $(pidof com.myapp) | grep "RenderThread" | awk '{print $2}') \
    -g -f 1000 --duration 5 \
    -o /data/local/tmp/render_thread.data

# Profile by thread name
adb shell /data/local/tmp/simpleperf record \
    -p $(pidof com.myapp) \
    --trace-offcpu \  # Also trace blocked time
    -g -f 1000 --duration 5
```

---

## 9. Off-CPU Analysis

```bash
# --trace-offcpu: samples khi thread bị block (sleep, I/O wait, lock wait)
# Useful để tìm:
#   - Lock contention
#   - Slow I/O
#   - Sleep calls

adb shell /data/local/tmp/simpleperf record \
    -p $(pidof com.myapp) \
    --trace-offcpu \
    -g -f 1000 --duration 10

python report_html.py -i perf.data --trace-offcpu -o offcpu.html
# Shows both on-CPU and off-CPU time in flame graph
```

---

## 10. Android Studio Integration

```
Profiler trong Android Studio:
  Run → Profile → CPU Profiler
  
  Record methods:
    Callstack Sample:    Simpleperf backend (low overhead)
    Java/Kotlin Method:  JVMTI instrumentation (accurate but heavy)
    System Trace:        Perfetto/Systrace backend
  
  Flame Chart (Call Chart):
    X-axis = time
    Y-axis = call depth
    
  Top Down / Bottom Up trees available
```

---

## 11. Pi4 Profile Notes

```bash
# Pi4 có ARM Cortex-A72 PMU
# PMU events tương thích: cpu-cycles, instructions, cache-*

# Thường gặp trên Pi4 với SD card:
#   I/O bound: trace-offcpu sẽ show nhiều thời gian ở I/O wait
#   Thermal throttle: cpu-cycles thấp khi bị throttle

# Check throttle state trong khi profile
adb shell cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq

# Disable thermal throttle (test only)
adb shell "echo performance > \
    /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor"
```

---

## 12. File quan trọng trong source

| File | Mô tả |
|------|-------|
| `system/extras/simpleperf/` | Simpleperf source |
| `system/extras/simpleperf/cmd_record.cpp` | Record command |
| `system/extras/simpleperf/report_utils.cpp` | Symbolication |
| `system/extras/simpleperf/scripts/` | Python helper scripts |
| `kernel/common/tools/perf/` | Linux perf source |
| `kernel/common/include/uapi/linux/perf_event.h` | perf_event API |
