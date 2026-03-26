# CN0577 Migration to Zed Board (FMC)

Migration of CN0577 Circuits from the Lab reference design to Xilinx Zed Board platform using FMC (FPGA Mezzanine Card) interface.

## Overview

This repository documents the migration of the [CN0577 reference design](https://www.analog.com/en/resources/reference-designs/circuits-from-the-lab/CN0577.html) from legacy DC2290A evaluation boards to a modern Xilinx Zynq-based platform. The new design uses a single FMC board compatible with the Zed Board that supports all 6 ADC variants.

### Migration Goals

- **Single Board Design**: One FMC card supporting all 6 ADC variants
- **Modern Platform**: Xilinx Zynq-7000 SoC (Zed Board)
- **Standard Interface**: FMC (VITA 57.1) connector
- **Improved Performance**: Leverage FPGA processing capabilities
- **Better Integration**: Direct integration with Xilinx tools and IPs

## CN0577 Overview

CN0577 is a precision data acquisition system reference design. This migration replaces the legacy DC2290A boards with a modern FMC-based solution.

### Legacy Platform (Current)
- **Boards**: DC2290A-A through DC2290A-F (6 variants)
- **Interface**: LVDS to DC718 USB adapter
- **Limitations**: Multiple boards, USB bottleneck, limited processing

### Target Platform (New)
- **Board**: Single FMC card for Zed Board
- **SoC**: Xilinx Zynq-7000 (XC7Z020-CLG484)
- **Interface**: FMC LPC (Low Pin Count) connector
- **Benefits**: Unified design, FPGA processing, extensibility

## ADC Variant Support

The FMC board supports 6 LTC238x ADC variants through component options. All variants share a common footprint and interface, using ADI's proven [AXI_LTC2387 IP core](https://analogdevicesinc.github.io/hdl/library/axi_ltc2387/index.html).

| Variant | ADC Part Number | Resolution | Sample Rate | Interface | Power |
|---------|-----------------|------------|-------------|-----------|-------|
| Config A | **LTC2387-18** | 18-bit | 15 Msps | 6-lane LVDS | 170 mW |
| Config B | **LTC2387-16** | 16-bit | 15 Msps | 6-lane LVDS | 170 mW |
| Config C | **LTC2386-18** | 18-bit | 10 Msps | 6-lane LVDS | 140 mW |
| Config D | **LTC2386-16** | 16-bit | 10 Msps | 6-lane LVDS | 140 mW |
| Config E | **LTC2385-18** | 18-bit | 5 Msps | 6-lane LVDS | 115 mW |
| Config F | **LTC2385-16** | 16-bit | 5 Msps | 6-lane LVDS | 115 mW |

**Key Features** (All Variants):
- Ultra-low noise: 89 dB SNR (typ)
- Low distortion: -103 dB THD (typ)
- No pipeline delay: Zero latency ADC
- Wide input bandwidth: 300 MHz (3 dB)
- Integrated digital interface with output randomizer
- Single 1.8V digital supply

## Features

### Hardware Design
- **FMC LPC Compliance**: VITA 57.1 standard interface
- **Multi-Variant Support**: Single board supports all 6 ADC configurations
- **Analog Front-End**: Precision signal conditioning optimized for each ADC variant
- **Power Management**: Efficient power delivery from FMC 12V/3.3V rails
- **Compact Form Factor**: Standard FMC card size

### FPGA Integration
- **ADI HDL IP Core**: Uses proven [AXI_LTC2387](https://analogdevicesinc.github.io/hdl/library/axi_ltc2387/) IP from ADI HDL library
- **Zynq PS**: ARM Cortex-A9 dual-core processor for control
- **FPGA Fabric**: High-speed LVDS interface and data capture
- **AXI Interfaces**: AXI-Lite control, AXI-Stream data output
- **DMA Engine**: Efficient data transfer to DDR memory via AXI DMAC
- **Linux Support**: PetaLinux with IIO framework drivers

### Software Stack
- **IIO Drivers**: Linux Industrial I/O framework
- **Python API**: High-level programming interface
- **MATLAB/Simulink**: HDL Coder integration
- **Web Interface**: Browser-based configuration and monitoring
- **LibIIO**: Cross-platform client library

## Quick Start

### Hardware Setup

1. **Connect FMC Card to Zed Board**
   - Align FMC card with LPC connector on Zed Board
   - Secure with standoffs
   - Connect 12V power supply to Zed Board

2. **Configure Boot Mode**
   - Set Zed Board boot mode to SD card
   - Insert SD card with PetaLinux image

3. **Connect Interfaces**
   - USB-UART for console access
   - Ethernet for network connectivity
   - Analog input signal to FMC SMA connectors

### Software Setup

```bash
# Clone the repository
git clone <repository-url>
cd dc2290a-migration

# Flash FPGA bitstream (from Zed Board Linux)
cd fpga/
./program_fpga.sh cn0577_fmc.bit

# Install Python support
cd software/python/
pip install -e .

# Run test capture
python examples/basic_capture.py --variant config-a
```

### Quick Test

```python
from cn0577_fmc import CN0577

# Initialize with variant configuration
adc = CN0577(variant='config-a', sample_rate=15e6)

# Capture data
data = adc.capture(num_samples=16384)

# Perform FFT analysis
adc.plot_fft(data)
```

## Repository Structure

```
.
├── .github/
│   └── workflows/           # CI/CD workflows
├── hardware/
│   ├── fmc_card/           # FMC card schematic and PCB design
│   ├── bom/                # Bill of materials
│   └── gerbers/            # PCB manufacturing files
├── fpga/
│   ├── hdl/                # Verilog/VHDL source files
│   ├── ip/                 # Custom IP cores
│   ├── constraints/        # Timing and pin constraints
│   └── vivado/             # Vivado project files
├── software/
│   ├── linux/              # Kernel drivers and device trees
│   ├── python/             # Python API and examples
│   ├── firmware/           # Bare-metal firmware
│   └── web/                # Web interface
├── docs/                   # Documentation and guides
├── scripts/                # Build and test automation
├── workflows/              # AI-driven design workflows
├── data/                   # Test results and benchmarks
└── README.md              # This file
```

## Documentation

### Design Documents
- [Hardware Design Guide](hardware/README.md) - FMC card schematic and layout
- [FPGA Architecture](fpga/README.md) - HDL design and AXI_LTC2387 IP integration
- [ADI IP Integration Guide](fpga/IP_INTEGRATION.md) - Step-by-step AXI_LTC2387 integration
- [Software Architecture](software/README.md) - Linux drivers and APIs
- [ADC Specifications](docs/adc-specifications.md) - LTC238x detailed specifications

### Migration Guides
- [Migration Checklist](docs/migration-checklist.md) - Step-by-step migration tasks
- [Comparison: Legacy vs New](docs/comparison-table.md) - DC2290A vs FMC design
- [Technical Specifications](docs/technical-specs.md) - Detailed specifications
- [Testing Procedures](docs/testing-guide.md) - Validation and characterization

### User Guides
- [Getting Started](docs/getting-started.md) - Quick start guide
- [Hardware Setup](docs/hardware-setup.md) - Assembly and connections
- [Software Installation](docs/software-install.md) - Driver and library installation
- [API Reference](docs/api-reference.md) - Python API documentation
- [Troubleshooting](docs/troubleshooting.md) - Common issues and solutions

## Contributing

Contributions are welcome! Please see [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

## License

This project is licensed under the Apache License 2.0 - see the [LICENSE](LICENSE) file for details.

## Key Design Decisions

### Why Zed Board?
- **Proven Platform**: Widely used Zynq-7000 development board
- **FMC Support**: Standard FMC LPC connector
- **Community**: Large user base and resources
- **Cost-Effective**: Affordable for development and deployment
- **Linux Support**: Mature PetaLinux BSP

### Why FMC Interface?
- **Standard**: VITA 57.1 industry standard
- **Portability**: Compatible with other FMC carrier boards
- **Pin Count**: FMC LPC sufficient for LVDS data and control
- **Mechanical**: Robust connector and mounting
- **Future-Proof**: Upgradeable to other Zynq platforms

### Single Board for 6 Variants
- **Component Options**: Populate different ADC parts
- **Software Configuration**: Runtime variant selection
- **Pin Compatibility**: ADCs share common pinout
- **Cost Efficiency**: Single PCB design and assembly
- **Simplified Inventory**: One board SKU

## Migration Advantages

| Aspect | Legacy (DC2290A) | New (FMC/Zed) | Improvement |
|--------|------------------|---------------|-------------|
| **Platform** | 6 separate boards | 1 FMC card | -83% hardware SKUs |
| **Interface** | USB 2.0 (480 Mbps) | AXI DMA (>1 Gbps) | >2x bandwidth |
| **Processing** | PC-based | FPGA + ARM | Real-time capable |
| **Latency** | ~10 ms (USB) | <1 µs (FPGA) | 10,000x faster |
| **Integration** | Standalone | Embedded system | Full control |
| **Expandability** | Limited | FPGA fabric | Unlimited |
| **Software** | Proprietary | Open source | Community support |

## Support

For questions or issues:
- **CN0577 Design**: Refer to [Analog Devices CN0577 page](https://www.analog.com/CN0577)
- **FMC Card Issues**: Open an issue in this repository
- **Zed Board Support**: [Xilinx/Avnet support forums](https://www.xilinx.com)
- **General Migration**: Contact the project team

## Acknowledgments

- **Analog Devices**: CN0577 reference design and technical support
- **Xilinx/AMD**: Zynq platform and development tools
- **Avnet**: Zed Board platform
- **Contributors**: See [CONTRIBUTORS.md](CONTRIBUTORS.md)
