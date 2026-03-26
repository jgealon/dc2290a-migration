# AXI_LTC2387 IP Core Integration Guide

Guide for integrating ADI's AXI_LTC2387 IP core into the DC2290A-x migration FMC design on Zed Board. This IP core is the same one used in the CN0577 reference design.

## Overview

The [AXI_LTC2387 IP core](https://analogdevicesinc.github.io/hdl/library/axi_ltc2387/index.html) from Analog Devices' HDL library provides a complete interface for the LTC2387/2386/2385 family of high-speed SAR ADCs.

### Supported ADCs

| ADC | Resolution | Sample Rate | Data Lanes |
|-----|------------|-------------|------------|
| LTC2387-18 | 18-bit | 15 Msps | 6 LVDS pairs |
| LTC2387-16 | 16-bit | 15 Msps | 6 LVDS pairs |
| LTC2386-18 | 18-bit | 10 Msps | 6 LVDS pairs |
| LTC2386-16 | 16-bit | 10 Msps | 6 LVDS pairs |
| LTC2385-18 | 18-bit | 5 Msps | 6 LVDS pairs |
| LTC2385-16 | 16-bit | 5 Msps | 6 LVDS pairs |

## IP Core Features

### Data Path
- **LVDS Receiver**: Uses ISERDES primitives for high-speed deserialization
- **Deserialization**: 6-lane LVDS to parallel 18-bit conversion
- **Data Formatting**: Supports both 16-bit and 18-bit modes
- **Output**: AXI-Stream interface for data
- **Clock Domain Crossing**: Handles ADC clock to AXI clock domain

### Control Interface
- **AXI-Lite Slave**: Configuration and status registers
- **SPI Master**: ADC configuration via SPI
- **GPIO**: CNV (convert) signal generation

### Performance
- **Throughput**: Up to 15 Msps × 18 bits = 270 Mbps
- **Latency**: Minimal (<10 ADC clock cycles)
- **Resource Usage**: ~500 LUTs, ~800 FFs, 0 BRAMs (for ISERDES version)

## IP Core Architecture

```
┌────────────────────────────────────────────────────────────┐
│                   AXI_LTC2387 IP Core                      │
│                                                            │
│  ┌──────────────────────┐      ┌──────────────────────┐  │
│  │   LVDS Receivers     │      │   AXI-Lite Slave     │  │
│  │   (6 LVDS Pairs)     │      │   (Control Regs)     │  │
│  │                      │      │                      │  │
│  │  - D0A/D0B          │      │  - Config            │  │
│  │  - D1A/D1B          │      │  - Status            │  │
│  │  - D2A/D2B          │      │  - Scratch           │  │
│  │  - DCO (Data Clock) │      │                      │  │
│  │  - FCO (Frame Clock)│      │                      │  │
│  └──────────┬───────────┘      └──────────┬───────────┘  │
│             │                              │              │
│  ┌──────────▼───────────┐      ┌──────────▼───────────┐  │
│  │   Deserialization    │      │   SPI Master         │  │
│  │   & Alignment        │      │   (ADC Config)       │  │
│  │                      │      │                      │  │
│  │  - Bit Slip          │      │  - SCLK              │  │
│  │  - Frame Sync        │      │  - MOSI/MISO         │  │
│  │  - 16/18-bit Mode    │      │  - CS#               │  │
│  └──────────┬───────────┘      └──────────────────────┘  │
│             │                                             │
│  ┌──────────▼───────────┐      ┌──────────────────────┐  │
│  │   CDC FIFO           │      │   CNV Generator      │  │
│  │   (Clock Domain      │      │   (Convert Pulse)    │  │
│  │    Crossing)         │      │                      │  │
│  └──────────┬───────────┘      └──────────────────────┘  │
│             │                                             │
│  ┌──────────▼───────────┐                                │
│  │   AXI-Stream Master  │                                │
│  │   (Data Output)      │                                │
│  │                      │                                │
│  │  - TDATA [31:0]      │                                │
│  │  - TVALID            │                                │
│  │  - TREADY            │                                │
│  └──────────────────────┘                                │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

## Obtaining the IP Core

### Method 1: Clone ADI HDL Repository

```bash
# Clone the ADI HDL repository
git clone https://github.com/analogdevicesinc/hdl.git

# Navigate to the LTC2387 library
cd hdl/library/axi_ltc2387/

# The IP source is in:
#   - axi_ltc2387.v (top level)
#   - axi_ltc2387_channel.v (per-lane deserializer)
#   - axi_ltc2387_cmos.v or axi_ltc2387_lvds.v (interface variant)
```

### Method 2: Use Pre-built IP

ADI provides pre-packaged IP for Vivado:

```bash
# Download from ADI HDL release
wget https://github.com/analogdevicesinc/hdl/releases/download/hdl_2023_r2/adi_hdl_2023_r2.zip

# Extract and locate IP
unzip adi_hdl_2023_r2.zip
cd hdl/library/
```

## Integration into Vivado Project

### Step 1: Add IP Repository

1. Open Vivado project
2. **Settings** → **IP** → **Repository**
3. Click **+** and add path: `<adi-hdl>/library/`
4. Vivado will scan and find `axi_ltc2387`

### Step 2: Instantiate IP Core

1. In **Block Design**, click **Add IP**
2. Search for `axi_ltc2387`
3. Double-click to add to design

### Step 3: Configure IP Parameters

| Parameter | Value | Description |
|-----------|-------|-------------|
| `LVDS_CMOS_N` | 1 | Use LVDS interface (not CMOS) |
| `NUM_CHANNELS` | 1 | Single ADC (not multi-channel) |
| `DATA_WIDTH` | 18 | 18-bit mode (supports 16-bit too) |

### Step 4: Connect Interfaces

#### External Ports (to FMC)

```tcl
# LVDS data lanes
create_bd_port -dir I -from 0 -to 0 d0a_p
create_bd_port -dir I -from 0 -to 0 d0a_n
create_bd_port -dir I -from 0 -to 0 d0b_p
create_bd_port -dir I -from 0 -to 0 d0b_n
create_bd_port -dir I -from 0 -to 0 d1a_p
create_bd_port -dir I -from 0 -to 0 d1a_n
create_bd_port -dir I -from 0 -to 0 d1b_p
create_bd_port -dir I -from 0 -to 0 d1b_n
create_bd_port -dir I -from 0 -to 0 d2a_p
create_bd_port -dir I -from 0 -to 0 d2a_n
create_bd_port -dir I -from 0 -to 0 d2b_p
create_bd_port -dir I -from 0 -to 0 d2b_n
create_bd_port -dir I -from 0 -to 0 dco_p
create_bd_port -dir I -from 0 -to 0 dco_n
create_bd_port -dir I -from 0 -to 0 fco_p
create_bd_port -dir I -from 0 -to 0 fco_n

# CNV signal
create_bd_port -dir O cnv

# SPI
create_bd_port -dir O spi_sclk
create_bd_port -dir O spi_mosi
create_bd_port -dir I spi_miso
create_bd_port -dir O spi_csn

# Connect to IP
connect_bd_net [get_bd_ports d0a_p] [get_bd_pins axi_ltc2387_0/d0a_p]
# ... (repeat for all LVDS pairs)
```

#### AXI Interfaces

```tcl
# AXI-Lite for control (connect to Zynq GP0)
connect_bd_intf_net [get_bd_intf_pins axi_ltc2387_0/s_axi] \
                    [get_bd_intf_pins axi_interconnect_0/M01_AXI]

# AXI-Stream for data (connect to DMA)
connect_bd_intf_net [get_bd_intf_pins axi_ltc2387_0/m_axis] \
                    [get_bd_intf_pins axi_dma_0/S_AXIS_S2MM]

# Clocks
connect_bd_net [get_bd_pins axi_ltc2387_0/s_axi_aclk] \
               [get_bd_pins processing_system7_0/FCLK_CLK0]
```

## Pin Constraints (XDC)

### FMC Pin Assignments

```tcl
# LTC2387 LVDS Data Lanes
set_property PACKAGE_PIN H19 [get_ports {d0a_p}]
set_property PACKAGE_PIN H20 [get_ports {d0a_n}]
set_property IOSTANDARD LVDS_25 [get_ports {d0a_p}]
set_property IOSTANDARD LVDS_25 [get_ports {d0a_n}]
set_property DIFF_TERM TRUE [get_ports {d0a_p}]

set_property PACKAGE_PIN G15 [get_ports {d0b_p}]
set_property PACKAGE_PIN G16 [get_ports {d0b_n}]
set_property IOSTANDARD LVDS_25 [get_ports {d0b_p}]
set_property IOSTANDARD LVDS_25 [get_ports {d0b_n}]
set_property DIFF_TERM TRUE [get_ports {d0b_p}]

# ... (repeat for d1a, d1b, d2a, d2b)

# Data Clock Output (DCO)
set_property PACKAGE_PIN D18 [get_ports {dco_p}]
set_property PACKAGE_PIN D19 [get_ports {dco_n}]
set_property IOSTANDARD LVDS_25 [get_ports {dco_p}]
set_property IOSTANDARD LVDS_25 [get_ports {dco_n}]
set_property DIFF_TERM TRUE [get_ports {dco_p}]

# Frame Clock Output (FCO)
set_property PACKAGE_PIN C20 [get_ports {fco_p}]
set_property PACKAGE_PIN B20 [get_ports {fco_n}]
set_property IOSTANDARD LVDS_25 [get_ports {fco_p}]
set_property IOSTANDARD LVDS_25 [get_ports {fco_n}]
set_property DIFF_TERM TRUE [get_ports {fco_p}]

# CNV (single-ended)
set_property PACKAGE_PIN G19 [get_ports {cnv}]
set_property IOSTANDARD LVCMOS25 [get_ports {cnv}]

# SPI
set_property PACKAGE_PIN G20 [get_ports {spi_sclk}]
set_property PACKAGE_PIN J18 [get_ports {spi_mosi}]
set_property PACKAGE_PIN J19 [get_ports {spi_miso}]
set_property PACKAGE_PIN K21 [get_ports {spi_csn}]
set_property IOSTANDARD LVCMOS25 [get_ports {spi_*}]
```

**Note**: Actual pin assignments must match your FMC pinout design.

### Timing Constraints

```tcl
# Create clock from DCO (data clock output from ADC)
# For LTC2387-18 at 15 Msps: DCO is 90 MHz (6× oversampling)
create_clock -period 11.111 -name dco_clk [get_ports dco_p]

# Input delays for LVDS data relative to DCO
set_input_delay -clock dco_clk -max 2.0 [get_ports {d0a_p d0a_n d0b_p d0b_n}]
set_input_delay -clock dco_clk -min 0.5 [get_ports {d0a_p d0a_n d0b_p d0b_n}]
set_input_delay -clock dco_clk -max 2.0 [get_ports {d1a_p d1a_n d1b_p d1b_n}]
set_input_delay -clock dco_clk -min 0.5 [get_ports {d1a_p d1a_n d1b_p d1b_n}]
set_input_delay -clock dco_clk -max 2.0 [get_ports {d2a_p d2a_n d2b_p d2b_n}]
set_input_delay -clock dco_clk -min 0.5 [get_ports {d2a_p d2a_n d2b_p d2b_n}]

# FCO (frame clock) constraints
set_input_delay -clock dco_clk -max 2.0 [get_ports {fco_p fco_n}]
set_input_delay -clock dco_clk -min 0.5 [get_ports {fco_p fco_n}]

# False path between async clocks (DCO and AXI)
set_false_path -from [get_clocks dco_clk] -to [get_clocks clk_fpga_0]
set_false_path -from [get_clocks clk_fpga_0] -to [get_clocks dco_clk]
```

## Register Map

The AXI_LTC2387 IP core provides AXI-Lite registers for configuration and status.

### Register Addresses (Offset from Base)

| Offset | Name | Type | Description |
|--------|------|------|-------------|
| 0x0000 | VERSION | RO | IP core version |
| 0x0004 | ID | RO | IP core ID |
| 0x0008 | SCRATCH | RW | Scratch register for testing |
| 0x0010 | CONFIG | RW | Configuration register |
| 0x0014 | STATUS | RO | Status register |
| 0x0020 | CNV_CTRL | RW | CNV pulse control |
| 0x0024 | CNV_PERIOD | RW | CNV period (sample rate) |

### Configuration Register (0x0010)

| Bits | Name | Description |
|------|------|-------------|
| [0] | ENABLE | Enable data capture (1=enable, 0=disable) |
| [1] | SOFTSPAN | Softspan mode select (0=normal, 1=softspan) |
| [2] | DATA_WIDTH | Data width (0=16-bit, 1=18-bit) |
| [3] | TEST_PATTERN | Enable test pattern mode |
| [7:4] | RESERVED | Reserved |

### Status Register (0x0014)

| Bits | Name | Description |
|------|------|-------------|
| [0] | LOCKED | DCO clock locked status |
| [1] | OVR | Overflow detected |
| [2] | UNR | Underrun detected |
| [7:3] | RESERVED | Reserved |

### CNV Control Register (0x0020)

| Bits | Name | Description |
|------|------|-------------|
| [0] | CNV_EN | Enable CNV pulse generation |
| [1] | CNV_MODE | 0=continuous, 1=one-shot |
| [7:2] | RESERVED | Reserved |

### CNV Period Register (0x0024)

| Bits | Name | Description |
|------|------|-------------|
| [31:0] | PERIOD | CNV period in clock cycles |

**Calculation**: `PERIOD = (AXI_CLK_FREQ / SAMPLE_RATE) - 1`

Example for 15 Msps with 100 MHz AXI clock:
- `PERIOD = (100e6 / 15e6) - 1 = 5.67 ≈ 6`

## Software Configuration

### Device Tree Entry

```dts
&axi {
    ltc2387_adc: axi-ltc2387@44a00000 {
        compatible = "adi,axi-ltc2387-1.0";
        reg = <0x44a00000 0x10000>;
        clocks = <&clkc 15>;
        clock-names = "adc_clk";

        adi,adc-variant = "ltc2387-18";  /* or ltc2387-16, ltc2386-18, etc. */
    };
};
```

### Linux Driver Configuration

The driver should read the variant from device tree and configure accordingly:

```c
static const struct ltc238x_chip_info ltc238x_chips[] = {
    [LTC2387_18] = {
        .resolution = 18,
        .max_sample_rate = 15000000,
        .num_lanes = 6,
    },
    [LTC2387_16] = {
        .resolution = 16,
        .max_sample_rate = 15000000,
        .num_lanes = 6,
    },
    [LTC2386_18] = {
        .resolution = 18,
        .max_sample_rate = 10000000,
        .num_lanes = 6,
    },
    // ... etc for all 6 variants
};
```

## Testing the IP Core

### Simulation

ADI provides testbenches in the HDL repository:

```bash
cd hdl/library/axi_ltc2387/
# Run simulation with ModelSim or Vivado XSIM
```

### Hardware Test

1. **Check Version Register**:
   ```bash
   devmem 0x44a00000
   # Should return IP version
   ```

2. **Test Scratch Register**:
   ```bash
   devmem 0x44a00008 32 0xDEADBEEF
   devmem 0x44a00008
   # Should return 0xDEADBEEF
   ```

3. **Enable Data Capture**:
   ```bash
   # Write to CONFIG register (enable + 18-bit mode)
   devmem 0x44a00010 32 0x00000005

   # Check STATUS register
   devmem 0x44a00014
   # Bit 0 should be 1 (locked)
   ```

4. **Configure CNV**:
   ```bash
   # Set sample rate to 1 Msps (with 100 MHz AXI clock)
   # Period = 100MHz / 1MHz - 1 = 99
   devmem 0x44a00024 32 99

   # Enable CNV
   devmem 0x44a00020 32 0x00000001
   ```

## Variant-Specific Considerations

### Sample Rate Configuration

| Variant | Max Sample Rate | DCO Frequency | CNV Period (100 MHz AXI) |
|---------|----------------|---------------|--------------------------|
| LTC2387-xx | 15 Msps | 90 MHz | 6 |
| LTC2386-xx | 10 Msps | 60 MHz | 9 |
| LTC2385-xx | 5 Msps | 30 MHz | 19 |

### 16-bit vs 18-bit Mode

The IP core supports both resolutions:
- **18-bit mode**: All 6 LVDS lanes used, full 18-bit data
- **16-bit mode**: Only 16 bits valid, MSBs zero-extended

Configure via register:
```c
// 18-bit mode
writel(0x00000005, base + CONFIG_REG);  // bit[2]=1, bit[0]=1

// 16-bit mode
writel(0x00000001, base + CONFIG_REG);  // bit[2]=0, bit[0]=1
```

## Troubleshooting

### DCO Clock Not Locking

**Symptoms**: STATUS[0] = 0 (not locked)

**Solutions**:
- Check LVDS connections to FMC
- Verify DCO timing constraints
- Check IDELAY calibration (IDELAYCTRL)
- Verify ADC is powered and clocked

### Data Corruption

**Symptoms**: Invalid or random data

**Solutions**:
- Check frame alignment (FCO signal)
- Adjust IDELAY taps for setup/hold
- Verify CNV pulse timing
- Check for PCB signal integrity issues

### Low Throughput

**Symptoms**: Data rate lower than expected

**Solutions**:
- Check AXI-Stream backpressure (TREADY)
- Verify DMA configuration
- Check for AXI interconnect bottlenecks
- Monitor FIFO overflow flags

## Resources

- [ADI HDL GitHub Repository](https://github.com/analogdevicesinc/hdl)
- [AXI_LTC2387 Documentation](https://analogdevicesinc.github.io/hdl/library/axi_ltc2387/)
- [LTC2387 Datasheet](https://www.analog.com/ltc2387)
- [Xilinx ISERDES User Guide](https://www.xilinx.com/support/documentation/)

## Support

For IP core issues:
- Open issue on [ADI HDL GitHub](https://github.com/analogdevicesinc/hdl/issues)
- ADI EngineerZone forums
- This repository's issue tracker

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2026-03-26 | Initial integration guide |
