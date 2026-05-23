# WiFi Flow — Android 16 on Raspberry Pi 4

## Hardware

- **Chip**: Broadcom/Cypress **CYW43455** (combo WiFi + Bluetooth)
- **WiFi**: 802.11ac dual-band (2.4GHz + 5GHz)
- **Bus**: SDIO (SPI-like, 4-bit data bus)
- **Kernel driver**: `brcmfmac` (Broadcom FullMAC WiFi)
- **Mode**: **FullMAC** — chip firmware tự quản lý 802.11 MAC, scan, connect; host chỉ gửi lệnh qua nl80211

---

## Software Stack

```
App
 └── WifiManager (android.net.wifi) — API stub
      └── Binder IPC
           └── WifiServiceImpl (packages/modules/Wifi/service/)
                └── WifiNative → JNI
                     └── SupplicantStaIfaceHalAidlImpl
                          └── AIDL IPC (socket)
                               └── wpa_supplicant (vendor APEX)
                                    └── nl80211 (netlink socket)
                                         └── cfg80211 (kernel)
                                              └── brcmfmac driver
                                                   └── SDIO bus → CYW43455
```

---

## Luồng chi tiết

```
┌──────────────────────────────────────────────────────────┐
│  APP PROCESS                                             │
│  WifiManager.connectNetwork() / startScan()              │
│  android.net.wifi.WifiManager  (API stub)                │
└────────────────────────┬─────────────────────────────────┘
                         │ Binder IPC
┌────────────────────────▼─────────────────────────────────┐
│  SYSTEM SERVER                                           │
│  WifiServiceImpl  ← packages/modules/Wifi/service/       │
│    ↓                                                     │
│  WifiNative (Java) → JNI                                 │
│    ↓                                                     │
│  SupplicantStaIfaceHalAidlImpl ─── AIDL IPC ──►         │
└────────────────────────┬─────────────────────────────────┘
                         │ AIDL (socket)
┌────────────────────────▼─────────────────────────────────┐
│  WPA_SUPPLICANT PROCESS (vendor APEX)                    │
│  wpa_supplicant ← device/brcm/rpi4/wifi/                │
│    interface: android.hardware.wifi.supplicant.ISupplicant│
│    socket: /data/vendor/wifi/wpa/sockets/wpa_wlan0       │
│    ↓                                                     │
│  nl80211 commands (via netlink socket, AF_NETLINK)       │
└────────────────────────┬─────────────────────────────────┘
                         │ Netlink socket
┌────────────────────────▼─────────────────────────────────┐
│  LINUX KERNEL                                            │
│  cfg80211 (generic wireless subsystem)                   │
│    ↓                                                     │
│  brcmfmac driver (Broadcom FullMAC)                      │
│    ↓                                                     │
│  SDIO bus driver                                         │
└────────────────────────┬─────────────────────────────────┘
                         │ SDIO (4-bit, ~50MHz)
┌────────────────────────▼─────────────────────────────────┐
│  CYW43455 WiFi chip                                      │
│  Firmware tự xử lý: 802.11 MAC, scan, auth, DHCP        │
└──────────────────────────────────────────────────────────┘
```

---

## Cấu hình RPi4

- `wifi.interface=wlan0` (vendor.prop)
- APEX: `com.android.hardware.wifi.supplicant.rpi`
- `wpa_supplicant_overlay.conf`:
  ```
  disable_scan_offload=1
  wowlan_triggers=any
  p2p_disabled=1
  filter_rssi=-75
  ```

## AudioPolicy Route

```
AudioPolicyService đọc audio_policy_configuration.xml
  → Khi kết nối WiFi Direct hoặc Miracast:
     module "r_submix" cho audio mirroring qua WiFi
```

## Điểm đặc biệt

- **FullMAC**: Khác với SoftMAC (host xử lý MAC layer), brcmfmac delegate toàn bộ 802.11 state machine cho firmware trên chip
- **wpa_supplicant** build thành APEX riêng, không phải HAL AIDL truyền thống
- **SDIO** không có DMA riêng — data đi qua CPU
- `wifi.interface=wlan0` hardcode trong vendor.prop, không auto-detect
