# Bluetooth Flow — Android 16 on Raspberry Pi 4

## Hardware

- **Chip**: Broadcom/Cypress **CYW43455** (combo WiFi + Bluetooth)
- **Bluetooth**: 5.0 (BLE + Classic)
- **Bus**: UART (`ttyAMA0`, 3Mbps)
- **Kernel driver**: `hci_uart` + `btbcm` (Broadcom HCI UART)

---

## Software Stack

```
App
 └── BluetoothAdapter (android.bluetooth)
      └── Binder IPC
           └── BluetoothManagerService (com.android.bluetooth process)
                └── libbluetooth.so (C++ native stack)
                     ├── btif/ — Bluetooth Interface Layer
                     ├── system/gd/ — Gabeldorsche stack
                     ├── HCI abstraction layer
                     └── AIDL IPC
                          └── android.hardware.bluetooth-service.rpi (HAL)
                               └── NetBluetoothMgmt → AF_BLUETOOTH socket
                                    └── hci_uart kernel driver → UART → CYW43455
```

---

## Luồng chi tiết

```
┌──────────────────────────────────────────────────────────┐
│  APP PROCESS                                             │
│  BluetoothAdapter.startDiscovery() / connectGatt()       │
└────────────────────────┬─────────────────────────────────┘
                         │ Binder IPC
┌────────────────────────▼─────────────────────────────────┐
│  COM.ANDROID.BLUETOOTH PROCESS                           │
│  BluetoothManagerService                                 │
│    ↓                                                     │
│  Profile managers (A2DP, GATT, HFP, HID…)               │
│    ↓ JNI                                                 │
│  libbluetooth.so (C++ native stack)                      │
│    btif/ → system/gd/ → HCI abstraction                 │
└────────────────────────┬─────────────────────────────────┘
                         │ AIDL IPC
                         │ android.hardware.bluetooth.IBluetoothHci
┌────────────────────────▼─────────────────────────────────┐
│  BLUETOOTH HAL PROCESS (vendor APEX)                     │
│  android.hardware.bluetooth-service.rpi                  │
│  ← device/brcm/rpi4/bluetooth/BluetoothHci.cpp           │
│                                                          │
│  BluetoothHci::initialize():                             │
│   1. rfKill(block=1)                                     │
│      → /sys/class/rfkill/rfkill*/state = '1'            │
│   2. waitHciDev(hci_interface=0)                         │
│      → socket(AF_BLUETOOTH, HCI_CHANNEL_CONTROL)         │
│      → MGMT_OP_READ_INDEX_LIST → chờ hci0 sẵn sàng      │
│   3. socket(AF_BLUETOOTH, HCI_CHANNEL_USER)              │
│      → exclusive access, bypass BlueZ                    │
│   4. H4Protocol wrapper                                  │
│      → đóng gói HCI packets theo H4 UART format          │
│         type byte: 0x01=CMD, 0x02=ACL, 0x03=SCO,        │
│                    0x04=EVENT, 0x05=ISO                  │
└────────────────────────┬─────────────────────────────────┘
                         │ AF_BLUETOOTH HCI socket
┌────────────────────────▼─────────────────────────────────┐
│  LINUX KERNEL                                            │
│  BlueZ HCI core (hci_core.c)                             │
│    ↓                                                     │
│  hci_uart driver + btbcm (Broadcom firmware loader)      │
│    ↓                                                     │
│  UART driver (ttyAMA0)                                   │
└────────────────────────┬─────────────────────────────────┘
                         │ UART 3Mbps
┌────────────────────────▼─────────────────────────────────┐
│  CYW43455 Bluetooth chip                                 │
│  HCI firmware: LMP, L2CAP, baseband                      │
└──────────────────────────────────────────────────────────┘
```

---

## H4 Protocol (HCI UART framing)

```
HCI Command packet:
  [0x01] [opcode 2B] [param_len 1B] [params...]

HCI ACL Data packet:
  [0x02] [handle+flags 2B] [data_len 2B] [data...]

HCI Event packet (từ chip lên):
  [0x04] [event_code 1B] [param_len 1B] [params...]
```

## Bluetooth Profiles được enable (vendor.prop)

```
bluetooth.profile.a2dp.source.enabled=true   ← stream nhạc ra loa BT
bluetooth.profile.a2dp.sink.enabled=true     ← nhận nhạc từ điện thoại
bluetooth.profile.hfp.ag.enabled=true        ← hands-free phone
bluetooth.profile.hid.host.enabled=true      ← BT keyboard/mouse
bluetooth.profile.gatt.enabled=true          ← BLE
bluetooth.profile.pan.nap.enabled=true       ← BT tethering
bluetooth.profile.bap.unicast.client=true    ← LE Audio (BT 5.2)
... (tổng 20+ profiles)
```

## Điểm đặc biệt RPi4

- HAL tự gọi rfkill thay vì dùng Power HAL — đặc trưng CYW43455
- `HCI_CHANNEL_USER`: HAL giữ exclusive ownership của HCI device, BlueZ kernel stack không tham gia
- `waitHciDev()`: HAL chờ kernel driver init chip xong qua `HCI_CHANNEL_CONTROL` management socket trước khi mở `HCI_CHANNEL_USER`
