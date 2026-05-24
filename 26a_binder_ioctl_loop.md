# IPCThreadState — Deep Dive: Cơ chế ioctl Loop

> File này phân tích chi tiết userspace engine của libbinder. Không lặp lại overview, không có AIDL, không có ServiceManager — chỉ tập trung vào cơ chế bên trong IPCThreadState.

---

## Section 1: `binder_write_read` — Giao thức song công trong một ioctl

Mọi tương tác với kernel đều đi qua một struct duy nhất:

```c
// kernel uapi: linux/android/binder.h
struct binder_write_read {
    binder_size_t    write_size;      // số byte cần ghi từ userspace xuống kernel
    binder_size_t    write_consumed;  // kernel đã đọc bao nhiêu (output)
    binder_uintptr_t write_buffer;    // con trỏ tới buffer chứa BC_* commands
    binder_size_t    read_size;       // kích thước buffer userspace cấp cho kernel ghi vào
    binder_size_t    read_consumed;   // kernel đã ghi bao nhiêu BR_* commands (output)
    binder_uintptr_t read_buffer;     // con trỏ tới buffer nhận BR_* commands
};
```

Trong `IPCThreadState`, hai Parcel tương ứng với hai buffer này:

- **`mOut`** (write buffer): accumulate các BC_* commands trước khi gọi ioctl. Thread ghi vào đây bằng `mOut.writeInt32(cmd)`, `mOut.write(&tr, sizeof(tr))`, v.v. Sau khi driver consume hết, `mOut.setDataSize(0)` reset về rỗng.
- **`mIn`** (read buffer): kernel ghi BR_* commands vào đây. Thread đọc từng command ra bằng `mIn.readInt32()`. Quan trọng: `mIn.dataPosition()` là con trỏ đọc — khi nó đuổi kịp `mIn.dataSize()`, buffer được coi là đã drain xong.

**Tại sao thiết kế song công?** Thay vì hai ioctl riêng (một để write, một để read), một ioctl `BINDER_WRITE_READ` thực hiện cả hai. Điều này giảm số lần context-switch kernel↔userspace xuống một nửa cho mỗi round-trip. Kernel sẽ xử lý toàn bộ write buffer trước, sau đó mới điền vào read buffer và wake up thread bên kia.

**Logic điều phối write/read:**

```cpp
// talkWithDriver(), line 1277-1294
const bool needRead = mIn.dataPosition() >= mIn.dataSize();
// needRead = true  → mIn đã drain, cần đọc thêm từ driver
// needRead = false → vẫn còn data trong mIn chưa xử lý hết

const size_t outAvail = (!doReceive || needRead) ? mOut.dataSize() : 0;
// Không write nếu:
//   doReceive = true  AND  needRead = false
// Tức là: nếu caller muốn nhận data VÀ mIn vẫn còn dữ liệu,
// thì không gửi thêm command mới — drain mIn trước đã.

bwr.write_size = outAvail;       // 0 hoặc mOut.dataSize()
if (doReceive && needRead) {
    bwr.read_size = mIn.dataCapacity();
    bwr.read_buffer = (uintptr_t)mIn.data();
} else {
    bwr.read_size = 0;
    bwr.read_buffer = 0;
    // Không yêu cầu read → driver sẽ không block thread này
}
```

Nếu cả `write_size == 0` lẫn `read_size == 0` → hàm return `NO_ERROR` ngay, không ioctl. Đây là guard tránh syscall thừa.

---

## Section 2: `talkWithDriver()` — Phân tích từng bước

```cpp
// IPCThreadState.cpp, line 1268-1385
status_t IPCThreadState::talkWithDriver(bool doReceive)
{
    if (mProcess->mDriverFD < 0) return -EBADF;

    binder_write_read bwr;
    const bool needRead = mIn.dataPosition() >= mIn.dataSize();
    const size_t outAvail = (!doReceive || needRead) ? mOut.dataSize() : 0;

    bwr.write_size   = outAvail;
    bwr.write_buffer = (uintptr_t)mOut.data();

    if (doReceive && needRead) {
        bwr.read_size   = mIn.dataCapacity();
        bwr.read_buffer = (uintptr_t)mIn.data();
    } else {
        bwr.read_size = 0; bwr.read_buffer = 0;
    }

    if ((bwr.write_size == 0) && (bwr.read_size == 0)) return NO_ERROR;

    bwr.write_consumed = 0;
    bwr.read_consumed  = 0;

    // Vòng retry EINTR
    do {
        if (ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr) >= 0)
            err = NO_ERROR;
        else
            err = -errno;
    } while (err == -EINTR);   // ← tại sao?
```

**Vòng `do...while(err == -EINTR)`**: Khi một signal (ví dụ SIGCHLD, SIGTERM) delivered tới thread đang block trong `ioctl()`, kernel trả về `-EINTR` và hủy syscall. Binder không có cơ chế partial-completion cho write buffer (driver guarantee: hoặc consume toàn bộ, hoặc không consume gì) — do đó retry là an toàn. Không retry sẽ làm mất BC_* commands trong `mOut` khi bị signal.

```cpp
    // Sau ioctl thành công:
    if (err >= NO_ERROR) {
        if (bwr.write_consumed > 0) {
            if (bwr.write_consumed < mOut.dataSize()) {
                // Driver chỉ consume một phần write buffer → BUG nghiêm trọng
                LOG_ALWAYS_FATAL("Driver did not consume write buffer. "
                                 "err: %s consumed: %zu of %zu.",
                                 statusToString(err).c_str(),
                                 (size_t)bwr.write_consumed, mOut.dataSize());
            } else {
                mOut.setDataSize(0);       // ← reset write buffer
                processPostWriteDerefs();  // ← SAU KHI driver consume xong
            }
        }
        if (bwr.read_consumed > 0) {
            mIn.setDataSize(bwr.read_consumed);
            mIn.setDataPosition(0);   // reset read pointer để parse từ đầu
        }
    }
```

**Tại sao `processPostWriteDerefs()` chỉ được gọi SAU khi driver consume xong?**

Khi userspace gọi `incStrongHandle()` (tức là gửi `BC_ACQUIRE` cho handle nào đó), nó tạm thời `incStrong()` trên proxy object rồi push vào `mPostWriteStrongDerefs`. Mục đích: giữ ref count > 0 trong khi command còn nằm trong `mOut` chưa được kernel xử lý. Nếu `decStrong()` trước khi driver xử lý `BC_ACQUIRE`, ref count có thể về 0 → destructor chạy → object bị free → kernel sau đó nhận `BC_ACQUIRE` với pointer/handle không hợp lệ. Chỉ sau khi `write_consumed == mOut.dataSize()` (kernel đã nhận command), lúc đó mới an toàn để drop ref tạm thời.

---

## Section 3: `writeTransactionData()` — Đóng gói Parcel thành BC_TRANSACTION

```cpp
// IPCThreadState.cpp, line 1387-1421
status_t IPCThreadState::writeTransactionData(int32_t cmd, uint32_t binderFlags,
    int32_t handle, uint32_t code, const Parcel& data, status_t* statusBuffer)
{
    binder_transaction_data tr;

    tr.target.ptr = 0;  // ← không truyền uninitialized stack data lên process khác
    tr.target.handle = handle;
    tr.code  = code;
    tr.flags = binderFlags;
    tr.cookie = 0;
    tr.sender_pid  = 0;   // kernel tự fill, userspace không được trust field này
    tr.sender_euid = 0;

    const status_t err = data.errorCheck();
    if (err == NO_ERROR) {
        // Path bình thường: trỏ thẳng vào Parcel internal buffer
        tr.data_size         = data.ipcDataSize();
        tr.data.ptr.buffer   = data.ipcData();
        tr.offsets_size      = data.ipcObjectsCount() * sizeof(binder_size_t);
        tr.data.ptr.offsets  = data.ipcObjects();
    } else if (statusBuffer) {
        // Path lỗi: Parcel bị corrupt hoặc có lỗi serialization
        tr.flags |= TF_STATUS_CODE;       // báo cho receiver biết đây là error code
        *statusBuffer = err;
        tr.data_size        = sizeof(status_t);
        tr.data.ptr.buffer  = reinterpret_cast<uintptr_t>(statusBuffer);
        tr.offsets_size     = 0;
        tr.data.ptr.offsets = 0;
    } else {
        return (mLastError = err);        // không có statusBuffer → propagate error
    }

    mOut.writeInt32(cmd);     // BC_TRANSACTION hoặc BC_REPLY
    mOut.write(&tr, sizeof(tr));
    return NO_ERROR;
}
```

**Tại sao `tr.target.ptr = 0` trước?**

`binder_transaction_data` là struct trên stack. Nếu không zero-initialize `tr.target.ptr`, các byte garbage từ stack frame trước sẽ nằm trong union `target` và được serialize vào kernel buffer, sau đó có thể leak sang process khác (information disclosure). Khi gọi từ phía client, chỉ `target.handle` được dùng (handle là integer index trong kernel); `target.ptr` (dành cho server-side BBinder pointer) không cần thiết, nhưng phải được zero tường minh để tránh leak.

**`ipcObjectsCount()` và offsets array:**

Khi Parcel chứa binder objects (IBinder*, file descriptor), mỗi object được serialize thành `flat_binder_object` struct tại một offset nào đó trong data buffer. `ipcObjects()` trả về array các offset đó. Kernel đọc offsets array để biết vị trí của từng `flat_binder_object` trong buffer, sau đó dịch chuyển handle/pointer phù hợp khi copy sang address space của receiver.

---

## Section 4: `waitForResponse()` — Vòng lặp xử lý phản hồi

Đây là trái tim của synchronous transaction. Sau khi `writeTransactionData()` nhét command vào `mOut`, thread client gọi `waitForResponse()` và spin ở đây cho đến khi có kết quả.

```cpp
// IPCThreadState.cpp, line 1163-1266
status_t IPCThreadState::waitForResponse(Parcel *reply, status_t *acquireResult)
{
    uint32_t cmd;
    int32_t err;

    while (1) {
        if ((err=talkWithDriver()) < NO_ERROR) break;
        err = mIn.errorCheck();
        if (err < NO_ERROR) break;
        if (mIn.dataAvail() == 0) continue;  // ioctl trả về nhưng read_consumed = 0

        cmd = (uint32_t)mIn.readInt32();

        switch (cmd) {
        case BR_ONEWAY_SPAM_SUSPECT:
            // Kernel phát hiện process gửi quá nhiều oneway calls → log callstack
            ALOGE("Process seems to be sending too many oneway calls.");
            CallStack::logStack("oneway spamming", ...);
            [[fallthrough]];              // ← rơi xuống BR_TRANSACTION_COMPLETE

        case BR_TRANSACTION_COMPLETE:
            if (!reply && !acquireResult) goto finish;
            // reply == nullptr → đây là oneway transaction, không cần chờ BR_REPLY
            // reply != nullptr → đây là two-way, tiếp tục vòng lặp chờ BR_REPLY
            break;
```

**`BR_TRANSACTION_COMPLETE` xuất hiện lúc nào?**
Kernel gửi `BR_TRANSACTION_COMPLETE` ngay khi nó đã enqueue transaction vào work queue của server thread — tức là **trước** khi server bắt đầu xử lý. Với oneway transaction (`reply == nullptr`), đây là tín hiệu hoàn thành; client có thể tiếp tục. Với two-way, đây chỉ là ACK "đã nhận, đang xử lý" — client phải tiếp tục loop chờ `BR_REPLY`.

```cpp
        case BR_DEAD_REPLY:
            err = DEAD_OBJECT;     // server process đã chết
            goto finish;

        case BR_FAILED_REPLY:
            err = FAILED_TRANSACTION;  // kernel không thể allocate buffer, v.v.
            goto finish;

        case BR_FROZEN_REPLY:
            ALOGW("Transaction failed because process frozen.");
            err = FAILED_TRANSACTION;  // server bị freeze (Android App Hibernate)
            goto finish;

        case BR_TRANSACTION_PENDING_FROZEN:
            ALOGW("Sending oneway calls to frozen process.");
            goto finish;  // oneway tới process frozen → drop silently

        case BR_REPLY:
            {
                binder_transaction_data tr;
                err = mIn.read(&tr, sizeof(tr));

                if (reply) {
                    if ((tr.flags & TF_STATUS_CODE) == 0) {
                        // Zero-copy: Parcel KHÔNG copy dữ liệu,
                        // chỉ lưu pointer vào shared buffer của kernel
                        reply->ipcSetDataReference(
                            reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),
                            tr.data_size,
                            reinterpret_cast<const binder_size_t*>(tr.data.ptr.offsets),
                            tr.offsets_size / sizeof(binder_size_t),
                            freeBuffer);   // callback giải phóng kernel buffer
                    } else {
                        // Server gửi error status_t thay vì data
                        err = *reinterpret_cast<const status_t*>(tr.data.ptr.buffer);
                        freeBuffer(...);   // giải phóng ngay vì không dùng data
                    }
                }
            }
            goto finish;

        default:
            // BR_ACQUIRE, BR_RELEASE, BR_DEAD_BINDER, BR_SPAWN_LOOPER, v.v.
            // Được xử lý TRONG KHI đang chờ reply!
            err = executeCommand(cmd);
            if (err != NO_ERROR) goto finish;
            break;
        }
    }
```

**`default: executeCommand(cmd)` — tại sao cần xử lý BR_* khác khi đang chờ reply?**

Binder là fully reentrant. Trong khi client thread đang block chờ `BR_REPLY` từ server A, kernel có thể gửi về:
- `BR_ACQUIRE` / `BR_RELEASE`: kernel cần userspace đồng bộ ref count trước khi tiếp tục
- `BR_DEAD_BINDER`: một binder object trong process này đã death-notify
- `BR_SPAWN_LOOPER`: kernel yêu cầu process spawn thêm thread

Nếu không handle các command này, kernel có thể bị deadlock (chờ `BC_ACQUIRE_DONE` nhưng userspace không bao giờ gửi). Do đó waitForResponse phải dispatch các non-reply commands trong khi chờ.

---

## Section 5: `executeCommand()` — Dispatch từng BR_* command

### BR_ACQUIRE — Kernel tăng strong ref

```cpp
case BR_ACQUIRE:
    refs = (RefBase::weakref_type*)mIn.readPointer();
    obj  = (BBinder*)mIn.readPointer();
    // Kernel đang promote weak ref lên strong ref cho BBinder này
    obj->incStrong(mProcess.get());
    // Phải reply đồng bộ trước khi tiếp tục
    mOut.writeInt32(BC_ACQUIRE_DONE);
    mOut.writePointer((uintptr_t)refs);
    mOut.writePointer((uintptr_t)obj);
    break;
```

Kernel gửi `BR_ACQUIRE` khi một remote process lần đầu tiên giữ strong reference tới BBinder này. Kernel **block** cho đến khi nhận `BC_ACQUIRE_DONE` — nếu userspace không reply, kernel transaction sẽ stall. `incStrong()` được gọi ngay tại đây (không defer) vì kernel cần biết chắc ref count đã tăng trước khi tiếp tục route transaction.

### BR_RELEASE — Defer decStrong()

```cpp
case BR_RELEASE:
    refs = (RefBase::weakref_type*)mIn.readPointer();
    obj  = (BBinder*)mIn.readPointer();
    // KHÔNG gọi obj->decStrong() ở đây!
    mPendingStrongDerefs.push(obj);  // ← defer
    break;
```

**Tại sao defer?** `decStrong()` có thể trigger destructor của BBinder nếu đây là ref cuối cùng. Destructor có thể gọi code bất kỳ, bao gồm outgoing binder transactions. Nếu ta đang ở giữa `executeCommand()` (tức là đang trong quá trình parse `mIn`), một outgoing transaction sẽ gọi `waitForResponse()` → `talkWithDriver()` → ioctl → kernel có thể gửi thêm BR_* commands → `mIn` bị overwrite trong khi vẫn đang được parse từ outer frame. Race condition này được ngăn bằng cách defer destructor-triggering operations sang `processPendingDerefs()`.

### BR_INCREFS / BR_DECREFS — Weak ref counterpart

```cpp
case BR_INCREFS:
    refs->incWeak(mProcess.get());
    mOut.writeInt32(BC_INCREFS_DONE);   // reply đồng bộ, giống BR_ACQUIRE
    ...

case BR_DECREFS:
    mPendingWeakDerefs.push(refs);   // defer, giống BR_RELEASE
    break;
```

Tương tự pattern trên nhưng cho weak ref. `decWeak()` cũng có thể trigger destructor (khi cả strong và weak về 0), nên cũng phải defer.

### BR_TRANSACTION — Inbound call tới server

```cpp
case BR_TRANSACTION:
    {
        // ... đọc binder_transaction_data tr từ mIn ...

        // Lưu caller identity từ kernel (đã được kernel verify, không giả mạo được)
        const pid_t origPid = mCallingPid;
        const uid_t origUid = mCallingUid;
        const char* origSid = mCallingSid;       // SELinux security context

        // Ghi đè bằng caller's identity từ transaction
        mCallingPid = tr.sender_pid;
        mCallingUid = tr.sender_euid;
        mCallingSid = reinterpret_cast<const char*>(tr_secctx.secctx);

        // Gọi onTransact() — đây là code của developer
        error = doTransactBinder(binder, tr.code, buffer, &reply, tr.flags);

        if ((tr.flags & TF_ONE_WAY) == 0) {
            // b/238777741: PHẢI clear buffer TRƯỚC khi sendReply
            buffer.setDataSize(0);
            sendReply(reply, tr.flags & TF_CLEAR_BUF);
        } else {
            // Oneway: không reply, chỉ log nếu có lỗi
            if (error != OK) ALOGI("oneway function error: ...");
        }

        // Restore caller identity về thread owner's identity
        mCallingPid = origPid;
        mCallingUid = origUid;
        mCallingSid = origSid;
    }
```

**Tại sao `buffer.setDataSize(0)` trước `sendReply()`?** (b/238777741)

`buffer` là Parcel trỏ vào kernel-allocated shared memory. Sau khi server gọi `sendReply()`, kernel báo cho client biết reply đã sẵn sàng và client bắt đầu đọc. Nếu server chưa release `buffer` (tức là buffer reference count > 0), kernel không thể reclaim vùng nhớ đó. Nếu client lập tức gửi một transaction mới, kernel cần allocate buffer mới trong pool — nhưng pool có thể cạn nếu buffer cũ chưa được free. Gọi `buffer.setDataSize(0)` không giải phóng buffer, nhưng nó trigger `freeBuffer` callback sớm hơn, release kernel buffer trước khi client bắt đầu transaction tiếp theo.

**Security model của `mCallingPid`/`mCallingUid`**: Kernel điền `sender_pid` và `sender_euid` từ credentials của calling process — userspace không thể tự khai báo. Khi `onTransact()` gọi `IPCThreadState::getCallingUid()`, nó đọc `mCallingUid` đã được set từ kernel data. Việc restore về `origUid` sau khi transaction xong là critical: nếu không restore, thread sẽ "nghĩ" mình đang serve một caller với uid khác trong lần transaction tiếp theo.

### BR_DEAD_BINDER — Death notification

```cpp
case BR_DEAD_BINDER:
    {
        BpBinder *proxy = (BpBinder*)mIn.readPointer();
        proxy->sendObituary();   // gọi tất cả death recipients đã đăng ký
        mOut.writeInt32(BC_DEAD_BINDER_DONE);
        mOut.writePointer((uintptr_t)proxy);  // reply đồng bộ
    }
    break;
```

Cũng reply đồng bộ (`BC_DEAD_BINDER_DONE`) để kernel biết userspace đã xử lý notification xong.

---

## Section 6: `processPendingDerefs()` — Tại sao cần list riêng

```cpp
// IPCThreadState.cpp, line 782-814
void IPCThreadState::processPendingDerefs()
{
    // Chỉ chạy khi mIn đã drain hoàn toàn
    if (mIn.dataPosition() >= mIn.dataSize()) {

        // Outer loop: vì decStrong() có thể trigger destructor
        // → destructor gửi outgoing transaction
        // → waitForResponse() gọi talkWithDriver() thêm lần nữa
        // → kernel có thể gửi thêm BR_RELEASE/BR_DECREFS
        // → mPendingStrongDerefs/mPendingWeakDerefs lại có thêm items
        while (mPendingWeakDerefs.size() > 0 || mPendingStrongDerefs.size() > 0) {

            // Xử lý weak derefs TRƯỚC
            while (mPendingWeakDerefs.size() > 0) {
                RefBase::weakref_type* refs = mPendingWeakDerefs[0];
                mPendingWeakDerefs.removeAt(0);
                refs->decWeak(mProcess.get());
            }

            // Chỉ xử lý MỘT strong deref rồi quay lại outer loop
            // Lý do: decStrong() có thể trigger code thêm decWeak() vào list
            // Nếu dùng while() ở đây, weak deref mới sẽ bị delay
            // → vi phạm thứ tự weak-before-strong
            if (mPendingStrongDerefs.size() > 0) {
                BBinder* obj = mPendingStrongDerefs[0];
                mPendingStrongDerefs.removeAt(0);
                obj->decStrong(mProcess.get());
                // Sau decStrong(), outer while sẽ check lại mPendingWeakDerefs
                // trước khi xử lý strong deref tiếp theo
            }
        }
    }
}
```

**Tại sao điều kiện `mIn.dataPosition() >= mIn.dataSize()`?**

`processPendingDerefs()` không được chạy khi vẫn còn BR_* commands chưa xử lý trong `mIn`. Nếu `decStrong()` trigger destructor → outgoing transaction → `talkWithDriver()` → kernel ghi thêm BR_* commands vào buffer `mIn` (overwrite hoặc shift position), thì caller đang parse `mIn` ở frame ngoài sẽ đọc sai vị trí. Guard này đảm bảo `processPendingDerefs()` chỉ được gọi từ `joinThreadPool()` loop (sau `getAndExecuteCommand()`) hoặc `handlePolledCommands()` (sau khi drain hết `mIn`).

**Hai list riêng biệt:**

- `mPendingStrongDerefs` (Vector\<BBinder*\>): cho `decStrong()` — có thể trigger destructor
- `mPendingWeakDerefs` (Vector\<weakref_type*\>): cho `decWeak()` — ít nguy hiểm hơn nhưng vẫn cần thứ tự
- `mPostWriteStrongDerefs` / `mPostWriteWeakDerefs`: **khác hoàn toàn** — đây là temp refs giữ trong khi BC_ACQUIRE/BC_INCREFS chưa được driver consume (xem Section 2)

---

## Section 7: `joinThreadPool()` — Vòng đời thread

```cpp
// IPCThreadState.cpp, line 839-881
void IPCThreadState::joinThreadPool(bool isMain)
{
    mProcess->checkExpectingThreadPoolStart();
    mProcess->mCurrentThreads++;

    // Phân biệt main thread và spawned thread
    mOut.writeInt32(isMain ? BC_ENTER_LOOPER : BC_REGISTER_LOOPER);
    // BC_ENTER_LOOPER: main thread, tự nguyện vào pool
    // BC_REGISTER_LOOPER: spawned bởi ProcessState khi nhận BR_SPAWN_LOOPER
    // Kernel dùng info này để quản lý thread count và có thể gửi BR_SPAWN_LOOPER
    // cho main looper nhưng không cho spawned loopers (tránh spawn storm)

    mIsLooper = true;
    status_t result;
    do {
        processPendingDerefs();        // drain deferred refs trước mỗi iteration
        result = getAndExecuteCommand();  // block trong talkWithDriver() cho đến khi có BR_*

        if (result < NO_ERROR && result != TIMED_OUT
                && result != -ECONNREFUSED && result != -EBADF) {
            LOG_ALWAYS_FATAL("getAndExecuteCommand returned unexpected error");
        }

        // Spawned thread: thoát nếu idle timeout
        if (result == TIMED_OUT && !isMain) {
            break;   // ← chỉ spawned threads mới được thoát sớm
        }
        // Main thread không bao giờ thoát vì TIMED_OUT
        // Main thread chỉ thoát khi driver FD bị đóng:
        // -ECONNREFUSED: driver đã close connection
        // -EBADF: FD không còn valid (process shutting down)
    } while (result != -ECONNREFUSED && result != -EBADF);

    mOut.writeInt32(BC_EXIT_LOOPER);  // báo kernel thread đang thoát
    mIsLooper = false;
    talkWithDriver(false);  // flush BC_EXIT_LOOPER xuống kernel
    mProcess->mCurrentThreads.fetch_sub(1);
}
```

**Starvation detection trong `getAndExecuteCommand()`:**

```cpp
// IPCThreadState.cpp, line 747-768
size_t newThreadsCount = mProcess->mExecutingThreadsCount.fetch_add(1) + 1;
if (newThreadsCount >= mProcess->mMaxThreads) {
    // Tất cả threads đang bận → ghi lại thời điểm bắt đầu starvation
    auto expected = ProcessState::never();
    mProcess->mStarvationStartTime
            .compare_exchange_strong(expected, std::chrono::steady_clock::now());
    // compare_exchange: chỉ set nếu chưa ai set trước (prevent overwrite)
}

result = executeCommand(cmd);   // xử lý command

newThreadsCount = mProcess->mExecutingThreadsCount.fetch_sub(1) - 1;
if (newThreadsCount < mProcess->mMaxThreads) {
    // Ít nhất một thread vừa free → đo duration starvation
    auto starvationStartTime =
            mProcess->mStarvationStartTime.exchange(ProcessState::never());
    if (starvationStartTime != ProcessState::never()) {
        auto starvationTime = std::chrono::steady_clock::now() - starvationStartTime;
        if (starvationTime > 100ms) {
            ALOGE("binder thread pool (%zu threads) starved for %" PRId64 " ms",
                  maxThreads, to_ms(starvationTime));
        }
    }
}
```

Cơ chế này measure thời gian mà **tất cả** threads của process đang bận. Nếu pool full > 100ms, đó là dấu hiệu thread pool không đủ lớn hoặc có slow transaction.

---

## Section 8: Full Cycle — Một Synchronous Transaction

```
Client thread                  Kernel binder driver          Server thread
─────────────────────────────────────────────────────────────────────────────

transact(handle=5, code=3, data)
  writeTransactionData(BC_TRANSACTION)
  → mOut: [BC_TRANSACTION | binder_transaction_data]
  waitForResponse(reply) {
    loop {
      talkWithDriver() {
        bwr.write_size = mOut.dataSize()  ──copy_from_user()──► alloc mmap buffer
        bwr.read_size  = mIn.capacity()                        enqueue to server
                                          ◄──BR_TRANSACTION_COMPLETE──
                                          wake_up(server_thread) ──────────────►
        ioctl returns                                           talkWithDriver()
      }                                                           bwr.write=0
      // mIn: [BR_TRANSACTION_COMPLETE]                           bwr.read=mIn
      // reply != nullptr → break, continue loop               ioctl blocks...
      talkWithDriver() {                                        ◄── ioctl returns
        bwr.write_size = 0     (mOut rỗng sau khi flush)       // mIn: [BR_TRANSACTION]
        bwr.read_size  = mIn.capacity()                        getAndExecuteCommand()
        ioctl blocks...                                          executeCommand(BR_TRANSACTION)
      }                                                            mCallingPid = tr.sender_pid
                                                                   mCallingUid = tr.sender_euid
                                                                   onTransact(code=3, data, &reply)
                                                                   buffer.setDataSize(0)  // b/238777741
                                                                   sendReply(reply) {
                                                                     writeTransactionData(BC_REPLY)
                                                                     waitForResponse(nullptr) {
                                                                       talkWithDriver() {
                                          ◄──BR_REPLY──────────────────ioctl (bwr.write=BC_REPLY)
      // ioctl returns                    enqueue reply to client       ioctl returns
      // mIn: [BR_REPLY | tr]             wake_up(client_thread)     }
      // BR_REPLY case:                                              // mIn: [BR_TRANSACTION_COMPLETE]
      reply->ipcSetDataReference(                                    // reply==nullptr → goto finish
          tr.data.ptr.buffer,    // zero copy: point vào kernel buf  return NO_ERROR
          tr.data_size,          // không malloc, không memcpy      }
          tr.data.ptr.offsets,                                     mCallingPid = origPid (restore)
          ..., freeBuffer)                                         mCallingUid = origUid (restore)
      goto finish
    }
  }
  return NO_ERROR
  // Caller đọc reply Parcel — data vẫn nằm trong kernel mmap buffer
  // freeBuffer() được gọi khi reply Parcel bị destroy → BC_FREE_BUFFER
```

**Điểm quan trọng trong diagram:**

1. Client nhận `BR_TRANSACTION_COMPLETE` **trước** khi server bắt đầu xử lý. Đây chỉ là ACK từ kernel, không phải kết quả.

2. Server's `waitForResponse(nullptr)` sau `sendReply()` cũng nhận `BR_TRANSACTION_COMPLETE` — đây là ACK rằng reply đã được kernel enqueue về phía client.

3. Zero-copy tại `ipcSetDataReference()`: `reply` Parcel không copy dữ liệu mà chỉ lưu raw pointer vào kernel-allocated mmap buffer. Buffer chỉ bị free khi `reply` Parcel bị destroy, lúc đó `freeBuffer` callback gửi `BC_FREE_BUFFER` tới driver.

4. Cả hai threads đều dùng `waitForResponse()` — client chờ `BR_REPLY`, server chờ `BR_TRANSACTION_COMPLETE`. Đây là đối xứng trong thiết kế.
