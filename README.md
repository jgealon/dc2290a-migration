# DC2290A-x Migration to Zed Board (FMC)

Migration of DC2290A-x legacy evaluation boards to Xilinx Zed Board platform using FMC (FPGA Mezzanine Card) interface, following [CN0577 reference design](https://www.analog.com/en/resources/reference-designs/circuits-from-the-lab/CN0577.html) best practices.

## Overview

This repository documents the migration of **six legacy DC2290A evaluation boards** (DC2290A-A through DC2290A-F) to a modern Xilinx Zynq-based platform. The goal is to consolidate all 6 variants into a **single FMC card** compatible with the Zed Board, using the same LTC238x ADC family and design approach proven in the CN0577 reference design.

### Migration Goals

- **Consolidate 6 Boards**: Replace DC2290A-A/B/C/D/E/F with single FMC design
- **Modern Platform**: Xilinx Zynq-7000 SoC (Zed Board)
- **Standard Interface**: FMC (VITA 57.1) connector for portability
- **Improved Performance**: Leverage FPGA processing vs. USB bottleneck
- **CN0577-Inspired**: Follow proven CN0577 design patterns and best practices

## Project Background

The DC2290A family consists of 6 legacy evaluation boards for high-speed ADC characterization. This project consolidates them into a modern, unified platform.

### Legacy Platform (DC2290A Boards)
- **Boards**: DC2290A-A, DC2290A-B, DC2290A-C, DC2290A-D, DC2290A-E, DC2290A-F (6 separate boards)
- **ADCs**: LTC2387/2386/2385 family (16/18-bit, 5/10/15 Msps)
- **Interface**: LVDS to DC718 USB adapter
- **Limitations**: 6 different boards, USB bottleneck, limited FPGA processing

### Target Platform (New FMC Design)
- **Board**: Single FMC card for Zed Board supporting all 6 variants
- **SoC**: Xilinx Zynq-7000 (XC7Z020-CLG484)
- **Interface**: FMC LPC (Low Pin Count) connector
- **Design Reference**: CN0577 (proven ADC characterization design)
- **Benefits**: One board, FPGA processing, better performance, easier inventory

## DC2290A Variant Support

Each DC2290A board variant will be supported on the single FMC design through component options or configuration. All LTC238x ADCs share a common footprint and interface, using ADI's proven [AXI_LTC2387 IP core](https://analogdevicesinc.github.io/hdl/library/axi_ltc2387/index.html).

| Legacy Board | ADC Part Number | Resolution | Sample Rate | Interface | Power |
|--------------|-----------------|------------|-------------|-----------|-------|
| **DC2290A-A** | LTC2387-18 | 18-bit | 15 Msps | 6-lane LVDS | 170 mW |
| **DC2290A-B** | LTC2387-16 | 16-bit | 15 Msps | 6-lane LVDS | 170 mW |
| **DC2290A-C** | LTC2386-18 | 18-bit | 10 Msps | 6-lane LVDS | 140 mW |
| **DC2290A-D** | LTC2386-16 | 16-bit | 10 Msps | 6-lane LVDS | 140 mW |
| **DC2290A-E** | LTC2385-18 | 18-bit | 5 Msps | 6-lane LVDS | 115 mW |
| **DC2290A-F** | LTC2385-16 | 16-bit | 5 Msps | 6-lane LVDS | 115 mW |

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
- **ADI HDL IP Core**: Uses proven [AXI_LTC2387](https://analogdevicesinc.github.io/hdl/library/axi_ltc2387/) IP (same as CN0577)
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
./program_fpga.sh dc2290a_fmc.bit

# Install Python support
cd software/python/
pip install -e .

# Run test capture (specify which DC2290A variant)
python examples/basic_capture.py --board dc2290a-a
```

### Quick Test

```python
from dc2290a_fmc import DC2290A

# Initialize with board variant (replaces physical board)
adc = DC2290A(variant='dc2290a-a', sample_rate=15e6)

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

### Why Consolidate into Single FMC Board?
- **Inventory**: One board SKU instead of 6 different DC2290A boards
- **Pin Compatibility**: All LTC238x ADCs share same footprint
- **Flexibility**: Software-configurable for any variant
- **Cost**: Single PCB design and assembly process
- **Maintenance**: One design to support and update

### Why Zed Board Platform?
- **Proven**: Widely used Zynq-7000 development board
- **FMC Support**: Standard FMC LPC connector
- **CN0577 Reference**: Follows proven evaluation board approach
- **Community**: Large user base and resources
- **Linux Support**: Mature PetaLinux BSP

### Why Follow CN0577 Design?
- **Proven Design**: CN0577 is a validated ADC characterization platform
- **Same ADCs**: Uses LTC238x family (same as DC2290A boards)
- **ADI IP Core**: Leverages AXI_LTC2387 (production-tested)
- **Best Practices**: Signal conditioning, layout, FPGA architecture
- **Reference**: Complete schematics and documentation available

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
- **DC2290A Boards**: Legacy board documentation and specifications
- **CN0577 Reference**: [Analog Devices CN0577 page](https://www.analog.com/CN0577) for design patterns
- **FMC Migration Issues**: Open an issue in this repository
- **Zed Board Support**: [Xilinx/Avnet support forums](https://www.xilinx.com)
- **Project Questions**: Contact the migration team

## Acknowledgments

- **Analog Devices**: DC2290A legacy boards, CN0577 reference design, HDL IP cores
- **Xilinx/AMD**: Zynq platform and development tools
- **Avnet**: Zed Board platform
- **Contributors**: See [CONTRIBUTORS.md](CONTRIBUTORS.md)
