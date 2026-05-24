# Binder Object Translation — flat_binder_object Deep Dive

Khi bạn truyền một `IBinder` qua Binder IPC, phép màu xảy ra trong kernel: một `BBinder*` (local pointer) ở process A biến thành một handle integer ở process B, và kernel tự quản lý mapping đó. File này giải thích cơ chế đó hoàn toàn.

---

## 1. flat_binder_object — cấu trúc trung tâm

```c
/* uapi/linux/android/binder.h */

struct binder_object_header {
    __u32 type;  /* BINDER_TYPE_* */
};

struct flat_binder_object {
    struct binder_object_header hdr;
    __u32 flags;
    union {
        binder_uintptr_t binder;  /* local object: BBinder* */
        __u32            handle;  /* remote reference: handle number */
    };
    binder_uintptr_t cookie;      /* user-defined tag (thường là same pointer) */
};
```

**7 loại object:**

| Type | Value | Ý nghĩa |
|------|-------|---------|
| `BINDER_TYPE_BINDER` | `'s','b','*',0x85` | Strong ref đến local BBinder |
| `BINDER_TYPE_WEAK_BINDER` | `'w','b','*',0x85` | Weak ref đến local BBinder |
| `BINDER_TYPE_HANDLE` | `'s','h','*',0x85` | Strong ref đến remote (handle) |
| `BINDER_TYPE_WEAK_HANDLE` | `'w','h','*',0x85` | Weak ref đến remote (handle) |
| `BINDER_TYPE_FD` | `'f','d','*',0x85` | File descriptor |
| `BINDER_TYPE_FDA` | `'f','d','a',0x85` | Array of file descriptors |
| `BINDER_TYPE_PTR` | `'p','t','*',0x85` | Scatter-gather buffer pointer |

---

## 2. FLAT_BINDER_FLAG_* — flags trong flat_binder_object

```c
enum flat_binder_object_flags {
    FLAT_BINDER_FLAG_PRIORITY_MASK    = 0xff,   // bits 0-7: nice priority của server thread
    FLAT_BINDER_FLAG_ACCEPTS_FDS      = 0x100,  // object cho phép nhận FD trong transaction
    FLAT_BINDER_FLAG_SCHED_POLICY_MASK = 3U << 9, // scheduling policy (SCHED_NORMAL, FIFO, RR)
    FLAT_BINDER_FLAG_INHERIT_RT       = 0x800,  // inherit real-time priority từ caller
    FLAT_BINDER_FLAG_TXN_SECURITY_CTX = 0x1000, // request secctx trong BR_TRANSACTION_SEC_CTX
};
```

**Priority inheritance qua flag:**

Khi ServiceManager register với `FLAT_BINDER_FLAG_TXN_SECURITY_CTX`, mỗi transaction đến service đó sẽ include SELinux security context của caller trong `BR_TRANSACTION_SEC_CTX` thay vì `BR_TRANSACTION`.

`FLAT_BINDER_FLAG_PRIORITY_MASK` và `FLAT_BINDER_FLAG_SCHED_POLICY_MASK`: kernel set priority của thread xử lý transaction bằng những flags này. Đây là cơ chế priority inheritance của Binder — caller không thể tùy ý set, đây là priority của **server** muốn được serve.

---

## 3. BINDER_TYPE_BINDER → HANDLE: phép biến đổi trong kernel

Đây là transformation quan trọng nhất. Khi process A muốn truyền BBinder của mình cho process B:

**Sender (A) viết:**
```c
flat_binder_object obj;
obj.hdr.type  = BINDER_TYPE_BINDER;  // Local object
obj.flags     = FLAT_BINDER_FLAG_ACCEPTS_FDS;
obj.binder    = (uintptr_t)myBBinder; // Địa chỉ thật trong process A
obj.cookie    = (uintptr_t)myBBinder; // Thường same
```

**Kernel xử lý (binder_translate_binder):**
```
1. Tìm/tạo binder_node cho ptr=obj.binder trong proc A
2. Tìm/tạo binder_ref cho node đó trong proc B
   → Cấp phát handle number mới (1, 2, 3, ...)
3. Thay đổi trong-place trong transaction buffer:
   obj.hdr.type = BINDER_TYPE_HANDLE  (từ BINDER_→HANDLE)
   obj.handle   = handle_number       (thay địa chỉ bằng handle)
4. Tăng strong ref: binder_inc_ref(ref, strong=1, ...)
```

**Receiver (B) đọc:**
```c
flat_binder_object obj;
// obj.hdr.type = BINDER_TYPE_HANDLE  ← kernel đã đổi
// obj.handle   = 3                   ← handle trong namespace của B
// Tạo BpBinder(handle=3) để giao tiếp với A
```

**Ngược lại — khi B gửi object ngược về A:**

Nếu B gửi `BINDER_TYPE_HANDLE` với handle=3, kernel:
1. Tìm binder_ref(handle=3) trong proc B
2. Ref đó point đến binder_node trong proc A
3. A chính là owner của node đó → kernel biết đây là "local" với A
4. Kernel đổi: `BINDER_TYPE_HANDLE` → `BINDER_TYPE_BINDER`, thay handle bằng ptr gốc

Kết quả: A nhận lại `BINDER_TYPE_BINDER` với ptr = địa chỉ BBinder của mình. A có thể dùng ptr trực tiếp thay vì đi qua Binder IPC.

---

## 4. BINDER_TYPE_FD — file descriptor transfer

FD crossing process boundary là feature đặc biệt mà Binder cung cấp, không thể làm với pipe hay socket thông thường.

```c
struct binder_fd_object {
    struct binder_object_header hdr;  /* BINDER_TYPE_FD */
    __u32 pad_flags;
    union {
        binder_uintptr_t pad_binder;
        __u32 fd;                      /* FD trong namespace của sender */
    };
    binder_uintptr_t cookie;
};
```

**Kernel thực hiện (binder_translate_fd):**

```
1. Lấy struct file* từ sender's file descriptor table:
   f = fget(obj.fd)

2. Tạo new FD trong receiver's file descriptor table:
   new_fd = get_unused_fd_flags(O_CLOEXEC)
   fd_install(new_fd, f)

3. Thay trong transaction buffer:
   obj.fd = new_fd  ← FD trong namespace của receiver

4. Receiver đọc obj.fd → mở được file tương ứng
```

**Tại sao FD sharing này quan trọng?**

```
Ví dụ thực tế: Gralloc buffer sharing
  SurfaceFlinger tạo buffer: fd = ashmem_create_region("gralloc", size)
  → Truyền qua Binder cho app (BINDER_TYPE_FD)
  → App nhận new_fd → cả hai process map cùng physical memory
  → Zero-copy graphics: app render, SurfaceFlinger composite trực tiếp
```

**Điều kiện:**

Object phải có `FLAT_BINDER_FLAG_ACCEPTS_FDS` set, nếu không kernel từ chối transaction chứa FD.

---

## 5. BINDER_TYPE_FDA — array of file descriptors

```c
struct binder_fd_array_object {
    struct binder_object_header hdr;  /* BINDER_TYPE_FDA */
    __u32 pad;
    binder_size_t num_fds;            /* số lượng FDs */
    binder_size_t parent;             /* index của BINDER_TYPE_PTR chứa FD array */
    binder_size_t parent_offset;      /* offset trong PTR buffer */
};
```

`BINDER_TYPE_FDA` luôn đi kèm `BINDER_TYPE_PTR`. FD array nằm trong scatter-gather buffer (PTR), và FDA cho kernel biết offset trong buffer đó chứa FD values cần translate.

---

## 6. BINDER_TYPE_PTR — scatter-gather (BC_TRANSACTION_SG)

Trước Android 8 (Treble), mỗi Parcel data phải copy thành một contiguous buffer trước khi gửi. Với HAL interface lớn (camera, audio), overhead copy này đáng kể.

```c
struct binder_buffer_object {
    struct binder_object_header hdr;  /* BINDER_TYPE_PTR */
    __u32 flags;                      /* BINDER_BUFFER_FLAG_HAS_PARENT */
    binder_uintptr_t buffer;          /* pointer đến data trong sender's address space */
    binder_size_t length;             /* size của buffer */
    binder_size_t parent;             /* parent buffer index (nếu HAS_PARENT) */
    binder_size_t parent_offset;      /* offset trong parent buffer */
};

struct binder_transaction_data_sg {
    struct binder_transaction_data transaction_data;
    binder_size_t buffers_size;       /* tổng kích thước scatter-gather buffers */
};
```

**BC_TRANSACTION_SG vs BC_TRANSACTION:**

```
BC_TRANSACTION:    1 contiguous buffer (tr.data.ptr.buffer)
BC_TRANSACTION_SG: 1 main buffer + N scatter-gather buffers (BINDER_TYPE_PTR objects)
```

**Kernel xử lý PTR object:**

```
1. Với mỗi BINDER_TYPE_PTR object:
   - Copy buffer từ sender's address space vào kernel binder buffer
   - Cập nhật pointer trong main transaction buffer
     (thay userspace pointer bằng pointer vào kernel buffer)
   
2. Receiver đọc main buffer:
   - Thấy pointer đã được resolve → có thể đọc trực tiếp
   
3. Nested buffers (HAS_PARENT):
   - Buffer có thể contain pointer đến buffer khác
   - parent + parent_offset: kernel cần update pointer trong parent buffer
```

**Ví dụ: Camera HAL truyền frame buffer**

```cpp
// HIDL/AIDL camera HAL gửi ICameraDeviceCallback::onResultReceived
// CaptureResult chứa nhiều buffer pointer riêng biệt:

CaptureResult result {
    .outputBuffers = {   // vector<StreamBuffer>
        StreamBuffer { .buffer = fd1, ... },
        StreamBuffer { .buffer = fd2, ... },
    },
    .physicalCameraMetadata = metadata_ptr,  // large metadata blob
};

// Với BC_TRANSACTION_SG:
// - metadata_ptr → BINDER_TYPE_PTR → kernel copy riêng
// - fd1, fd2 → BINDER_TYPE_FD → kernel dup riêng
// Không cần serialize toàn bộ vào 1 contiguous Parcel buffer
```

---

## 7. TF_CLEAR_BUF — security flag

```c
enum transaction_flags {
    TF_ONE_WAY          = 0x01,
    TF_ROOT_OBJECT      = 0x04,
    TF_STATUS_CODE      = 0x08,
    TF_ACCEPT_FDS       = 0x10,
    TF_CLEAR_BUF        = 0x20,  /* ← zero out buffer sau khi dùng */
    TF_UPDATE_TXN       = 0x40,
};
```

Khi `TF_CLEAR_BUF` set, kernel zero-out transaction buffer sau khi receiver đọc xong (khi `BC_FREE_BUFFER` được gọi). Dùng cho sensitive data như cryptographic keys, PIN.

Trong code:
```cpp
// IPCThreadState.cpp — sau khi sendReply()
// buffer.setDataSize(0) xóa Parcel trước khi reply
// TF_CLEAR_BUF được forward từ incoming transaction:
constexpr uint32_t kForwardReplyFlags = TF_CLEAR_BUF;
status_t error2 = sendReply(reply, (tr.flags & kForwardReplyFlags));
```

---

## 8. BR_TRANSACTION_SEC_CTX — SELinux security context

```c
struct binder_transaction_data_secctx {
    struct binder_transaction_data transaction_data;
    binder_uintptr_t secctx;  /* pointer đến SELinux context string */
};
```

Khi service register với `FLAT_BINDER_FLAG_TXN_SECURITY_CTX`:
1. Kernel include SELinux context của caller (ví dụ: `"u:r:untrusted_app:s0:c512,c768"`)
2. Server nhận `BR_TRANSACTION_SEC_CTX` thay vì `BR_TRANSACTION`
3. Server access qua `Binder.getCallerSecurityContext()`
4. Service có thể enforce SELinux policy tầng application (không chỉ kernel)

```cpp
// executeCommand() — khi BR_TRANSACTION_SEC_CTX:
mCallingSid = reinterpret_cast<const char*>(tr_secctx.secctx);
// → callable từ service: IPCThreadState::self()->getCallingSid()
```

---

## 9. Offsets array — kernel biết đâu là object

Khi gửi transaction, Parcel phải cung cấp offsets array:

```
Transaction buffer layout:
┌────────────────────────────────────────────┐
│  Regular data (int, string, ...)           │
│  ...                                       │
│  flat_binder_object #1 ← offset[0] = 32   │
│  ...                                       │
│  flat_binder_object #2 ← offset[1] = 80   │
│  ...                                       │
│  flat_binder_object #3 ← offset[2] = 120  │
└────────────────────────────────────────────┘

Offsets array: [32, 80, 120]
```

```cpp
// writeTransactionData() trong IPCThreadState:
tr.data_size = data.ipcDataSize();
tr.data.ptr.buffer  = data.ipcData();        // buffer chứa data + objects
tr.offsets_size = data.ipcObjectsCount() * sizeof(binder_size_t);
tr.data.ptr.offsets = data.ipcObjects();     // array of offsets
```

Kernel iterate qua offsets → xử lý từng `flat_binder_object` theo type (translate BINDER, dup FD, copy PTR). Data ngoài offsets được copy nguyên vẹn (regular ints, strings).

---

## 10. Parcel::writeStrongBinder() → flat_binder_object

Chuỗi đầy đủ khi AIDL code gọi `writeStrongBinder(myService)`:

```
Parcel::writeStrongBinder(sp<IBinder> val)
  ↓
  val->flattenBinder(parcel)         // method trên IBinder
  ↓
  BBinder::flattenBinder():
    obj.hdr.type = BINDER_TYPE_BINDER
    obj.binder   = reinterpret_cast<uintptr_t>(this)
    obj.cookie   = reinterpret_cast<uintptr_t>(this)
    parcel->writeObject(obj)         // append to buffer, record offset
  
  BpBinder::flattenBinder():
    obj.hdr.type = BINDER_TYPE_HANDLE
    obj.handle   = mHandle           // đã là handle, truyền handle
    parcel->writeObject(obj)
  ↓
  writeTransactionData() → mOut
  ↓
  talkWithDriver() → ioctl(BINDER_WRITE_READ)
  ↓
  [Kernel: binder_translate_binder() / binder_translate_handle()]
  ↓
  Receiver: Parcel::readStrongBinder()
    → readObject() đọc flat_binder_object
    → type == BINDER_TYPE_HANDLE → ProcessState::getStrongProxyForHandle(handle)
    → returns BpBinder(handle)
```
