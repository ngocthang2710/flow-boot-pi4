# Kernel Clock Framework — clk Subsystem & BCM2711

## 1. Tổng quan

Clock framework (Common Clock Framework - CCF) quản lý tất cả clock sources trong SoC: PLL, dividers, gates, muxes.

```
Crystal (xtal) 54MHz
  │
  ├─ PLL_A → 787.5MHz ──→ Audio, I2S
  ├─ PLL_C → 1000MHz ──→ GPU (V3D), SDRAM
  ├─ PLL_D → 594MHz  ──→ HDMI, Display (PLLH)
  ├─ PLL_H → (HDMI PLL)
  └─ PLLB  → 3000MHz ──→ ARM CPU
       │
       ├─ /arm_prescaler → 1500MHz
       ├─ /2 → 750MHz
       └─ ARM CPU (up to 1500MHz for Pi4)
```

---

## 2. Common Clock Framework (CCF)

```c
/* Clock provider registers with CCF */
#include <linux/clk-provider.h>

/* Clock types: */
/* clk_fixed_rate: fixed frequency (crystal) */
/* clk_gate: enable/disable a clock */
/* clk_divider: divide parent clock */
/* clk_mux: select between parent clocks */
/* clk_fixed_factor: fixed multiplication/division */

/* Example: register a gate clock */
struct clk *my_clk = clk_register_gate(
    dev,
    "my_gate_clk",          /* clock name */
    "parent_clk",           /* parent clock name */
    0,                       /* flags */
    reg_base + 0x100,        /* register address */
    5,                       /* bit offset */
    0,                       /* gate flags */
    &my_lock                 /* spinlock */
);

/* Consumer uses clock */
struct clk *clk = devm_clk_get(dev, "my_clk");
clk_prepare_enable(clk);
clk_set_rate(clk, 100000000); /* 100 MHz */
/* ... */
clk_disable_unprepare(clk);
```

---

## 3. BCM2711 Clock Sources

```
BCM2711 (Raspberry Pi 4) clock hierarchy:

Crystal: 54 MHz (xtal)
  │
  ├── PLLB (ARM PLL)
  │     vpll_fold → arm_core_clk → Cortex-A72
  │     Range: 600MHz - 1800MHz (depends on temp/voltage)
  │
  ├── PLLC (Core PLL) = 1GHz typically
  │     pllc_core0 → core_clk → GPU, ISP
  │     pllc_core1 → core1_clk
  │     pllc_core2 → core2_clk
  │     pllc_per   → sdram_clk, emmc_clk
  │
  ├── PLLD (DSI PLL) = 500MHz
  │     plld_core → pixel_clk → Display pixel clock
  │     plld_per  → uart_clk, spi_clk, i2c_clk, pwm_clk
  │
  ├── PLLH (HDMI PLL)
  │     pllh_rcal → HDMI audio
  │     pllh_aux  → HDMI pixel clock
  │
  └── PLLA (Audio PLL)
        plla_core → audio_clk
        plla_per  → pcm_clk
```

---

## 4. Clock Driver (clk-bcm2835.c)

```c
/* drivers/clk/bcm/clk-bcm2835.c */

/* BCM2835/BCM2711 define all clocks: */
static const struct bcm2835_clk_desc clk_desc_array[] = {
    /* ARM Core Clock */
    [BCM2835_CLOCK_ARM] = REGISTER_PLL_DIV(
        .name = "arm",
        .source_pll = "pllb",
        .cm_reg = CM_ARMCTL,
        .a2w_reg = A2W_PLLB_ARM,
        .load_mask = CM_ARMCTL_PLLB_SET,
        .hold_mask = 0,
        .fixed_divider = 1,
        .flags = CLK_SET_RATE_PARENT,
    ),
    /* GPU Core Clock */
    [BCM2835_CLOCK_V3D] = REGISTER_CLK(
        .name = "v3d",
        .parents = bcm2835_clock_per_parents,
        .num_mux_parents = ARRAY_SIZE(bcm2835_clock_per_parents),
        .ctl_reg = CM_V3DCTL,
        .div_reg = CM_V3DDIV,
        .int_bits = 4,
        .frac_bits = 8,
    ),
    /* ... many more ... */
};
```

---

## 5. Clock Control via Firmware (Pi4)

```c
/* Pi4 có VideoCore firmware control some clocks */
/* raspberrypi-firmware-clocks driver bridges firmware ↔ CCF */

/* Firmware clock IDs */
#define RPI_FIRMWARE_CLK_EMMC    1
#define RPI_FIRMWARE_CLK_UART    2
#define RPI_FIRMWARE_CLK_ARM     3   /* ARM CPU clock */
#define RPI_FIRMWARE_CLK_CORE    4   /* GPU core clock */
#define RPI_FIRMWARE_CLK_V3D     5   /* 3D graphics */
#define RPI_FIRMWARE_CLK_H264    6
#define RPI_FIRMWARE_CLK_ISP     7   /* Camera ISP */
#define RPI_FIRMWARE_CLK_SDRAM   8
#define RPI_FIRMWARE_CLK_PIXEL   9   /* Display pixel clock */
#define RPI_FIRMWARE_CLK_PWM    10
#define RPI_FIRMWARE_CLK_HEVC   17
#define RPI_FIRMWARE_CLK_EMMC2  18   /* SD card (bcm2711) */

/* Set clock rate via mailbox */
bcm2835_firmware_property(
    firmware,
    RPI_FIRMWARE_SET_CLOCK_RATE,
    &msg, sizeof(msg)
);
```

---

## 6. Debug Clock Framework

```bash
# List all clocks
adb shell cat /sys/kernel/debug/clk/clk_summary
# clock                    enabled  prepared  protected  rate         accuracy phase
# xtal                            1        1          0     54000000         0     0
# pllb                            1        1          0   3000000000         0     0
# pllb_arm                        1        1          0   3000000000         0     0
# arm                             1        1          0   1500000000         0     0
# pllc                            1        1          0   1000000000         0     0
# ...

# Xem clock tree
adb shell cat /sys/kernel/debug/clk/clk_dump

# Specific clock info
adb shell cat /sys/kernel/debug/clk/arm/clk_rate
# 1500000000

# Thông qua vcgencmd (Pi4 firmware)
adb shell vcgencmd measure_clock arm    # ARM clock
adb shell vcgencmd measure_clock core   # GPU core clock
adb shell vcgencmd measure_clock v3d    # V3D clock
adb shell vcgencmd measure_clock emmc   # SD card clock
adb shell vcgencmd measure_clock uart   # UART clock
adb shell vcgencmd measure_clock pixel  # Display pixel clock
```

---

## 7. Clock Rate Setting

```bash
# ARM clock (controlled by cpufreq, usually via firmware)
# Xem section 19_kernel_cpufreq_dvfs.md

# Core (GPU) clock
adb shell vcgencmd set_clock core 500000000   # 500MHz
adb shell vcgencmd get_clock core

# Overclock config.txt (persistent)
# /boot/config.txt:
# arm_freq=1800        ← ARM CPU
# core_freq=750        ← GPU core
# v3d_freq=750         ← V3D GPU
# h264_freq=300        ← H264 codec
# isp_freq=250         ← Camera ISP
# sdram_freq=3200      ← SDRAM

# Emergency: check throttle due to overclock
adb shell vcgencmd get_throttled
# 0x0: no throttle
# 0x50000: throttled (bit 17=ARM freq capped, bit 18=temp limit)
```

---

## 8. Device Tree Clock Binding

```dts
/* arch/arm64/boot/dts/broadcom/bcm2711.dtsi */

clocks: cprman@7e101000 {
    compatible = "brcm,bcm2711-cprman";
    #clock-cells = <1>;         /* one clock ID per phandle */
    reg = <0x7e101000 0x2000>;
    
    clocks = <&xosc>,           /* crystal */
             <&dsi0_byte_clk>, <&dsi0_ddr2_clk>, ...;
};

/* Consumer references clocks via phandle + ID */
uart0: serial@7e201000 {
    compatible = "brcm,bcm2835-pl011";
    clocks = <&clocks BCM2835_CLOCK_UART>,  /* clock for UART */
             <&clocks BCM2835_CLOCK_VPU>;   /* bus clock */
    clock-names = "uartclk", "apb_pclk";
};
```

---

## 9. Clock Prepare/Enable Two-Phase

```c
/* CCF two-phase enable: */
/* Phase 1: clk_prepare() — may sleep, set up PLL */
/* Phase 2: clk_enable() — atomic, cannot sleep */

/* Use: */
clk_prepare_enable(clk);  /* convenience wrapper */

/* Disable: */
clk_disable_unprepare(clk);  /* convenience wrapper */

/* Atomic context: */
clk_enable(clk);   /* only if already prepared */
clk_disable(clk);  /* only disable, not unprepare */

/* Rate change: */
clk_round_rate(clk, rate);   /* what rate is achievable? */
clk_set_rate(clk, rate);     /* set the rate */
clk_get_rate(clk);           /* current rate */
```

---

## 10. File quan trọng trong source

| File | Mô tả |
|------|-------|
| `kernel/common/drivers/clk/bcm/clk-bcm2835.c` | BCM2835/2711 clock driver |
| `kernel/common/drivers/clk/bcm/clk-raspberrypi.c` | Pi firmware clock |
| `kernel/common/drivers/firmware/raspberrypi.c` | Firmware mailbox |
| `kernel/common/include/linux/clk-provider.h` | Clock provider API |
| `kernel/common/include/linux/clk.h` | Clock consumer API |
| `kernel/common/drivers/clk/clk.c` | CCF core |
| `kernel/common/arch/arm64/boot/dts/broadcom/bcm2711.dtsi` | Clock DT nodes |
