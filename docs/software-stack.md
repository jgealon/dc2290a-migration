# DC2290A Software Stack (Based on CN0577)

Complete software stack documentation for DC2290A migration, based on the proven CN0577 software architecture.

## Overview

The DC2290A software stack follows the CN0577 reference implementation, leveraging ADI's Linux distribution and IIO framework. This document details each layer and highlights where adaptations are needed for DC2290A.

## Software Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     USER APPLICATIONS                           │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │   Python     │  │   MATLAB     │  │     Web      │         │
│  │   Scripts    │  │   Scripts    │  │   Interface  │         │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘         │
│         │                  │                  │                  │
├─────────┼──────────────────┼──────────────────┼─────────────────┤
│         │                  │                  │                  │
│  ┌──────▼──────────────────▼──────────────────▼─────┐          │
│  │          LibIIO / pylibiio Client Library         │          │
│  └────────────────────────┬──────────────────────────┘          │
│                           │                                      │
├───────────────────────────┼──────────────────────────────────────┤
│                           │                                      │
│  ┌────────────────────────▼───────────────────────────┐         │
│  │          IIO Daemon (iiod) - Optional              │         │
│  │          Network access to IIO devices             │         │
│  └────────────────────────┬───────────────────────────┘         │
│                           │                                      │
├───────────────────────────┼──────────────────────────────────────┤
│                    LINUX KERNEL                                  │
│  ┌────────────────────────▼───────────────────────────┐         │
│  │     Industrial I/O (IIO) Subsystem                 │         │
│  │  ┌──────────────┐  ┌──────────────┐  ┌─────────┐  │         │
│  │  │ IIO Core     │  │   Buffers    │  │ Triggers│  │         │
│  │  └──────┬───────┘  └──────┬───────┘  └────┬────┘  │         │
│  │         │                  │               │        │         │
│  │  ┌──────▼──────────────────▼───────────────▼────┐  │         │
│  │  │    LTC2387 IIO Driver (ltc2387.c)            │  │         │
│  │  │    - Sysfs interfaces                         │  │         │
│  │  │    - Buffer management                        │  │         │
│  │  │    - DMA integration                          │  │         │
│  │  └───────────────────┬──────────────────────────┘  │         │
│  └────────────────────────┼─────────────────────────────┘       │
│                           │                                      │
│  ┌────────────────────────▼─────────────────────────────┐       │
│  │    AXI Device Drivers (DMA, SPI, etc.)               │       │
│  └────────────────────────┬─────────────────────────────┘       │
│                           │                                      │
├───────────────────────────┼──────────────────────────────────────┤
│                    HARDWARE LAYER                                │
│  ┌────────────────────────▼─────────────────────────────┐       │
│  │         FPGA (Programmable Logic)                     │       │
│  │  ┌──────────────┐  ┌──────────────┐  ┌───────────┐  │       │
│  │  │ AXI_LTC2387  │  │  AXI_DMAC    │  │AXI_SYSID  │  │       │
│  │  │   IP Core    │  │   DMA        │  │           │  │       │
│  │  └──────┬───────┘  └──────┬───────┘  └───────────┘  │       │
│  └─────────┼──────────────────┼──────────────────────────┘       │
│            │                  │                                  │
│  ┌─────────▼──────────────────▼──────────────────────┐          │
│  │       ARM Processing System (PS)                   │          │
│  │       - DDR3 Memory                                │          │
│  │       - ARM Cortex-A9 x2                           │          │
│  └────────────────────────────────────────────────────┘          │
└─────────────────────────────────────────────────────────────────┘
```

## CN0577 Reference Software Components

### 1. HDL Project
**Reference**: https://github.com/analogdevicesinc/hdl/tree/main/projects/cn0577

**CN0577 Implementation**:
- Project: `hdl/projects/cn0577/`
- Supported carriers: Zed Board
- IP cores: AXI_LTC2387, AXI_DMAC, AXI_SYSID
- Build system: ADI HDL build infrastructure
- Block design: Vivado BD with AXI interconnect

**DC2290A Adaptation Required** ⚠️:
```
Location: hdl/projects/dc2290a/
Changes:
1. Project name: cn0577 → dc2290a
2. Board definition: Use same zed platform
3. IP configuration: Reuse CN0577 IP setup (no changes needed)
4. System_bd.tcl: Copy from CN0577, rename project references
5. Makefile: Update project name

Status: Create new project folder, copy CN0577 structure
```

### 2. Linux Kernel Driver
**Reference**: https://github.com/analogdevicesinc/linux/blob/main/drivers/iio/adc/ltc2387.c

**CN0577 Implementation**:
- Driver: `drivers/iio/adc/ltc2387.c`
- Type: IIO (Industrial I/O) ADC driver
- Compatible string: `"adi,ltc2387"`
- Features:
  - Buffered capture via DMA
  - Sysfs attributes for configuration
  - 16-bit and 18-bit mode support
  - Continuous sampling

**DC2290A Adaptation Required** ⚠️:
```
Location: drivers/iio/adc/ltc2387.c
Changes: NONE - Driver is ADC-specific, not board-specific
        LTC2387/2386/2385 all use same driver

Action: Use CN0577 driver as-is
Verification: Check driver supports all LTC238x variants
```

### 3. Device Tree
**Reference**: https://github.com/analogdevicesinc/linux/blob/main/arch/arm/boot/dts/xilinx/zynq-zed-adv7511-cn0577.dts

**CN0577 Device Tree Structure**:
```dts
/dts-v1/;

#include "zynq-zed.dtsi"
#include "zynq-zed-adv7511.dtsi"

/ {
    model = "Analog Devices CN0577 on Zed";
    compatible = "xlnx,zynq-zed", "xlnx,zynq-7000";
};

// FPGA AXI devices
&axi {
    rx_dma: dma@7c400000 {
        compatible = "adi,axi-dmac-1.00.a";
        reg = <0x7c400000 0x10000>;
        #dma-cells = <1>;
        interrupts = <0 57 IRQ_TYPE_LEVEL_HIGH>;
        clocks = <&clkc 16>;

        adi,channels {
            #size-cells = <0>;
            #address-cells = <1>;
            dma-channel@0 {
                reg = <0>;
                adi,source-bus-width = <16>;
                adi,destination-bus-width = <64>;
                adi,source-bus-type = <2>;
                adi,destination-bus-type = <0>;
            };
        };
    };

    axi_ltc2387: axi-ltc2387@7c420000 {
        compatible = "adi,axi-ltc2387-1.0";
        reg = <0x7c420000 0x10000>;
        dmas = <&rx_dma 0>;
        dma-names = "rx";

        clocks = <&clkc 15>;
    };
};

// IIO ADC device
&spi0 {
    status = "okay";

    ltc2387: adc@0 {
        compatible = "adi,ltc2387";
        reg = <0>;
        spi-max-frequency = <10000000>;

        adi,use-dma;
        adi,axi-adc-handle = <&axi_ltc2387>;
    };
};
```

**DC2290A Adaptation Required** ⚠️:
```
Location: arch/arm/boot/dts/xilinx/zynq-zed-adv7511-dc2290a.dts
Changes:
1. Model string: "Analog Devices CN0577 on Zed"
                 → "Analog Devices DC2290A on Zed"
2. File name: zynq-zed-adv7511-cn0577.dts
              → zynq-zed-adv7511-dc2290a.dts
3. IP addresses: Verify match FPGA design (usually same)
4. ADC compatible: Keep "adi,ltc2387" (supports all variants)
5. Add variant property (optional):
   adi,board-variant = "dc2290a-a";  // Indicates populated ADC

Action: Copy CN0577 DTS, rename, update model string
```

### 4. IIO Driver Documentation
**Reference**: https://wiki.analog.com/resources/tools-software/linux-drivers/iio-adc/ltc2387

**CN0577 Driver Features**:
- Compatible strings: `"adi,ltc2387"`, `"adi,ltc2386"`, `"adi,ltc2385"`
- Sysfs attributes:
  - `in_voltage0_raw` - Read ADC value
  - `in_voltage0_scale` - Voltage scale factor
  - `sampling_frequency` - Sample rate
  - `buffer/enable` - Enable buffered capture
  - `buffer/length` - Buffer size
- DMA support: High-speed continuous capture
- Resolution: Auto-detected (16-bit or 18-bit)

**DC2290A Adaptation Required** ⚠️:
```
Location: Driver documentation and usage
Changes: NONE - Documentation applies directly to DC2290A

Action: Reference CN0577 IIO driver documentation
Note: Add DC2290A-specific examples in user guide
```

## Complete Software Stack Breakdown

### Layer 1: Bootloader (FSBL + U-Boot)

#### First Stage Boot Loader (FSBL)
**CN0577 Reference**: Standard Xilinx FSBL

```c
// Location: hw_platform/ps7_init.c (generated by Vivado)
// Function: Initialize PS (DDR, clocks, peripherals)
```

**DC2290A Adaptation** ⚠️:
```
Changes: NONE - FSBL is generated from Vivado hardware design
Action: Export hardware from DC2290A Vivado project
        Generate FSBL in Vitis
```

#### U-Boot
**CN0577 Reference**: ADI Kuiper Linux U-Boot

```bash
# Boot command
bootcmd=run $modeboot
# Load kernel and device tree
bootargs=console=ttyPS0,115200 root=/dev/mmcblk0p2 rw rootwait
```

**DC2290A Adaptation** ⚠️:
```
Changes: Device tree name in U-Boot environment
From: zynq-zed-adv7511-cn0577.dtb
To:   zynq-zed-adv7511-dc2290a.dtb

Location: u-boot environment variables
Action: Update bootargs to load dc2290a device tree
```

### Layer 2: Linux Kernel

#### Kernel Configuration
**CN0577 Reference**: ADI Linux kernel with IIO enabled

```kconfig
# Required kernel options
CONFIG_IIO=y
CONFIG_IIO_BUFFER=y
CONFIG_IIO_BUFFER_DMA=y
CONFIG_IIO_TRIGGERED_BUFFER=y
CONFIG_LTC2387=m
CONFIG_AXI_DMAC=y
```

**DC2290A Adaptation** ⚠️:
```
Changes: NONE - Same kernel configuration as CN0577
Action: Use ADI Linux kernel branch with LTC2387 driver
```

#### LTC2387 Driver Details

**Driver File**: `drivers/iio/adc/ltc2387.c`

```c
// Supported device IDs
static const struct spi_device_id ltc2387_id[] = {
    { "ltc2387-16", ID_LTC2387_16 },
    { "ltc2387-18", ID_LTC2387_18 },
    { "ltc2386-16", ID_LTC2386_16 },
    { "ltc2386-18", ID_LTC2386_18 },
    { "ltc2385-16", ID_LTC2385_16 },
    { "ltc2385-18", ID_LTC2385_18 },
    {}
};

// Device tree compatible strings
static const struct of_device_id ltc2387_of_match[] = {
    { .compatible = "adi,ltc2387" },
    { .compatible = "adi,ltc2386" },
    { .compatible = "adi,ltc2385" },
    {}
};

// IIO channels
static const struct iio_chan_spec ltc2387_channels[] = {
    {
        .type = IIO_VOLTAGE,
        .indexed = 1,
        .channel = 0,
        .info_mask_separate = BIT(IIO_CHAN_INFO_RAW) |
                              BIT(IIO_CHAN_INFO_SCALE),
        .scan_index = 0,
        .scan_type = {
            .sign = 's',
            .realbits = 18,  // Or 16, detected at runtime
            .storagebits = 32,
            .shift = 0,
            .endianness = IIO_BE,
        },
    },
};
```

**DC2290A Adaptation** ⚠️:
```
Changes: NONE - Driver supports all DC2290A variants
Note: Driver auto-detects 16-bit vs 18-bit from device tree or SPI ID
```

#### Device Tree Bindings

**Complete DC2290A Device Tree**:

```dts
/dts-v1/;

#include "zynq-zed.dtsi"
#include "zynq-zed-adv7511.dtsi"

/ {
    model = "Analog Devices DC2290A on Zed";
    compatible = "adi,dc2290a", "xlnx,zynq-zed", "xlnx,zynq-7000";

    chosen {
        bootargs = "console=ttyPS0,115200 earlycon root=/dev/mmcblk0p2 rw rootwait";
        stdout-path = "serial0:115200n8";
    };

    aliases {
        serial0 = &uart1;
        spi0 = &spi0;
    };
};

&fpga_axi {
    rx_dma: dma@7c400000 {
        compatible = "adi,axi-dmac-1.00.a";
        reg = <0x7c400000 0x10000>;
        #dma-cells = <1>;
        interrupts = <0 57 IRQ_TYPE_LEVEL_HIGH>;
        clocks = <&clkc 16>;

        adi,channels {
            #size-cells = <0>;
            #address-cells = <1>;

            dma-channel@0 {
                reg = <0>;
                adi,source-bus-width = <32>;
                adi,destination-bus-width = <64>;
                adi,source-bus-type = <2>;  // FIFO
                adi,destination-bus-type = <0>;  // Memory
            };
        };
    };

    axi_ltc2387: axi-ltc2387@7c420000 {
        compatible = "adi,axi-ltc2387-1.0";
        reg = <0x7c420000 0x10000>;
        dmas = <&rx_dma 0>;
        dma-names = "rx";
        clocks = <&clkc 15>;

        #io-channel-cells = <1>;
    };

    axi_sysid_0: axi-sysid-0@7c440000 {
        compatible = "adi,axi-sysid-1.00.a";
        reg = <0x7c440000 0x10000>;
    };
};

&spi0 {
    status = "okay";
    num-cs = <4>;
    is-decoded-cs = <0>;

    ltc2387@0 {
        compatible = "adi,ltc2387";  // Generic, works for 2387/2386/2385
        reg = <0>;
        spi-max-frequency = <10000000>;
        spi-cpol;
        spi-cpha;

        /* Link to AXI ADC core for DMA */
        adi,axi-adc-handle = <&axi_ltc2387>;

        /* Optional: Specify exact variant for documentation */
        adi,adc-model = "ltc2387-18";  // Update per DC2290A variant
        adi,board-variant = "dc2290a-a";  // Indicates DC2290A-A board

        /* Reference voltage in millivolts */
        adi,vref-mv = <5000>;
    };
};

&uart1 {
    status = "okay";
};

&usb0 {
    status = "okay";
    dr_mode = "host";
};

&sdhci0 {
    status = "okay";
};

&gem0 {
    status = "okay";
    phy-mode = "rgmii-id";
    phy-handle = <&ethernet_phy>;

    ethernet_phy: ethernet-phy@0 {
        reg = <0>;
    };
};
```

**DC2290A Customization Points** ⚠️:
```
1. Model string: Update to "DC2290A on Zed"
2. Compatible: Add "adi,dc2290a" for identification
3. adi,adc-model: Set to actual ADC (ltc2387-18, ltc2387-16, etc.)
4. adi,board-variant: Set to DC2290A variant (dc2290a-a through dc2290a-f)
5. Memory addresses: Verify against DC2290A FPGA design
```

### Layer 3: User Space Software

#### IIO Tools (Command Line)

**CN0577 Usage**:
```bash
# List IIO devices
iio_info

# Read device attributes
iio_attr -d ltc2387

# Capture data
iio_readdev -b 16384 ltc2387 voltage0 > capture.bin
```

**DC2290A Adaptation** ⚠️:
```
Changes: NONE - Commands work identically
Action: Use same IIO tools from CN0577 examples
```

#### Python (pylibiio)

**CN0577 Example**:
```python
import iio

# Create context
ctx = iio.Context('local:')

# Get ADC device
dev = ctx.find_device('ltc2387')

# Read attributes
sample_rate = dev.attrs['sampling_frequency'].value
print(f"Sample rate: {sample_rate} Hz")

# Enable channels and buffers
chn = dev.find_channel('voltage0')
chn.enabled = True

# Create buffer
buf = iio.Buffer(dev, 16384)

# Read data
buf.refill()
data = buf.read()
```

**DC2290A Adaptation** ⚠️:
```
Changes: Minimal - Add DC2290A-specific class wrapper
Location: software/python/dc2290a_fmc/

Example wrapper:
class DC2290A:
    def __init__(self, variant='dc2290a-a', uri='local:'):
        self.variant = variant
        self.ctx = iio.Context(uri)
        self.dev = self.ctx.find_device('ltc2387')
        self._configure_variant()

    def _configure_variant(self):
        # Set variant-specific parameters
        variant_configs = {
            'dc2290a-a': {'adc': 'ltc2387-18', 'max_rate': 15e6},
            'dc2290a-b': {'adc': 'ltc2387-16', 'max_rate': 15e6},
            # ... etc
        }
        config = variant_configs[self.variant]
        self.adc_model = config['adc']
        self.max_sample_rate = config['max_rate']
```

## Software Build Process

### 1. HDL Build (FPGA Bitstream)

**CN0577 Build**:
```bash
cd hdl/projects/cn0577/zed
make
```

**DC2290A Build** ⚠️:
```bash
# Create DC2290A project
cd hdl/projects/
cp -r cn0577 dc2290a

# Update project files
cd dc2290a/
# Edit system_project.tcl: set project_name "dc2290a"
# Edit Makefile: PROJECT_NAME = dc2290a

# Build
cd zed/
make

# Output: dc2290a_zed.bit, system_top.xsa
```

### 2. Linux Kernel Build

**CN0577 Kernel**:
```bash
git clone https://github.com/analogdevicesinc/linux.git
cd linux
git checkout main  # Or specific ADI release branch

# Configure
make ARCH=arm zynq_xcomm_adv7511_defconfig

# Enable LTC2387
make ARCH=arm menuconfig
# Device Drivers → Industrial I/O → ADC drivers → LTC2387

# Build
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- zImage dtbs modules
```

**DC2290A Adaptation** ⚠️:
```
Changes:
1. Add DC2290A device tree: zynq-zed-adv7511-dc2290a.dts
2. Update arch/arm/boot/dts/xilinx/Makefile to build DC2290A DTB

Action:
# Copy and modify device tree
cp arch/arm/boot/dts/xilinx/zynq-zed-adv7511-cn0577.dts \
   arch/arm/boot/dts/xilinx/zynq-zed-adv7511-dc2290a.dts

# Edit dc2290a.dts with DC2290A-specific settings

# Add to Makefile
echo "dtb-$(CONFIG_ARCH_ZYNQ) += zynq-zed-adv7511-dc2290a.dtb" >> \
    arch/arm/boot/dts/xilinx/Makefile

# Build
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- dtbs
```

### 3. Root Filesystem

**CN0577 Rootfs**: ADI Kuiper Linux
```bash
# Use pre-built Kuiper image or build with Buildroot/Yocto
# Includes: IIO tools, pylibiio, libad9361-iio, etc.
```

**DC2290A Adaptation** ⚠️:
```
Changes: NONE - Use same ADI Kuiper rootfs
Action: Add DC2290A-specific Python scripts to /usr/local/bin/
```

### 4. Boot Image (BOOT.BIN)

**Creation**:
```bash
# Combine FSBL, bitstream, U-Boot
bootgen -image boot.bif -o BOOT.BIN

# boot.bif contents:
the_ROM_image:
{
    [bootloader] fsbl.elf
    [load=0x00100000] system_top.bit
    u-boot.elf
}
```

**DC2290A Adaptation** ⚠️:
```
Changes:
1. Use DC2290A bitstream (dc2290a_zed.bit)
2. Generate FSBL from DC2290A Vivado project

Action: Standard Zynq boot flow, replace bitstream
```

## Software Adaptation Summary

### ✅ No Changes Required (Reuse from CN0577)
1. **LTC2387 IIO Driver** - Works for all LTC238x variants
2. **AXI DMA Driver** - Standard ADI driver
3. **IIO Framework** - Kernel subsystem
4. **LibIIO / pylibiio** - User-space libraries
5. **IIO Tools** - Command-line utilities
6. **Kernel Configuration** - Same config as CN0577

### ⚠️ Adaptations Required (DC2290A-Specific)
1. **HDL Project Name**
   - Location: `hdl/projects/dc2290a/`
   - Change: Project name cn0577 → dc2290a

2. **Device Tree File**
   - Location: `arch/arm/boot/dts/xilinx/zynq-zed-adv7511-dc2290a.dts`
   - Changes:
     - Model string: "DC2290A on Zed"
     - Board variant property: `adi,board-variant = "dc2290a-a"`
     - ADC model property: `adi,adc-model = "ltc2387-18"`

3. **Python Wrapper Class**
   - Location: `software/python/dc2290a_fmc/`
   - Purpose: Add variant-aware interface
   - Changes: Create DC2290A() class wrapping pylibiio

4. **Documentation**
   - Location: `docs/`
   - Changes: Add DC2290A-specific examples and guides

### 🔧 Optional Enhancements (Not in CN0577)
1. **Variant Auto-Detection**
   - Read ADC ID via SPI at boot
   - Update device tree property dynamically

2. **Web Interface**
   - Add DC2290A board selection UI
   - Show variant-specific specs

3. **Calibration Storage**
   - Store per-variant calibration data
   - EEPROM or flash storage

## Testing and Validation

### Layer-by-Layer Test Plan

#### 1. FPGA/Hardware Layer
```bash
# Test AXI registers accessible
devmem 0x7c420000  # Should read AXI_LTC2387 version

# Test DMA
cat /sys/class/dma/dma0chan0/bytes_transferred
```

#### 2. Kernel Driver Layer
```bash
# Check driver loaded
lsmod | grep ltc2387

# Check device enumeration
ls /sys/bus/iio/devices/iio:device0/

# Read attributes
cat /sys/bus/iio/devices/iio:device0/name
# Expected: ltc2387
```

#### 3. IIO Layer
```bash
# IIO info
iio_info | grep ltc2387

# Test single capture
cat /sys/bus/iio/devices/iio:device0/in_voltage0_raw

# Test buffered capture
iio_readdev -b 1024 ltc2387 voltage0 > /dev/null
# Should complete without errors
```

#### 4. Application Layer
```python
# Python test
from dc2290a_fmc import DC2290A
adc = DC2290A(variant='dc2290a-a')
data = adc.capture(16384)
assert len(data) == 16384
print("✓ DC2290A software stack validated")
```

## References

### CN0577 Software Resources
- **HDL Project**: https://github.com/analogdevicesinc/hdl/tree/main/projects/cn0577
- **Linux Driver**: https://github.com/analogdevicesinc/linux/blob/main/drivers/iio/adc/ltc2387.c
- **Device Tree**: https://github.com/analogdevicesinc/linux/blob/main/arch/arm/boot/dts/xilinx/zynq-zed-adv7511-cn0577.dts
- **Driver Documentation**: https://wiki.analog.com/resources/tools-software/linux-drivers/iio-adc/ltc2387

### ADI Software Repositories
- **HDL**: https://github.com/analogdevicesinc/hdl
- **Linux Kernel**: https://github.com/analogdevicesinc/linux
- **LibIIO**: https://github.com/analogdevicesinc/libiio
- **PyADI-IIO**: https://github.com/analogdevicesinc/pyadi-iio

### Documentation
- **IIO Documentation**: https://www.kernel.org/doc/html/latest/driver-api/iio/
- **Zynq TRM**: https://www.xilinx.com/support/documentation/user_guides/ug585-Zynq-7000-TRM.pdf
- **ADI Wiki**: https://wiki.analog.com/

## Appendix: Complete File Locations

### DC2290A Software Files

```
dc2290a-migration/
│
├── hdl/projects/dc2290a/           ⚠️ NEW - Copy from CN0577
│   ├── zed/
│   │   ├── Makefile
│   │   └── system_project.tcl
│   └── common/
│       └── dc2290a_bd.tcl
│
├── linux/                          ⚠️ Fork ADI Linux
│   ├── arch/arm/boot/dts/xilinx/
│   │   └── zynq-zed-adv7511-dc2290a.dts  ⚠️ NEW
│   └── drivers/iio/adc/
│       └── ltc2387.c               ✅ USE AS-IS from CN0577
│
└── software/                       ⚠️ NEW Application layer
    ├── python/dc2290a_fmc/
    │   ├── __init__.py
    │   ├── dc2290a.py              ⚠️ Wrapper class
    │   └── examples/
    ├── web/
    └── scripts/
```

**Legend**:
- ✅ **USE AS-IS**: Reuse from CN0577 without modification
- ⚠️ **NEW/ADAPT**: Create new or adapt from CN0577
- 🔧 **OPTIONAL**: Enhancement beyond CN0577

---

**Document Version**: 1.0
**Last Updated**: 2026-03-26
**Based On**: CN0577 Reference Design Software Architecture
