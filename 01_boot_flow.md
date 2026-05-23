# Android 16 Boot Flow — Raspberry Pi 4

## Partition Layout (SD card / USB)

```
mmcblk0p1  [boot]     FAT32   128MB  ← BCM firmware + kernel Image + ramdisk.img
mmcblk0p3  [userdata] ext4    (còn lại) ← /data
mmcblk0p5  [/]        ext4    3GB    ← /system
mmcblk0p6  [vendor]   ext4    384MB  ← /vendor (HAL, drivers RPi4)
mmcblk0p7  [metadata] ext4    16MB   ← encryption metadata
```

---

## Stage 0 — BCM2711 ROM Bootloader

- CPU ARM Cortex-A72 reset → PC = 0x0 (ROM, không thể thay đổi)
- **GPU (VideoCore IV) boot trước ARM** — đây là đặc trưng của BCM
- ROM load `bootcode.bin` từ SD card vào SRAM 16KB on-chip

## Stage 1 — GPU Firmware (`start4.elf`)

Đọc `config.txt` (device/brcm/rpi4/boot/config.txt):

```ini
arm_64bit=1                    # AArch64 mode
kernel=Image                   # Linux kernel
initramfs ramdisk.img followkernel
dtoverlay=vc4-kms-v3d          # DRM/KMS display driver
disable_fw_kms_setup=1         # Kernel tự quản lý display
dtoverlay=dwc2,dr_mode=peripheral  # USB gadget
camera_auto_detect=1           # CSI camera auto-detect
enable_uart=1                  # Serial console
```

GPU firmware thực hiện:
1. Init LPDDR4 DRAM
2. Load Device Tree Blob (.dtb)
3. Apply dt-overlays (vc4-kms-v3d, dwc2, audio…)
4. Release ARM CPU → jump đến `Image`

## Stage 2 — Linux Kernel Boot

```
arch/arm64/kernel/head.S
  → setup MMU, page tables, exception vectors
  → start_kernel()
     ├── setup_arch()   ← parse Device Tree, init memory map
     ├── mm_init()      ← buddy allocator, slab
     ├── sched_init()   ← CFS scheduler
     ├── irq_init()     ← GIC interrupt controller
     └── init_drivers() ← probe theo DT compatible
            ├── vc4-kms-v3d    → DRM/KMS + HDMI
            ├── bcm2835-isp    → ISP camera
            ├── unicam         → MIPI CSI-2 receiver
            ├── brcmfmac       → WiFi SDIO
            ├── hci_uart       → Bluetooth UART
            ├── bcm2835-audio  → 3.5mm jack
            └── dwc2           → USB controller
```

`kernel_init()` → PID 1 → `execv("/init")`

## Stage 3 — Android Init First Stage

File: `system/core/init/first_stage_init.cpp`

1. Mount: `tmpfs`, `devtmpfs`, `proc`, `sysfs`, `selinuxfs`
2. `LoadKernelModules()` — load .ko từ ramdisk
3. `DoFirstStageMount()` — đọc `fstab.rpi4`:
   - `/dev/mmcblk0p5` → `/system` (ext4, ro, `first_stage_mount`)
   - `/dev/mmcblk0p6` → `/vendor` (ext4, ro, `first_stage_mount`)
   - `/dev/mmcblk0p7` → `/metadata` (ext4, `first_stage_mount`)
4. `execv("/init", ["selinux_setup"])`

## Stage 4 — SELinux Setup

- `LoadSelinuxPolicy()` — đọc policy từ `/system/etc/selinux/`
- Load vào kernel: `selinux_load_policy()`
- Chuyển sang enforcing mode
- `execv("/init", ["second_stage"])`

## Stage 5 — Android Init Second Stage

File: `system/core/init/init.cpp::SecondStageMain()`

1. `property_init()` — khởi động property service
2. Parse RC files:
   - `/init.rc`
   - `/vendor/etc/init/hw/init.rpi4.rc`
   - `/vendor/etc/init/hw/init.rpi4.usb.rc`
   - `/apex/*/init.rc`

3. **Trigger `early-init`:**
   - start `ueventd` (đọc `ueventd.rpi4.rc` → tạo `/dev/*` nodes)
   - `/dev/video*` → camera, `/dev/rfkill` → BT, `/dev/cec0` → HDMI CEC

4. **Trigger `init`:**
   - start `logd`, `servicemanager`, `hwservicemanager`, `vndservicemanager`
   - `mount_all fstab.rpi4 --early`
   - mount `/data` (`mmcblk0p3`)

5. **Trigger `late-init`** → `fs` → `post-fs-data` → **`zygote-start`**:
   - start `zygote` (`app_process64 /system/bin --zygote`)
   - start `zygote_secondary` (32-bit)

6. **Trigger `boot`** — start vendor HAL services (song song):
   ```
   vendor.audio-rpi            ← Audio HAL
   vendor.camera.provider-ext  ← Camera HAL
   vendor.bluetooth            ← Bluetooth HAL
   vendor.drm-hwcomposer       ← HWC/Display HAL
   vendor.health-service       ← Health HAL
   vendor.light-service        ← Light HAL
   vendor.thermal              ← Thermal HAL
   wpa_supplicant              ← WiFi (sau apex.all.ready=true)
   ```

## Stage 6 — Zygote

File: `frameworks/base/core/java/com/android/internal/os/ZygoteInit.java`

1. `AndroidRuntime::start("ZygoteInit")` — khởi động ART (JVM)
2. `preloadClasses()` — preload ~7000 Java classes vào heap
3. `preloadResources()` — load drawable, layout, string resources
4. `forkSystemServer()` → fork() → child = system_server process
5. `zygoteServer.runSelectLoop()` — lắng nghe `/dev/socket/zygote`
   - Mỗi app launch = nhận fork command → fork() → child kế thừa preloaded classes (Copy-on-Write)

## Stage 7 — System Server

File: `frameworks/base/services/java/com/android/server/SystemServer.java`

### `startBootstrapServices()` — Phase 100–480

| Phase | Constant | Services |
|-------|----------|----------|
| 100 | `PHASE_WAIT_FOR_DEFAULT_DISPLAY` | Chờ SurfaceFlinger (đã start native) |
| — | — | `ActivityTaskManagerService` (ATMS) |
| — | — | `ActivityManagerService` (AMS) |
| — | — | `PowerManagerService` |
| — | — | `ThermalManagerService` |
| — | — | `PackageManagerService` ← scan `/system/app`, `/data/app` |
| — | — | `UserManagerService` |

### `startCoreServices()` — Phase 500

| Phase | Services |
|-------|----------|
| `PHASE_SYSTEM_SERVICES_READY` (500) | `BatteryService`, `WebViewUpdateService` |

### `startOtherServices()` — Phase 520–1000

| Phase | Constant | Services / Actions |
|-------|----------|--------------------|
| 520 | `PHASE_DEVICE_SPECIFIC_SERVICES_READY` | `WindowManagerService`, `InputManagerService` |
| — | — | `NetworkManagementService`, `WifiService` |
| — | — | `AudioService`, `CameraService`, `BluetoothManagerService` |
| 550 | `PHASE_ACTIVITY_MANAGER_READY` | `AMS.systemReady()` → **launch Launcher app** |
| 600 | `PHASE_THIRD_PARTY_APPS_CAN_START` | Broadcast `ACTION_LOCKED_BOOT_COMPLETED` |
| 1000 | `PHASE_BOOT_COMPLETED` | Broadcast `ACTION_BOOT_COMPLETED` |

## Stage 8 — Home Screen

```
AMS.startHomeOnAllDisplays()
  → Zygote fork → Launcher process
  → Launcher.onCreate() → inflate layout → draw UI
  → SurfaceFlinger composite
  → DRM atomic commit (drm_hwcomposer)
  → vc4 HDMI → màn hình hiện home screen
```

---

## Timeline tổng hợp

| Thời gian | Sự kiện |
|-----------|---------|
| t=0ms | BCM2711 ROM boot |
| t=~500ms | GPU firmware (start4.elf) xong, release ARM |
| t=~1s | Linux kernel boot, PID 1 = /init |
| t=~2s | First stage: mount /system, /vendor |
| t=~3s | SELinux policy loaded |
| t=~4s | servicemanager, ueventd sẵn sàng |
| t=~5s | Zygote start, preload classes |
| t=~8s | SystemServer fork, bootstrap services up |
| t=~12s | PackageManager scan xong |
| t=~15s | AMS.systemReady() → Launcher launch |
| t=~18s | **Home screen hiển thị** |
| t=~20s+ | `ACTION_BOOT_COMPLETED` broadcast |

---

## So sánh RPi4 vs Android Phone

| | Android Phone | RPi4 Android |
|---|---|---|
| Bootloader | ABL / Qualcomm XBL | BCM ROM + GPU firmware |
| Verified Boot | Android AVB (ký số) | Không có |
| A/B partition | Thường có | Không có (single slot) |
| Ramdisk | `vendor_boot` partition | `initramfs` trong FAT32 |
| Kernel load | Từ `boot.img` (android format) | Trực tiếp `kernel=Image` |
| Partition scheme | `super` (dynamic partitions) | ext4 static partitions |
| GPU/CPU boot order | ARM boot trước | **GPU boot trước**, release ARM |
