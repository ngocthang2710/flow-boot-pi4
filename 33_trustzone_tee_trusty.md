# TrustZone / ARM TEE / Trusty — Android Security

## 1. Tổng quan

ARM TrustZone chia processor thành 2 "world" hoàn toàn cách ly về phần cứng:

```
┌─────────────────────────────────────────────────┐
│              ARM Cortex-A72 (Pi4)               │
│                                                 │
│  ┌──────────────────┐  ┌──────────────────────┐ │
│  │   Normal World   │  │    Secure World      │ │
│  │   EL0: App       │  │   EL0: Trusted App   │ │
│  │   EL1: Linux     │  │   EL1: Trusty OS     │ │
│  │   EL2: Hypervisor│  │                      │ │
│  └──────────────────┘  └──────────────────────┘ │
│              EL3: Secure Monitor (BL31)          │
│         (switch giữa Normal ↔ Secure)           │
└─────────────────────────────────────────────────┘
```

**Trusty trên Pi4:** `kernel/common-modules/trusty/`  
**Config:** `CONFIG_TRUSTY=y`, `CONFIG_TRUSTY_LOG=y`

---

## 2. ARM Exception Levels

```
EL3 (Secure Monitor Mode):
  - Firmware (ATF - ARM Trusted Firmware)
  - Switch Normal ↔ Secure via SMC instruction
  - Highest privilege, always Secure

EL2 (Hypervisor):
  - Hypervisor (KVM trên Pi4)
  - Manage VMs

EL1 (OS/Kernel):
  - Normal: Linux kernel
  - Secure: Trusty OS

EL0 (User):
  - Normal: Android apps
  - Secure: Trusted Apps (TA) — keymaster, gatekeeper, DRM
```

---

## 3. Trusty TEE — Google's TEE OS

```
Normal World:          Secure World (Trusty):
                       
Linux Kernel           Trusty OS (microkernel)
  │                      │
  │ /dev/trusty-ipc-dev  │
  ├──────────────────────►│ Trusted Apps:
  │  Trusty IPC          │  ├── keymaster (key generation/sign)
  │                       │  ├── gatekeeper (PIN/password verify)
  │                       │  ├── DRM (Widevine)
  │                       │  ├── Secure storage
  │                       │  └── Custom TAs
  │                       │
  └──────────────────────◄│
     Results (encrypted)
```

---

## 4. Trusty trên Pi4

Pi4 có Trusty support qua `kernel/common-modules/trusty/`:

```bash
# Trusty kernel modules
ls kernel/common-modules/trusty/drivers/
# trusty-core/     ← Core IPC
# trusty-log/      ← Logging
# trusty-mem/      ← Shared memory
# trusty-virtio/   ← VirtIO transport

# Trên device
ls /dev/trusty*
# /dev/trusty-ipc-dev0
```

---

## 5. SMC (Secure Monitor Call)

```c
/* Gọi vào Secure World từ Normal World */
/* Linux kernel: arch/arm64/kernel/smccc-call.S */

/* SMC convention: */
arm_smccc_smc(
    0x83000001,   /* function ID */
    arg0, arg1, arg2, arg3,
    &res
);

/* ATF (BL31) nhận SMC, dispatch sang Trusty nếu cần */
/* Trusty nhận, process trong Secure World, return result */
```

---

## 6. Trusty IPC — Giao tiếp Android ↔ Trusty

```
Android (Normal World)          Trusty (Secure World)
                                
tipc_connect("keymaster")  ──►  Keymaster TA nhận kết nối
tipc_send(key_request)     ──►  TA generate key
tipc_recv(key_material)    ◄──  TA return encrypted key
tipc_close()
```

```c
/* Userspace TIPC (Trusty IPC) */
#include <trusty/tipc.h>

int fd = tipc_connect("/dev/trusty-ipc-dev0",
                      "com.android.trusty.keymaster");

uint8_t request[] = { KEYMASTER_GENERATE_KEY, ... };
write(fd, request, sizeof(request));

uint8_t response[1024];
read(fd, response, sizeof(response));

tipc_close(fd);
```

---

## 7. Keymaster / Keymint trong TEE

```
Android Keystore API (Java)
        │
        ▼
KeystoreService (system_server)
        │  AIDL
        ▼
keystore2 daemon
        │  TIPC
        ▼
Keymaster TA (trong Trusty, Secure World)
        │  Hardware operations
        ▼
Crypto hardware (nếu có) / ARM crypto extensions
```

```java
// Tạo key trong TEE
KeyPairGenerator kpg = KeyPairGenerator.getInstance(
    KeyProperties.KEY_ALGORITHM_EC, "AndroidKeyStore");

kpg.initialize(new KeyGenParameterSpec.Builder(
    "my_key",
    KeyProperties.PURPOSE_SIGN | KeyProperties.PURPOSE_VERIFY)
    .setDigests(KeyProperties.DIGEST_SHA256)
    .build());

KeyPair keyPair = kpg.generateKeyPair();
// Key được tạo và lưu trong Trusty, KHÔNG BAO GIỜ ra ngoài Normal World
```

---

## 8. Secure Storage

```
Trusty có vùng lưu trữ riêng:
  - Encrypted bằng device-specific key
  - Chỉ Trusty apps mới access được
  - Không visible với Linux filesystem

Trên Pi4:
  /data/ss/  ← Trusty secure storage (encrypted blob)
```

---

## 9. DRM (Widevine) trong TEE

```
Video streaming:
  Server → encrypted video → Android
              │
              ▼ Decryption key request
         Widevine TA (Trusty Secure World)
              │ Key chỉ trong Secure World
              ▼
         Decrypted video → Display
         (không thể capture bởi Normal World)

Đây là lý do Android có thể play L1 DRM content
```

---

## 10. ATF (ARM Trusted Firmware) trên Pi4

```
Pi4 boot với ATF (BL31):
  BL1 (ROM) → BL2 (UEFI/U-Boot) → BL31 (ATF) → Linux

BL31 chạy ở EL3, cung cấp:
  - PSCI (Power State Coordination Interface): CPU on/off, suspend
  - SMC handler: dispatch sang Trusty
  - SCMI: System Control and Management Interface
```

```bash
# PSCI trên Pi4
cat /sys/devices/system/cpu/cpu0/cpuidle/state*/name
# WFI (Wait For Interrupt) - EL1 idle
# cpu-sleep                - PSCI CPU_SUSPEND

# PSCI version
cat /sys/devices/system/cpu/cpu0/../power/psci_version 2>/dev/null
```

---

## 11. Kiểm tra Trusty trên Pi4

```bash
# Trusty modules loaded?
lsmod | grep trusty
# trusty_core     ...
# trusty_log      ...

# Trusty IPC device
ls -la /dev/trusty*

# Keystore hoạt động
adb shell keystore_cli_v2 list

# Test TEE
adb shell cmd keystore get-key-characteristics my_key

# Xem Trusty logs
adb logcat -s trusty
```

---

## 12. File quan trọng trong source

| File | Mô tả |
|------|-------|
| `kernel/common-modules/trusty/drivers/trusty/` | Trusty kernel driver |
| `kernel/common-modules/trusty/drivers/trusty/trusty-ipc.c` | TIPC IPC driver |
| `system/core/trusty/` | Trusty userspace utilities |
| `hardware/interfaces/security/keymint/` | Keymint AIDL HAL |
| `system/keymaster/` | Keymaster implementation |
| `vendor/brcm/rpi4/proprietary/boot/` | ATF binaries (start4.elf) |
