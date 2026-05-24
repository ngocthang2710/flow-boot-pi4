# Kernel Regulator Framework — PMIC & Power Domains

## 1. Tổng quan

Regulator framework quản lý voltage và current regulators (PMIC, LDO, DC-DC converters). Pi4 dùng Raspberry Pi PMIC (MXL7704) hoặc discrete regulators.

```
Kernel driver (I2C client driver)
  │ regulator_register()
  ▼
Regulator Core (kernel/drivers/regulator/core.c)
  │ regulator_enable/set_voltage
  ▼
Physical PMIC (via I2C/SPI)
  │
  ├── VDD_ARM  → CPU core voltage (0.8V - 1.1V)
  ├── VDD_GPU  → GPU voltage
  ├── VDD_SDRAM → SDRAM voltage
  ├── 3V3      → 3.3V rail (GPIO, SPI, I2C peripherals)
  └── 1V8      → 1.8V rail (some interfaces)
```

---

## 2. Regulator Types

```
Regulator types:
  REGULATOR_VOLTAGE:  Controls output voltage (LDO, Buck, Boost)
  REGULATOR_CURRENT:  Controls output current (LED driver, charger)

Topology:
  LDO (Low Dropout):  Simple, low efficiency (linear)
  Buck (Step-down):   Switching, high efficiency (V_out < V_in)
  Boost (Step-up):    Switching, V_out > V_in
  Buck-Boost:         V_out can be above or below V_in
```

---

## 3. Regulator Consumer API

```c
#include <linux/regulator/consumer.h>

/* Get regulator */
struct regulator *reg = devm_regulator_get(dev, "vdd");
if (IS_ERR(reg))
    return PTR_ERR(reg);

/* Enable */
ret = regulator_enable(reg);

/* Set voltage range */
ret = regulator_set_voltage(reg,
    1100000,  /* min_uV: 1.1V */
    1200000   /* max_uV: 1.2V */
);

/* Get current voltage */
int uV = regulator_get_voltage(reg);

/* Disable */
regulator_disable(reg);

/* Current limiting */
ret = regulator_set_current_limit(reg,
    500000,  /* min_uA: 500mA */
    1000000  /* max_uA: 1A */
);
```

---

## 4. Regulator Provider (PMIC Driver)

```c
#include <linux/regulator/driver.h>
#include <linux/regulator/machine.h>

/* Describe regulator constraints */
static struct regulator_desc my_reg_desc = {
    .name     = "vdd_cpu",
    .type     = REGULATOR_VOLTAGE,
    .n_voltages = 32,
    .min_uV   = 800000,   /* 0.8V */
    .uV_step  = 25000,    /* 25mV steps */
    .ops      = &my_reg_ops,
    .owner    = THIS_MODULE,
};

/* Operations */
static const struct regulator_ops my_reg_ops = {
    .enable         = my_reg_enable,
    .disable        = my_reg_disable,
    .is_enabled     = my_reg_is_enabled,
    .set_voltage    = my_reg_set_voltage,
    .get_voltage    = my_reg_get_voltage,
    .list_voltage   = regulator_list_voltage_linear,
};

/* Register */
struct regulator_dev *rdev = devm_regulator_register(
    dev, &my_reg_desc, &config);
```

---

## 5. Pi4 Power Architecture

```
Raspberry Pi 4B Power Supply:
  
  USB-C (5V/3A) → MXL7704 PMIC (SPI interface)
    │
    ├── BUCK1 → VDD_CORE_GPU (GPU, VideoCore)
    │          0.87V-1.1V (load/temp dependent)
    │
    ├── BUCK2 → VDD_SDRAM (LPDDR4)
    │          1.1V (LPDDR4X) or 1.2V (LPDDR4)
    │
    ├── BUCK3 → VDD_3V3 (3.3V rail)
    │          USB, GPIO, SD card, HDMI
    │
    ├── BUCK4 → VDD_1V8 (1.8V rail)
    │          Some I/O standards
    │
    └── VDD_ARM → Processor Core
               Via ARM PMU + voltage scaling
               
Note: Pi4 không có dedicated voltage scaling cho ARM
  → ARM voltage tied to core voltage
  → Oversimplified compared to phone SoC
```

---

## 6. Device Tree Regulator

```dts
/* Typical PMIC in DT */
&i2c1 {
    pmic: pmic@1a {
        compatible = "mxl,mxl7704";
        reg = <0x1a>;
        
        regulators {
            vdd_cpu: buck1 {
                regulator-name = "vdd-cpu";
                regulator-min-microvolt = <800000>;
                regulator-max-microvolt = <1100000>;
                regulator-always-on;              /* never disable */
                regulator-boot-on;                /* on at boot */
                regulator-ramp-delay = <500>;     /* 500 uV/us */
            };
            
            vdd_3v3: buck3 {
                regulator-name = "vdd-3v3";
                regulator-min-microvolt = <3300000>;
                regulator-max-microvolt = <3300000>;
                regulator-always-on;
                regulator-boot-on;
            };
        };
    };
};

/* Consumer references regulator */
sd_card: mmc@7e300000 {
    vmmc-supply  = <&vdd_3v3>;   /* VCC power supply */
    vqmmc-supply = <&vdd_1v8>;   /* VCC_IO power supply */
};
```

---

## 7. Power Domains

```c
/* Power domain = group of devices powered together */
/* When all devices in domain idle → domain powers off */

#include <linux/pm_domain.h>

/* Pi4 power domains (VideoCore firmware managed): */
/* RPI_POWER_DOMAIN_I2C0 */
/* RPI_POWER_DOMAIN_I2C1 */
/* RPI_POWER_DOMAIN_USB  */
/* RPI_POWER_DOMAIN_V3D  */  /* GPU */
/* RPI_POWER_DOMAIN_ISP  */  /* Camera */
/* RPI_POWER_DOMAIN_H264 */  /* Video decoder */
/* RPI_POWER_DOMAIN_JPEG */

/* Device tree binding */
/*
v3d: v3d@7ec04000 {
    power-domains = <&pm RPI_POWER_DOMAIN_V3D>;
};
*/

/* Driver: power domain controlled via firmware mailbox */
/* bcm2835-power.c handles Pi power domains */
```

---

## 8. Debug Regulators

```bash
# List all regulators
adb shell cat /sys/kernel/debug/regulator/regulator_summary
# regulator                      use open bypass  min_uV  max_uV  cmin_uA  cmax_uA  opmode
# regulator-dummy                  0    0      0       0       0        0        0   off
# 3v3                              3    3      0 3300000 3300000        0        0   on
# vdd-core                         1    1      0  900000 1100000        0        0   on

# Specific regulator
adb shell ls /sys/class/regulator/
# regulator.0  regulator.1  ...

adb shell cat /sys/class/regulator/regulator.0/name
adb shell cat /sys/class/regulator/regulator.0/microvolts
adb shell cat /sys/class/regulator/regulator.0/state  # enabled/disabled

# Pi4 firmware voltage
adb shell vcgencmd measure_volts core    # GPU/core voltage
adb shell vcgencmd measure_volts sdram_c # SDRAM controller voltage
adb shell vcgencmd measure_volts sdram_i # SDRAM I/O voltage
adb shell vcgencmd measure_volts sdram_p # SDRAM phy voltage
```

---

## 9. Dynamic Voltage and Frequency Scaling (DVFS)

```
DVFS = Voltage + Frequency scale together:
  Higher frequency → needs higher voltage
  Lower voltage → must lower frequency

Pi4 simplified DVFS:
  CPU freq scales via cpufreq governor (see 19_kernel_cpufreq_dvfs.md)
  CPU voltage: fixed by firmware per freq OPP
  
Phone SoC (e.g., Snapdragon 8 Gen 3):
  Multiple voltage rails per cluster
  Fine-grained OPP table
  PMIC negotiation per voltage step
  
Pi4 limitation:
  Less granular → less efficient at low power
  Phone PMIC more sophisticated
```

---

## 10. File quan trọng trong source

| File | Mô tả |
|------|-------|
| `kernel/common/drivers/regulator/core.c` | Regulator framework core |
| `kernel/common/drivers/regulator/fixed.c` | Fixed voltage regulator |
| `kernel/common/drivers/mfd/rpi-sense-hat.c` | Pi sense HAT |
| `kernel/common/drivers/power/bcm2835-power.c` | Pi power domains |
| `kernel/common/include/linux/regulator/consumer.h` | Consumer API |
| `kernel/common/include/linux/regulator/driver.h` | Provider API |
| `kernel/common/arch/arm64/boot/dts/broadcom/bcm2711.dtsi` | Pi4 power DT |
