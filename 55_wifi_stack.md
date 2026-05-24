# WiFi Stack — cfg80211, mac80211, wpa_supplicant, WifiService

## 1. Tổng quan

```
App (WifiManager API)
  │ Binder
  ▼
WifiService (system_server)
  │ D-Bus / Socket
  ▼
wpa_supplicant (userspace daemon)
  │ nl80211 (netlink)
  ▼
cfg80211 (kernel WiFi abstraction)
  │
  ├── mac80211 (software MAC for FullMAC/SoftMAC drivers)
  │     │
  │     └── wireless driver (brcmfmac, ath9k, iwlwifi, ...)
  │
  └── FullMAC driver (MAC in firmware, driver just nl80211)
        └── brcmfmac (Broadcom, Pi4)
```

---

## 2. Pi4 WiFi Hardware

```
Pi4B: Cypress CYW43455 (combo chip)
  WiFi: 802.11ac (WiFi 5)
    2.4GHz: b/g/n
    5GHz:   a/n/ac (up to 300 Mbps)
  Bluetooth: 5.0 (same chip)
  
Interface: SDIO (Secure Digital I/O) — connected via internal SDIO bus
Driver: brcmfmac (Broadcom FullMAC driver)
Firmware: brcm/brcmfmac43455-sdio.bin (loaded at boot)

wlan0 = WiFi station (client) interface
wlan0 can also be AP (access point) for hotspot
```

---

## 3. cfg80211 + nl80211

```
cfg80211 = kernel WiFi management framework
nl80211  = netlink protocol to talk to cfg80211 from userspace

nl80211 operations (some):
  NL80211_CMD_GET_WIPHY       ← get driver capabilities
  NL80211_CMD_SCAN            ← trigger scan
  NL80211_CMD_AUTHENTICATE    ← authenticate to AP
  NL80211_CMD_ASSOCIATE       ← associate to AP
  NL80211_CMD_CONNECT         ← shorthand auth+assoc
  NL80211_CMD_DISCONNECT      ← disconnect
  NL80211_CMD_SET_INTERFACE   ← change to AP/monitor mode
  NL80211_CMD_START_AP        ← start access point
  NL80211_CMD_NEW_STATION     ← new client in AP mode
```

---

## 4. brcmfmac — Broadcom FullMAC Driver

```
FullMAC = MAC layer runs in firmware (not in Linux mac80211)
  → firmware handles: scanning, authentication, association, power save
  → Driver just bridges nl80211 ↔ firmware commands
  
brcmfmac driver:
  drivers/net/wireless/broadcom/brcm80211/brcmfmac/
  
  SDIO interface → firmware download → WiFi ready
  Commands via IOCTL to firmware (not standard nl80211 internally)
  
  P2P (WiFi Direct): virtual p2p0 interface
  AP mode: virtual ap0 interface
```

---

## 5. wpa_supplicant

```
wpa_supplicant handles:
  WPA/WPA2/WPA3 authentication
  PMKSA caching (fast roaming)
  EAP (enterprise WiFi: PEAP, TLS, TTLS)
  SAE (WPA3 Simultaneous Authentication of Equals)
  WPS (WiFi Protected Setup)
  
Config: /data/misc/wifi/wpa_supplicant.conf

network={
    ssid="MyNetwork"
    psk="mypassword"
    key_mgmt=WPA-PSK
}

# WPA3
network={
    ssid="Secure"
    psk="mypassword"
    key_mgmt=SAE
    ieee80211w=2   ← Protected Management Frames (required for WPA3)
}
```

```bash
# wpa_cli control interface
adb shell wpa_cli -i wlan0 status
# bssid=aa:bb:cc:dd:ee:ff
# freq=5180
# ssid=MyNetwork
# id=0
# mode=station
# pairwise_cipher=CCMP
# group_cipher=CCMP
# key_mgmt=WPA2-PSK
# wpa_state=COMPLETED
# ip_address=192.168.1.101

adb shell wpa_cli -i wlan0 scan
adb shell wpa_cli -i wlan0 scan_results

adb shell wpa_cli -i wlan0 list_networks
# network id / ssid / bssid / flags
# 0       MyNetwork  any     [CURRENT]

adb shell wpa_cli -i wlan0 add_network
adb shell wpa_cli -i wlan0 set_network 1 ssid '"NewSSID"'
adb shell wpa_cli -i wlan0 set_network 1 psk '"password"'
adb shell wpa_cli -i wlan0 enable_network 1
adb shell wpa_cli -i wlan0 select_network 1
```

---

## 6. WifiService + WifiManager (Android Framework)

```
WifiService manages:
  Wifi enable/disable
  Scan results
  Network config (WifiConfiguration / WifiNetworkSuggestion)
  Connecting, disconnecting
  Hotspot (SoftAP) mode
  WiFi Direct (P2P)
  
Android 12+ changes:
  WifiNetworkSuggestion API replaces WifiConfiguration
  Random MAC addresses by default (privacy)
```

```java
// Java WiFi API
WifiManager wm = getSystemService(WifiManager.class);

// Scan (Android 6+: requires CHANGE_WIFI_STATE + LOCATION)
wm.startScan();
List<ScanResult> results = wm.getScanResults();

// Connect via suggestion (Android 10+)
WifiNetworkSuggestion suggestion = new WifiNetworkSuggestion.Builder()
    .setSsid("MyNetwork")
    .setWpa2Passphrase("mypassword")
    .setIsAppInteractionRequired(false)
    .build();
wm.addNetworkSuggestions(List.of(suggestion));

// Check connection
WifiInfo info = wm.getConnectionInfo();
info.getSSID();        // "MyNetwork"
info.getBSSID();       // "aa:bb:cc:dd:ee:ff"
info.getRssi();        // -65 dBm
info.getLinkSpeed();   // 300 Mbps
```

---

## 7. Hotspot / SoftAP

```java
// Create WiFi Hotspot (Android 8+, requires WRITE_SETTINGS or privileged)
SoftApConfiguration config = new SoftApConfiguration.Builder()
    .setSsid("Pi4Hotspot")
    .setPassphrase("mypassword", SoftApConfiguration.SECURITY_TYPE_WPA2_PSK)
    .setBand(SoftApConfiguration.BAND_5GHZ)
    .build();

wm.startTethering(TetheringManager.TETHERING_WIFI, true, callback);
```

```bash
# Check hotspot state
adb shell dumpsys wifi | grep -A5 "SoftAp"

# Manual AP via wpa_supplicant (hostapd mode)
# Or via tethering in Settings → Hotspot
```

---

## 8. WiFi Power Management

```
Power save modes:
  LEGACY_PSM:  Legacy 802.11 PS (beacon listen intervals)
  DTIM:        Delivery Traffic Indication Message (wake for multicast)
  
Pi4 behavior:
  WiFi can enter power save → adds latency
  Disable for low-latency (e.g., gaming):
    iw wlan0 set power_save off
  
Android WifiLock:
  WifiManager.WifiLock lock = wm.createWifiLock(
      WifiManager.WIFI_MODE_FULL_HIGH_PERF, "MyApp::Download");
  lock.acquire();  // prevents PS during critical transfer
```

```bash
# Check power save state
adb shell iw wlan0 get power_save
# Power save: off

# Signal quality
adb shell iw wlan0 link
# Connected to aa:bb:cc:dd:ee:ff
# SSID: MyNetwork
# freq: 5180
# RX: 1234567 bytes (1234 packets)
# TX: 234567 bytes (234 packets)
# signal: -55 dBm
# rx bitrate: 300.0 MBit/s MCS 15 40MHz short GI
# tx bitrate: 130.0 MBit/s MCS 7 40MHz short GI

# Channel info
adb shell iw dev wlan0 info
```

---

## 9. Debug WiFi

```bash
# Full WiFi dump
adb shell dumpsys wifi

# WifiService state
adb shell dumpsys wifi | head -100

# Scan results
adb shell dumpsys wifi | grep -A3 "Latest scan results"

# Connection events log
adb shell dumpsys wifi | grep -A50 "WifiConfigManager"

# Kernel WiFi events
adb shell dmesg | grep -E "brcmfmac|wlan|cfg80211" | tail -30

# WiFi interface stats
adb shell cat /sys/class/net/wlan0/statistics/rx_bytes
adb shell cat /sys/class/net/wlan0/statistics/tx_bytes

# Signal monitor
adb shell while true; do \
    adb shell iw wlan0 link | grep signal; sleep 1; done
```

---

## 10. File quan trọng trong source

| File | Mô tả |
|------|-------|
| `packages/modules/Wifi/service/java/com/android/server/wifi/WifiServiceImpl.java` | WifiService |
| `packages/modules/Wifi/service/java/com/android/server/wifi/WifiConnectivityManager.java` | Auto-connect |
| `kernel/common/drivers/net/wireless/broadcom/brcm80211/brcmfmac/` | brcmfmac driver |
| `kernel/common/net/wireless/cfg80211.c` | cfg80211 core |
| `kernel/common/net/mac80211/` | mac80211 software MAC |
| `external/wpa_supplicant_8/` | wpa_supplicant source |
