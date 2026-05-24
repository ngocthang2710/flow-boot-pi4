# Android Services & Processes

## 1. Các loại Service (5 loại chính)

### A. SystemService
- Chạy trong process `system_server` (dùng chung 1 process lớn)
- Kế thừa từ class `SystemService`
- Có vòng đời theo boot phase
- **Số lượng:** 167 class trong Android-16

```java
// frameworks/base/services/core/java/com/android/server/BatteryService.java
public final class BatteryService extends SystemService {
    public void onStart() {
        publishBinderService("battery", mBinderService);
        publishLocalService(BatteryManagerInternal.class, new LocalService());
    }
}
```

Ví dụ: `BatteryService`, `PowerManagerService`, `ActivityManagerService`, `WindowManagerService`, `PackageManagerService`, `ConnectivityService`, `AudioService`...

### B. Application Service
- Kế thừa từ `android.app.Service`
- Chạy trong process của ứng dụng
- Do ActivityManager quản lý vòng đời
- **Số lượng:** 240 class

### C. JobService
- Kế thừa từ `android.app.job.JobService`
- Chạy khi điều kiện thỏa mãn (battery, network, idle...)
- **Số lượng:** 40+ class

```java
public class BatteryIdleJob extends JobService { ... }
public class CameraStatsJobService extends JobService { ... }
```

### D. Specialized Service
Các service chuyên biệt cho từng chức năng:

| Loại | Vai trò |
|------|---------|
| `InputMethodService` | Bàn phím, phương thức nhập liệu |
| `AccessibilityService` | Hỗ trợ khả năng tiếp cận |
| `NotificationListenerService` | Lắng nghe thông báo |
| `WallpaperService` | Hình nền động |

**Số lượng:** 65 class

### E. Binder/Stub Service
- Implements AIDL interface (`IService.Stub`)
- Cung cấp giao diện IPC qua Binder giữa các process
- **Số lượng:** 376 class

---

## 2. Cơ chế giao tiếp giữa Service

| Cơ chế | Dùng khi |
|--------|---------|
| **Binder IPC** | Giao tiếp giữa các process khác nhau (dùng AIDL) |
| **Local Service** | Giao tiếp trong cùng process (`publishLocalService`) |
| **ServiceManager** | Đăng ký và tra cứu named service tập trung |

---

## 3. Boot Phases của SystemService

```
PHASE 100  → Wait for default display
PHASE 200  → Wait for sensor service
PHASE 480  → Lock settings ready
PHASE 500  → System services ready
PHASE 520  → Device-specific services ready
PHASE 550  → ActivityManager ready
PHASE 600  → Third-party apps can start
PHASE 1000 → Boot completed
```

---

## 4. Các Process

### Native Process (Init RC files)
- **1106 file .rc** → **878 dòng service definition** → **443 service name unique**
- Mỗi service = 1 Linux process riêng, viết bằng C/C++

| Nhóm | Ví dụ process |
|------|--------------|
| Core | `init`, `zygote`, `system_server`, `logd`, `adbd` |
| Storage | `vold` |
| Network | `netd`, `wificond` |
| Media/HAL | `audioserver`, `cameraserver`, `mediaserver` |
| Security | `keystore2`, `credstore` |
| ART | `artd`, `dexopt_chroot_setup` |
| Tracing | `traced`, `traced_probes`, `heapprofd` |
| Update | `update_engine`, `apexd` |

### Android Process (AndroidManifest.xml)
- **207 định nghĩa `android:process`**

```xml
<application android:process="system">
  <service android:process=":ui" />
  <service android:process=":background" />
</application>
```

Các process đặc biệt: `system`, `com.android.systemui`, `com.android.phone`,  
Private process: `:ui`, `:service`, `:background`, `:remote`, `:renderer`

---

## 5. Phân tầng tổng quan

```
┌────────────────────────────────────────────────┐
│          Applications (Java/Kotlin)             │
├────────────────────────────────────────────────┤
│  System Server — 167 SystemService              │  ← 1 process chứa nhiều service
├────────────────────────────────────────────────┤
│  Native Services (C/C++) — 443 process          │  ← Mỗi service = 1 process riêng
├────────────────────────────────────────────────┤
│  HAL (Hardware Abstraction Layer)               │
├────────────────────────────────────────────────┤
│  Linux Kernel                                   │
└────────────────────────────────────────────────┘
```

---

## 6. Số liệu tổng kết

| Chỉ số | Số lượng |
|--------|----------|
| Loại service | **5 loại** |
| SystemService | 167 |
| Application Service | 240 |
| JobService | 40+ |
| Specialized Service | 65 |
| Binder/Stub | 376 |
| Service module framework | 47 module |
| Native process (RC files) | 443 unique |
| Android:process definitions | 207 |
