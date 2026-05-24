# netd & Android Networking

## 1. Tổng quan

netd (Network Daemon) là native daemon quản lý mọi cấu hình mạng của Android: routing, firewall, DNS, bandwidth control.

```
App (socket, HttpURLConnection)
  │
  ▼
ConnectivityService (system_server)
  │ Binder AIDL
  ▼
netd (/system/bin/netd)
  │
  ├── iptables/nftables  ← Firewall, NAT
  ├── ip rule/route      ← Routing
  ├── tc (traffic control) ← Bandwidth
  ├── DNS resolver       ← /etc/resolv.conf equivalent
  └── Network namespaces ← VPN isolation
```

---

## 2. netd Architecture

```
netd daemon:
  ├── NetlinkHandler  ← Nhận kernel network events
  ├── CommandListener ← Nhận lệnh từ framework (qua socket)
  ├── BandwidthController ← Giới hạn bandwidth per app
  ├── FirewallController  ← iptables rules
  ├── NatController       ← NAT/tethering
  ├── RouteController     ← Per-UID routing tables
  └── DnsResolver         ← DNS-over-TLS, caching
```

---

## 3. Per-UID Firewall (Network Policy)

```
Android dùng iptables owner module để firewall theo UID:

App (uid=10123)
  │ socket()
  ▼
iptables OUTPUT chain:
  -m owner --uid-owner 10123 -j ACCEPT   ← allowed
  
Background process (uid=10456, Doze):
  -m owner --uid-owner 10456 -j REJECT   ← blocked

Firewall chains:
  fw_INPUT    ← inbound packets
  fw_OUTPUT   ← outbound packets
  fw_FORWARD  ← forwarded (tethering)
  fw_dozable  ← Doze mode restrictions
  fw_standby  ← App standby restrictions
  fw_restricted ← Restricted bucket
```

```bash
# Xem firewall rules
adb shell iptables -L fw_OUTPUT -n

# Xem per-UID rules
adb shell iptables -L bw_penalty_box -n

# Xem bandwidth usage per UID
adb shell cat /proc/net/xt_qtaguid/stats | head -20
```

---

## 4. Routing — Multiple Networks

```
Android hỗ trợ nhiều network cùng lúc:
  WiFi (wlan0) + Mobile (rmnet0) + VPN (tun0)

Per-UID routing:
  App uid=10123 → route qua WiFi (default)
  App uid=10456 → route qua VPN (forced)

Routing tables:
  Table 1000: main WiFi routes
  Table 1001: mobile data routes
  Table 1002: VPN routes
  Table 99: per-UID rules

ip rule list:
  0: from all lookup local
  9000: from all fwmark .../0xffff lookup VPN table
  9001: from all oif wlan0 lookup WiFi table
  32766: from all lookup main
```

```bash
# Xem routing rules
adb shell ip rule list
adb shell ip route show table all

# Xem network state
adb shell dumpsys connectivity | head -100
adb shell dumpsys netstats | head -50
```

---

## 5. DNS Resolver

```
Android 9+: DNS resolver trong netd
  → DNS-over-TLS (DoT) support
  → Per-network DNS servers
  → Cache

Config:
  net.dns1=8.8.8.8
  net.dns2=8.8.4.4

Private DNS (DoT):
  Settings → Network → Private DNS → dns.google
  → netd kết nối TLS port 853
```

```bash
# Xem DNS config
adb shell getprop net.dns1
adb shell ndc resolver getnetdns <netId>

# Test DNS
adb shell nslookup google.com

# DNS stats
adb shell dumpsys dnsresolver
```

---

## 6. Tethering & NAT

```
Pi4 as WiFi hotspot:
  Client → wlan0 → Pi4 → eth0 → Internet

iptables NAT:
  POSTROUTING -o eth0 -j MASQUERADE
  FORWARD -i wlan0 -o eth0 -j ACCEPT
  FORWARD -i eth0 -o wlan0 -m state --state RELATED,ESTABLISHED -j ACCEPT

netd NatController:
  enableNat(intIface="wlan0", extIface="eth0")
```

---

## 7. VPN

```
VpnService (app-level VPN):
  App tạo TUN interface
  → All traffic được redirect qua TUN
  → App encrypt/forward theo VPN protocol

Kernel VPN:
  struct tun_pi → /dev/tun
  ip tuntap add mode tun → tun0
  ip rule add → route all traffic qua tun0
```

```java
// Android VpnService
VpnService.Builder builder = new VpnService.Builder();
builder.addAddress("10.0.0.2", 24)
       .addRoute("0.0.0.0", 0)     // Default route qua VPN
       .addDnsServer("1.1.1.1")
       .setSession("MyVPN");

ParcelFileDescriptor vpnInterface = builder.establish();
// Đọc/ghi packet từ vpnInterface.getFileDescriptor()
```

---

## 8. Bandwidth Control

```bash
# Xem bandwidth stats per app
adb shell cat /proc/net/xt_qtaguid/stats

# Set bandwidth limit cho app (background)
# ConnectivityManager.restrictBackgroundData()
# netd: tc qdisc, tc class, tc filter

# Xem qdisc
adb shell tc qdisc show dev wlan0
adb shell tc class show dev wlan0
```

---

## 9. Android Network Security

```xml
<!-- Network Security Config (res/xml/network_security_config.xml) -->
<network-security-config>
    <!-- Trust only system CAs -->
    <base-config cleartextTrafficPermitted="false">
        <trust-anchors>
            <certificates src="system" />
        </trust-anchors>
    </base-config>

    <!-- Allow debug server (debug builds only) -->
    <debug-overrides>
        <trust-anchors>
            <certificates src="user" />
        </trust-anchors>
    </debug-overrides>
</network-security-config>
```

---

## 10. Pi4 Network Interfaces

```bash
# Ethernet
ip addr show eth0
# eth0: <BROADCAST,MULTICAST,UP,LOWER_UP>
#   inet 192.168.1.100/24

# WiFi
ip addr show wlan0
# wlan0: <BROADCAST,MULTICAST,UP,LOWER_UP>
#   inet 192.168.1.101/24

# WiFi supplicant
adb shell wpa_cli status
# bssid=aa:bb:cc:dd:ee:ff
# ssid=MyNetwork
# ip_address=192.168.1.101

# Test connectivity
adb shell ping -c 4 8.8.8.8
adb shell curl -v https://google.com
```

---

## 11. File quan trọng trong source

| File | Mô tả |
|------|-------|
| `system/netd/server/` | netd server implementation |
| `system/netd/server/RouteController.cpp` | Per-UID routing |
| `system/netd/server/FirewallController.cpp` | iptables rules |
| `system/netd/server/BandwidthController.cpp` | Traffic control |
| `system/netd/resolv/` | DNS resolver |
| `frameworks/base/services/core/java/com/android/server/connectivity/` | ConnectivityService |
