# Raspberry Pi 4 USB Stack — xHCI, dwc2, USB-C

## 1. Tổng quan

Pi4 có hai USB subsystems:

```
BCM2711 SoC
  │
  ├── dwc2 (DesignWare USB 2.0 OTG)
  │     USB-C port (GPIO 8: USB OTG)
  │     Host + Device mode
  │
  └── PCIe bus → VIA VL805 xHCI controller
          │
          ├── USB 3.0 port A (5 Gbps)
          ├── USB 3.0 port B (5 Gbps)
          ├── USB 2.0 port C (480 Mbps)
          └── USB 2.0 port D (480 Mbps)
```

---

## 2. VL805 xHCI Host Controller

```
VIA VL805:
  USB 3.2 Gen 1 (5 Gbps) xHCI controller
  Connected via PCIe gen 2 x1 (5 Gbps PCIe)
  
  4 ports:
    Port 1,2: USB 3.0 SuperSpeed (5 Gbps)
    Port 3,4: USB 2.0 HighSpeed (480 Mbps)
    All share USB 2.0 (HS) root hub
    
Driver:
  xhci-hcd (xhci.c) → generic xHCI driver
  xhci-pci.c → PCIe binding
  Firmware load: VL805 needs firmware from Pi4
```

```bash
# VL805 PCIe presence
adb shell lspci
# 01:00.0 USB controller: VIA Technologies, Inc. VL805/806 xHCI USB 3.0 Controller (rev 01)

# xHCI root hubs
adb shell lsusb
# Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
# Bus 002 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub
```

---

## 3. dwc2 OTG Controller

```
DesignWare USB 2.0 OTG:
  Supports: Host, Device, OTG modes
  Max speed: HighSpeed (480 Mbps) — USB 2.0 only
  
Pi4 USB-C:
  Connected to dwc2
  When connected to PC → Device mode (ADB, MTP)
  When used as OTG → Host mode (USB drives, keyboards)
  
Mode selection:
  VBUS detection + ID pin
  Or forced via device tree property
  
Limitation:
  dwc2 Host mode: shared with internal USB hub
  → Pi4 actual USB-A ports use VL805
  → dwc2 host mode mainly for USB-C OTG use
```

---

## 4. USB 3.0 vs USB 2.0 Architecture

```
USB Protocol Stack:
  Application (bulk transfer, control)
    │
  USB core (usb.c, hub.c)
    │
  Host Controller Driver (xHCI/OHCI/EHCI)
    │
  Hardware (xHCI register interface)

USB Speeds:
  USB 3.2 Gen 1  = SuperSpeed     = 5 Gbps
  USB 3.1 Gen 1  = SuperSpeed     = 5 Gbps (same)
  USB 3.0        = SuperSpeed     = 5 Gbps (same)
  USB 2.0        = HighSpeed      = 480 Mbps
  USB 1.1        = FullSpeed/Low  = 12/1.5 Mbps

Pi4 max real throughput:
  USB 3.0 disk: ~400 MB/s
  USB 2.0 disk: ~35 MB/s
  USB 2.0 keyboard: < 1 KB/s
```

---

## 5. USB Device Framework

```
USB device structure:
  Device → Configurations → Interfaces → Endpoints
  
  Device: one device (1 VID:PID)
  Configuration: power/feature set (usually 1)
  Interface: function (e.g., keyboard=1 interface, webcam=2)
  Endpoint: data pipe (IN/OUT, Bulk/Interrupt/Isoch/Control)
  
Example: USB Webcam:
  Device: VID=046d, PID=0825
  Config 1:
    Interface 0: Video Control (setup, camera control)
      Endpoint 0: Control (bidirectional)
    Interface 1: Video Streaming (frames)
      Endpoint 1: Bulk/Isoch IN (video data)
    Interface 2: Audio (microphone)
      Endpoint 2: Isoch IN (audio data)
```

---

## 6. USB Transfer Types

```
Control:    Setup, then IN/OUT data
            Used for: enumeration, configuration
            Max 64 bytes/packet (HS)

Interrupt:  Guaranteed max latency polling
            Used for: HID (keyboard, mouse)
            USB 2.0: 1ms minimum interval

Bulk:       Large data, no timing guarantee
            Used for: mass storage, ADB
            USB 3.0: up to 1024 bytes/packet

Isochronous: Fixed bandwidth, no error recovery
             Used for: audio, video (streaming)
             USB 2.0: up to 1024 bytes/packet at 1ms
```

---

## 7. USB Class Drivers

```bash
# HID (Human Interface Device) — keyboard, mouse
# Driver: hid-generic, usbhid
adb shell ls /dev/input/  # keyboard → /dev/input/eventX

# USB Mass Storage
# Driver: usb-storage, uas (USB Attached SCSI)
adb shell ls /dev/block/sd*  # USB drive → /dev/sda, /dev/sdb

# USB Ethernet (RNDIS/CDC)
# Driver: rndis_host, cdc_ether
adb shell ip link show  # usb0 or eth1

# USB Audio
# Driver: snd-usb-audio
adb shell cat /proc/asound/cards

# USB Serial (FTDI, CP210x)
# Driver: ftdi_sio, cp210x
adb shell ls /dev/ttyUSB*

# USB Webcam (UVC)
# Driver: uvcvideo
adb shell ls /dev/video*
```

---

## 8. USB Power Management

```bash
# USB autosuspend (power off idle devices)
# Default: enabled for most devices

# Check autosuspend
adb shell cat /sys/bus/usb/devices/1-1/power/autosuspend_delay_ms
# 2000  (suspend after 2 seconds idle)

adb shell cat /sys/bus/usb/devices/1-1/power/runtime_status
# suspended  or  active

# Disable autosuspend for device
adb shell echo -1 > /sys/bus/usb/devices/1-1/power/autosuspend_delay_ms

# USB power budget (Pi4 USB-A ports)
# Default: 600mA per port
# With max_usb_current=1 in config.txt: 1.2A per port
adb shell cat /sys/bus/usb/devices/usb1/power/usb3_hardware_lpm_active
```

---

## 9. ADB USB Protocol

```
ADB (Android Debug Bridge) protocol over USB:

  USB layer: Bulk IN/OUT endpoints
  
  ADB message format:
    command (4 bytes) : CNXN, OPEN, CLOSE, WRITE, SYNC, AUTH
    arg0    (4 bytes)
    arg1    (4 bytes)
    data_length (4 bytes)
    data_check  (4 bytes)
    magic   (4 bytes) : command ^ 0xffffffff
    data    (variable)
    
  Setup:
    Host sends: CNXN (version, max_data, "host::...")
    Device responds: CNXN ("device:rpi4:Android16:...")
    
  Authentication (Android 4.4+):
    Host sends AUTH(TOKEN)
    Device sends AUTH(RSA_SIGN, signature)
    Host verifies with stored RSA public key
```

---

## 10. Debug USB

```bash
# USB kernel events
adb shell dmesg | grep -E "usb|xhci|dwc2" | tail -30

# USB device tree
adb shell cat /sys/kernel/debug/usb/devices
# T:  Bus=01 Lev=00 Prnt=00 Port=00 Cnt=00 Dev#=  1 Spd=480 MxCh= 1
# D:  Ver= 2.00 Cls=09(hub  ) Sub=00 Prot=01 MxPS=64 #Cfgs=  1
# ...

# xHCI registers
adb shell ls /sys/kernel/debug/xhci/

# USB bus reset (test)
adb shell echo 1 > /sys/bus/usb/devices/usb1/authorized_default

# USB traffic sniffing (usbmon kernel module)
adb shell modprobe usbmon
adb shell ls /dev/usbmon*
# /dev/usbmon0  /dev/usbmon1  /dev/usbmon2

# Monitor USB traffic (Wireshark format)
adb shell cat /sys/kernel/debug/usb/usbmon/1u | head -50
```

---

## 11. USB Gadget (Device Mode)

```bash
# ADB connected state
adb shell getprop sys.usb.state
# adb  or  mtp,adb

adb shell getprop sys.usb.config
# adb

# USB gadget backend
adb shell ls /config/usb_gadget/

# VID:PID shown to host
adb shell getprop sys.usb.vid
# 18d1  (Google)
adb shell getprop sys.usb.pid
# 4ee2  (Android ADB+MTP)
```

---

## 12. File quan trọng trong source

| File | Mô tả |
|------|-------|
| `kernel/common/drivers/usb/host/xhci.c` | xHCI host driver |
| `kernel/common/drivers/usb/dwc2/` | dwc2 OTG driver |
| `kernel/common/drivers/usb/core/hub.c` | USB hub driver |
| `kernel/common/drivers/usb/core/urb.c` | URB (USB Request Block) |
| `kernel/common/drivers/usb/storage/usb.c` | USB Mass Storage |
| `kernel/common/drivers/hid/usbhid/usbhid.c` | USB HID |
| `system/core/adb/` | ADB daemon |
| `kernel/common/arch/arm64/boot/dts/broadcom/bcm2711-rpi-4-b.dts` | Pi4 USB DT |
