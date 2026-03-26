# CN0577 FMC FPGA Design

FPGA design for the CN0577 FMC card on Xilinx Zed Board (Zynq-7000 SoC), using ADI's **AXI_LTC2387** IP core.

## Overview

The FPGA design leverages Analog Devices' proven [AXI_LTC2387 IP core](https://analogdevicesinc.github.io/hdl/library/axi_ltc2387/) to interface with LTC2387/2386/2385 high-speed SAR ADCs. The IP handles LVDS deserialization, data formatting, and AXI-Stream output. All 6 ADC variants (LTC2387/2386/2385 in 16-bit and 18-bit) are supported through configuration.

**Key Benefits of Using ADI HDL IP**:
- ✅ Proven, production-tested design
- ✅ Optimized LVDS timing and alignment
- ✅ Built-in clock domain crossing
- ✅ AXI-Stream DMA ready
- ✅ Active ADI support and updates

### Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                     Zynq-7000 SoC                            │
│  ┌───────────────────────────────────────────────────────┐   │
│  │          Processing System (PS)                       │   │
│  │  ┌────────────┐    ┌─────────────┐                   │   │
│  │  │ ARM Cortex │    │   DDR3      │                   │   │
│  │  │  A9 x2     │───▶│   512 MB    │                   │   │
│  │  └────────────┘    └─────────────┘                   │   │
│  │         │                                             │   │
│  │         │ AXI GP0 (Control) / HP0 (Data)             │   │
│  └─────────┼─────────────────────────────────────────────┘   │
│            │                                                  │
│  ┌─────────▼──────────────────────────────────────────────┐  │
│  │          Programmable Logic (PL)                       │  │
│  │                                                         │  │
│  │  ┌──────────────────────────────────────────────────┐  │  │
│  │  │        ADI AXI_LTC2387 IP Core                   │  │  │
│  │  │  ┌────────────┐  ┌────────────┐  ┌───────────┐  │  │  │
│  │  │  │   LVDS     │  │Deserialize │  │   CDC     │  │  │  │
│  │  │  │  Receivers │─▶│  & Align   │─▶│   FIFO    │  │  │  │
│  │  │  │ (6 Lanes)  │  │            │  │           │  │  │  │
│  │  │  └────────────┘  └────────────┘  └─────┬─────┘  │  │  │
│  │  │         ▲                               │        │  │  │
│  │  │         │ DCO/FCO                       │        │  │  │
│  │  │         │                               ▼        │  │  │
│  │  │  ┌──────┴───────┐             ┌────────────┐    │  │  │
│  │  │  │   CNV Gen    │             │ AXI-Stream │    │  │  │
│  │  │  │  (Convert)   │             │   Master   │────┼──┼─┐│
│  │  │  └──────────────┘             └────────────┘    │  │ ││
│  │  │  ┌──────────────┐                               │  │ ││
│  │  │  │  SPI Master  │                               │  │ ││
│  │  │  │  (ADC Cfg)   │                               │  │ ││
│  │  │  └──────────────┘                               │  │ ││
│  │  └─────────┬────────────────────────────────────────┘  │ ││
│  │            │ AXI-Lite (Control)                        │ ││
│  │  ┌─────────▼────────────────────────────────────────┐  │ ││
│  │  │          AXI Interconnect                        │  │ ││
│  │  │  ┌──────────────┐        ┌──────────────┐       │  │ ││
│  │  │  │  AXI DMAC    │◀───────│  AXI HP0     │       │  │ ││
│  │  │  │  (High Perf) │        │   Port       │       │  │ ││
│  │  │  └──────────────┘        └──────────────┘       │  │ ││
│  │  └──────────────────────────────────────────────────┘  │ ││
│  │                                                         │ ││
│  └─────────────────────────────────────────────────────────┘ ││
│                                                               ││
└───────────────────────────────────────────────────────────────┘│
                │                                                │
                ▼ (to FMC)                          (from FMC)  │
         ┌──────────────┐                         ┌─────────────┘
         │ CNV, SPI     │                         │ LVDS Data
         │              │                         │ DCO, FCO
         └──────────────┘                         └─────────────
```

## Directory Structure

```
fpga/
├── hdl/
│   ├── rtl/                    # Verilog/VHDL source
│   │   ├── adc_interface.v     # LVDS receiver and deserializer
│   │   ├── data_packer.v       # 16/18-bit to AXI width conversion
│   │   ├── trigger_control.v   # ADC conversion trigger
│   │   ├── spi_controller.v    # ADC configuration interface
│   │   └── top.v               # Top-level design
│   ├── sim/                    # Simulation testbenches
│   └── include/                # Header files
├── ip/
│   ├── axi_dma/                # AXI DMA IP configuration
│   ├── selectio/               # LVDS deserializer IP
│   └── clocking/               # Clock management IP
├── constraints/
│   ├── pins.xdc                # Pin assignments for FMC
│   ├── timing.xdc              # Timing constraints
│   └── physical.xdc            # Physical layout constraints
├── vivado/
│   ├── create_project.tcl      # Project creation script
│   ├── build.tcl               # Build automation
│   └── cn0577_fmc.xpr          # Vivado project (gitignored)
├── sdk/
│   ├── fsbl/                   # First Stage Boot Loader
│   ├── devicetree/             # Device tree sources
│   └── examples/               # Bare-metal examples
├── petalinux/
│   ├── project-spec/           # PetaLinux configuration
│   ├── recipes/                # Custom recipes
│   └── build/                  # Build outputs (gitignored)
└── README.md                   # This file
```

## Design Approach

### Using ADI HDL IP Core (Recommended)

The **recommended approach** is to use ADI's pre-built AXI_LTC2387 IP core from the [ADI HDL GitHub repository](https://github.com/analogdevicesinc/hdl). This IP is production-tested, actively maintained, and optimized for the LTC238x ADC family.

**See [IP_INTEGRATION.md](IP_INTEGRATION.md) for complete integration guide.**

### Custom HDL Implementation (Alternative)

If custom modifications are needed, you can implement your own interface. Below are reference modules:

## HDL Modules (Reference - Use ADI IP Instead)

### 1. ADC Interface (`adc_interface.v`) - Reference Only

**⚠️ Note: Use AXI_LTC2387 IP core instead. This is for reference only.**

Receives LVDS data from LTC238x ADC and deserializes to parallel format.

**Features:**
- LVDS to LVCMOS conversion using SelectIO primitives
- Deserializer (DDR to SDR conversion)
- Bit-slip for alignment
- Configurable for 16-bit or 18-bit mode
- Frame synchronization

**Ports:**
```verilog
module adc_interface #(
    parameter DATA_WIDTH = 18
)(
    input  wire                     clk,        // System clock
    input  wire                     rst,        // Reset
    // LVDS inputs from ADC
    input  wire [DATA_WIDTH-1:0]    adc_data_p,
    input  wire [DATA_WIDTH-1:0]    adc_data_n,
    input  wire                     adc_clk_p,
    input  wire                     adc_clk_n,
    // Parallel output
    output wire [DATA_WIDTH-1:0]    data_out,
    output wire                     data_valid,
    // Status
    output wire                     locked
);
```

### 2. Data Packer (`data_packer.v`)

Converts ADC data width to AXI stream format and manages FIFO buffering.

**Features:**
- Width conversion (16/18-bit to 32/64-bit AXI)
- FIFO buffering (8K samples)
- Overflow/underflow detection
- Backpressure handling

**Ports:**
```verilog
module data_packer #(
    parameter ADC_WIDTH = 18,
    parameter AXI_WIDTH = 32
)(
    input  wire                     adc_clk,
    input  wire                     axi_clk,
    input  wire                     rst,
    // ADC data input
    input  wire [ADC_WIDTH-1:0]     adc_data,
    input  wire                     adc_valid,
    // AXI Stream output
    output wire [AXI_WIDTH-1:0]     m_axis_tdata,
    output wire                     m_axis_tvalid,
    input  wire                     m_axis_tready,
    output wire                     m_axis_tlast,
    // Status
    output wire                     fifo_overflow,
    output wire [15:0]              fifo_count
);
```

### 3. Trigger Control (`trigger_control.v`)

Generates ADC conversion triggers and manages acquisition timing.

**Features:**
- Programmable sample rate divider
- External trigger support
- Burst capture mode
- Continuous capture mode

**Ports:**
```verilog
module trigger_control (
    input  wire         clk,
    input  wire         rst,
    // Configuration
    input  wire [31:0]  sample_rate_div,    // Clock divider
    input  wire         mode,               // 0=continuous, 1=burst
    input  wire [31:0]  burst_length,       // Samples per burst
    input  wire         ext_trigger_en,
    input  wire         ext_trigger,
    // Control
    input  wire         start,
    input  wire         stop,
    // ADC control
    output reg          adc_cnv,
    // Status
    output reg          busy,
    output reg [31:0]   sample_count
);
```

### 4. SPI Controller (`spi_controller.v`)

Configures ADC registers via SPI interface.

**Features:**
- SPI master mode
- Configurable clock rate
- Byte-oriented transactions
- Busy/done status

### 5. Top Module (`top.v`)

Integrates all modules and connects to Zynq PS.

**Features:**
- AXI-Lite slave for control registers
- AXI-Stream master for data transfer
- Interrupt generation
- Status monitoring

## IP Cores

### ADI IP Cores

- **[AXI_LTC2387](https://analogdevicesinc.github.io/hdl/library/axi_ltc2387/)**: LTC238x ADC interface (primary IP)
- **[AXI_DMAC](https://analogdevicesinc.github.io/hdl/library/axi_dmac/)**: High-performance DMA controller from ADI

### Xilinx IP

- **AXI Interconnect**: Connect multiple AXI slaves/masters
- **Clock Wizard**: Generate required clock domains
- **Processing System**: Zynq PS configuration (ARM + peripherals)
- **IDELAYCTRL**: Required for LVDS IDELAY calibration

### IP Integration Notes

1. **AXI_LTC2387** provides complete ADC interface
2. **AXI_DMAC** provides streaming DMA (alternative to Xilinx DMA)
3. Both ADI IPs are optimized to work together
4. See [IP_INTEGRATION.md](IP_INTEGRATION.md) for step-by-step guide

## Clock Domains

| Clock | Frequency | Source | Usage |
|-------|-----------|--------|-------|
| clk_100 | 100 MHz | PS | AXI buses, control logic |
| clk_adc | 15-200 MHz | PL | ADC sample clock |
| clk_ref | 200 MHz | PL | IDELAYCTRL reference |
| clk_ddr | 200 MHz | PL | DDR deserialization |

## Memory Map

### AXI-Lite Control Registers (Base: 0x43C0_0000)

| Offset | Register | Access | Description |
|--------|----------|--------|-------------|
| 0x00 | CTRL | R/W | Control register (start/stop/reset) |
| 0x04 | STATUS | RO | Status register (busy/overflow/locked) |
| 0x08 | CONFIG | R/W | Configuration (mode/data_width) |
| 0x0C | SAMPLE_DIV | R/W | Sample rate divider |
| 0x10 | BURST_LEN | R/W | Burst capture length |
| 0x14 | FIFO_COUNT | RO | FIFO fill level |
| 0x18 | INT_EN | R/W | Interrupt enable |
| 0x1C | INT_STATUS | R/W1C | Interrupt status |

### Control Register (0x00)

| Bit | Name | Description |
|-----|------|-------------|
| 0 | START | Start acquisition |
| 1 | STOP | Stop acquisition |
| 2 | RESET | Software reset |
| 3 | MODE | 0=continuous, 1=burst |
| 4 | EXT_TRIG_EN | Enable external trigger |
| [31:5] | Reserved | - |

## Building the Design

### Prerequisites

- **Xilinx Vivado** 2022.2 or later (2023.1+ recommended)
- **ADI HDL Repository**: Clone from [GitHub](https://github.com/analogdevicesinc/hdl)
- **Xilinx SDK/Vitis** (for software)
- **PetaLinux Tools** 2022.2+ (for Linux build)
- **Zed Board files**: Included with Vivado

### Step 0: Get ADI HDL Library

```bash
# Clone ADI HDL repository
cd /opt/
git clone https://github.com/analogdevicesinc/hdl.git
cd hdl
git checkout hdl_2023_r2  # Or latest stable release

# Set environment variable
export ADI_HDL_DIR=/opt/hdl
```

### Quick Build

```bash
# Source Vivado settings
source /opt/Xilinx/Vivado/2022.2/settings64.sh

# Create Vivado project (will import ADI IPs)
cd fpga/vivado/
vivado -mode batch -source create_project.tcl

# Build bitstream
vivado -mode batch -source build.tcl

# Outputs:
# - Bitstream: fpga/vivado/cn0577_fmc.runs/impl_1/system_wrapper.bit
# - Hardware platform: fpga/vivado/system_wrapper.xsa
```

### Detailed Build Steps

#### 1. Create Block Design

```tcl
# In Vivado GUI or Tcl console

# Add IP repository path
set_property ip_repo_paths $env(ADI_HDL_DIR)/library [current_project]
update_ip_catalog

# Create block design
create_bd_design "system"

# Add Zynq PS
create_bd_cell -type ip -vlnv xilinx.com:ip:processing_system7:5.5 processing_system7_0

# Configure PS (enable HP0, GP0, FCLK_CLK0=100MHz)
apply_bd_automation -rule xilinx.com:bd_rule:processing_system7 -config {
    make_external "FIXED_IO, DDR"
    apply_board_preset "1"
} [get_bd_cells processing_system7_0]

set_property -dict [list \
    CONFIG.PCW_USE_S_AXI_HP0 {1} \
    CONFIG.PCW_FPGA0_PERIPHERAL_FREQMHZ {100} \
] [get_bd_cells processing_system7_0]

# Add AXI_LTC2387 IP
create_bd_cell -type ip -vlnv analog.com:user:axi_ltc2387:1.0 axi_ltc2387_0

# Configure for 18-bit, LVDS mode
set_property -dict [list \
    CONFIG.LVDS_CMOS_N {1} \
    CONFIG.NUM_CHANNELS {1} \
    CONFIG.DATA_WIDTH {18} \
] [get_bd_cells axi_ltc2387_0]

# Add AXI DMA (or use AXI_DMAC from ADI)
create_bd_cell -type ip -vlnv analog.com:user:axi_dmac:1.0 axi_dmac_0

# Configure DMA for streaming receive
set_property -dict [list \
    CONFIG.DMA_TYPE_SRC {2} \
    CONFIG.DMA_TYPE_DEST {0} \
    CONFIG.CYCLIC {0} \
    CONFIG.DMA_DATA_WIDTH_SRC {32} \
] [get_bd_cells axi_dmac_0]

# Run connection automation
apply_bd_automation -rule xilinx.com:bd_rule:axi4 -config {
    Master "/processing_system7_0/M_AXI_GP0"
} [get_bd_intf_pins axi_ltc2387_0/s_axi]

apply_bd_automation -rule xilinx.com:bd_rule:axi4 -config {
    Slave "/processing_system7_0/S_AXI_HP0"
} [get_bd_intf_pins axi_dmac_0/m_dest_axi]

# Connect data stream
connect_bd_intf_net [get_bd_intf_pins axi_ltc2387_0/m_axis] \
                    [get_bd_intf_pins axi_dmac_0/s_axis]

# Make external ports for FMC connections
make_bd_intf_pins_external [get_bd_intf_pins axi_ltc2387_0/lvds]

# Validate and save
validate_bd_design
save_bd_design
```

#### 2. Add Constraints

Add pin constraints from [IP_INTEGRATION.md](IP_INTEGRATION.md#pin-constraints-xdc).

#### 3. Generate Bitstream

```tcl
# Generate HDL wrapper
make_wrapper -files [get_files system.bd] -top
add_files -norecurse [file normalize [lindex [glob [get_property directory \
    [current_project]]/system.srcs/sources_1/bd/system/hdl/system_wrapper.v] 0]]

# Run synthesis
launch_runs synth_1 -jobs 4
wait_on_run synth_1

# Run implementation
launch_runs impl_1 -to_step write_bitstream -jobs 4
wait_on_run impl_1

# Export hardware
write_hw_platform -fixed -include_bit -force \
    -file [file normalize [get_property directory [current_project]]/system_wrapper.xsa]
```

## Simulation

### Running Testbenches

```bash
cd fpga/hdl/sim/

# Compile and simulate with ModelSim
vsim -do sim_adc_interface.do

# Or with Vivado simulator
xvlog ../rtl/adc_interface.v tb_adc_interface.v
xelab -debug typical tb_adc_interface
xsim tb_adc_interface -gui
```

### Test Scenarios

1. **ADC Interface Test**: Verify LVDS deserialization and data capture
2. **Data Packer Test**: Verify width conversion and FIFO operation
3. **End-to-End Test**: Complete data path from ADC to DMA

## Timing Closure

### Critical Paths

1. **ADC Data Path**: LVDS input to FIFO write
   - Constraint: < 6.6 ns (15 MHz ADC clock)
   - Strategy: Pipeline registers, IDELAY tuning

2. **AXI Interface**: Control register access
   - Constraint: < 10 ns (100 MHz AXI clock)
   - Strategy: Standard AXI timing

### Timing Constraints

```tcl
# ADC clock (worst case: 15 MHz)
create_clock -period 66.667 -name adc_clk [get_ports adc_clk_p]

# AXI clock from PS
create_clock -period 10.000 -name axi_clk [get_pins PS/FCLK_CLK0]

# Input delays for LVDS data
set_input_delay -clock adc_clk -max 2.0 [get_ports adc_data_p]
set_input_delay -clock adc_clk -min 0.5 [get_ports adc_data_p]

# False paths between async clock domains
set_false_path -from [get_clocks adc_clk] -to [get_clocks axi_clk]
```

## PetaLinux Integration

### Device Tree

Add device tree entry for ADC controller:

```dts
&axi {
    cn0577_adc@43c00000 {
        compatible = "adi,cn0577-adc-1.0";
        reg = <0x43c00000 0x10000>;
        interrupts = <0 29 4>;
        interrupt-parent = <&intc>;
        clocks = <&clkc 15>;
        clock-names = "s_axi_aclk";
    };
};
```

### Kernel Driver

See `software/linux/` for IIO driver implementation.

## Debugging

### ILA (Integrated Logic Analyzer)

Insert ILA cores to debug:

```tcl
# Mark nets for debugging
mark_debug [get_nets {adc_data[*]}]
mark_debug [get_nets {data_valid}]

# Create debug core
create_debug_core ila_0 ila
```

### ChipScope/ILA Signals

Recommended signals to probe:
- ADC data bus
- Data valid signals
- FIFO flags (full/empty/overflow)
- AXI handshake signals (valid/ready)

## Performance

### Resource Utilization (Zynq-7020)

| Resource | Used | Available | Utilization |
|----------|------|-----------|-------------|
| LUTs | ~8,500 | 53,200 | 16% |
| FFs | ~12,000 | 106,400 | 11% |
| BRAM | 15 | 140 | 11% |
| DSP | 0 | 220 | 0% |

### Throughput

- **Maximum Sample Rate**: 15 Msps
- **Data Width**: 18-bit
- **Throughput**: 270 Mbps (18-bit × 15 MHz)
- **AXI Bandwidth**: 800 Mbps (plenty of headroom)

## Known Issues

1. **Bit Alignment**: LVDS data may require IDELAY calibration at power-up
2. **Clock Jitter**: External clock input preferred for lowest jitter
3. **EMI**: Shield high-speed signals to avoid interference

## Future Enhancements

- [ ] Add DSP processing in FPGA (FFT, filtering)
- [ ] Multi-channel support
- [ ] Compressed data streaming
- [ ] Advanced triggering (pattern, threshold)
- [ ] Real-time sample rate conversion

## Resources

- [Zynq-7000 TRM](https://www.xilinx.com/support/documentation/user_guides/ug585-Zynq-7000-TRM.pdf)
- [Vivado Design Suite User Guide](https://www.xilinx.com/support/documentation/sw_manuals/xilinx2022_2/ug893-vivado-ip.pdf)
- [AXI Reference Guide](https://www.xilinx.com/support/documentation/ip_documentation/axi_ref_guide/latest/ug1037-vivado-axi-reference-guide.pdf)

## Support

For FPGA design questions:
- Open an issue with the `fpga` label
- Check simulation testbenches for examples
- Refer to Xilinx documentation

## Status

| Component | Status | Notes |
|-----------|--------|-------|
| ADC Interface | In Progress | Testing bit alignment |
| Data Packer | Complete | Verified in simulation |
| Trigger Control | Complete | - |
| SPI Controller | Complete | - |
| Top-level Integration | In Progress | Timing closure pending |
| Vivado Project | In Progress | IP integration ongoing |
| Simulation | Complete | All modules tested |
| Timing Closure | Pending | Target Q2 2026 |
