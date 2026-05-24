# Native Services

## 1. Native Service là gì?

Native Services là các service viết bằng **C/C++**, chạy trực tiếp trên Linux kernel, **không qua Java Virtual Machine hay Android Runtime (ART)**.

Chúng được khai báo trong file `.rc` (Init Script), mỗi service tương ứng 1 file binary trong `/system/bin/`:

```bash
# /system/core/rootdir/init.rc
service logd /system/bin/logd
    class core
    user logd
    group logd system

service vold /system/bin/vold
    class core
    user root
    group root
```

---

## 2. So sánh Native Service vs Java Service

| Đặc điểm | Native Service | Java SystemService |
|-----------|---------------|-------------------|
| Ngôn ngữ | C / C++ | Java / Kotlin |
| Runtime | Linux process trực tiếp | JVM / ART |
| Process | Mỗi service = 1 process riêng | Nhiều service dùng chung `system_server` |
| Khởi động | `init` đọc `.rc`, fork process | `SystemServer.java` khởi tạo |
| Giao tiếp | Binder IPC, Socket, netlink | Binder IPC, Local Service |

---

## 3. Các Native Service quan trọng trong Android-16

### Core
| Service | Binary | Vai trò |
|---------|--------|---------|
| `logd` | `/system/bin/logd` | Thu thập log toàn hệ thống |
| `adbd` | `/system/bin/adbd` | ADB daemon (debug) |
| `vold` | `/system/bin/vold` | Quản lý storage, mount/unmount |

### Network
| Service | Binary | Vai trò |
|---------|--------|---------|
| `netd` | `/system/bin/netd` | Cấu hình mạng, firewall iptables |
| `wificond` | `/system/bin/wificond` | Giao tiếp WiFi driver |

### Media / HAL
| Service | Binary | Vai trò |
|---------|--------|---------|
| `audioserver` | `/system/bin/audioserver` | Xử lý âm thanh tầng HAL |
| `cameraserver` | `/system/bin/cameraserver` | Quản lý Camera HAL |
| `mediaserver` | `/system/bin/mediaserver` | Media framework |
| `mediadrmserver` | `/system/bin/mediadrmserver` | DRM decryption |

### Security
| Service | Binary | Vai trò |
|---------|--------|---------|
| `keystore2` | `/system/bin/keystore2` | Lưu trữ khóa mã hóa |
| `credstore` | `/system/bin/credstore` | Credential storage |

### ART / Package
| Service | Binary | Vai trò |
|---------|--------|---------|
| `artd` | `/system/bin/artd` | ART daemon |
| `apexd` | `/system/bin/apexd` | Quản lý APEX packages |
| `dexopt_chroot_setup` | — | Dex optimization |

### Tracing / Debug
| Service | Binary | Vai trò |
|---------|--------|---------|
| `traced` | `/system/bin/traced` | Perfetto trace daemon |
| `traced_probes` | `/system/bin/traced_probes` | Kernel trace probes |
| `heapprofd` | `/system/bin/heapprofd` | Heap memory profiling |

### Update
| Service | Binary | Vai trò |
|---------|--------|---------|
| `update_engine` | `/system/bin/update_engine` | OTA update manager |
| `gsid` | `/system/bin/gsid` | Generic System Image daemon |

---

## 4. Vị trí trong kiến trúc Android

```
┌─────────────────────────────────────┐
│         Applications (Java)         │
├─────────────────────────────────────┤
│   System Server (Java SystemService)│  ← BatteryService, WindowManagerService...
├─────────────────────────────────────┤
│   Native Services (C/C++)           │  ← audioserver, cameraserver, vold...
├─────────────────────────────────────┤
│   HAL (Hardware Abstraction Layer)  │
├─────────────────────────────────────┤
│   Linux Kernel                      │
└─────────────────────────────────────┘
```

Native Services là tầng **gần hardware nhất** trong userspace.  
Chúng xử lý các tác vụ đòi hỏi hiệu năng cao hoặc cần truy cập trực tiếp kernel/hardware mà Java không thể làm hiệu quả.

---

## 5. Cách khai báo trong RC file

```bash
service <tên>  <đường_dẫn_binary>  [tham_số]
    class    <core|main|hal|...>    # Nhóm khởi động
    user     <username>             # Chạy dưới user nào
    group    <group...>             # Group permissions
    socket   <name> <type> <mode>  # Unix socket (nếu cần)
    oneshot                         # Chỉ chạy 1 lần, không restart
    disabled                        # Không tự khởi động
```

Ví dụ đầy đủ:
```bash
service vold /system/bin/vold \
    --blkid_context=u:r:blkid:s0
    class core
    socket vold stream 0660 root mount
    socket cryptd stream 0660 root mount
    ioprio be 2
```

---

## 6. Class của Native Service

| Class | Khởi động khi | Ví dụ |
|-------|--------------|-------|
| `core` | Sớm nhất, ngay sau init | `logd`, `vold`, `netd` |
| `main` | Sau core | `audioserver`, `cameraserver` |
| `hal` | HAL services | Hardware abstraction |
| `late_start` | Sau boot completed | Các service không cần thiết sớm |
