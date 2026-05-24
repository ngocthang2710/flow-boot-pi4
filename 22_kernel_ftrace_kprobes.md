# Kernel Ftrace & Kprobes — Raspberry Pi 4

## 1. Tổng quan

Ftrace và Kprobes là 2 công cụ tracing kernel mạnh mẽ nhất — cho phép trace function call, đo latency, debug driver mà **không cần rebuild kernel**.

**Config đã enabled:**
```
CONFIG_FTRACE=y
CONFIG_FUNCTION_TRACER=y         ← Trace function calls
CONFIG_FUNCTION_GRAPH_TRACER=y   ← Trace call graph + thời gian
CONFIG_KPROBES=y                 ← Dynamic probe tại bất kỳ địa chỉ
CONFIG_KRETPROBES=y              ← Probe return values
CONFIG_FPROBE=y                  ← Fast probe (dùng ftrace hook)
CONFIG_UPROBE_EVENTS=y           ← Probe trong userspace binary
CONFIG_PROBE_EVENTS=y            ← Kprobe/uprobe events
CONFIG_HIST_TRIGGERS=y           ← Histogram từ trace events
CONFIG_STACK_TRACER=y            ← Stack trace mỗi function call
CONFIG_IRQSOFF_TRACER=y          ← Đo thời gian IRQ bị disable
CONFIG_PREEMPTOFF_TRACER=y       ← Đo thời gian preemption bị disable
CONFIG_LATENCY_HIST=y            ← Latency histogram
```

**Interface:** `/sys/kernel/debug/tracing/`

---

## 2. Ftrace — Function Tracer

### 2.1 Basic Usage

```bash
TRACE=/sys/kernel/debug/tracing

# Xem available tracers
cat $TRACE/available_tracers
# blk function_graph wakeup_rt wakeup irqsoff function nop

# Bật function tracer
echo function > $TRACE/current_tracer

# Bật tracing
echo 1 > $TRACE/tracing_on

# Làm gì đó... rồi stop
echo 0 > $TRACE/tracing_on

# Xem kết quả
cat $TRACE/trace | head -50
# tracer: function
# TASK    PID    CPU  FLAGS  TIMESTAMP    FUNCTION
# <idle>    0    [001]  ...  1234.567890: cpu_idle <-arch_cpu_idle
```

---

### 2.2 Filter theo function cụ thể

```bash
# Chỉ trace bcm2835 SPI functions
echo "bcm2835_spi*" > $TRACE/set_ftrace_filter

# Trace GPIO functions
echo "gpiod_set_value" > $TRACE/set_ftrace_filter

# Thêm nhiều function
echo "i2c_bcm2835*" >> $TRACE/set_ftrace_filter

# Xem danh sách filter
cat $TRACE/set_ftrace_filter

# Bỏ filter (trace tất cả)
echo > $TRACE/set_ftrace_filter

# Blacklist một số function (loại trừ)
echo "do_page_fault" > $TRACE/set_ftrace_notrace
```

---

### 2.3 Function Graph Tracer — Xem call chain + thời gian

```bash
echo function_graph > $TRACE/current_tracer

# Giới hạn depth
echo 5 > $TRACE/max_graph_depth

# Filter chỉ 1 function và tất cả sub-functions
echo "bcm2835_spi_transfer_one" > $TRACE/set_graph_function

echo 1 > $TRACE/tracing_on
# trigger SPI transfer...
echo 0 > $TRACE/tracing_on

cat $TRACE/trace
# # DURATION         FUNCTION CALLS
# # |      |          |   |   |   |
#  134.301 us |    bcm2835_spi_transfer_one() {
#   12.456 us |      bcm2835_spi_dma_init()
#  118.230 us |      wait_for_completion_timeout()
#             |    }
```

---

### 2.4 IRQ Off Tracer — Đo latency interrupt bị block

```bash
# Tìm đoạn code giữ IRQ lâu nhất
echo irqsoff > $TRACE/current_tracer
echo 1 > $TRACE/tracing_on

# Chạy workload...
sleep 5
echo 0 > $TRACE/tracing_on

# Xem max latency
cat $TRACE/tracing_max_latency
# 234 us

cat $TRACE/trace
# Latency trace từng function đã giữ IRQ bao lâu
```

---

## 3. Kprobes — Dynamic Probe

Kprobes cho phép chèn probe handler vào **bất kỳ địa chỉ** trong kernel mà không cần rebuild.

### 3.1 Kprobe qua sysfs (kprobe events)

```bash
TRACE=/sys/kernel/debug/tracing

# Tạo kprobe tại đầu function bcm2835_spi_transfer_one
# p = probe at entry, r = return probe
echo 'p:spi_entry bcm2835_spi_transfer_one len=%x2' \
    > $TRACE/kprobe_events

# Tạo return probe — capture return value
echo 'r:spi_return bcm2835_spi_transfer_one ret=$retval' \
    >> $TRACE/kprobe_events

# Xem events đã tạo
cat $TRACE/kprobe_events

# Enable events
echo 1 > $TRACE/events/kprobes/spi_entry/enable
echo 1 > $TRACE/events/kprobes/spi_return/enable

echo 1 > $TRACE/tracing_on
# Trigger SPI transfer...
echo 0 > $TRACE/tracing_on

cat $TRACE/trace
# kprobe-0    [000] d...  12.345: spi_entry: (bcm2835_spi_transfer_one) len=0x4
# kprobe-0    [000] d...  12.346: spi_return: (bcm2835_spi_transfer_one <- ...) ret=0x0

# Xóa probe
echo '-:spi_entry' >> $TRACE/kprobe_events
echo '-:spi_return' >> $TRACE/kprobe_events
```

---

### 3.2 Kprobe argument capture

```bash
# Probe tại sys_openat — capture filename
# x0-x5 là registers ARM64
echo 'p:my_open do_sys_openat2 flags=%x2:u64 mode=%x3:u64' \
    > $TRACE/kprobe_events

# Capture string argument
echo 'p:my_open do_sys_openat2 filename=+0(%x1):string' \
    > $TRACE/kprobe_events

echo 1 > $TRACE/events/kprobes/my_open/enable

# Xem files được mở
cat $TRACE/trace | grep my_open
# my_open: filename="/sys/class/thermal/thermal_zone0/temp"
```

---

### 3.3 Kprobe Module — Code thực tế

```c
/* my_kprobe.c — Probe GPIO set value */
#include <linux/kprobes.h>
#include <linux/module.h>
#include <linux/gpio/consumer.h>

static struct kprobe kp;

/* Handler chạy TRƯỚC khi gpiod_set_value thực thi */
static int pre_handler(struct kprobe *p, struct pt_regs *regs)
{
    /* ARM64: x0=gpio_desc, x1=value */
    struct gpio_desc *desc = (struct gpio_desc *)regs->regs[0];
    int value = (int)regs->regs[1];

    pr_info("GPIO set: desc=%p value=%d\n", desc, value);
    return 0;
}

/* Handler chạy SAU khi function return */
static void post_handler(struct kprobe *p, struct pt_regs *regs,
                          unsigned long flags)
{
    pr_info("GPIO set completed\n");
}

static int __init kprobe_init(void)
{
    kp.symbol_name = "gpiod_set_value";
    kp.pre_handler  = pre_handler;
    kp.post_handler = post_handler;

    int ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("register_kprobe failed: %d\n", ret);
        return ret;
    }
    pr_info("Planted kprobe at %pS\n", kp.addr);
    return 0;
}

static void __exit kprobe_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("kprobe removed\n");
}

module_init(kprobe_init);
module_exit(kprobe_exit);
MODULE_LICENSE("GPL");
```

---

## 4. Histogram Triggers — Latency Analysis

```bash
# Đo latency từ I2C transfer start đến complete
# Tạo histogram theo thời gian giữa 2 events

echo 'hist:keys=common_pid:ts0=common_timestamp.usecs' \
    > $TRACE/events/i2c/i2c_write/trigger

echo 'hist:keys=common_pid:lat=common_timestamp.usecs-$ts0:sort=lat' \
    > $TRACE/events/i2c/i2c_result/trigger

# Trigger I2C activity...
i2cget -y 1 0x68 0x75

# Xem histogram
cat $TRACE/events/i2c/i2c_result/hist
# { pid:1234 } hitcount: 45  lat: 285 usecs
# { pid:1234 } hitcount: 12  lat: 312 usecs
```

---

## 5. Stack Tracer

```bash
# Enable stack tracing — tìm function nào dùng stack nhiều nhất
echo 1 > $TRACE/options/sym-userobj
echo 1 > /proc/sys/kernel/stack_tracer_enabled

# Xem max stack usage
cat $TRACE/stack_trace
# Depth    Size   Location    (48 entries)
# -----    ----   --------
#     0     240   ftrace_call+0x4/0x8
#     1     240   ...
#    ...
#    47    3984   start_kernel+0x58/0x5c
```

---

## 6. Trace Pi4-specific Drivers

```bash
# Theo dõi thermal throttling
echo 1 > $TRACE/events/thermal/enable

# Theo dõi cpufreq transitions
echo 1 > $TRACE/events/power/cpu_frequency/enable

# Trace DRM V3D GPU jobs
echo 1 > $TRACE/events/gpu/enable

# Trace block I/O (SD card)
echo 1 > $TRACE/events/block/enable

# Xem tất cả trace events có sẵn
ls $TRACE/events/
```

---

## 7. Perf + Kprobes

```bash
# Dùng perf để trace kprobe
perf probe --add 'bcm2835_spi_transfer_one len=%x2'
perf record -e probe:bcm2835_spi_transfer_one -aR sleep 5
perf script

# Tạo flame graph
perf record -g -F 99 sleep 10
perf script | stackcollapse-perf.pl | flamegraph.pl > flame.svg
```

---

## 8. Tracing Workflow thực tế trên Pi4

```bash
#!/bin/bash
# trace_gpio_latency.sh — Đo latency GPIO toggle

TRACE=/sys/kernel/debug/tracing

# Setup
echo nop > $TRACE/current_tracer
echo > $TRACE/trace
echo > $TRACE/kprobe_events

# Tạo probes
echo 'p:gpio_start gpiod_set_value' >> $TRACE/kprobe_events
echo 'r:gpio_done  gpiod_set_value' >> $TRACE/kprobe_events

# Enable
echo 1 > $TRACE/events/kprobes/gpio_start/enable
echo 1 > $TRACE/events/kprobes/gpio_done/enable

# Histogram
echo 'hist:keys=common_pid:ts=common_timestamp.usecs' \
    >> $TRACE/events/kprobes/gpio_start/trigger
echo 'hist:keys=common_pid:lat=common_timestamp.usecs-$ts,buckets=10' \
    >> $TRACE/events/kprobes/gpio_done/trigger

echo 1 > $TRACE/tracing_on

# Toggle GPIO 1000 lần
for i in $(seq 1 1000); do
    gpioset gpiochip0 17=1
    gpioset gpiochip0 17=0
done

echo 0 > $TRACE/tracing_on
cat $TRACE/events/kprobes/gpio_done/hist
```

---

## 9. File quan trọng trong source

| File | Mô tả |
|------|-------|
| `kernel/trace/ftrace.c` | Ftrace core (220KB) |
| `kernel/trace/trace.c` | Trace buffer management |
| `kernel/trace/trace_kprobe.c` | Kprobe events |
| `kernel/trace/trace_hist.c` | Histogram triggers |
| `kernel/kprobes.c` | Kprobe core (76KB) |
| `arch/arm64/kernel/probes/kprobes.c` | ARM64 kprobe implementation |
| `kernel/trace/trace_irqsoff.c` | IRQ/preempt off tracer |
