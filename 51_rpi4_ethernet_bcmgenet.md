# bcmgenet — Raspberry Pi 4 Gigabit Ethernet

## 1. Tổng quan

Pi4 dùng BCM54213PE Gigabit PHY kết nối với BCM2711's GENET (Gigabit Ethernet) controller.

```
BCM2711 SoC
  │
  ├── GENET controller (in-silicon)
  │     │ RGMII (Reduced GMII)
  │     │
  └── BCM54213PE (PHY chip)
          │ RJ45 connector
          │
          Gigabit Ethernet (up to 1 Gbps)
```

---

## 2. GENET Hardware

```
GENET (Broadcom GENET v5):
  Full duplex Gigabit Ethernet
  Hardware checksum offload (TCP/UDP/IP)
  Hardware VLAN tagging/stripping
  Multiple RX/TX queues (for QoS)
  Wake-on-LAN (WoL)
  
DMA:
  Internal DMA engine
  Buffer descriptors in RAM
  Zero-copy RX possible via DMA
  
Interrupts:
  TX done, RX done, error, link change
```

---

## 3. Driver Architecture (bcmgenet.c)

```c
/* drivers/net/ethernet/broadcom/genet/bcmgenet.c */

struct bcmgenet_priv {
    void __iomem *base;           /* Registers base */
    struct net_device *dev;
    struct phy_device *phydev;    /* BCM54213 PHY */
    
    /* TX queues (16 HW + 1 default) */
    struct bcmgenet_tx_ring tx_rings[17];
    
    /* RX queues (16 HW + 1 default) */
    struct bcmgenet_rx_ring rx_rings[17];
    
    struct clk *clk;
    struct clk *clk_eee;          /* Energy Efficient Ethernet */
    
    /* WoL */
    u32 wolopts;
    
    /* NAPI (budget-based interrupt coalescing) */
    struct napi_struct rx_napi;
    struct napi_struct tx_napi;
};

/* Key operations: */
static const struct net_device_ops bcmgenet_netdev_ops = {
    .ndo_open            = bcmgenet_open,
    .ndo_stop            = bcmgenet_close,
    .ndo_start_xmit      = bcmgenet_xmit,       /* TX */
    .ndo_set_rx_mode     = bcmgenet_set_rx_mode, /* Multicast */
    .ndo_set_mac_address = bcmgenet_set_mac_addr,
    .ndo_do_ioctl        = bcmgenet_ioctl,
    .ndo_set_features    = bcmgenet_set_features,
};
```

---

## 4. PHY — BCM54213PE

```
BCM54213PE is a GPHY (Gigabit PHY):
  Interfaces: RGMII (1000BASE-T) or MII (100BASE-TX)
  Auto-negotiation: 10/100/1000 Mbps
  MDI/MDIX: Auto-crossover
  EEE: Energy Efficient Ethernet (802.3az)
  WoL: Wake-on-LAN
  
MDIO (Management Data I/O):
  BCM2711 GENET includes MDIO master
  PHY registers accessible via MDIO bus
  
Linux: Generic PHY driver + bcm_phy_lib.c for Broadcom specifics
```

---

## 5. NAPI — Interrupt Coalescing

```c
/* NAPI (New API) = hybrid interrupt/polling: */
/* 1. First packet: interrupt fires */
/* 2. Disable interrupt, schedule poll */
/* 3. Poll until budget exhausted or no more packets */
/* 4. Re-enable interrupt */

/* This prevents interrupt storm on heavy traffic */

static int bcmgenet_rx_poll(struct napi_struct *napi, int budget) {
    struct bcmgenet_rx_ring *ring = 
        container_of(napi, struct bcmgenet_rx_ring, napi);
    
    unsigned int work_done = bcmgenet_desc_rx(ring, budget);
    
    if (work_done < budget) {
        napi_complete_done(napi, work_done);
        /* Re-enable RX interrupt */
        bcmgenet_rdma_ring_writel(priv, ring->index,
            DMA_MBUF_DONE_THRESH, 1);
    }
    return work_done;
}
```

---

## 6. RX/TX Buffer Flow

```
TX (sending):
  sk_buff (socket buffer) from networking stack
    │ bcmgenet_xmit()
    ▼
  DMA descriptor → points to sk_buff data (no copy!)
    │ Hardware DMA
    ▼
  GENET FIFO → RGMII → PHY → Cable
    │ TX done interrupt
    ▼
  Free sk_buff

RX (receiving):
  Cable → PHY → RGMII → GENET FIFO
    │ DMA to pre-allocated sk_buff
    ▼
  RX interrupt → NAPI poll
    │ bcmgenet_desc_rx()
    ▼
  netif_receive_skb() → kernel networking stack
    │
    ▼
  TCP/IP → socket → App recv()
```

---

## 7. Hardware Offloads

```
bcmgenet HW offloads (reduce CPU usage):

TX checksum:
  NETIF_F_IP_CSUM    ← IPv4 TCP/UDP checksum offload
  NETIF_F_IPV6_CSUM  ← IPv6 TCP/UDP checksum offload
  NETIF_F_SG         ← Scatter-gather (multi-buffer TX)

RX checksum:
  NETIF_F_RXCSUM     ← RX checksum verification

Features disabled on Pi4:
  TSO (TCP Segmentation Offload) — limited HW support
  GRO (Generic Receive Offload) — SW based
```

---

## 8. Energy Efficient Ethernet (EEE)

```
IEEE 802.3az — EEE:
  When link is idle → PHY enters Low Power Idle (LPI)
  Wakes up for bursts, sleeps between
  
  ~50% power savings on idle link
  
Pi4 EEE config:
  Enabled by default if PHY supports it
  Can be disabled: ethtool --set-eee eth0 eee off

adb shell ethtool --show-eee eth0
# EEE settings for eth0:
#   EEE status: enabled - active
#   Tx LPI: 1000 [us]
#   Advertised EEE link modes: 1000baseT/Full
#   Link partner advertised EEE link modes: 1000baseT/Full
```

---

## 9. Debug Network

```bash
# Interface info
adb shell ip link show eth0
# 2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq
#     link/ether dc:a6:32:xx:xx:xx brd ff:ff:ff:ff:ff:ff

adb shell ip addr show eth0
# inet 192.168.1.100/24 brd 192.168.1.255

# PHY info via ethtool
adb shell ethtool eth0
# Settings for eth0:
#   Speed: 1000Mb/s
#   Duplex: Full
#   Port: MII
#   Link detected: yes

# PHY registers (MDIO)
adb shell ethtool --phy-statistics eth0

# Statistics
adb shell ethtool --statistics eth0
adb shell cat /proc/net/dev | grep eth0

# Driver info
adb shell ethtool --driver eth0
# driver: bcmgenet
# version: [kernel version]

# Network performance test
adb shell iperf3 -s            # server mode
# iperf3 -c 192.168.1.100      # from PC → device

# Interface queue stats
adb shell tc -s qdisc show dev eth0
```

---

## 10. Wake-on-LAN

```bash
# Enable WoL on Pi4
adb shell ethtool -s eth0 wol g    # magic packet

# Send magic packet from PC (to wake Pi4)
# etherwake <Pi4-MAC>  (Linux)
# wakeonlan <Pi4-MAC>  (wakeonlan tool)

# Check WoL capability
adb shell ethtool eth0 | grep Wake
# Supports Wake-on: g
# Wake-on: g
```

---

## 11. File quan trọng trong source

| File | Mô tả |
|------|-------|
| `kernel/common/drivers/net/ethernet/broadcom/genet/bcmgenet.c` | GENET driver |
| `kernel/common/drivers/net/ethernet/broadcom/genet/bcmgenet.h` | GENET headers |
| `kernel/common/drivers/net/phy/bcm-phy-lib.c` | Broadcom PHY library |
| `kernel/common/drivers/net/phy/broadcom.c` | BCM54213 PHY driver |
| `kernel/common/include/linux/netdevice.h` | Network device API |
| `kernel/common/arch/arm64/boot/dts/broadcom/bcm2711-rpi-4-b.dts` | Pi4 DT (Ethernet node) |
