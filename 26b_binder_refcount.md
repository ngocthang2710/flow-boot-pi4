# Binder Reference Counting — Deep Dive

Đây là phần tinh tế nhất của Binder. Kernel và userspace phối hợp để duy trì strong/weak reference counting xuyên process boundaries, đảm bảo object không bị free khi vẫn còn client giữ reference.

---

## 1. Tại sao cần cross-process refcount?

Trong cùng một process, `sp<T>` (strong pointer) tự động quản lý lifetime qua `RefBase`. Nhưng xuyên process:

```
Client Process A          Kernel           Server Process B
  BpBinder(handle=3)  ←──────────────────  BBinder* (addr=0x7f...)
  sp<IBinder> ref         binder_ref         binder_node
```

- Client giữ `BpBinder` với handle number (integer)
- Kernel giữ `binder_ref` → `binder_node` → `BBinder*` trong server
- **Vấn đề:** Nếu client destroy BpBinder mà không báo kernel, kernel vẫn giữ `binder_node`, server không bao giờ biết để free BBinder

Cross-process refcount giải quyết: mỗi `BpBinder` tồn tại ↔ kernel increment ref trên node ↔ kernel báo server giữ strong ref.

---

## 2. Bốn lệnh refcount BC_*/BR_*

```
Direction      Command          Meaning
──────────────────────────────────────────────────────
Client→Kernel  BC_INCREFS       client muốn tăng WEAK ref trên handle
Client→Kernel  BC_ACQUIRE       client muốn tăng STRONG ref trên handle
Client→Kernel  BC_DECREFS       client release WEAK ref
Client→Kernel  BC_RELEASE       client release STRONG ref

Kernel→Server  BR_INCREFS       kernel yêu cầu server tăng weak ref trên BBinder
Kernel→Server  BR_ACQUIRE       kernel yêu cầu server tăng strong ref trên BBinder
Kernel→Server  BR_RELEASE       kernel yêu cầu server release strong ref
Kernel→Server  BR_DECREFS       kernel yêu cầu server release weak ref

Server→Kernel  BC_INCREFS_DONE  server xác nhận đã tăng weak ref
Server→Kernel  BC_ACQUIRE_DONE  server xác nhận đã tăng strong ref
```

**Protocol invariant:** Kernel chỉ send `BR_ACQUIRE` đến server SAU KHI nhận `BC_ACQUIRE` từ client, và chỉ send `BR_RELEASE` khi tất cả clients release strong ref.

---

## 3. Vòng đời BpBinder — client side

### 3.1 BpBinder được tạo (ProcessState::getStrongProxyForHandle)

```cpp
// ProcessState.cpp::getStrongProxyForHandle()
sp<IBinder> ProcessState::getStrongProxyForHandle(int32_t handle)
{
    handle_entry* e = lookupHandleLocked(handle);

    if (e != nullptr) {
        IBinder* b = e->binder;
        if (b == nullptr || !e->refs->attemptIncWeak(this)) {
            // Lần đầu: tạo BpBinder mới
            // Không BC_ACQUIRE ngay — BpBinder constructor sẽ gọi
            // incWeakHandle() / incStrongHandle() khi cần
            BpBinder *b = BpBinder::create(handle);
            e->binder = b;
            e->refs = b->getWeakRefs();
        }
        // ...
    }
}
```

### 3.2 incStrongHandle() — BC_ACQUIRE với temp reference

```cpp
// IPCThreadState.cpp::incStrongHandle()
void IPCThreadState::incStrongHandle(int32_t handle, BpBinder *proxy)
{
    mOut.writeInt32(BC_ACQUIRE);
    mOut.writeInt32(handle);

    if (!flushIfNeeded()) {
        // Chưa flush ngay → BC_ACQUIRE chưa tới kernel
        // Nếu proxy bị destroy trước khi flush → kernel không biết
        // → Giữ 1 temp strong ref để đảm bảo proxy sống đến khi flush
        proxy->incStrong(mProcess.get());
        mPostWriteStrongDerefs.push(proxy);
        // processPostWriteDerefs() sẽ gọi proxy->decStrong() SAU khi flush
    }
}
```

**Tại sao cần temp reference?**

Timeline nguy hiểm nếu không có temp ref:
```
t=0: BC_ACQUIRE viết vào mOut
t=1: Caller drop sp<BpBinder> → refcount=0 → ~BpBinder() gọi BC_RELEASE
t=2: talkWithDriver() flush: kernel nhận BC_RELEASE trước BC_ACQUIRE
     → kernel: invalid state, crash
```

Temp reference đảm bảo `proxy` sống ít nhất đến khi `talkWithDriver()` flush xong `mOut`.

### 3.3 decStrongHandle() — BC_RELEASE

```cpp
void IPCThreadState::decStrongHandle(int32_t handle)
{
    mOut.writeInt32(BC_RELEASE);
    mOut.writeInt32(handle);
    flushIfNeeded();
    // Không cần temp ref: chỉ cần BC_RELEASE tới kernel
    // không có object nào bị ảnh hưởng từ phía client
}
```

---

## 4. BR_ACQUIRE / BR_RELEASE — server side

### 4.1 BR_ACQUIRE: đồng bộ, phải reply ngay

```cpp
// executeCommand() trong IPCThreadState.cpp
case BR_ACQUIRE:
    refs = (RefBase::weakref_type*)mIn.readPointer();
    obj = (BBinder*)mIn.readPointer();

    ALOG_ASSERT(refs->refBase() == obj, ...);

    obj->incStrong(mProcess.get());   // Tăng strong ref của BBinder ngay lập tức

    mOut.writeInt32(BC_ACQUIRE_DONE); // Phải reply đồng bộ
    mOut.writePointer((uintptr_t)refs);
    mOut.writePointer((uintptr_t)obj);
    break;
```

**Tại sao phải reply ngay?** Kernel đang hold `binder_node` với `pending_strong_ref = 1`. Kernel không thể gửi thêm `BR_ACQUIRE` cho cùng node cho đến khi nhận `BC_ACQUIRE_DONE`. Nếu server crash trước khi reply → kernel cleanup node.

### 4.2 BR_RELEASE: defer, KHÔNG gọi decStrong() ngay

```cpp
case BR_RELEASE:
    refs = (RefBase::weakref_type*)mIn.readPointer();
    obj = (BBinder*)mIn.readPointer();

    // KHÔNG gọi obj->decStrong() ở đây
    mPendingStrongDerefs.push(obj);
    // processPendingDerefs() sẽ xử lý sau khi drain mIn
    break;
```

**Tại sao defer?**

`decStrong()` có thể trigger destructor của `BBinder`. Destructor có thể:
1. Gọi outgoing Binder transaction (ví dụ notify cleanup)
2. Transaction đó cần `talkWithDriver()`, modify `mIn`/`mOut`
3. Nếu đang trong giữa `executeCommand()` switch → state corruption

Deferring đến sau khi drain `mIn` đảm bảo không có re-entrancy.

### 4.3 BR_INCREFS / BR_DECREFS: weak ref

```cpp
case BR_INCREFS:
    refs = (RefBase::weakref_type*)mIn.readPointer();
    obj = (BBinder*)mIn.readPointer();
    refs->incWeak(mProcess.get());    // tăng weak ref
    mOut.writeInt32(BC_INCREFS_DONE); // reply đồng bộ
    mOut.writePointer((uintptr_t)refs);
    mOut.writePointer((uintptr_t)obj);
    break;

case BR_DECREFS:
    refs = (RefBase::weakref_type*)mIn.readPointer();
    obj = (BBinder*)mIn.readPointer(); // consume pointer (discard)
    // NOTE: obj có thể đã bị free rồi (ptr không còn valid)
    // Đó là lý do tại sao không có assertion ở đây
    mPendingWeakDerefs.push(refs);    // defer decWeak()
    break;
```

`BR_DECREFS` không có assertion vì `BBinder` có thể đã bị destroy (strong ref = 0) trước khi weak ref về 0. Lúc đó `obj` pointer là garbage, chỉ `refs` (weakref_type) vẫn valid cho đến khi weak ref = 0.

---

## 5. processPendingDerefs() — thứ tự quan trọng

```cpp
// IPCThreadState.cpp::processPendingDerefs()
void IPCThreadState::processPendingDerefs()
{
    if (mIn.dataPosition() >= mIn.dataSize()) {
        // Chỉ chạy khi mIn đã được drain hoàn toàn
        // → Không re-enter giữa chừng executeCommand() loop

        while (mPendingWeakDerefs.size() > 0 || mPendingStrongDerefs.size() > 0) {
            // Xử lý weak trước — vì decStrong() có thể add thêm vào
            // mPendingWeakDerefs qua destructor
            while (mPendingWeakDerefs.size() > 0) {
                RefBase::weakref_type* refs = mPendingWeakDerefs[0];
                mPendingWeakDerefs.removeAt(0);
                refs->decWeak(mProcess.get());
            }

            if (mPendingStrongDerefs.size() > 0) {
                // Chỉ xử lý 1 strong deref mỗi iteration outer loop
                // vì decStrong() có thể queue thêm weakDeref
                // → outer loop sẽ drain weak trước strong tiếp theo
                BBinder* obj = mPendingStrongDerefs[0];
                mPendingStrongDerefs.removeAt(0);
                obj->decStrong(mProcess.get());
            }
        }
    }
}
```

**Tại sao outer loop?**

```
Scenario:
  decStrong(A) → ~A() → hủy member sp<B>
  → B->decStrong() thêm B vào mPendingStrongDerefs
  → B's destructor cần decWeak(C) thêm C vào mPendingWeakDerefs
  → ...
```

Nếu chỉ dùng inner loop, C sẽ không được xử lý. Outer loop tiếp tục đến khi cả hai lists rỗng.

---

## 6. processPostWriteDerefs() — sau khi flush

```cpp
// IPCThreadState.cpp::processPostWriteDerefs()
void IPCThreadState::processPostWriteDerefs()
{
    mIsProcessingPostWriteDerefs = true;

    // Xử lý weak derefs trước
    for (size_t i = 0; i < mPostWriteWeakDerefs.size(); i++) {
        RefBase::weakref_type* refs = mPostWriteWeakDerefs[i];
        refs->decWeak(mProcess.get());
    }
    mPostWriteWeakDerefs.clear();

    // Rồi strong derefs
    for (size_t i = 0; i < mPostWriteStrongDerefs.size(); i++) {
        RefBase* obj = mPostWriteStrongDerefs[i];
        obj->decStrong(mProcess.get());
    }
    mPostWriteStrongDerefs.clear();

    mIsProcessingPostWriteDerefs = false;
}
```

**Hai list riêng biệt: pendingDerefs vs postWriteDerefs**

| List | Thời điểm populate | Thời điểm drain | Source |
|------|-------------------|-----------------|--------|
| `mPendingStrongDerefs` | `executeCommand(BR_RELEASE)` | Sau khi drain `mIn` | Server nhận BR_RELEASE |
| `mPendingWeakDerefs` | `executeCommand(BR_DECREFS)` | Sau khi drain `mIn` | Server nhận BR_DECREFS |
| `mPostWriteStrongDerefs` | `incStrongHandle()` temp ref | Sau khi kernel consume `mOut` | Client BC_ACQUIRE temp |
| `mPostWriteWeakDerefs` | `incWeakHandle()` temp ref | Sau khi kernel consume `mOut` | Client BC_INCREFS temp |

---

## 7. Kernel side — binder_node refcount states

Kernel track 4 trạng thái trong `binder_node`:

```c
struct binder_node {
    int internal_strong_refs;  // strong refs từ các process khác (BC_ACQUIRE count)
    int local_weak_refs;       // weak refs giữ bởi kernel khi có pending work
    int local_strong_refs;     // strong refs giữ bởi kernel (transaction in-flight)
    unsigned has_strong_ref:1;   // đã notify server (BR_ACQUIRE) chưa
    unsigned pending_strong_ref:1; // đang chờ server reply BC_ACQUIRE_DONE
    unsigned has_weak_ref:1;
    unsigned pending_weak_ref:1;
};
```

**State machine của has_strong_ref:**

```
internal_strong_refs: 0 → 1       has_strong_ref: 0 → 1
  └─ Kernel send BR_ACQUIRE          pending_strong_ref = 1
     └─ Server reply BC_ACQUIRE_DONE  pending_strong_ref = 0

internal_strong_refs: 1 → 0       has_strong_ref: 1 → 0
  └─ Kernel send BR_RELEASE          pending_strong_ref = 1
     └─ Server ack (implicit)         pending_strong_ref = 0
```

Nếu `internal_strong_refs` tăng từ 0 → 1 trong khi `pending_strong_ref = 1` (đang chờ ack), kernel không gửi `BR_ACQUIRE` thứ hai mà chỉ increment counter. Server chỉ nhận 1 `BR_ACQUIRE` cho mỗi "0→1 transition".

---

## 8. Deadlock scenario và cách tránh

**Scenario nguy hiểm:**

```
Client A                    Server B
─────────────────────────────────────
  sp<IFoo> foo = ...
  foo->bar()               ← BC_TRANSACTION
  (waiting for BR_REPLY)
                            onTransact() {
                              // cần lấy IBar từ A
                              sp<IBar> bar = A->getBar(); ← BC_TRANSACTION vào A
                            }
  // A đang blocked chờ B ← A không thể serve B's request
  // → DEADLOCK
```

Binder không tự detect deadlock này ở userspace. Nhưng kernel có **transaction stack**: khi B gọi vào A và A đang chờ B reply, kernel có thể detect cycle qua `binder_transaction::from` chain. Nếu phát hiện cycle → trả về `BR_FAILED_REPLY`.

**Phòng tránh:**
- Dùng `oneway` cho callbacks khi có thể
- Không gọi synchronous Binder trong `onTransact()` back vào caller

---

## 9. Quan sát refcount trực tiếp

```bash
# Xem node với strong/weak ref count
adb shell cat /proc/binder/proc/$(pidof system_server) | grep "node "
# node 42: u0000007b2c... c0000007b... hs 1 hw 1 ls 0 lw 0 is 2 iw 3 tr 0
#   hs=has_strong_ref, hw=has_weak_ref
#   ls=local_strong_refs, lw=local_weak_refs
#   is=internal_strong_refs, iw=internal_weak_refs

# Xem references từ process khác
adb shell cat /proc/binder/proc/$(pidof system_server) | grep "ref "
# ref 5: desc 5 node 42 s 1 w 1 d 0000...
#   s=strong, w=weak, d=death cookie

# Theo dõi BC_ACQUIRE/BR_ACQUIRE qua ftrace
adb shell "echo 1 > /sys/kernel/debug/tracing/events/binder/binder_transaction/enable"
adb shell cat /sys/kernel/debug/tracing/trace | grep "ACQUIRE"
```
