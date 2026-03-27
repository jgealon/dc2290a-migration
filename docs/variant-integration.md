# DC2290A Variant Integration Guide

## Overview

The DC2290A migration uses a **single unified design** that supports all 6 board variants (A through F) at the hardware, FPGA, and software levels. This document explains how one design can support multiple ADC variants and what adaptations are required.

## Quick Answer

**YES** - We use the **same HDL IP (AXI_LTC2387)** for all 6 variants because:
- All LTC238x ADCs share identical LVDS interface (6-lane, DDR)
- Same digital protocol and timing
- Pin-compatible footprint
- Differences are only: resolution (16-bit vs 18-bit) and max sample rate (5/10/15 Msps)

## Hardware Layer: One Board, Six Variants

### PCB Design Approach

```
┌─────────────────────────────────────────────────────────┐
│        DC2290A FMC Card (Single PCB Design)             │
│                                                          │
│  ┌────────────────────────────────────────────────┐    │
│  │   LTC238x Footprint (Shared)                   │    │
│  │   ┌──────────────────────────────────────┐     │    │
│  │   │  Variant determined by which ADC     │     │    │
│  │   │  part is populated:                  │     │    │
│  │   │                                       │     │    │
│  │   │  Option A: LTC2387-18 installed      │     │    │
│  │   │  Option B: LTC2387-16 installed      │     │    │
│  │   │  Option C: LTC2386-18 installed      │     │    │
│  │   │  Option D: LTC2386-16 installed      │     │    │
│  │   │  Option E: LTC2385-18 installed      │     │    │
│  │   │  Option F: LTC2385-16 installed      │     │    │
│  │   └──────────────────────────────────────┘     │    │
│  │                                                 │    │
│  │  All parts share:                              │    │
│  │  - Same 32-pin package (LQFP-32)               │    │
│  │  - Same pinout                                 │    │
│  │  - Same LVDS interface (6 data lanes)          │    │
│  │  - Same power requirements (1.8V digital)      │    │
│  └────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
```

**Key Point**: Like the legacy DC2290A, the new FMC board has **one PCB design** with 6 SKUs based on which ADC is installed during manufacturing.

### LTC238x Pin Compatibility

All six ADC variants are pin-compatible:

| Pin Function    | LTC2387 | LTC2386 | LTC2385 | Notes |
|-----------------|---------|---------|---------|-------|
| **LVDS Outputs**| 6 lanes | 6 lanes | 6 lanes | Identical |
| **DCO/FCO**     | Yes     | Yes     | Yes     | Clocking |
| **CNV**         | Yes     | Yes     | Yes     | Convert |
| **SDI/SDO/SCK** | Yes     | Yes     | Yes     | SPI config |
| **Power**       | 1.8V    | 1.8V    | 1.8V    | Same rails |

**Result**: The same PCB layout works for all variants.

## FPGA Layer: Single HDL Design for All Variants

### AXI_LTC2387 IP Core

ADI's **AXI_LTC2387 IP core** natively supports the entire LTC238x family:

```verilog
// From ADI HDL Library
// https://github.com/analogdevicesinc/hdl/tree/main/library/axi_ltc2387

module axi_ltc2387 #(
  parameter FPGA_TECHNOLOGY = 0,
  parameter DELAY_REFCLK_FREQUENCY = 200,
  parameter NUM_CHANNELS = 1,
  parameter DATA_WIDTH = 18,        // ← Configurable: 16 or 18
  parameter SAMPLE_RATE = 15000000  // ← Max rate per variant
) (
  // LVDS interface - same for all variants
  input       [NUM_CHANNELS-1:0]  lvds_dco,
  input       [NUM_CHANNELS-1:0]  lvds_fco,
  input       [5:0]               lvds_d,   // 6 lanes (all variants)

  // AXI-Stream output - auto-sized
  output      [DATA_WIDTH-1:0]    adc_data,
  output                          adc_valid,

  // Control
  output                          adc_cnv,
  // ...
);
```

**Key Features**:
- **Configurable data width**: 16-bit or 18-bit via parameter
- **Same LVDS interface**: All variants use 6-lane LVDS
- **Auto-adaptation**: IP handles both resolutions
- **Sample rate flexible**: Software-controlled, respects variant max

### FPGA Integration: One Bitstream, Six Variants

**Option 1: Runtime Configuration (Recommended)**
```tcl
# Vivado project - configure for highest capability (18-bit)
set_property -dict [list \
  CONFIG.DATA_WIDTH {18} \
  CONFIG.NUM_CHANNELS {1} \
] [get_bd_cells axi_ltc2387]

# Software/driver detects actual variant at runtime
# and uses 16 or 18 bits as appropriate
```

**Option 2: Build-Time Variants**
```bash
# Generate 2 bitstreams (one for 16-bit, one for 18-bit variants)
make VARIANT=16bit  # For DC2290A-B/D/F
make VARIANT=18bit  # For DC2290A-A/C/E

# User loads appropriate bitstream for their variant
```

**Recommended**: Option 1 (Runtime) - Single bitstream, software adapts.

### Why One IP Works for All Variants

1. **Identical LVDS Protocol**
   ```
   All LTC238x ADCs use the same LVDS signaling:
   - 6 data lanes (D0-D5)
   - DDR (data on both edges)
   - DCO (data clock output)
   - FCO (frame clock output)
   ```

2. **Data Width Handling**
   ```
   16-bit ADC: Uses bits [17:2], bits [1:0] = 0
   18-bit ADC: Uses all bits [17:0]

   → IP configured for 18-bit captures both cases
   → Software masks off unused bits for 16-bit variants
   ```

3. **Sample Rate Control**
   ```
   All variants: Software controls CNV (convert) rate
   - Driver enforces max rate per variant
   - Same timing logic in FPGA
   ```

## Software Layer: Variant-Aware Stack

### Linux Kernel: Single Driver for All Variants

The **ltc2387.c** IIO driver already supports all LTC238x variants:

```c
// From ADI Linux kernel
// drivers/iio/adc/ltc2387.c

static const struct spi_device_id ltc2387_id[] = {
    { "ltc2387-16", ID_LTC2387_16 },  // DC2290A-B
    { "ltc2387-18", ID_LTC2387_18 },  // DC2290A-A
    { "ltc2386-16", ID_LTC2386_16 },  // DC2290A-D
    { "ltc2386-18", ID_LTC2386_18 },  // DC2290A-C
    { "ltc2385-16", ID_LTC2385_16 },  // DC2290A-F
    { "ltc2385-18", ID_LTC2385_18 },  // DC2290A-E
    {}
};

static const struct of_device_id ltc2387_of_match[] = {
    { .compatible = "adi,ltc2387" },  // Generic - all variants
    { .compatible = "adi,ltc2386" },
    { .compatible = "adi,ltc2385" },
    {}
};

static int ltc2387_probe(struct spi_device *spi) {
    // Auto-detect resolution from device ID
    switch (id) {
    case ID_LTC2387_16:
    case ID_LTC2386_16:
    case ID_LTC2385_16:
        st->resolution = 16;
        break;
    case ID_LTC2387_18:
    case ID_LTC2386_18:
    case ID_LTC2385_18:
        st->resolution = 18;
        break;
    }

    // Set max sample rate per variant
    st->max_sample_rate = ltc238x_max_rates[id];
}
```

**Driver automatically adapts** based on device tree specification.

### Device Tree: Variant Specification

**Approach 1: Generic Compatible String (Recommended)**
```dts
&spi0 {
    ltc2387: adc@0 {
        compatible = "adi,ltc2387";  // Generic, works for all
        reg = <0>;
        spi-max-frequency = <10000000>;

        adi,axi-adc-handle = <&axi_ltc2387>;

        // Optional: Explicit variant identification
        adi,adc-model = "ltc2387-18";        // Actual ADC installed
        adi,board-variant = "dc2290a-a";     // Board SKU
    };
};
```

**Approach 2: Specific Compatible String**
```dts
&spi0 {
    ltc2387: adc@0 {
        compatible = "adi,ltc2387-18";  // Specific to DC2290A-A
        // ... rest same
    };
};
```

**Approach 3: Runtime Detection (Future Enhancement)**
```dts
&spi0 {
    ltc2387: adc@0 {
        compatible = "adi,ltc2387";
        // No model specified
        // Driver reads ADC ID register via SPI to auto-detect variant
    };
};
```

### Variant Detection Flow

```
┌─────────────────────────────────────────────────────────┐
│                     Boot Sequence                        │
└─────────────────────────────────────────────────────────┘
                           │
                           ▼
        ┌──────────────────────────────────────┐
        │  U-Boot loads device tree            │
        │  - Contains variant hint if set      │
        └──────────────────┬───────────────────┘
                           │
                           ▼
        ┌──────────────────────────────────────┐
        │  Kernel boots, ltc2387 driver loads  │
        │  - Reads compatible string           │
        │  - Or reads SPI ID register          │
        └──────────────────┬───────────────────┘
                           │
                           ▼
        ┌──────────────────────────────────────┐
        │  Driver configures per variant:      │
        │  - Resolution: 16 or 18 bits         │
        │  - Max sample rate: 5/10/15 Msps     │
        │  - IIO attributes                    │
        └──────────────────┬───────────────────┘
                           │
                           ▼
        ┌──────────────────────────────────────┐
        │  User space reads IIO sysfs:         │
        │  cat /sys/bus/iio/devices/           │
        │      iio:device0/name                │
        │  → "ltc2387-18"                      │
        └──────────────────────────────────────┘
```

### Python API: Variant-Aware Wrapper

Create a DC2290A-specific wrapper that enforces variant constraints:

```python
# software/python/dc2290a_fmc/dc2290a.py

import iio

class DC2290A:
    """DC2290A FMC board interface supporting all 6 variants"""

    # Variant specifications
    VARIANTS = {
        'dc2290a-a': {'adc': 'ltc2387-18', 'resolution': 18, 'max_rate': 15e6},
        'dc2290a-b': {'adc': 'ltc2387-16', 'resolution': 16, 'max_rate': 15e6},
        'dc2290a-c': {'adc': 'ltc2386-18', 'resolution': 18, 'max_rate': 10e6},
        'dc2290a-d': {'adc': 'ltc2386-16', 'resolution': 16, 'max_rate': 10e6},
        'dc2290a-e': {'adc': 'ltc2385-18', 'resolution': 18, 'max_rate': 5e6},
        'dc2290a-f': {'adc': 'ltc2385-16', 'resolution': 16, 'max_rate': 5e6},
    }

    def __init__(self, variant='auto', uri='local:'):
        """
        Initialize DC2290A board

        Args:
            variant: Board variant ('dc2290a-a' through 'dc2290a-f')
                     or 'auto' to detect from device tree
            uri: IIO context URI
        """
        self.ctx = iio.Context(uri)
        self.dev = self.ctx.find_device('ltc2387')

        if variant == 'auto':
            variant = self._detect_variant()

        if variant not in self.VARIANTS:
            raise ValueError(f"Invalid variant: {variant}")

        self.variant = variant
        self.config = self.VARIANTS[variant]
        self._configure()

    def _detect_variant(self):
        """Auto-detect variant from IIO attributes"""
        # Try to read variant from device tree property
        try:
            variant_attr = self.dev.find_attribute('board_variant')
            if variant_attr:
                return variant_attr.value
        except:
            pass

        # Fall back to detecting from resolution
        resolution = self._read_resolution()
        max_rate = self._read_max_rate()

        # Match against known configs
        for var, cfg in self.VARIANTS.items():
            if cfg['resolution'] == resolution and cfg['max_rate'] == max_rate:
                return var

        raise RuntimeError("Could not auto-detect variant")

    def _read_resolution(self):
        """Read ADC resolution from IIO"""
        chan = self.dev.find_channel('voltage0')
        return chan.attrs['resolution'].value if hasattr(chan.attrs, 'resolution') else 18

    def _read_max_rate(self):
        """Read max sample rate from IIO"""
        if 'max_sampling_frequency' in self.dev.attrs:
            return float(self.dev.attrs['max_sampling_frequency'].value)
        return 15e6  # Default

    def _configure(self):
        """Configure based on variant"""
        self.resolution = self.config['resolution']
        self.max_sample_rate = self.config['max_rate']
        self.adc_model = self.config['adc']

    @property
    def sample_rate(self):
        """Get sample rate in Hz"""
        return float(self.dev.attrs['sampling_frequency'].value)

    @sample_rate.setter
    def sample_rate(self, rate):
        """Set sample rate with variant checking"""
        if rate > self.max_sample_rate:
            raise ValueError(
                f"Rate {rate/1e6:.1f} Msps exceeds max for {self.variant}: "
                f"{self.max_sample_rate/1e6:.1f} Msps"
            )
        self.dev.attrs['sampling_frequency'].value = str(int(rate))

    def capture(self, num_samples, timeout_ms=5000):
        """
        Capture ADC samples

        Returns:
            numpy array with correct bit-width for variant
        """
        import numpy as np

        # Enable channel
        chn = self.dev.find_channel('voltage0')
        chn.enabled = True

        # Create buffer
        buf = iio.Buffer(self.dev, num_samples)

        # Capture
        buf.refill()
        data = np.frombuffer(buf.read(), dtype=np.int32)

        # Mask to actual resolution
        if self.resolution == 16:
            data = data >> 2  # 16-bit in 18-bit field
            data = data & 0xFFFF
        else:
            data = data & 0x3FFFF

        return data

    def get_variant_info(self):
        """Get variant information"""
        return {
            'variant': self.variant,
            'adc_model': self.adc_model,
            'resolution': self.resolution,
            'max_sample_rate': self.max_sample_rate,
        }
```

### Usage Example

```python
from dc2290a_fmc import DC2290A

# Explicit variant
adc_a = DC2290A(variant='dc2290a-a')
print(adc_a.get_variant_info())
# {'variant': 'dc2290a-a', 'adc_model': 'ltc2387-18',
#  'resolution': 18, 'max_sample_rate': 15000000.0}

# Sample rate enforcement
adc_a.sample_rate = 15e6  # OK
adc_a.sample_rate = 20e6  # Raises ValueError

# Capture with correct resolution
data = adc_a.capture(16384)
print(f"Data range: {data.min()} to {data.max()}")
# For 18-bit: 0 to 262143 (2^18-1)

# Different variant
adc_e = DC2290A(variant='dc2290a-e')
adc_e.sample_rate = 5e6   # OK
adc_e.sample_rate = 10e6  # Raises ValueError (exceeds LTC2385 max)
```

## Summary: Integration Strategy

### What Stays the Same (Reusable from CN0577)

| Layer | Component | Status |
|-------|-----------|--------|
| **FPGA** | AXI_LTC2387 IP core | ✅ Use as-is |
| **FPGA** | HDL project structure | ✅ Copy from CN0577 |
| **FPGA** | Bitstream | ✅ One bitstream for all variants |
| **Linux** | ltc2387.c driver | ✅ Already supports all variants |
| **Linux** | IIO framework | ✅ No changes |
| **Python** | pylibiio library | ✅ Use as-is |

### What Needs Adaptation (DC2290A-Specific)

| Layer | Component | Changes Required |
|-------|-----------|------------------|
| **Device Tree** | Board DTS file | ⚠️ Create dc2290a-{a,b,c,d,e,f}.dts variants OR single .dts with variant parameter |
| **Python** | API wrapper | ⚠️ Create DC2290A() class with variant enforcement |
| **Documentation** | User guides | ⚠️ Document variant selection process |
| **Build System** | SD card images | ⚠️ Option: Build 6 images (one per variant) OR one universal image |

## Deployment Options

### Option A: Universal Image (Recommended)

**Single SD card image works for all 6 variants:**

```
kuiper-dc2290a-universal.img
├── BOOT.BIN (single bitstream, supports all variants)
├── devicetree/
│   └── dc2290a.dtb (generic, variant detected at runtime)
└── rootfs/
    └── usr/bin/dc2290a-detect  (script to detect and configure variant)
```

**Pros**: Single image, easy deployment, no user confusion
**Cons**: Requires variant detection mechanism

### Option B: Per-Variant Images

**Six SD card images, one per variant:**

```
kuiper-dc2290a-a.img  (for DC2290A-A boards)
kuiper-dc2290a-b.img  (for DC2290A-B boards)
...
kuiper-dc2290a-f.img  (for DC2290A-F boards)
```

**Pros**: Simpler, no detection needed, optimized per variant
**Cons**: User must select correct image, 6 images to maintain

### Option C: Hybrid

**Base image + variant overlays:**

```
kuiper-dc2290a-base.img  (common base)
├── BOOT.BIN
├── rootfs/
└── overlays/
    ├── dc2290a-a.dtbo (device tree overlay for variant A)
    ├── dc2290a-b.dtbo
    ...
    └── dc2290a-f.dtbo

User selects overlay in /boot/config.txt
```

**Pros**: One base image, easy variant switching
**Cons**: Requires overlay mechanism

## Implementation Checklist

### Phase 1: Hardware (No Changes Needed)
- [x] PCB designed for pin-compatible LTC238x footprint
- [x] All 6 variants use same FMC pinout
- [x] BOM variants documented

### Phase 2: FPGA
- [ ] Copy CN0577 HDL project → dc2290a project
- [ ] Configure AXI_LTC2387 for 18-bit (max capability)
- [ ] Verify LVDS constraints for all variants
- [ ] Build single bitstream
- [ ] Test: Load same bitstream with different ADC variants

### Phase 3: Linux Kernel
- [ ] Create device tree: `zynq-zed-adv7511-dc2290a.dts`
- [ ] Add variant property: `adi,board-variant = "dc2290a-a"`
- [ ] Verify ltc2387 driver loads for all variants
- [ ] Test IIO interface for 16-bit and 18-bit modes

### Phase 4: Python API
- [ ] Create `dc2290a_fmc/` package
- [ ] Implement `DC2290A()` class with variant support
- [ ] Add sample rate enforcement per variant
- [ ] Add resolution masking for 16-bit variants
- [ ] Create examples for each variant
- [ ] Write unit tests

### Phase 5: Documentation
- [ ] Document variant selection process
- [ ] Create quick-start per variant
- [ ] Document API differences between variants
- [ ] Add troubleshooting for variant mismatch

### Phase 6: Build System
- [ ] Decide: Universal image vs per-variant images
- [ ] Create SD card image build scripts
- [ ] Test images with all 6 variants
- [ ] Document image flashing process

## Testing Matrix

Test each layer with all 6 variants:

| Test | DC2290A-A | DC2290A-B | DC2290A-C | DC2290A-D | DC2290A-E | DC2290A-F |
|------|-----------|-----------|-----------|-----------|-----------|-----------|
| **FPGA boots** | ☐ | ☐ | ☐ | ☐ | ☐ | ☐ |
| **Driver loads** | ☐ | ☐ | ☐ | ☐ | ☐ | ☐ |
| **IIO device appears** | ☐ | ☐ | ☐ | ☐ | ☐ | ☐ |
| **Correct resolution detected** | ☐ (18) | ☐ (16) | ☐ (18) | ☐ (16) | ☐ (18) | ☐ (16) |
| **Max rate enforced** | ☐ (15M) | ☐ (15M) | ☐ (10M) | ☐ (10M) | ☐ (5M) | ☐ (5M) |
| **Data capture works** | ☐ | ☐ | ☐ | ☐ | ☐ | ☐ |
| **Python API works** | ☐ | ☐ | ☐ | ☐ | ☐ | ☐ |

## Variant Detection Implementation

**⭐ RECOMMENDED: Hardware Resistor Coding**

The preferred method is **resistor coding** using R33, R34, R35 on the FMC card (similar to legacy DC2290A). This provides automatic, zero-cost variant detection at boot time.

**See**: [Hardware Variant Detection Guide](hardware-variant-detection.md) for complete implementation details.

### Method 1: Device Tree Property (Simple)

User specifies variant when flashing SD card or in boot config:

```bash
# During image creation
export DC2290A_VARIANT=dc2290a-a
./build-image.sh

# Or at boot, U-Boot selects DT
setenv board_variant dc2290a-a
saveenv
```

### Method 2: SPI ID Register Read (Automatic)

Read ADC ID register via SPI to auto-detect:

```c
// In ltc2387.c probe function
static int ltc2387_detect_variant(struct spi_device *spi) {
    uint8_t id_reg;

    // Read ID register (address 0x00)
    ltc2387_spi_read(spi, 0x00, &id_reg);

    // Parse ID: bits [7:4] = family, bits [3:0] = resolution
    switch (id_reg) {
    case 0x78: return ID_LTC2387_18;
    case 0x76: return ID_LTC2387_16;
    case 0x68: return ID_LTC2386_18;
    case 0x66: return ID_LTC2386_16;
    case 0x58: return ID_LTC2385_18;
    case 0x56: return ID_LTC2385_16;
    default:   return -ENODEV;
    }
}
```

### Method 3: Resistor Coding on FMC (RECOMMENDED)

**Resistor coding using R33, R34, R35** provides instant, zero-cost variant identification:

```
3-bit GPIO code using resistor population:
  R33 (0Ω or DNI) → GPIO bit 0
  R34 (0Ω or DNI) → GPIO bit 1
  R35 (0Ω or DNI) → GPIO bit 2

Resistor populated → GPIO reads 0 (pulled to GND)
Resistor DNI → GPIO reads 1 (pulled high)

Example: DC2290A-B has R33=0Ω, R34=DNI, R35=DNI
  → Reads as 011 (binary) = 0x3 = dc2290a-b
```

**Benefits**:
- Zero cost (just resistor population)
- Instant detection at boot
- No external components needed
- Manufacturing friendly

**See**: [Hardware Variant Detection Guide](hardware-variant-detection.md) for complete software stack implementation.

### Method 4: EEPROM on FMC Card (Alternative)

Store variant info in FMC EEPROM:

```
FMC EEPROM:
  Offset 0x00: "DC2290A"
  Offset 0x08: Variant letter ('A', 'B', 'C', 'D', 'E', 'F')
  Offset 0x10: ADC part number
  Offset 0x20: Calibration data
```

Cost: ~$0.50 per board, adds complexity.

## Conclusion

**The answer is YES**: The same HDL IP (AXI_LTC2387), driver (ltc2387.c), and FPGA bitstream work for all 6 DC2290A variants because:

1. **Hardware**: All LTC238x ADCs are pin-compatible with identical LVDS interface
2. **FPGA**: AXI_LTC2387 IP supports configurable data width (16/18-bit)
3. **Software**: Linux driver auto-adapts based on device identification
4. **API**: Python wrapper enforces variant-specific constraints

**Minimal changes needed**:
- Device tree variant specification (or detection)
- Python wrapper class for variant awareness
- Documentation for variant selection

This unified approach dramatically simplifies maintenance compared to having separate designs for each variant.

---

**Document Version**: 1.0
**Last Updated**: 2026-03-27
**Status**: Complete integration guide
