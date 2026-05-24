# A/B Update & Virtual A/B (VAB)

## 1. Traditional A/B Update

### Ý tưởng: 2 slot song song

```
Flash Storage:
┌──────────┬──────────┬──────────┐
│  boot_a  │  boot_b  │          │
│ system_a │ system_b │ userdata │  ← Không có slot (dùng chung)
│ vendor_a │ vendor_b │          │
└──────────┴──────────┴──────────┘
   Slot A     Slot B
  (active)   (inactive)
```

### Luồng update

```
1. Device đang chạy Slot A
         │
         ▼
2. OTA download → ghi thẳng vào Slot B
   (device vẫn dùng bình thường từ Slot A)
         │
         ▼
3. update_engine gọi setActiveBootSlot(B)
   Bootloader ghi vào BCB (Bootloader Control Block)
         │
         ▼
4. User reboot → Bootloader đọc BCB → boot Slot B
         │
         ▼
5a. Boot OK → markBootSuccessful() → Slot B thành active
5b. Boot fail → bootloader timeout → tự rollback về Slot A
```

### Boot Control HAL Interface

```aidl
// hardware/interfaces/boot/aidl/IBootControl.aidl
interface IBootControl {
    int  getCurrentSlot()              // slot đang chạy (0=A, 1=B)
    int  getActiveBootSlot()           // slot sẽ boot kế tiếp
    int  getNumberSlots()              // luôn = 2
    void setActiveBootSlot(int slot)   // yêu cầu bootloader switch
    void markBootSuccessful()          // xác nhận slot này OK
    void setSlotAsUnbootable(int slot) // đánh dấu slot lỗi
    bool isSlotBootable(int slot)
    MergeStatus getSnapshotMergeStatus()
    void setSnapshotMergeStatus(MergeStatus)
}
```

### Nhược điểm
Cần **2x dung lượng** cho system + vendor + product. Thiết bị 32GB có thể mất 6–8GB chỉ cho slot dự phòng.

---

## 2. Virtual A/B (VAB) — Tiết kiệm bộ nhớ

### Ý tưởng: 1 slot + COW snapshot

```
Flash Storage (VAB):
┌───────────────────────────────────┬──────────────┐
│           super partition         │   userdata   │
│  ┌────────┬────────┬───────────┐  │  ┌─────────┐ │
│  │ system │ vendor │  product  │  │  │COW files│ │
│  └────────┴────────┴───────────┘  │  └─────────┘ │
│  (base partitions - 1 bản duy nhất)│  (snapshot)  │
└───────────────────────────────────┴──────────────┘
```

### Copy-on-Write (COW) hoạt động thế nào

Khi OTA ghi block mới vào `system`:
- Block gốc **giữ nguyên** → dùng để rollback
- Thay đổi mới ghi vào **COW file** trong `/data`
- Kernel ghép 2 lại → thiết bị thấy partition "mới"

```
Đọc block 100 (chưa thay đổi):  đọc thẳng từ base partition
Đọc block 200 (đã OTA):         đọc từ COW file → trả về data mới
Ghi block 300 (OTA đang apply): ghi vào COW, không đụng base
```

### Các loại COW Operation

```cpp
// system/core/fs_mgr/libsnapshot/libsnapshot_cow/cow_format.cpp
kCowCopyOp    // copy block từ vị trí này sang vị trí khác
kCowReplaceOp // ghi raw bytes mới (data từ OTA server)
kCowZeroOp    // zero out block
kCowXorOp     // XOR block cũ với delta (tiết kiệm hơn Replace)
kCowLabelOp   // checkpoint để resume nếu bị gián đoạn
```

### COW Size Formula

```
Total COW = cow_partition_size + cow_file_size

cow_partition_size = min(calculated_size, free_region_in_super)
cow_file_size      = max(0, calculated_size - cow_partition_size)
```

### Merge Driver — 3 biến thể

```cpp
// system/core/fs_mgr/libsnapshot/snapshot.h
enum SnapshotDriver {
    DM_SNAPSHOT,  // Kernel dm-snapshot (truyền thống)
    DM_USER,      // snapuserd daemon (userspace, linh hoạt)
    UBLK          // io_uring based (mới nhất, hiệu năng cao)
}
```

---

## 3. Toàn bộ luồng VAB OTA

### Phase 1: Apply OTA (device dùng bình thường)

```
update_engine
    ├─ PreparePartitionsForUpdate()
    │     ├─ Tính COW size cần thiết
    │     ├─ Allocate COW space trong super (nếu có dư)
    │     └─ Allocate COW files trong /data
    │
    ├─ Download payload từ server
    │
    └─ DeltaPerformer apply operations:
          kReplaceOp  → cow_writer.AddRawBlocks()
          kSourceCopy → cow_writer.AddCopy()
          kXorOp      → cow_writer.AddXorBlocks()

UpdateState: None → Initiated → Unverified
```

### Phase 2: Boot vào "slot mới" (base + COW)

```
First-stage init
    ├─ Đọc snapshot metadata từ /metadata/ota/
    ├─ Tạo dm-snapshot devices:
    │     /dev/block/mapper/system ← base + COW
    │     /dev/block/mapper/vendor ← base + COW
    └─ Mount: thiết bị thấy OS mới

markBootSuccessful() → xác nhận boot OK
UpdateState: Unverified → Merging
```

### Phase 3: Merge (chạy nền)

```
snapuserd daemon (hoặc kernel dm-snapshot)
    ├─ Đọc COW operations theo thứ tự
    ├─ Ghi data mới trực tiếp vào base partition
    └─ Xóa COW block đã merge

CleanupPreviousUpdateAction theo dõi progress
UpdateState: Merging → MergeCompleted → None
```

---

## 4. UpdateState Lifecycle

```
None
 │ OTA bắt đầu
 ▼
Initiated ──→ Cancelled (user cancel)
 │ OTA apply xong
 ▼
Unverified ──→ Cancelled (boot fail → rollback)
 │ Boot OK + markBootSuccessful()
 ▼
Merging ──→ MergeFailed (I/O error)
 │ Merge xong
 ▼
MergeNeedsReboot
 │ Reboot
 ▼
MergeCompleted
 │ Cleanup
 ▼
None
```

---

## 5. Rollback

| Thời điểm | Cơ chế | Kết quả |
|-----------|--------|---------|
| Boot Phase 2 thất bại | Bootloader timeout → boot lại từ base | OS cũ, COW bị xóa |
| Đang merge | `HandleCancelledUpdate()` → xóa snapshots | OS cũ phục hồi |
| Merge hoàn tất | **Không thể rollback** | Phải factory reset |

---

## 6. MergePhase — Thứ tự merge

```protobuf
// system/core/fs_mgr/libsnapshot/android/snapshot/snapshot.proto
enum MergePhase {
    NO_MERGE     = 0  // Không merge
    FIRST_PHASE  = 1  // Merge partitions nhỏ hơn trước
    SECOND_PHASE = 2  // Merge partitions lớn hơn sau
}
```

---

## 7. System Properties quan trọng

| Property | Ý nghĩa |
|----------|---------|
| `ro.virtual_ab.enabled` | Enable VAB |
| `ro.virtual_ab.compression.enabled` | Enable COW compression |
| `ro.virtual_ab.compression.xor.enabled` | Enable XOR compression |
| `ro.virtual_ab.userspace.snapshots.enabled` | Dùng dm-user/snapuserd |
| `ro.boot.dynamic_partitions` | Enable dynamic partitions |
| `ro.boot.slot_suffix` | Current slot (`_a` hoặc `_b`) |

---

## 8. File chính trong source

| File | Vai trò |
|------|---------|
| `hardware/interfaces/boot/aidl/IBootControl.aidl` | Boot Control HAL interface |
| `system/update_engine/aosp/boot_control_android.cc` | Boot Control implementation |
| `system/update_engine/aosp/dynamic_partition_control_android.cc` | Dynamic partition + VAB control |
| `system/update_engine/aosp/update_attempter_android.cc` | Main OTA apply logic |
| `system/update_engine/aosp/cleanup_previous_update_action.cc` | Merge & rollback cleanup |
| `system/core/fs_mgr/libsnapshot/snapshot.cpp` | SnapshotManager core |
| `system/core/fs_mgr/libsnapshot/partition_cow_creator.cpp` | COW size calculation |
| `system/core/fs_mgr/libsnapshot/libsnapshot_cow/cow_format.cpp` | COW format & operations |

---

## 9. So sánh tổng kết

| Tiêu chí | Traditional A/B | Virtual A/B |
|----------|----------------|-------------|
| Số slot vật lý | 2 | 1 |
| Dung lượng thêm | ~100% (2x) | ~20% (COW) |
| OTA apply time | Nhanh | Nhanh |
| Rollback sau merge | Luôn được | Không được |
| Phức tạp implementation | Thấp | Cao |
| Bắt buộc từ | Android 8 | Android 11 |
