# Binder Kernel Driver — drivers/android/binder.c

## 1. Tổng quan

Binder kernel driver là tầng thấp nhất của Android IPC — cung cấp zero-copy message passing, object identity tracking, thread management cho Binder framework.

```
User space (libbinder)
  │ ioctl(fd, BINDER_WRITE_READ, ...)
  ▼
/dev/binder, /dev/hwbinder, /dev/vndbinder
  │
  ▼
binder.c (kernel driver)
  │ Zero-copy via mmap
  │ Reference counting
  └── Thread management
```

---

## 2. Binder Device Files

```
/dev/binder      ← Framework Binder (apps, system_server)
/dev/hwbinder    ← Hardware Binder (HAL services)
/dev/vndbinder   ← Vendor Binder (vendor services)

Separation đảm bảo:
  vendor HAL không access framework Binder namespace
  SELinux enforces access to each device
```

---

## 3. Kernel Data Structures

```c
/* binder.c — Key structures */

/* Một Binder node (object trong 1 process) */
struct binder_node {
    int debug_id;
    spinlock_t lock;
    struct binder_work work;
    union {
        struct rb_node rb_node;    /* node trong proc->nodes */
        struct hlist_node dead_node;
    };
    struct binder_proc *proc;      /* owner process */
    struct hlist_head refs;        /* references from other procs */
    int internal_strong_refs;
    int local_weak_refs;
    int local_strong_refs;
    binder_uintptr_t ptr;          /* user-space pointer */
    binder_uintptr_t cookie;       /* user-space cookie */
    unsigned has_strong_ref:1;
    unsigned pending_strong_ref:1;
    unsigned has_weak_ref:1;
    unsigned pending_weak_ref:1;
};

/* Reference từ process khác đến node */
struct binder_ref {
    struct binder_ref_data data;
    struct rb_node rb_node_desc;   /* rb tree by descriptor */
    struct rb_node rb_node_node;   /* rb tree by node */
    struct hlist_node node_entry;
    struct binder_proc *proc;
    struct binder_node *node;
    struct binder_ref_death *death;
};

/* Per-process binder state */
struct binder_proc {
    struct hlist_node proc_node;
    struct rb_root threads;         /* binder_thread per tid */
    struct rb_root nodes;           /* local binder_nodes */
    struct rb_root refs_by_desc;    /* refs by handle number */
    struct rb_root refs_by_node;
    struct list_head waiting_threads;
    int pid;
    struct task_struct *tsk;
    struct files_struct *files;
    struct mutex files_lock;
    struct hlist_node deferred_work_node;
    int deferred_work;
    bool is_dead;
    struct list_head todo;          /* pending work */
    wait_queue_head_t freeze_wait;
    struct binder_stats stats;
    struct list_head delivered_death;
    int max_threads;
    int requested_threads;
    int requested_threads_started;
    int tmp_ref;
    long default_priority;
    struct dentry *debugfs_entry;
    struct binder_alloc alloc;      /* mmap allocator */
    struct binder_context *context; /* /dev/binder vs /dev/hwbinder */
};
```

---

## 4. Zero-Copy mmap Mechanism

```c
/* Process mmap /dev/binder → shared buffer */
/* binder_alloc.c manages this shared region */

/*
  User process address space:
  ┌──────────────────────────────────┐
  │  ... other mappings ...          │
  │  mmap region [0x7f000000]        │ ← User VA
  │   ↕ same physical pages          │
  │  kernel VA (vmalloc area)        │ ← Kernel VA
  └──────────────────────────────────┘
  
  Sender writes data → kernel maps into RECEIVER's mmap region
  Receiver reads from its own address space
  → Zero extra copy (kernel never copies data)
*/

/* binder_alloc_new_buf(): allocate buffer in target process's mmap */
struct binder_buffer *binder_alloc_new_buf(
    struct binder_alloc *alloc,
    size_t data_size,
    size_t offsets_size,
    size_t extra_buffers_size,
    int is_async,
    int pid);
```

---

## 5. Transaction Flow (ioctl)

```c
/* Step 1: Caller writes BC_TRANSACTION */
struct binder_transaction_data {
    union {
        __u32 handle;           /* target handle (for call) */
        binder_uintptr_t ptr;   /* target ptr (for reply) */
    } target;
    binder_uintptr_t cookie;    /* target object cookie */
    __u32 code;                 /* method code */
    __u32 flags;
    __s32 sender_pid;
    __u32 sender_euid;
    binder_size_t data_size;
    binder_size_t offsets_size;
    union {
        struct { ... } ptr;
        __u8 buf[8];
    } data;
};

/* Kernel allocates buffer in target's mmap space */
/* Copies transaction data → target buffer */
/* Wakes target thread */

/* Step 2: Target thread's ioctl returns BR_TRANSACTION */
/* Target reads from its mmap region (zero copy) */
/* Target processes, sends BC_REPLY */

/* Step 3: Original caller's ioctl returns BR_REPLY */
```

---

## 6. Thread Pool Management

```c
/* Binder spawns threads on demand */

/* When server gets BR_SPAWN_LOOPER: */
/*   IPCThreadState::joinThreadPool() */
/*   → creates new thread */
/*   → registers with kernel via BC_REGISTER_LOOPER */

/* Kernel tracks: */
/*   max_threads (default 15) */
/*   requested_threads (threads kernel requested) */
/*   requested_threads_started (threads user created) */

/* main thread = BC_ENTER_LOOPER (não thể idle) */
/* spawned thread = BC_REGISTER_LOOPER (có thể exit khi idle) */
```

---

## 7. Debug /proc/binder/

```bash
# Binder stats (all processes)
adb shell cat /proc/binder/stats

# Per-process binder state
adb shell cat /proc/binder/proc/$(pidof system_server)

# Binder state overview
adb shell cat /proc/binder/state
# proc 1234 (system_server)
#   thread 1234: l 12
#   node 1: u00007f... c00007f... hs 1 hw 1 ls 0 lw 0 is 2 iw 0 tr 0
#   ref 2: desc 2 node 5 s 1 w 1 d 000...
#   buffer 12345: 1024 size: active

# Binder failed reply stats
adb shell cat /proc/binder/failed_transaction_log

# Recent transactions
adb shell cat /proc/binder/transaction_log
```

---

## 8. Binder Death Notification

```c
/* Register for death notification */
/* BC_REQUEST_DEATH_NOTIFICATION */

/* When server process dies: */
/* kernel iterates all refs → sends BR_DEAD_BINDER */
/* Client receives death notification */
/* client calls BC_DEAD_BINDER_DONE when handled */

/* In Java: */
/*
   IBinder.DeathRecipient:
     binder.linkToDeath(recipient, 0);
     // recipient.binderDied() called on death
*/

/* Ví dụ: ConnectivityService watch cho netd die */
```

---

## 9. Binder Security

```c
/* Kernel fills in sender credentials automatically */
/* Cannot be spoofed by user space */

/* In transaction: */
/*   sender_pid  = current->tgid */
/*   sender_euid = from_kuid(&init_user_ns, current_euid()) */

/* Java: Binder.getCallingPid(), Binder.getCallingUid() */
/* These are filled by kernel, trustworthy */

/* SELinux check in binder: */
/* security_binder_transaction(caller, target) */
/* → checked for each transaction */
```

---

## 10. Binder Freeze (Android 12+)

```c
/* Process can be "frozen" (for cached apps) */
/* binder_proc_freeze(): */
/*   Set proc->is_frozen = true */
/*   Pending async transactions queued */
/*   Sync transactions → EAGAIN (caller handles) */

/* Freezer used by: */
/*   ActivityManagerService → freeze cached apps */
/*   CachedAppOptimizer (Android 12+) */
/*   → Reduces CPU usage of background apps */
```

```bash
# Xem frozen processes
adb shell dumpsys activity processes | grep -A3 "frozen"

# /proc/<pid>/freezer (cgroup freezer)
adb shell cat /sys/fs/cgroup/freezer/frozen/cgroup.procs
```

---

## 11. File quan trọng trong source

| File | Mô tả |
|------|-------|
| `kernel/common/drivers/android/binder.c` | Binder driver (7000+ lines) |
| `kernel/common/drivers/android/binder_alloc.c` | mmap allocator |
| `kernel/common/include/uapi/linux/android/binder.h` | User-space interface |
| `frameworks/native/libs/binder/IPCThreadState.cpp` | User-space Binder loop |
| `frameworks/native/libs/binder/ProcessState.cpp` | Open /dev/binder, mmap |
| `frameworks/native/libs/binder/Binder.cpp` | BBinder (service side) |
| `frameworks/native/libs/binder/BpBinder.cpp` | BpBinder (proxy side) |
