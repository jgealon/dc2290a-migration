# DC2290A Software Adaptations for 6-Variant Support

## Overview

CN0577 reference design only supports **LTC2387-18** (18-bit, 15 Msps). The DC2290A FMC migration must support **all 6 ADC variants**. This document details the necessary software stack changes to support all variants.

## Variant Differences Summary

| Variant | ADC Part | Resolution | Max Rate | Conversion Time | DCO Frequency |
|---------|----------|------------|----------|-----------------|---------------|
| DC2290A-A | LTC2387-18 | 18-bit | 15 Msps | 66.7 ns | 90 MHz |
| DC2290A-B | LTC2387-16 | 16-bit | 15 Msps | 66.7 ns | 90 MHz |
| DC2290A-C | LTC2386-18 | 18-bit | 10 Msps | 100 ns | 60 MHz |
| DC2290A-D | LTC2386-16 | 16-bit | 10 Msps | 100 ns | 60 MHz |
| DC2290A-E | LTC2385-18 | 18-bit | 5 Msps | 200 ns | 30 MHz |
| DC2290A-F | LTC2385-16 | 16-bit | 5 Msps | 200 ns | 30 MHz |

**Key Differences**:
- **Resolution**: 16-bit vs 18-bit (affects data format and scaling)
- **Sample Rate**: 15/10/5 Msps (affects timing and throughput)
- **DCO Clock**: 90/60/30 MHz (affects FPGA timing constraints)

---

## 1. Linux Kernel Driver Changes

### Current State (CN0577)
The existing `ltc2387.c` driver only supports LTC2387-18:

```c
static const struct of_device_id ltc2387_of_match[] = {
    { .compatible = "adi,ltc2387" },
    {}
};
```

### Required Changes

#### A. Add Chip Info Structure
Define variant-specific parameters:

```c
struct ltc2387_chip_info {
    const char *name;
    unsigned int resolution;      /* 16 or 18 bits */
    unsigned int max_rate;        /* Hz */
    unsigned int dco_frequency;   /* Hz */
    unsigned int conv_time_ns;    /* nanoseconds */
};

static const struct ltc2387_chip_info ltc2387_18_info = {
    .name = "ltc2387-18",
    .resolution = 18,
    .max_rate = 15000000,
    .dco_frequency = 90000000,
    .conv_time_ns = 67,
};

static const struct ltc2387_chip_info ltc2387_16_info = {
    .name = "ltc2387-16",
    .resolution = 16,
    .max_rate = 15000000,
    .dco_frequency = 90000000,
    .conv_time_ns = 67,
};

static const struct ltc2387_chip_info ltc2386_18_info = {
    .name = "ltc2386-18",
    .resolution = 18,
    .max_rate = 10000000,
    .dco_frequency = 60000000,
    .conv_time_ns = 100,
};

static const struct ltc2387_chip_info ltc2386_16_info = {
    .name = "ltc2386-16",
    .resolution = 16,
    .max_rate = 10000000,
    .dco_frequency = 60000000,
    .conv_time_ns = 100,
};

static const struct ltc2387_chip_info ltc2385_18_info = {
    .name = "ltc2385-18",
    .resolution = 18,
    .max_rate = 5000000,
    .dco_frequency = 30000000,
    .conv_time_ns = 200,
};

static const struct ltc2387_chip_info ltc2385_16_info = {
    .name = "ltc2385-16",
    .resolution = 16,
    .max_rate = 5000000,
    .dco_frequency = 30000000,
    .conv_time_ns = 200,
};
```

#### B. Update Device ID Table
Add all 6 variants:

```c
static const struct of_device_id ltc2387_of_match[] = {
    { .compatible = "adi,ltc2387-18", .data = &ltc2387_18_info },
    { .compatible = "adi,ltc2387-16", .data = &ltc2387_16_info },
    { .compatible = "adi,ltc2386-18", .data = &ltc2386_18_info },
    { .compatible = "adi,ltc2386-16", .data = &ltc2386_16_info },
    { .compatible = "adi,ltc2385-18", .data = &ltc2385_18_info },
    { .compatible = "adi,ltc2385-16", .data = &ltc2385_16_info },
    { .compatible = "adi,ltc2387", .data = &ltc2387_18_info },  /* Legacy */
    {}
};
MODULE_DEVICE_TABLE(of, ltc2387_of_match);
```

#### C. Update IIO Channel Definition
Adjust based on resolution:

```c
static int ltc2387_read_raw(struct iio_dev *indio_dev,
                            struct iio_chan_spec const *chan,
                            int *val, int *val2, long mask)
{
    struct ltc2387_state *st = iio_priv(indio_dev);

    switch (mask) {
    case IIO_CHAN_INFO_RAW:
        /* Read based on resolution */
        if (st->chip_info->resolution == 18) {
            *val = /* 18-bit data */;
        } else {
            *val = /* 16-bit data */;
        }
        return IIO_VAL_INT;

    case IIO_CHAN_INFO_SCALE:
        /* Scale based on resolution */
        if (st->chip_info->resolution == 18) {
            *val = 5000;  /* mV */
            *val2 = 18;   /* bits */
        } else {
            *val = 5000;
            *val2 = 16;
        }
        return IIO_VAL_FRACTIONAL_LOG2;

    case IIO_CHAN_INFO_SAMP_FREQ:
        *val = st->sampling_freq;
        return IIO_VAL_INT;

    default:
        return -EINVAL;
    }
}
```

#### D. Update Sampling Frequency Validation
Check against variant-specific max rate:

```c
static int ltc2387_write_raw(struct iio_dev *indio_dev,
                             struct iio_chan_spec const *chan,
                             int val, int val2, long mask)
{
    struct ltc2387_state *st = iio_priv(indio_dev);

    switch (mask) {
    case IIO_CHAN_INFO_SAMP_FREQ:
        if (val > st->chip_info->max_rate) {
            dev_err(&st->spi->dev,
                    "Requested rate %d Hz exceeds max %d Hz for %s\n",
                    val, st->chip_info->max_rate, st->chip_info->name);
            return -EINVAL;
        }
        st->sampling_freq = val;
        return 0;

    default:
        return -EINVAL;
    }
}
```

#### E. Update Buffer Scan Type
Adjust realbits and storagebits:

```c
static int ltc2387_probe(struct spi_device *spi)
{
    struct ltc2387_state *st;
    struct iio_dev *indio_dev;
    const struct of_device_id *match;

    match = of_match_device(ltc2387_of_match, &spi->dev);
    if (!match)
        return -ENODEV;

    st->chip_info = match->data;

    /* Configure IIO channels based on resolution */
    indio_dev->channels = ltc2387_channels;
    indio_dev->num_channels = ARRAY_SIZE(ltc2387_channels);

    /* Adjust scan_type based on resolution */
    if (st->chip_info->resolution == 18) {
        ltc2387_channels[0].scan_type.realbits = 18;
        ltc2387_channels[0].scan_type.storagebits = 32;
    } else {
        ltc2387_channels[0].scan_type.realbits = 16;
        ltc2387_channels[0].scan_type.storagebits = 16;
    }

    /* Set max sample rate */
    st->max_rate = st->chip_info->max_rate;

    return iio_device_register(indio_dev);
}
```

**File to Modify**: `linux/drivers/iio/adc/ltc2387.c`

---

## 2. Device Tree Changes

### Current State (CN0577)
Only specifies LTC2387-18:

```dts
&axi_ltc2387 {
    compatible = "adi,axi-ltc2387-1.0";
};

ltc2387@0 {
    compatible = "adi,ltc2387";
    reg = <0>;
};
```

### Required Changes

#### A. Add Variant-Specific Properties
Create device tree for each DC2290A variant:

```dts
/* DC2290A-A: LTC2387-18, 15 Msps */
&fmc_spi {
    ltc2387_18: adc@0 {
        compatible = "adi,ltc2387-18";
        reg = <0>;
        spi-max-frequency = <10000000>;

        /* Board and ADC identification */
        adi,board-variant = "dc2290a-a";
        adi,adc-model = "ltc2387-18";

        /* Link to AXI IP core */
        adi,axi-adc-handle = <&axi_ltc2387>;

        /* ADC configuration */
        adi,vref-mv = <5000>;
        adi,resolution-bits = <18>;
        adi,max-sample-rate = <15000000>;
    };
};

&axi_ltc2387 {
    compatible = "adi,axi-ltc2387-1.0";
    reg = <0x44a00000 0x10000>;

    /* Configure for 18-bit variant */
    adi,data-width = <18>;
    adi,num-channels = <1>;
};
```

#### B. Create DTS Overlay for Each Variant
**File**: `arch/arm/boot/dts/xilinx/zynq-zed-dc2290a-a.dts` (through F)

```dts
/dts-v1/;
/plugin/;

/ {
    compatible = "xlnx,zynq-zed", "xlnx,zynq-7000";
};

/* DC2290A-A Overlay (LTC2387-18) */
&fmc_spi {
    status = "okay";

    dc2290a_adc: adc@0 {
        compatible = "adi,ltc2387-18";
        reg = <0>;
        spi-max-frequency = <10000000>;

        adi,board-variant = "dc2290a-a";
        adi,adc-model = "ltc2387-18";
        adi,axi-adc-handle = <&axi_ltc2387>;
        adi,vref-mv = <5000>;
        adi,resolution-bits = <18>;
        adi,max-sample-rate = <15000000>;
    };
};

&axi_ltc2387 {
    status = "okay";
    adi,data-width = <18>;
};
```

**Repeat for all variants** with appropriate compatible strings and parameters.

**Files to Create**:
- `zynq-zed-dc2290a-a.dts` → LTC2387-18
- `zynq-zed-dc2290a-b.dts` → LTC2387-16
- `zynq-zed-dc2290a-c.dts` → LTC2386-18
- `zynq-zed-dc2290a-d.dts` → LTC2386-16
- `zynq-zed-dc2290a-e.dts` → LTC2385-18
- `zynq-zed-dc2290a-f.dts` → LTC2385-16

---

## 3. HDL/FPGA Configuration Changes

### Current State (CN0577)
Fixed configuration for LTC2387-18:

```tcl
set_property -dict [list \
    CONFIG.DATA_WIDTH {18} \
    CONFIG.NUM_CHANNELS {1} \
] [get_bd_cells axi_ltc2387_0]
```

### Required Changes

#### A. Parameterize Data Width
Create different FPGA builds for 16-bit vs 18-bit variants:

**File**: `fpga/projects/dc2290a/system_bd.tcl`

```tcl
# Get data width from environment or default to 18
if {[info exists ::env(ADC_DATA_WIDTH)]} {
    set data_width $::env(ADC_DATA_WIDTH)
} else {
    set data_width 18
}

# Configure AXI_LTC2387 IP core
set_property -dict [list \
    CONFIG.DATA_WIDTH $data_width \
    CONFIG.NUM_CHANNELS {1} \
    CONFIG.LVDS_CMOS_N {1} \
] [get_bd_cells axi_ltc2387_0]

puts "INFO: Configured AXI_LTC2387 for ${data_width}-bit data width"
```

#### B. Create Build Variants
**File**: `fpga/projects/dc2290a/Makefile`

```makefile
# Build all variants
all: build_18bit build_16bit

build_18bit:
	$(MAKE) ADC_DATA_WIDTH=18 VARIANT=18bit build
	cp system_top.bit system_top_18bit.bit

build_16bit:
	$(MAKE) ADC_DATA_WIDTH=16 VARIANT=16bit build
	cp system_top.bit system_top_16bit.bit

# Individual variant targets
dc2290a-a dc2290a-b: build_18bit
dc2290a-c dc2290a-d dc2290a-e dc2290a-f: build_16bit
```

#### C. Update Constraints
Adjust timing for different DCO frequencies:

**File**: `fpga/projects/dc2290a/system_constr.xdc`

```tcl
# DCO clock constraints vary by ADC variant
# LTC2387: 90 MHz
# LTC2386: 60 MHz
# LTC2385: 30 MHz

# Get DCO frequency from environment
if {[info exists ::env(DCO_FREQ_MHZ)]} {
    set dco_freq_mhz $::env(DCO_FREQ_MHZ)
} else {
    set dco_freq_mhz 90
}

set dco_period [expr {1000.0 / $dco_freq_mhz}]

create_clock -period $dco_period -name dco_clk [get_ports fmc_la_p[6]]

puts "INFO: DCO clock configured for ${dco_freq_mhz} MHz (period ${dco_period} ns)"
```

**Files to Modify**:
- `hdl/projects/dc2290a/system_bd.tcl`
- `hdl/projects/dc2290a/system_constr.xdc`
- `hdl/projects/dc2290a/Makefile`

---

## 4. Python API Changes

### Current State (CN0577)
Hardcoded for LTC2387-18:

```python
class CN0577:
    def __init__(self):
        self.resolution = 18
        self.max_rate = 15000000
```

### Required Changes

#### A. Add Variant Detection
**File**: `software/python/dc2290a_fmc/dc2290a.py`

```python
class DC2290A:
    """DC2290A FMC evaluation board supporting all 6 ADC variants"""

    # Variant specifications
    VARIANTS = {
        'dc2290a-a': {'adc': 'ltc2387-18', 'bits': 18, 'max_rate': 15e6},
        'dc2290a-b': {'adc': 'ltc2387-16', 'bits': 16, 'max_rate': 15e6},
        'dc2290a-c': {'adc': 'ltc2386-18', 'bits': 18, 'max_rate': 10e6},
        'dc2290a-d': {'adc': 'ltc2386-16', 'bits': 16, 'max_rate': 10e6},
        'dc2290a-e': {'adc': 'ltc2385-18', 'bits': 18, 'max_rate': 5e6},
        'dc2290a-f': {'adc': 'ltc2385-16', 'bits': 16, 'max_rate': 5e6},
    }

    def __init__(self, uri='local:', variant=None):
        """
        Initialize DC2290A board

        Args:
            uri: IIO context URI (default: 'local:')
            variant: Board variant (dc2290a-a through dc2290a-f)
                     If None, auto-detect from device tree
        """
        self.ctx = iio.Context(uri)
        self.dev = self.ctx.find_device('ltc2387')

        # Auto-detect variant from device tree if not specified
        if variant is None:
            variant = self._detect_variant()

        if variant not in self.VARIANTS:
            raise ValueError(f"Invalid variant: {variant}")

        self.variant = variant
        self.spec = self.VARIANTS[variant]
        self.resolution = self.spec['bits']
        self.max_rate = self.spec['max_rate']

        print(f"Initialized {variant}: {self.spec['adc']}, "
              f"{self.resolution}-bit, {self.max_rate/1e6:.0f} Msps")

    def _detect_variant(self):
        """Auto-detect board variant from device tree"""
        try:
            # Read from IIO device attribute
            variant_attr = self.dev.find_attr('board_variant')
            if variant_attr:
                return variant_attr.value
        except:
            pass

        # Try reading from device tree directly
        try:
            with open('/proc/device-tree/amba/spi@e0006000/adc@0/adi,board-variant', 'r') as f:
                return f.read().strip('\x00')
        except:
            pass

        # Default to variant A if detection fails
        print("Warning: Could not auto-detect variant, defaulting to dc2290a-a")
        return 'dc2290a-a'

    @property
    def sample_rate(self):
        """Get current sample rate"""
        return self.dev.channels[0].attrs['sampling_frequency'].value

    @sample_rate.setter
    def sample_rate(self, rate):
        """Set sample rate with validation"""
        if rate > self.max_rate:
            raise ValueError(f"Rate {rate} Hz exceeds max {self.max_rate} Hz for {self.variant}")
        self.dev.channels[0].attrs['sampling_frequency'].value = str(int(rate))

    def capture(self, num_samples):
        """Capture data samples"""
        buf = iio.Buffer(self.dev, num_samples)
        buf.refill()

        # Read and convert based on resolution
        if self.resolution == 18:
            data = np.frombuffer(buf.read(), dtype=np.int32)
            # Sign extend from 18-bit to 32-bit
            data = np.where(data & 0x20000, data | 0xFFFC0000, data)
        else:
            data = np.frombuffer(buf.read(), dtype=np.int16)

        # Convert to voltage
        voltage = data * (5.0 / (2 ** self.resolution))
        return voltage

    def calculate_enob(self, data):
        """Calculate ENOB accounting for resolution"""
        # FFT analysis
        fft = np.fft.fft(data * np.hanning(len(data)))
        magnitude = np.abs(fft)[:len(fft)//2]

        # Find signal and noise
        signal_bin = np.argmax(magnitude[1:]) + 1
        signal_power = magnitude[signal_bin] ** 2

        noise_power = np.sum(magnitude ** 2) - signal_power
        snr = 10 * np.log10(signal_power / noise_power)

        # ENOB = (SNR - 1.76) / 6.02
        enob = (snr - 1.76) / 6.02
        return enob
```

#### B. Update Example Scripts
**File**: `software/python/examples/basic_capture.py`

```python
#!/usr/bin/env python3
"""Basic data capture for all DC2290A variants"""

import argparse
from dc2290a_fmc import DC2290A

def main():
    parser = argparse.ArgumentParser(description='DC2290A basic capture')
    parser.add_argument('--variant', choices=[
        'dc2290a-a', 'dc2290a-b', 'dc2290a-c',
        'dc2290a-d', 'dc2290a-e', 'dc2290a-f'
    ], help='Board variant (auto-detect if not specified)')
    parser.add_argument('--rate', type=float, help='Sample rate in Hz')
    parser.add_argument('--samples', type=int, default=16384, help='Number of samples')
    args = parser.parse_args()

    # Initialize with variant (or auto-detect)
    adc = DC2290A(variant=args.variant)

    # Set sample rate (validate against variant max)
    if args.rate:
        adc.sample_rate = args.rate
    else:
        adc.sample_rate = adc.max_rate  # Use max for this variant

    print(f"Capturing {args.samples} samples at {adc.sample_rate/1e6:.1f} Msps...")
    data = adc.capture(args.samples)

    # Analyze
    enob = adc.calculate_enob(data)
    print(f"ENOB: {enob:.2f} bits (theoretical max: {adc.resolution} bits)")

if __name__ == '__main__':
    main()
```

**Files to Modify**:
- `software/python/dc2290a_fmc/dc2290a.py`
- `software/python/examples/basic_capture.py`
- `software/python/examples/performance_test.py`

---

## 5. Build System Changes

### A. Kernel Build
**File**: `software/linux/build_kernel.sh`

```bash
#!/bin/bash
# Build kernel with all variant support

# Enable all LTC238x variants in defconfig
cat >> arch/arm/configs/xilinx_zynq_defconfig <<EOF
CONFIG_LTC2387=y
CONFIG_LTC2387_18=y
CONFIG_LTC2387_16=y
CONFIG_LTC2386_18=y
CONFIG_LTC2386_16=y
CONFIG_LTC2385_18=y
CONFIG_LTC2385_16=y
EOF

make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- xilinx_zynq_defconfig
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- -j$(nproc) zImage modules dtbs

# Build all variant device trees
for variant in a b c d e f; do
    echo "Building device tree for DC2290A-${variant}..."
    make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- \
        xilinx/zynq-zed-dc2290a-${variant}.dtb
done
```

### B. FPGA Build
**File**: `fpga/build_all_variants.sh`

```bash
#!/bin/bash
# Build FPGA bitstreams for all variants

cd projects/dc2290a

# Build 18-bit variants (A, C, E)
echo "Building 18-bit bitstream..."
export ADC_DATA_WIDTH=18
export DCO_FREQ_MHZ=90  # Will be overridden per variant
make clean all
cp build/system_top.bit ../../boot/dc2290a_18bit.bit

# Build 16-bit variants (B, D, F)
echo "Building 16-bit bitstream..."
export ADC_DATA_WIDTH=16
make clean all
cp build/system_top.bit ../../boot/dc2290a_16bit.bit

echo "Built 2 bitstreams: dc2290a_18bit.bit, dc2290a_16bit.bit"
echo "Use 18-bit for variants A/C/E, 16-bit for variants B/D/F"
```

---

## Summary of Required Changes

### Critical Changes (Must Have)

| Component | File | Change Type | Reason |
|-----------|------|-------------|---------|
| **Kernel Driver** | `drivers/iio/adc/ltc2387.c` | Add chip_info for 6 variants | Support different resolutions and rates |
| **Device Tree** | `arch/arm/boot/dts/xilinx/zynq-zed-dc2290a-*.dts` | Create 6 variant DTS files | Configure per-variant parameters |
| **Python API** | `software/python/dc2290a_fmc/dc2290a.py` | Add variant detection and validation | Enforce rate limits per variant |
| **FPGA Config** | `hdl/projects/dc2290a/system_bd.tcl` | Parameterize DATA_WIDTH | Support 16-bit and 18-bit ADCs |

### Important Changes (Recommended)

| Component | File | Change Type | Reason |
|-----------|------|-------------|---------|
| **FPGA Constraints** | `hdl/projects/dc2290a/system_constr.xdc` | Parameterize DCO timing | Support 90/60/30 MHz DCO |
| **Build Scripts** | `software/linux/build_kernel.sh` | Build all variant DTBs | Easy deployment |
| **Example Code** | `software/python/examples/*.py` | Add variant argument | User convenience |

### Optional Changes (Nice to Have)

| Component | File | Change Type | Reason |
|-----------|------|-------------|---------|
| **Web Interface** | `software/web/` | Add variant selector | Visual variant selection |
| **Documentation** | `docs/variant-guide.md` | Per-variant setup guide | User documentation |
| **Test Suite** | `software/tests/` | Variant-specific tests | Validation |

---

## Implementation Priority

### Phase 1: Core Support (Week 1-2)
1. ✅ Update kernel driver with chip_info structure
2. ✅ Create 6 device tree files
3. ✅ Parameterize FPGA DATA_WIDTH
4. ✅ Update Python API with variant detection

### Phase 2: Validation (Week 3)
1. Test each variant individually
2. Verify rate limits are enforced
3. Validate data format (16-bit vs 18-bit)
4. Performance characterization per variant

### Phase 3: Polish (Week 4)
1. Update all example scripts
2. Add variant selection to web interface
3. Create per-variant documentation
4. Build automated test suite

---

## Testing Strategy

### Per-Variant Testing

```bash
# Test DC2290A-A (LTC2387-18, 15 Msps)
python3 basic_capture.py --variant dc2290a-a --rate 15000000

# Test DC2290A-B (LTC2387-16, 15 Msps)
python3 basic_capture.py --variant dc2290a-b --rate 15000000

# Test DC2290A-C (LTC2386-18, 10 Msps)
python3 basic_capture.py --variant dc2290a-c --rate 10000000

# Test DC2290A-D (LTC2386-16, 10 Msps)
python3 basic_capture.py --variant dc2290a-d --rate 10000000

# Test DC2290A-E (LTC2385-18, 5 Msps)
python3 basic_capture.py --variant dc2290a-e --rate 5000000

# Test DC2290A-F (LTC2385-16, 5 Msps)
python3 basic_capture.py --variant dc2290a-f --rate 5000000
```

### Rate Limit Validation

```python
# Should fail - exceeds max for variant E
adc = DC2290A(variant='dc2290a-e')  # 5 Msps max
adc.sample_rate = 15e6  # Should raise ValueError

# Should succeed
adc.sample_rate = 5e6  # OK
```

---

## References

- CN0577 Reference: https://github.com/analogdevicesinc/hdl/tree/main/projects/cn0577
- LTC2387 Driver: https://github.com/analogdevicesinc/linux/blob/main/drivers/iio/adc/ltc2387.c
- AXI_LTC2387 IP: https://analogdevicesinc.github.io/hdl/library/axi_ltc2387/
- IIO Documentation: https://wiki.analog.com/resources/tools-software/linux-drivers/iio-adc/ltc2387
