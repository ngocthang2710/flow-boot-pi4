# Inter-Core Communication — Android 16 on Raspberry Pi 4

## Hardware Overview

- **SoC**: BCM2711
- **CPU**: 4x Cortex-A72 trong 1 cluster
- **Cache**: L1 I-cache 48KB + D-cache 32KB (per core), L2 1MB (shared)
- **Bus**: ACE (AXI Coherency Extension) — hardware cache coherency tự động giữa 4 cores
- **Interrupt controller**: GICv2 (`RPI4_GIC_GICD_BASE` / `RPI4_GIC_GICC_BASE`)

```
┌─ Core 0 ────┐  ┌─ Core 1 ────┐  ┌─ Core 2 ────┐  ┌─ Core 3 ────┐
│ L1 I: 48KB  │  │ L1 I: 48KB  │  │ L1 I: 48KB  │  │ L1 I: 48KB  │
│ L1 D: 32KB  │  │ L1 D: 32KB  │  │ L1 D: 32KB  │  │ L1 D: 32KB  │
└──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘
       └─────────────────┴─────────────────┴─────────────────┘
                          L2 Cache 1MB (shared)
                          ACE bus — hardware coherency
                               │
                             GICv2
                    GICD_SGIR → SGI interrupt
```

> **Lưu ý quan trọng**: Cache coherency (hardware đảm bảo) và memory ordering (cần barrier) là **hai vấn đề khác nhau**. ACE giải quyết coherency nhưng không giải quyết reordering của CPU.

---

## Cơ chế 1: IPI — Inter-Processor Interrupt

IPI là con đường nhanh nhất để một core ra lệnh cho core khác.

### Các loại IPI trên ARM64

```c
// kernel/common/arch/arm64/kernel/smp.c

enum ipi_msg_type {
    IPI_RESCHEDULE,    // "Mày có task mới, schedule lại đi"
    IPI_CALL_FUNC,     // "Chạy hàm này trên core mày ngay bây giờ"
    IPI_CPU_STOP,      // "Dừng lại — hệ thống panic/kexec"
    IPI_CPU_STOP_NMI,  // Như trên nhưng qua NMI (không thể block)
    IPI_TIMER,         // Broadcast timer tick
    IPI_IRQ_WORK,      // Chạy irq_work queue
    IPI_CPU_BACKTRACE, // Dump stack trace của core đó (debugging)
    IPI_KGDB_ROUNDUP,  // Kernel debugger tập hợp tất cả cores
};
```

### Luồng gửi IPI (ví dụ: reschedule)

```
Core 0 wake up task → phải chạy trên Core 1:

resched_curr(rq)                    ← kernel/sched/core.c:1185
    └── smp_send_reschedule(cpu=1)
         └── smp_cross_call(cpumask_of(1), IPI_RESCHEDULE)
              └── __ipi_send_mask(ipi_desc[IPI_RESCHEDULE], target)
                   └── GICv2: gic_ipi_send_mask()
                        └── writel(SGI_TARGET_CPU1 | IPI_RESCHEDULE,
                                   GICD_SGIR)   ← 1 write vào GIC register
```

### Luồng nhận IPI

```
Core 1 nhận GICv2 SGI interrupt:

ipi_handler(irq, data)              ← percpu IRQ handler đã register
    └── do_handle_IPI(ipinr)
         └── switch (ipinr):
              IPI_RESCHEDULE  → scheduler_ipi()
                                 → set_tsk_need_resched(current)
                                    → preempt tại interrupt return
              IPI_CALL_FUNC   → generic_smp_call_function_interrupt()
              IPI_CPU_STOP    → local_cpu_stop(cpu)
              IPI_TIMER       → tick_receive_broadcast()
              IPI_IRQ_WORK    → irq_work_run()
```

### smp_call_function — chạy code trên core khác

```c
// Ví dụ: TLB flush trên tất cả cores sau khi unmap page
smp_call_function_many(mask, flush_tlb_func, &info, 1 /*wait*/);
    → gửi IPI_CALL_FUNC đến tất cả core trong mask
    → mỗi core nhận interrupt → chạy flush_tlb_func(&info)
    → nếu wait=1: Core 0 spin chờ tất cả cores hoàn thành
```

---

## Cơ chế 2: Atomic Operations — LDXR/STXR (Load-Link / Store-Conditional)

Nền tảng của mọi synchronization primitive. Không cần lock, không cần IPI.

### LL/SC trên ARM64

```c
// arch/arm64/include/asm/atomic_ll_sc.h

// atomic_add(5, &counter) biên dịch thành:
asm volatile(
    "prfm  pstl1strm, %2\n"   // prefetch hint: chuẩn bị store exclusive
"1: ldxr   %w0, %2\n"         // Load eXclusive: đặt "monitor" trên địa chỉ
    "add    %w0, %w0, %w3\n"  // tính toán (không ảnh hưởng monitor)
    "stxr   %w1, %w0, %2\n"   // Store eXclusive:
                               //   w1=0: thành công (không ai ghi vào địa chỉ đó)
                               //   w1=1: thất bại (core khác đã can thiệp)
    "cbnz   %w1, 1b\n"        // nếu thất bại → retry từ ldxr
);
```

**Cơ chế hardware exclusive monitor:**
```
Core 0: ldxr [addr]  → local monitor: EXCLUSIVE
Core 1: ldxr [addr]  → cả 2 cùng monitor địa chỉ đó

Core 1: stxr [addr]  → thành công, global monitor của Core 0 bị CLEAR
Core 0: stxr [addr]  → THẤT BẠI (monitor đã clear) → retry

→ Chỉ 1 core "thắng" mỗi lần, không deadlock, không starvation
```

### LSE (Large System Extensions) — ARM8.1, BCM2711 hỗ trợ

```c
// arch/arm64/include/asm/atomic_lse.h — nhanh hơn LL/SC

// Thay vì retry loop, dùng 1 instruction atomic:
"ldadd  %w0, %w2, %1\n"   // atomic Load-and-ADD
"ldset  %w0, %w2, %1\n"   // atomic OR
"ldclr  %w0, %w2, %1\n"   // atomic AND-NOT
"swp    %w0, %w2, %1\n"   // atomic swap

// Kernel tự chọn LSE hay LL/SC qua CPU feature detection:
ALTERNATIVE(ll_sc_version, lse_version, ARM64_HAS_LSE_ATOMICS)
```

---

## Cơ chế 3: Memory Barriers

Cache coherency (ACE bus) đảm bảo tất cả cores thấy cùng giá trị, nhưng **không đảm bảo thứ tự**. CPU và compiler có thể reorder instructions.

### Các barrier trên ARM64

```c
// arch/arm64/include/asm/barrier.h

// Inter-core (Inner Shareable domain = cluster):
#define __smp_mb()   dmb(ish)    // full barrier: loads và stores
#define __smp_rmb()  dmb(ishld)  // barrier chỉ cho loads
#define __smp_wmb()  dmb(ishst)  // barrier chỉ cho stores

// Full system barrier (kể cả VideoCore GPU, DMA):
#define __mb()       dsb(sy)     // chặn mọi thứ, chờ hoàn thành

// Instruction pipeline:
#define isb()        asm("isb")  // flush pipeline, re-fetch instructions
```

### Shareability domain trên BCM2711

```
Inner Shareable  = 4x Cortex-A72 + L2 cache
                 → dmb(ish) đủ cho inter-core sync

Outer Shareable  = toàn SoC (kể cả VideoCore/GPU)
                 → dmb(osh) cần khi sync với GPU hoặc DMA

Full System (sy) = tất cả observers kể cả I/O
                 → dsb(sy) cho MMIO, device registers
```

### Ví dụ: tại sao cần barrier

```
Không có barrier — Core 1 có thể thấy sai thứ tự:
  Core 0: data = 42          Core 1: while (!ready) {}
          ready = 1                  use(data)  ← có thể thấy data=0 !
                                                   (CPU reorder write)

Với barrier — đúng:
  Core 0: WRITE_ONCE(data, 42)
          smp_wmb()           → dmb(ishst): đảm bảo data ghi ra trước
          WRITE_ONCE(ready, 1)

  Core 1: while (!READ_ONCE(ready)) {}
          smp_rmb()           → dmb(ishld): đảm bảo ready đọc trước data
          use(data)           ← luôn thấy data=42
```

### Store-Release / Load-Acquire (hiệu quả hơn dmb)

```c
// Thay vì dmb + store, dùng STLR (Store-Release):
asm("stlr %w1, %0");   // write + implicit barrier phía sau

// Thay vì load + dmb, dùng LDAR (Load-Acquire):
asm("ldar %w0, %1");   // read + implicit barrier phía trước

// Linux dùng trong __smp_store_release / __smp_load_acquire
// → spinlock unlock dùng stlr thay vì dmb + str
```

---

## Cơ chế 4: Queued Spinlock (qspinlock / MCS)

Linux không dùng test-and-set spinlock đơn giản (gây cache line bouncing). Thay vào đó dùng **MCS-based qspinlock**.

### Cấu trúc lock 32-bit

```
 31         17  16       9   8         2   1        0
 ┌───────────┬───────────┬─────────────┬────────────┐
 │   tail    │ tail cpu  │    zero     │  pending   │  locked  │
 └───────────┴───────────┴─────────────┴────────────┘
```

### 3 trường hợp lấy lock

```
kernel/locking/qspinlock.c

[CASE 1] Lock rảnh (val = 0):
    atomic_cmpxchg(lock, 0, _Q_LOCKED_VAL)  → xong ngay, 0 contention

[CASE 2] 1 waiter — dùng pending bit (tránh cấp phát MCS node):
    set pending bit
    spin on locked bit (chờ owner unlock)
    khi owner unlock: clear_pending_set_locked()
    → chỉ 1 cache line, không cần queue

[CASE 3] Nhiều waiters — MCS queue:
    mỗi core cấp phát MCS node trên per-cpu array (stack riêng)
    xchg(lock->tail, my_node)    ← enqueue bản thân vào tail
    spin on MY node->locked      ← KHÔNG spin trên lock chính
                                    → không cache thrashing!
    core trước release: next->locked = 1  ← thông báo 1:1
    → O(1) wakeup, cache line chỉ di chuyển giữa 2 cores kề nhau
```

**Tại sao MCS tốt hơn test-and-set?**
```
test-and-set (cũ):     Core 1,2,3 đều spin trên lock->val
                       → mỗi lần unlock: cache line "bouncing" qua 3 cores

MCS queue (hiện tại):  Core 1 spin trên node1->locked (địa chỉ riêng)
                       Core 2 spin trên node2->locked (địa chỉ riêng)
                       → unlock chỉ write vào next node, 1 cache line transfer
```

---

## Cơ chế 5: Scheduler — Task Migration

Cách kernel quyết định core nào chạy task nào và migrate task giữa cores.

### Wake-up path

```
try_to_wake_up(task)             ← kernel/sched/core.c
    └── select_task_rq_fair()    ← CFS chọn core
         ├── ưu tiên core đang idle
         ├── ưu tiên core cùng LLC (L2 cache — cache affinity)
         └── xem xét CPU capacity, load balancing
    └── nếu task đang sleep trên core khác:
         ttwu_queue(p, target_cpu)
              └── smp_send_reschedule(target_cpu)
                   └── IPI_RESCHEDULE → GICv2 SGI
```

### Load balancing (mỗi tick)

```
scheduler_tick()                 ← per-core timer interrupt
    └── trigger_load_balance()
         └── run_rebalance_domains()
              └── load_balance()
                   ├── tìm core busy nhất (src) và idle nhất (dst)
                   ├── pull tasks từ src sang dst
                   └── migrate_task_to(task, dst_cpu)
                        └── smp_call_function_single(src, move_task_fn)
                             └── IPI_CALL_FUNC → src core thực hiện migration
```

---

## Tổng hợp: Khi nào dùng cơ chế nào

| Tình huống | Cơ chế | Latency |
|---|---|---|
| Đếm, cộng trừ giá trị | `atomic_add()` — LDXR/STXR hoặc LSE | ~5-10 ns |
| Bảo vệ critical section ngắn | `spin_lock()` — qspinlock MCS | ~20-50 ns |
| Thông báo core khác reschedule | IPI `IPI_RESCHEDULE` → GICv2 SGI | ~1-2 µs |
| Chạy hàm trên core khác | `smp_call_function()` → `IPI_CALL_FUNC` | ~2-5 µs |
| Đảm bảo thứ tự ghi/đọc | `smp_mb()` / `smp_wmb()` / `stlr` / `ldar` | 0 (in-order) |
| Bảo vệ critical section dài | `mutex_lock()` → futex (sleep nếu contend) | ~10+ µs |
| TLB flush đồng bộ tất cả cores | `smp_call_function_many()` + wait | ~5-10 µs |

---

## Nguồn source code

| File | Nội dung |
|------|---------|
| `kernel/common/arch/arm64/kernel/smp.c` | IPI types, `do_handle_IPI()`, `smp_cross_call()` |
| `kernel/common/arch/arm64/include/asm/atomic_ll_sc.h` | LDXR/STXR atomic ops |
| `kernel/common/arch/arm64/include/asm/atomic_lse.h` | LSE single-instruction atomics |
| `kernel/common/arch/arm64/include/asm/barrier.h` | `dmb`/`dsb`/`isb`, `stlr`/`ldar` |
| `kernel/common/arch/arm64/include/asm/spinlock.h` | ARM64 spinlock → qspinlock |
| `kernel/common/kernel/locking/qspinlock.c` | MCS queue spinlock implementation |
| `kernel/common/kernel/sched/core.c` | `resched_curr()`, `try_to_wake_up()`, `load_balance()` |
| `external/arm-trusted-firmware/plat/rpi/common/rpi3_pm.c` | PSCI CPU on/off (ATF level) |
| `external/arm-trusted-firmware/plat/rpi/rpi4/include/platform_def.h` | Power state definitions |
