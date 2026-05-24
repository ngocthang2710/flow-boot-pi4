# USB Gadget Framework — ADB over USB & dwc2

## 1. Tổng quan

Pi4 dùng USB theo 2 vai trò:
- **USB Host** (xHCI): kết nối USB devices (keyboard, USB drive, hub)
- **USB Gadget** (dwc2): Pi4 hoạt động như USB device (ADB, MTP, RNDIS)

```
Pi4 USB-C port → dwc2 OTG controller
  │
  ├── Host mode: Pi4 kết nối tới PC → USB hub, peripherals
  └── Device mode: Pi4 connected TO PC → ADB, fastboot, MTP
  
4× USB-A ports → VIA VL805 xHCI host controller
  → USB 3.0 host mode only
```

---

## 2. USB Host — xHCI (VL805)

```
Pi4 USB 3.0 topology:
  BCM2711 PCIe → VL805 (USB 3.0 controller)
    │
    ├── USB3-A port 1 (5 Gbps)
    ├── USB3-A port 2 (5 Gbps)
    ├── USB2-A port 3 (480 Mbps)
    └── USB2-A port 4 (480 Mbps)

Driver: xhci-hcd (xhci.c)
USB stack: usb-core → hub → device drivers
```

```bash
# Xem USB devices
adb shell lsusb
# Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
# Bus 002 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub
# Bus 001 Device 002: ID 046d:c52b Logitech Wireless Receiver

# USB device details
adb shell lsusb -v -d 046d:c52b

# Xem USB topology
adb shell cat /sys/kernel/debug/usb/devices
```

---

## 3. USB Gadget Framework

```
Linux USB Gadget Stack:

[ USB Host (PC side) ]
        ↕ USB cable
[ dwc2 UDC (USB Device Controller) ]
        ↕
[ USB Gadget Core (gadget.c) ]
        ↕
[ Composite Gadget (g_android) ]
  ├── ADB function
  ├── MTP function
  ├── RNDIS function (network over USB)
  └── Mass Storage function
```

---

## 4. dwc2 — Dual Role Controller

```c
/* dwc2 driver: drivers/usb/dwc2/ */
/* Supports Host, Device, OTG modes */

/* Pi4 USB-C port uses dwc2 */
/* DT node: */
/*
usb: usb@7e980000 {
    compatible = "brcm,bcm2835-usb";
    reg = <0x7e980000 0x10000>;
    interrupts = <GIC_SPI 9 IRQ_TYPE_LEVEL_HIGH>;
    clocks = <&clocks BCM2835_CLOCK_USB>;
    phys = <&usbphy>;
};
*/

/* dwc2 modes: */
/*   DWC2_FORCE_MODE_HOST: always host */
/*   DWC2_FORCE_MODE_DEVICE: always device */
/*   DWC2_AUTO: detect via ID pin / VBUS */
```

---

## 5. Android USB Gadget — ADB

```
Android USB gadget composition:
  /sys/class/android_usb/android0/

  When ADB connected:
    functions = adb
    enable = 1
    
  When ADB + MTP:
    functions = mtp,adb

  Protocol:
    adb_usb.c (kernel function) ↔ adbd (user-space daemon)
    → Bulk IN/OUT endpoints
    → ADB protocol over USB
```

```bash
# Xem USB gadget state
adb shell cat /sys/class/android_usb/android0/state
# CONFIGURED

adb shell cat /sys/class/android_usb/android0/functions
# mtp,adb

# Xem USB device from PC side
# lsusb  (on Linux host)
# Bus 001 Device 012: ID 18d1:4ee2 Google Inc. Nexus/Pixel Device (ADB + MTP)

# ADB connection debug
adb shell cat /sys/kernel/debug/usb/usbmon/1u
```

---

## 6. ConfigFS USB Gadget (Modern)

```bash
# Android 12+ dùng ConfigFS (thay legacy android_usb)
# /config/usb_gadget/g1/

ls /config/usb_gadget/
# g1/

ls /config/usb_gadget/g1/
# UDC  bcdDevice  bcdUSB  bDeviceClass  configs/  functions/  idProduct  idVendor  strings/

# Tạo ADB function
mkdir /config/usb_gadget/g1/functions/ffs.adb
# → Creates /dev/usb-ffs/adb/ endpoints

# Tạo MTP function
mkdir /config/usb_gadget/g1/functions/mtp.gs0

# Bind to UDC (USB Device Controller)
echo "fe980000.usb" > /config/usb_gadget/g1/UDC
# → Activates the gadget
```

---

## 7. USB Gadget Functions

```
Android USB functions:

ADB (Android Debug Bridge):
  Protocol: ADB protocol over bulk USB
  Endpoints: BULK IN + BULK OUT
  → adb shell, adb push/pull, adb install

MTP (Media Transfer Protocol):
  Protocol: MTP/PTP over USB
  → Transfer photos, music from device to PC
  → Windows Explorer shows device

RNDIS (Remote NDIS):
  → USB Tethering (sharing mobile data)
  → Creates network interface on PC (usb0)
  
NCM (Network Control Model):
  → Better USB network than RNDIS (Linux/Mac)

Mass Storage:
  → Expose partition as USB drive (legacy)
  → Rarely used in modern Android
  
FunctionFS (ffs):
  → User-space USB function implementation
  → ADB uses ffs since Android 12
```

---

## 8. ADB over TCP/IP (Alternative)

```bash
# Khi không có USB gadget, dùng ADB over network
# Pi4 usually connects via Ethernet/WiFi

# Trên Android device:
adb tcpip 5555

# Từ host (PC)
adb connect 192.168.1.100:5555

# Persistent (set property)
adb shell setprop service.adb.tcp.port 5555
adb shell stop adbd && adb shell start adbd
```

---

## 9. USB Power Delivery (Pi4)

```
Pi4 USB-C port:
  - Supports USB 2.0 (no USB 3.0 on USB-C)
  - Power Delivery: requires 5V/3A for Pi4 operation
  - OTG: can function as device mode
  
USB power flags in config.txt:
  max_usb_current=1  → Enable 1.2A on USB-A ports (was 600mA)
  usb_max_current_enable=1  → Same (newer name)
  
USB power issues:
  Undervoltage → throttling → performance drop
  → use adb shell vcgencmd get_throttled
```

---

## 10. Debug USB

```bash
# USB kernel debug
adb shell mount -t debugfs none /sys/kernel/debug
adb shell ls /sys/kernel/debug/usb/

# USB device stats
adb shell cat /sys/kernel/debug/usb/usbmon/0s

# USB endpoint state (device mode)
adb shell ls /sys/kernel/debug/dwc2/

# dmesg for USB events
adb shell dmesg | grep -E "usb|dwc2|xhci" | tail -20

# Watch USB connections (udevadm on host)
# udevadm monitor --subsystem-match=usb

# USB gadget state
adb shell cat /config/usb_gadget/g1/UDC
# fe980000.usb  → bound and active
```

---

## 11. File quan trọng trong source

| File | Mô tả |
|------|-------|
| `kernel/common/drivers/usb/dwc2/` | dwc2 OTG driver |
| `kernel/common/drivers/usb/host/xhci.c` | xHCI host driver |
| `kernel/common/drivers/usb/gadget/` | Gadget framework |
| `kernel/common/drivers/usb/gadget/function/f_fs.c` | FunctionFS (ADB) |
| `kernel/common/drivers/usb/gadget/function/f_mtp.c` | MTP function |
| `kernel/common/drivers/usb/gadget/legacy/android.c` | Legacy android USB |
| `system/core/adb/` | ADB daemon source |
| `device/brcm/rpi4/` → USB config | Pi4 USB gadget init |
