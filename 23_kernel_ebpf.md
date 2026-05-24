# Kernel eBPF — Extended Berkeley Packet Filter — Raspberry Pi 4

## 1. Tổng quan

eBPF là máy ảo trong kernel cho phép chạy chương trình sandbox an toàn mà không cần viết kernel module. Được dùng để trace, monitor network, implement security policy.

**Config đã enabled:**
```
CONFIG_BPF=y
CONFIG_BPF_SYSCALL=y         ← bpf() syscall
CONFIG_BPF_JIT=y             ← JIT compiler (ARM64 native code)
CONFIG_BPF_JIT_ALWAYS_ON=y   ← Luôn dùng JIT
CONFIG_BPF_EVENTS=y          ← Attach vào trace events
CONFIG_BPF_LSM=y             ← LSM hooks (security)
CONFIG_NET_CLS_BPF=y         ← Network traffic classifier
CONFIG_NET_ACT_BPF=y         ← Network action
CONFIG_BPF_STREAM_PARSER=y   ← Stream parser
CONFIG_XDP_SOCKETS=y         ← XDP sockets (AF_XDP)
CONFIG_HAVE_EBPF_JIT=y       ← Platform hỗ trợ JIT
```

**Core files:**
```
kernel/common/kernel/bpf/
├── core.c          ← BPF interpreter và verifier cơ bản
├── verifier.c      ← Static verifier (30000+ lines)
├── syscall.c       ← bpf() syscall handler
├── jit.c           ← JIT compilation
├── helpers.c       ← BPF helper functions
├── tracing.c       ← Tracing integration
├── bpf_lsm.c       ← LSM hooks
└── net.c           ← Network integration

kernel/common/kernel/trace/bpf_trace.c  ← BPF tracing (93KB)
```

---

## 2. BPF Program Types

| Type | Hook point | Dùng để |
|------|-----------|---------|
| `BPF_PROG_TYPE_KPROBE` | Kprobe/kretprobe | Trace kernel function |
| `BPF_PROG_TYPE_TRACEPOINT` | Trace events | Trace static tracepoints |
| `BPF_PROG_TYPE_PERF_EVENT` | Perf events | Profiling, sampling |
| `BPF_PROG_TYPE_SOCKET_FILTER` | Socket | Filter packets |
| `BPF_PROG_TYPE_XDP` | NIC driver | Fast packet processing |
| `BPF_PROG_TYPE_CGROUP_SKB` | Cgroup | Per-cgroup network policy |
| `BPF_PROG_TYPE_LSM` | LSM hooks | Security policy |
| `BPF_PROG_TYPE_FENTRY/FEXIT` | Function entry/exit | Low-overhead tracing |

---

## 3. BPFtrace — High-level Tracing

BPFtrace dùng ngôn ngữ AWK-like để viết eBPF program nhanh:

```bash
# Trace tất cả write() syscall
bpftrace -e 'tracepoint:syscalls:sys_enter_write {
    printf("%s wrote %d bytes\n", comm, args->count);
}'

# Đếm syscalls theo process
bpftrace -e 'tracepoint:syscalls:sys_enter_* { @[comm] = count(); }'

# Histogram latency của read()
bpftrace -e '
tracepoint:syscalls:sys_enter_read { @start[tid] = nsecs; }
tracepoint:syscalls:sys_exit_read  {
    @lat = hist(nsecs - @start[tid]);
    delete(@start[tid]);
}'

# Trace GPIO trên Pi4
bpftrace -e 'kprobe:gpiod_set_value {
    printf("GPIO set: value=%d\n", arg1);
}'

# Đo I2C transfer latency
bpftrace -e '
kprobe:i2c_bcm2835_xfer       { @t[tid] = nsecs; }
kretprobe:i2c_bcm2835_xfer    {
    printf("I2C latency: %d us\n", (nsecs-@t[tid])/1000);
    delete(@t[tid]);
}'

# Top files được open nhiều nhất
bpftrace -e '
tracepoint:syscalls:sys_enter_openat {
    @[str(args->filename)] = count();
}
END { print(@, 10); }'

# Monitor network traffic
bpftrace -e 'kprobe:tcp_sendmsg {
    printf("TCP send: pid=%d size=%d\n", pid, arg2);
}'
```

---

## 4. BCC (BPF Compiler Collection) Tools

```bash
# Trace block I/O latency (SD card trên Pi4)
biolatency -d mmcblk0

# Top processes theo CPU
cpudist

# Network connections
tcpconnect

# File open trace
opensnoop

# Kernel function latency
funclatency bcm2835_spi_transfer_one

# Stack trace profiling (flame graph)
profile -F 99 -f 30 | flamegraph.pl > cpu.svg

# Memory allocation trace
memleak -p $(pgrep system_server)
```

---

## 5. Viết BPF Program bằng C (libbpf)

```c
/* gpio_monitor.bpf.c */
#include <linux/bpf.h>
#include <bpf/bpf_helpers.h>
#include <bpf/bpf_tracing.h>

/* Map để lưu thống kê */
struct {
    __uint(type, BPF_MAP_TYPE_HASH);
    __type(key,  u32);   /* GPIO number */
    __type(value, u64);  /* toggle count */
    __uint(max_entries, 64);
} gpio_stats SEC(".maps");

/* Probe tại gpiod_set_value */
SEC("kprobe/gpiod_set_value")
int BPF_KPROBE(trace_gpio_set, struct gpio_desc *desc, int value)
{
    u32 gpio_num = 17; /* simplified */
    u64 *count = bpf_map_lookup_elem(&gpio_stats, &gpio_num);
    if (count) {
        __sync_fetch_and_add(count, 1);
    } else {
        u64 init = 1;
        bpf_map_update_elem(&gpio_stats, &gpio_num, &init, BPF_ANY);
    }

    bpf_printk("GPIO %d set to %d\n", gpio_num, value);
    return 0;
}

/* Tracepoint tại thermal throttle */
SEC("tracepoint/thermal/thermal_zone_trip")
int trace_thermal(struct trace_event_raw_thermal_zone_trip *ctx)
{
    bpf_printk("Thermal trip on zone %s, temp=%d\n",
               ctx->thermal_zone, ctx->temp);
    return 0;
}

char LICENSE[] SEC("license") = "GPL";
```

```c
/* gpio_monitor.c — Userspace loader */
#include <bpf/libbpf.h>
#include "gpio_monitor.skel.h"

int main(void)
{
    struct gpio_monitor_bpf *skel = gpio_monitor_bpf__open_and_load();
    gpio_monitor_bpf__attach(skel);

    printf("Monitoring GPIO... Press Ctrl+C to stop\n");

    /* Read map periodically */
    while (1) {
        sleep(1);
        u32 key = 17;
        u64 count;
        bpf_map__lookup_elem(skel->maps.gpio_stats,
                             &key, sizeof(key),
                             &count, sizeof(count), 0);
        printf("GPIO 17 toggles: %llu\n", count);
    }

    gpio_monitor_bpf__destroy(skel);
    return 0;
}
```

---

## 6. XDP — Fast Packet Processing

XDP (eXpress Data Path) chạy BPF program ngay tại driver NIC, trước khi packet vào network stack:

```c
/* xdp_drop_icmp.bpf.c — Drop ICMP packets */
#include <linux/bpf.h>
#include <linux/if_ether.h>
#include <linux/ip.h>
#include <bpf/bpf_helpers.h>

SEC("xdp")
int xdp_prog(struct xdp_md *ctx)
{
    void *data_end = (void *)(long)ctx->data_end;
    void *data     = (void *)(long)ctx->data;

    struct ethhdr *eth = data;
    if (eth + 1 > data_end)
        return XDP_PASS;

    if (eth->h_proto != htons(ETH_P_IP))
        return XDP_PASS;

    struct iphdr *ip = (void *)(eth + 1);
    if (ip + 1 > data_end)
        return XDP_PASS;

    if (ip->protocol == IPPROTO_ICMP)
        return XDP_DROP;   /* Drop ICMP */

    return XDP_PASS;
}

char LICENSE[] SEC("license") = "GPL";
```

```bash
# Load XDP program vào interface
ip link set eth0 xdp obj xdp_drop_icmp.o sec xdp

# Test: ping bị drop
ping 8.8.8.8  # No response

# Remove
ip link set eth0 xdp off
```

---

## 7. BPF Maps — Chia sẻ data giữa kernel và userspace

```c
/* Các loại map thường dùng */

/* Hash map — key/value lookup */
struct { __uint(type, BPF_MAP_TYPE_HASH); ... } my_hash SEC(".maps");

/* Array — index-based */
struct { __uint(type, BPF_MAP_TYPE_ARRAY); ... } my_arr SEC(".maps");

/* Ring buffer — efficient event streaming */
struct { __uint(type, BPF_MAP_TYPE_RINGBUF); ... } events SEC(".maps");

/* Per-CPU array — no locking needed */
struct { __uint(type, BPF_MAP_TYPE_PERCPU_ARRAY); ... } percpu SEC(".maps");

/* Stack trace */
struct { __uint(type, BPF_MAP_TYPE_STACK_TRACE); ... } stacks SEC(".maps");
```

---

## 8. BPF LSM — Security Policy

```c
/* deny_gpio_write.bpf.c — Deny GPIO writes từ process không có quyền */
#include <linux/bpf.h>
#include <bpf/bpf_helpers.h>
#include <bpf/bpf_tracing.h>

SEC("lsm/file_permission")
int BPF_PROG(restrict_gpio, struct file *file, int mask)
{
    /* Kiểm tra nếu đang access /dev/gpiochip* */
    char fname[32];
    bpf_d_path(&file->f_path, fname, sizeof(fname));

    if (bpf_strncmp(fname, 12, "/dev/gpiochi") == 0) {
        u32 uid = bpf_get_current_uid_gid() & 0xFFFFFFFF;
        /* Chỉ cho phép UID 1000 (Android system) */
        if (uid != 1000) {
            bpf_printk("GPIO access denied for UID %d\n", uid);
            return -EPERM;
        }
    }
    return 0;
}

char LICENSE[] SEC("license") = "GPL";
```

---

## 9. Debug BPF Programs

```bash
# Xem BPF programs đang loaded
bpftool prog list

# Xem BPF maps
bpftool map list

# Dump BPF bytecode
bpftool prog dump xlated id 42

# Dump JIT native code (ARM64)
bpftool prog dump jited id 42

# Xem BPF program stats
bpftool prog show id 42 -j | jq .

# Trace BPF verifier
echo 1 > /proc/sys/net/core/bpf_jit_enable
dmesg | grep BPF
```

---

## 10. Thực hành: Monitor Pi4 với eBPF

```bash
#!/usr/bin/env bpftrace
# pi4_monitor.bt — Monitor toàn diện Pi4

# CPU temperature check mỗi giây
interval:s:1 {
    // Không thể đọc file trực tiếp từ bpftrace
    // Dùng tracepoint thay thế
    printf("=== Pi4 Monitor ===\n");
}

# SPI transfer count
kprobe:bcm2835_spi_transfer_one {
    @spi_count = count();
}

# I2C transfer count
kprobe:i2c_bcm2835_xfer {
    @i2c_count = count();
}

# GPU job submission
tracepoint:gpu:gpu_job_enqueue {
    @gpu_jobs = count();
}

# Thermal trips
tracepoint:thermal:thermal_zone_trip {
    printf("THERMAL TRIP: %s temp=%d\n", str(args->thermal_zone), args->temp);
}

# CPU freq change
tracepoint:power:cpu_frequency {
    printf("CPU freq: %d -> %d MHz\n",
           @last_freq / 1000, args->state / 1000);
    @last_freq = args->state;
}

END {
    printf("SPI transfers: %d\n", @spi_count);
    printf("I2C transfers: %d\n", @i2c_count);
    printf("GPU jobs: %d\n", @gpu_jobs);
}
```

---

## 11. File quan trọng trong source

| File | Mô tả |
|------|-------|
| `kernel/bpf/core.c` | BPF interpreter core |
| `kernel/bpf/verifier.c` | Static verifier (safety checker) |
| `kernel/bpf/syscall.c` | bpf() syscall |
| `kernel/bpf/bpf_lsm.c` | LSM hooks implementation |
| `kernel/trace/bpf_trace.c` | BPF tracing integration |
| `arch/arm64/net/bpf_jit_comp.c` | ARM64 JIT compiler |
| `net/core/filter.c` | Network BPF filter |
| `include/linux/bpf.h` | BPF core types và API |
| `include/uapi/linux/bpf.h` | Userspace BPF API |
