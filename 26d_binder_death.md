# Binder Death Notification — Cơ chế Full Chain

Khi một service process chết bất ngờ (crash, OOM kill), mọi client đang giữ reference đến service đó cần được thông báo. Binder death notification là cơ chế làm điều này một cách reliable, ngay cả khi process die đột ngột mà không có chance cleanup.

---

## 1. Tại sao death notification là hard problem

```
Client A                   Server B (crash)
  sp<IFoo> service → BpBinder(handle=5)
  service->doWork()        ← BC_TRANSACTION
                           [B process killed by OOM killer]
  // A đang blocked chờ BR_REPLY
  // B đã chết, không ai reply
  // A block mãi mãi? → phải có mechanism
```

Kernel phát hiện process die vì `task_struct` cleanup → kernel có thể iterate tất cả `binder_ref` pointing đến nodes trong process đó và notify clients.

---

## 2. Chuỗi đăng ký — linkToDeath()

**Java/Kotlin:**
```java
IBinder service = ServiceManager.getService("foo");
IBinder.DeathRecipient recipient = new IBinder.DeathRecipient() {
    @Override
    public void binderDied() {
        // Reconnect hoặc cleanup
        reconnectToService();
    }
};
service.linkToDeath(recipient, 0);
```

**Java → JNI → C++:**
```
android.os.Binder.linkToDeath()
  → android_os_BinderProxy_linkToDeath() [JNI]
  → BpBinder::linkToDeath(recipient, cookie=JavaDeathRecipient)
  → IPCThreadState::requestDeathNotification(handle, proxy)
```

**C++ BpBinder::linkToDeath():**
```cpp
// frameworks/native/libs/binder/BpBinder.cpp
status_t BpBinder::linkToDeath(
        const sp<DeathRecipient>& recipient, void* cookie, uint32_t flags)
{
    // Store recipient in mObituaries list
    Obituary ob;
    ob.recipient = recipient;
    ob.cookie    = cookie;
    ob.flags     = flags;

    {
        AutoMutex _l(mLock);
        if (!mObitsSent) {
            // Chưa nhận death notification → đăng ký
            if (!mObituaries) {
                mObituaries = new Vector<Obituary>;
                // Lần đầu: yêu cầu kernel theo dõi
                getWeakRefs()->incWeak(this);  // giữ weak ref
                IPCThreadState* self = IPCThreadState::self();
                self->requestDeathNotification(mHandle, this);
                self->flushCommands();         // flush ngay để kernel biết
            }
            mObituaries->push(ob);
        } else {
            // Server đã chết rồi → gọi binderDied() ngay lập tức
            recipient->binderDied(this);
        }
    }
}
```

**IPCThreadState::requestDeathNotification():**
```cpp
status_t IPCThreadState::requestDeathNotification(
        int32_t handle, BpBinder* proxy)
{
    mOut.writeInt32(BC_REQUEST_DEATH_NOTIFICATION);
    mOut.writeInt32((int32_t)handle);
    mOut.writePointer((uintptr_t)proxy);  // cookie = BpBinder pointer
    return NO_ERROR;
}
```

**Kernel nhận BC_REQUEST_DEATH_NOTIFICATION:**
```c
/* binder.c — kernel side */
static int binder_thread_write(..., BC_REQUEST_DEATH_NOTIFICATION) {
    __u32 handle;
    binder_uintptr_t cookie;  /* = BpBinder* */

    // Tìm binder_ref cho handle trong client process
    ref = binder_get_ref_olocked(proc, handle, false);

    // Cấp phát binder_ref_death struct
    death = kzalloc(sizeof(*death), GFP_KERNEL);
    death->cookie = cookie;  /* lưu BpBinder* */

    // Gắn death object vào ref
    ref->death = death;

    // Nếu node đã chết (race condition: server die trước khi register)
    if (ref->node->proc == NULL) {
        // Deliver death notification ngay
        binder_send_death(proc, thread, ref);
    }
}
```

---

## 3. Kernel phát hiện process die

Khi process B chết, kernel cleanup thông qua `binder_proc_dec_tmpref()` → `binder_deferred_func()`:

```c
/* Khi proc->is_dead = true và tmp_ref drops to 0 */
static void binder_deferred_release(struct binder_proc *proc)
{
    /* Iterate tất cả binder_nodes trong proc */
    while ((n = rb_first(&proc->nodes)) != NULL) {
        struct binder_node *node = rb_entry(n, ...);
        node->proc = NULL;  /* mark node as dead */

        /* Với mỗi binder_ref pointing đến node này */
        hlist_for_each_entry_safe(ref, ..., &node->refs, ...) {
            if (ref->death) {
                /* Queue death notification đến client */
                binder_send_death(ref->proc, NULL, ref);
            }
        }
    }
    /* ...cleanup allocations, threads... */
}

static void binder_send_death(struct binder_proc *client_proc,
                               struct binder_thread *client_thread,
                               struct binder_ref *ref)
{
    struct binder_ref_death *death = ref->death;

    /* Enqueue BR_DEAD_BINDER work vào client process */
    death->work.type = BINDER_WORK_DEAD_BINDER;
    binder_enqueue_work(client_proc, &death->work);
    /* Wake up client's binder thread */
    wake_up_interruptible(&client_proc->wait);
}
```

---

## 4. Client nhận BR_DEAD_BINDER

**Kernel → client process:**
```
Client thread blocked trong talkWithDriver()
  ← kernel deliver BR_DEAD_BINDER (cookie = BpBinder*)
  ← ioctl returns
```

**executeCommand() xử lý BR_DEAD_BINDER:**
```cpp
case BR_DEAD_BINDER:
{
    BpBinder *proxy = (BpBinder*)mIn.readPointer();
    proxy->sendObituary();            // notify tất cả recipients

    mOut.writeInt32(BC_DEAD_BINDER_DONE);
    mOut.writePointer((uintptr_t)proxy);  // ack về kernel
}
break;
```

**Note quan trọng:** `BR_DEAD_BINDER` có thể đến trong `waitForResponse()` khi client đang chờ reply từ service khác. Case `default:` trong `waitForResponse()` gọi `executeCommand()`, xử lý death notification mid-wait rồi tiếp tục chờ reply từ service kia. Death notification không interrupt đợi transaction đang chạy.

---

## 5. BpBinder::sendObituary() — notify all recipients

```cpp
void BpBinder::sendObituary()
{
    mAlive = 0;

    Vector<Obituary>* obits = nullptr;
    {
        AutoMutex _l(mLock);
        if (!mObitsSent) {
            mObitsSent = 1;        // ngăn linkToDeath() mới gọi requestDeathNotification
            obits = mObituaries;   // lấy list, clear khỏi proxy
            mObituaries = nullptr;
        }
    }

    if (obits != nullptr) {
        const size_t N = obits->size();
        for (size_t i = 0; i < N; i++) {
            reportOneDeath(obits->itemAt(i));
        }
        delete obits;
    }
}

void BpBinder::reportOneDeath(const Obituary& obit)
{
    sp<DeathRecipient> recipient = obit.recipient.promote();
    if (recipient == nullptr) return;
    recipient->binderDied(this);
}
```

---

## 6. BC_DEAD_BINDER_DONE → kernel cleanup

Sau khi client xử lý death:

```cpp
mOut.writeInt32(BC_DEAD_BINDER_DONE);
mOut.writePointer((uintptr_t)proxy);
```

**Kernel nhận BC_DEAD_BINDER_DONE:**
```c
static int binder_thread_write(..., BC_DEAD_BINDER_DONE) {
    binder_uintptr_t cookie;
    /* Tìm death object với cookie này trong delivered_death list */
    struct binder_ref_death *death = ...;

    if (death->work.type == BINDER_WORK_DEAD_BINDER_AND_CLEAR) {
        /* Client đã gọi unlinkToDeath trước khi death arrived */
        /* Hoàn tất clear operation */
        death->work.type = BINDER_WORK_CLEAR_DEATH_NOTIFICATION;
        binder_enqueue_work(proc, &death->work);
    }
    /* Free death object nếu không còn cần */
}
```

---

## 7. unlinkToDeath() — hủy đăng ký

```java
service.unlinkToDeath(recipient, 0);
```

**Kernel nhận BC_CLEAR_DEATH_NOTIFICATION:**
```c
/* Nếu server chưa chết: */
/*   xóa ref->death, free binder_ref_death */
/*   send BR_CLEAR_DEATH_NOTIFICATION_DONE về client */

/* Nếu server đã chết và BR_DEAD_BINDER đang pending: */
/*   đổi work.type thành BINDER_WORK_DEAD_BINDER_AND_CLEAR */
/*   client sẽ nhận death notification rồi sau đó clear */
```

**Client nhận BR_CLEAR_DEATH_NOTIFICATION_DONE:**
```cpp
case BR_CLEAR_DEATH_NOTIFICATION_DONE:
{
    BpBinder *proxy = (BpBinder*)mIn.readPointer();
    proxy->getWeakRefs()->decWeak(proxy);
    /* release weak ref được giữ khi đăng ký linkToDeath */
}
break;
```

---

## 8. Race condition: server die trước linkToDeath()

```
t=0: Client getService("foo") → BpBinder(handle=5)
t=1: Server B crash
t=2: Client linkToDeath(recipient)
```

Trong `BpBinder::linkToDeath()`:
```cpp
self->requestDeathNotification(mHandle, this);
self->flushCommands();
// BC_REQUEST_DEATH_NOTIFICATION → kernel
// Kernel: ref->node->proc == NULL (server đã chết)
//       → deliver death ngay → BR_DEAD_BINDER
// Client thread nhận ngay trong flush hoặc next talkWithDriver()
```

Trong `linkToDeath()` sau khi requestDeathNotification() và flush, nếu `mObitsSent == 1` → gọi `recipient->binderDied()` ngay lập tức. Race này được handle đúng.

---

## 9. Threading model của death notification

Death notification được deliver trên **binder thread** của client — bất kỳ thread nào đang blocked trong `talkWithDriver()` khi `BR_DEAD_BINDER` arrives.

**Implication:** `binderDied()` callback chạy trên binder thread pool thread, không phải main thread. Nếu callback cần update UI, phải dispatch về main thread.

```java
@Override
public void binderDied() {
    // Đây là binder thread — KHÔNG touch UI trực tiếp
    mainHandler.post(() -> {
        // Safe to update UI here
        showReconnectDialog();
    });
}
```

**Deadlock risk:** Nếu `binderDied()` gọi synchronous Binder transaction đến service vừa chết → `BR_DEAD_REPLY` ngay. An toàn. Nhưng nếu gọi vào service khác → có thể block binder thread pool nếu pool đã đầy.

---

## 10. Full sequence diagram

```
Client (A)              Kernel                  Server (B)
────────────────────────────────────────────────────────────────────
linkToDeath(r)
  BC_REQUEST_DEATH_NOTIFICATION ──►
  (cookie=BpBinder*, handle=5)     create binder_ref_death
                                   ref->death = death_obj

                                                    [crash / OOM kill]
                                                    task cleanup
                                   binder_deferred_release()
                                     node->proc = NULL
                                     death->work = DEAD_BINDER
                                     enqueue death work
                                     wake_up client thread
talkWithDriver() returns
  mIn: BR_DEAD_BINDER (cookie=BpBinder*)
executeCommand(BR_DEAD_BINDER)
  proxy->sendObituary()
    → recipient->binderDied()  [callback!]
  BC_DEAD_BINDER_DONE ──────────►
  (cookie=BpBinder*)             cleanup death_obj
  proxy->getWeakRefs()->decWeak()
```

---

## 11. Debug death notification

```bash
# Xem death registrations
adb shell cat /proc/binder/proc/$(pidof system_server) | grep "death"
# ref 3: desc 3 node 42 s 1 w 1 d 0x7f3a4b00  ← d = death cookie (non-zero = registered)

# Theo dõi khi process die
adb shell "echo 1 > /sys/kernel/debug/tracing/events/binder/binder_dead_binder_send/enable"
adb shell cat /sys/kernel/debug/tracing/trace

# Simulate service crash để test death notification
adb shell kill -9 $(pidof com.example.myservice)
# → logcat sẽ show binderDied() callback

# Xem BR_DEAD_BINDER trong transaction log
adb shell cat /proc/binder/transaction_log | grep "DEAD"
```
