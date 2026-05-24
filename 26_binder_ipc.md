# Binder IPC — Xương sống giao tiếp Android

## 1. Tổng quan

Binder là cơ chế IPC (Inter-Process Communication) đặc trưng của Android. **Mọi** giao tiếp giữa app ↔ system service ↔ HAL đều đi qua Binder.

```
Không có Binder = không có Android:
  startActivity()        → Binder → ActivityManagerService
  getSystemService()     → Binder → SystemServiceManager
  camera.open()          → Binder → CameraService
  AudioTrack.write()     → Binder → AudioFlinger
  Sensor.registerListener → Binder → SensorService
```

**Kernel driver:** `kernel/common/drivers/android/binder.c` (13000+ lines)  
**Userspace:** `frameworks/native/libs/binder/`

---

## 2. Kiến trúc Binder

```
Process A (client)              Process B (server)
┌──────────────┐                ┌──────────────────────┐
│  IBinder     │                │  BBinder (service)    │
│  BpBinder    │                │  onTransact()         │
│  (proxy)     │                │                       │
└──────┬───────┘                └──────────┬────────────┘
       │ ioctl(BC_TRANSACTION)             │
       ▼                                  ▼
┌─────────────────────────────────────────────────────┐
│          /dev/binder  (kernel driver)               │
│                                                     │
│  binder_transaction()                               │
│  ├─ Copy data từ A → kernel buffer                  │
│  ├─ Map kernel buffer vào address space của B       │
│  └─ Wake up B's binder thread                       │
└─────────────────────────────────────────────────────┘

Key insight: chỉ 1 lần copy (A→kernel), B đọc trực tiếp
             qua memory mapping → ZERO COPY cho receiver
```

---

## 3. Kernel Driver — Cơ chế thực tế

### 3.1 Binder nodes và refs

```c
/* drivers/android/binder.c */

/* Mỗi service = 1 binder_node */
struct binder_node {
    int           debug_id;
    struct rb_node rb_node;     /* node trong rb-tree của proc */
    struct binder_proc *proc;   /* owner process */
    struct hlist_head refs;     /* danh sách client references */
    int           internal_strong_refs;
    binder_uintptr_t ptr;       /* userspace pointer (BBinder*) */
    binder_uintptr_t cookie;
    struct {
        u8 has_strong_ref;
        u8 pending_strong_ref;
        u8 has_weak_ref;
        u8 accept_fds;
    };
};

/* Mỗi client reference đến service = 1 binder_ref */
struct binder_ref {
    struct binder_ref_data data;  /* desc (handle number) */
    struct binder_node    *node;  /* → service node */
    struct binder_proc    *proc;  /* owner client */
};
```

### 3.2 Transaction flow trong kernel

```c
static void binder_transaction(struct binder_proc *proc,
                               struct binder_thread *thread,
                               struct binder_transaction_data *tr)
{
    /* 1. Tìm target process qua handle */
    target_node = binder_get_node_from_ref(proc, tr->target.handle, ...);
    target_proc = target_node->proc;

    /* 2. Cấp phát buffer trong target process */
    t->buffer = binder_alloc_new_buf(&target_proc->alloc,
                                      tr->data_size, tr->offsets_size, ...);

    /* 3. Copy data từ sender (1 lần duy nhất) */
    copy_from_user(t->buffer->user_data, tr->data.ptr.buffer, tr->data_size);

    /* 4. Process flat_binder_objects (fix up pointers, transfer fds) */
    binder_transaction_buffer_release(target_proc, NULL, t->buffer, ...);

    /* 5. Enqueue transaction vào target thread/process */
    binder_enqueue_thread_work(target_thread, &t->work);

    /* 6. Wake up target */
    wake_up_interruptible_sync(target_thread->wait);
}
```

### 3.3 Memory mapping — zero copy

```c
/* Binder dùng mmap để map kernel buffer vào target process */
static int binder_mmap(struct file *filp, struct vm_area_struct *vma)
{
    struct binder_proc *proc = filp->private_data;

    /* Giới hạn: mỗi process tối đa 4MB mapped */
    if (vma->vm_end - vma->vm_start > SZ_4M)
        vma->vm_end = vma->vm_start + SZ_4M;

    /* Tạo virtual area cho binder buffer */
    proc->alloc.vma = vma;
    proc->alloc.vma_vm_mm = vma->vm_mm;
    /* ... */
}
/* Kết quả: sender copy 1 lần vào kernel,
            receiver đọc trực tiếp qua mmap = zero copy */
```

---

## 4. Userspace Binder — AIDL Stack

```
App Java/Kotlin
  ↓ generated AIDL stub
BpInterface (Proxy)      ← Phía client: gọi transact()
  ↓ Parcel serialize
BpBinder::transact()
  ↓ ioctl(BINDER_WRITE_READ, BC_TRANSACTION)
/dev/binder              ← Kernel
  ↑ ioctl(BR_TRANSACTION)
BBinder::transact()
  ↑ onTransact() dispatch
BnInterface (Stub)       ← Phía server: implement method
  ↑ Parcel deserialize
Service Implementation
```

---

## 5. AIDL — Interface Definition

```java
// IMyService.aidl
package com.example;
interface IMyService {
    int add(int a, int b);
    String hello(String name);
    void registerCallback(IMyCallback cb);
}
```

**Generated code (simplified):**

```java
// Proxy (client side)
public static class Proxy implements IMyService {
    private IBinder mRemote;

    public int add(int a, int b) throws RemoteException {
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        data.writeInt(a);
        data.writeInt(b);
        mRemote.transact(TRANSACTION_add, data, reply, 0);
        return reply.readInt();
    }
}

// Stub (server side)
public static abstract class Stub extends Binder {
    public boolean onTransact(int code, Parcel data, Parcel reply, int flags) {
        switch (code) {
            case TRANSACTION_add:
                int a = data.readInt();
                int b = data.readInt();
                reply.writeInt(add(a, b));
                return true;
        }
    }
}
```

---

## 6. ServiceManager — Registry

```
ServiceManager là process đặc biệt (handle=0),
đóng vai trò DNS cho Binder services:

addService("window", mWindowManagerBinder)    ← đăng ký
IBinder b = getService("window")              ← tìm kiếm
```

```java
// Đăng ký service (system server)
ServiceManager.addService(Context.WINDOW_SERVICE, wm);

// Lookup service (client)
IBinder b = ServiceManager.getService(Context.WINDOW_SERVICE);
IWindowManager wm = IWindowManager.Stub.asInterface(b);
```

---

## 7. Binder Thread Pool

```
Mỗi process có thread pool phục vụ Binder transactions:

ProcessState::self()->startThreadPool();  ← Khởi tạo pool

Mặc định: tối đa 15 threads/process (BINDER_SET_MAX_THREADS)
Main thread + binder threads = tổng số threads phục vụ

Khi tất cả threads bận → transaction block (synchronous)
→ Deadlock risk nếu A chờ B mà B chờ A (trực tiếp/gián tiếp)
```

---

## 8. Binder Object Types

| Type | Mô tả |
|------|-------|
| `BINDER_TYPE_BINDER` | Strong reference đến BBinder local |
| `BINDER_TYPE_WEAK_BINDER` | Weak reference |
| `BINDER_TYPE_HANDLE` | Remote handle (reference đến process khác) |
| `BINDER_TYPE_FD` | File descriptor (được dup sang process kia) |
| `BINDER_TYPE_FDA` | Array of FDs |
| `BINDER_TYPE_PTR` | Buffer pointer (scatter-gather) |

---

## 9. One-way vs Two-way Transaction

```java
// Two-way (synchronous, default): chờ reply
int result = service.add(1, 2);  // block đến khi có kết quả

// One-way (async): không chờ
// Khai báo trong AIDL:
oneway void fireAndForget(String msg);
// Gọi: không block, được queue trong server
service.fireAndForget("hello");
```

---

## 10. Debug Binder

```bash
# Xem Binder state
cat /sys/kernel/debug/binder/state

# Stats mỗi process
cat /sys/kernel/debug/binder/stats

# Transactions đang pending
cat /sys/kernel/debug/binder/transactions

# Failed transactions
cat /sys/kernel/debug/binder/failed_transaction_log

# Dùng dumpsys để xem service
adb shell dumpsys window          # WindowManagerService
adb shell dumpsys activity        # ActivityManagerService
adb shell dumpsys meminfo         # Memory per process

# Trace Binder transactions
adb shell atrace binder_driver binder_lock
```

---

## 11. Binder trong Kernel — Key Constants

```c
/* drivers/android/binder.c */
#define BINDER_SET_MAX_THREADS    _IOW('b', 5, __u32)
#define BINDER_VERSION            _IOWR('b', 9, struct binder_version)

/* Transaction commands (BC_* = client → driver) */
#define BC_TRANSACTION            _IOW('c', 0, struct binder_transaction_data)
#define BC_REPLY                  _IOW('c', 1, struct binder_transaction_data)
#define BC_FREE_BUFFER            _IOW('c', 3, binder_uintptr_t)
#define BC_ACQUIRE                _IOW('c', 4, __u32)
#define BC_RELEASE                _IOW('c', 5, __u32)

/* Return commands (BR_* = driver → client) */
#define BR_TRANSACTION            _IOR('r', 2, struct binder_transaction_data)
#define BR_REPLY                  _IOR('r', 3, struct binder_transaction_data)
#define BR_DEAD_BINDER            _IOR('r', 10, binder_uintptr_t)
#define BR_FAILED_REPLY           _IOR('r', 14, int)
```

---

## 12. File quan trọng trong source

| File | Mô tả |
|------|-------|
| `kernel/common/drivers/android/binder.c` | Kernel driver (13000+ lines) |
| `kernel/common/drivers/android/binder_alloc.c` | Buffer allocator |
| `frameworks/native/libs/binder/BpBinder.cpp` | Client proxy |
| `frameworks/native/libs/binder/BBinder.cpp` | Server stub base |
| `frameworks/native/libs/binder/ProcessState.cpp` | Process init, mmap /dev/binder |
| `frameworks/native/libs/binder/IPCThreadState.cpp` | Per-thread IPC state, ioctl loop |
| `frameworks/native/cmds/servicemanager/` | ServiceManager daemon |
| `frameworks/base/core/java/android/os/Binder.java` | Java Binder |
