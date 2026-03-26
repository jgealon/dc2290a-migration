# DC2290A-x FMC Card Hardware Design

Hardware design files for the DC2290A-x migration FMC card compatible with Xilinx Zed Board. This design consolidates 6 legacy DC2290A boards into a single FMC card, following CN0577 reference design patterns.

## Design Overview

The FMC card replaces 6 legacy DC2290A evaluation boards with a single VITA 57.1 compliant FMC Low Pin Count (LPC) design. It supports all 6 DC2290A board variants (A through F) through component options, following the proven CN0577 reference design approach.

### Key Features

- **FMC LPC Interface**: 68 user-defined pins, compliant with VITA 57.1
- **ADC Support**: 16/18-bit SAR ADCs at 5/10/15 Msps
- **Analog Front-End**: Precision amplifiers and signal conditioning
- **Power**: Derived from FMC 12V and 3.3V rails
- **Clock**: Onboard oscillator + optional external clock input
- **Form Factor**: Standard FMC dimensions (69mm x 76mm)

## Directory Structure

```
hardware/
├── fmc_card/
│   ├── schematic/          # OrCAD/Altium schematic files
│   ├── pcb/                # PCB layout files
│   ├── libraries/          # Component libraries
│   └── pdf/                # PDF schematics and assembly drawings
├── bom/
│   ├── bom_master.xlsx     # Complete bill of materials
│   ├── bom_variant_a.csv   # Config A specific components
│   ├── bom_variant_b.csv   # Config B specific components
│   └── ...
├── gerbers/
│   ├── fabrication/        # Gerber files for PCB fab
│   ├── assembly/           # Pick-and-place and assembly drawings
│   └── stackup.pdf         # PCB stackup specification
├── 3d_models/
│   ├── step/               # STEP files for mechanical design
│   └── renders/            # 3D renders
├── testing/
│   ├── test_procedures/    # Hardware test procedures
│   └── calibration/        # Calibration procedures
└── README.md               # This file
```

## FMC Pin Assignment

### High-Speed Signals (ADC Data)

LTC238x uses 6-lane LVDS output (18-bit data serialized):

| FMC Pin | Signal | Direction | Description |
|---------|--------|-----------|-------------|
| LA00_P/N | D0A_P/N | Input | LVDS data lane 0A |
| LA01_P/N | D0B_P/N | Input | LVDS data lane 0B |
| LA02_P/N | D1A_P/N | Input | LVDS data lane 1A |
| LA03_P/N | D1B_P/N | Input | LVDS data lane 1B |
| LA04_P/N | D2A_P/N | Input | LVDS data lane 2A |
| LA05_P/N | D2B_P/N | Input | LVDS data lane 2B |
| LA06_P/N | DCO_P/N | Input | Data clock output (LVDS) |
| LA07_P/N | FCO_P/N | Input | Frame clock output (LVDS) |

### Control Signals

| FMC Pin | Signal | Direction | Description |
|---------|--------|-----------|-------------|
| LA19_P | ADC_CNV | Output | ADC convert trigger |
| LA19_N | ADC_RESET | Output | ADC reset |
| LA20_P | SPI_SCLK | Output | SPI clock for ADC config |
| LA20_N | SPI_MOSI | Output | SPI data to ADC |
| LA21_P | SPI_MISO | Input | SPI data from ADC |
| LA21_N | SPI_CS | Output | SPI chip select |

### Analog Signals

| FMC Pin | Signal | Type | Description |
|---------|--------|------|-------------|
| J2-1/2 | AIN_P/N | SMA | Differential analog input |
| J3-1/2 | VREF_OUT | SMA | Reference voltage output |
| J4-1/2 | CLK_IN | SMA | External clock input (optional) |

## Power Budget

| Rail | Source | Current | Power |
|------|--------|---------|-------|
| +12V | FMC P12V0 | 150 mA | 1.8 W |
| +3.3V_FMC | FMC P3V3 | 300 mA | 1.0 W |
| +5V_ANALOG | LDO from 12V | 200 mA | 1.0 W |
| -5V_ANALOG | Inverting DC-DC | 50 mA | 0.25 W |
| +1.8V_DIGITAL | LDO from 3.3V | 100 mA | 0.18 W |
| **Total** | | | **4.23 W** |

## PCB Specifications

- **Layers**: 6-layer stackup
- **Dimensions**: 69mm x 76mm (standard FMC)
- **Thickness**: 1.6mm
- **Material**: FR-4, Tg 170°C
- **Copper**: 1 oz outer layers, 0.5 oz inner layers
- **Surface Finish**: ENIG (gold) for critical analog paths
- **Impedance Control**: 100Ω differential, 50Ω single-ended

### Layer Stackup

```
Layer 1 (Top):       Signal - Components, routing
Layer 2 (GND):       Ground plane - AGND/DGND split
Layer 3 (Power):     Power distribution - 3.3V, 5V, 1.8V
Layer 4 (Power):     Power distribution - 12V, -5V
Layer 5 (GND):       Ground plane - solid AGND
Layer 6 (Bottom):    Signal - routing, test points
```

## Component Selection

### DC2290A Board to ADC Mapping

| Legacy Board | ADC Part Number | Resolution | Sample Rate | Package | Pin Compatible |
|--------------|-----------------|------------|-------------|---------|----------------|
| **DC2290A-A** | LTC2387-18 | 18-bit | 15 Msps | 24-Lead QFN (4mm × 4mm) | Yes |
| **DC2290A-B** | LTC2387-16 | 16-bit | 15 Msps | 24-Lead QFN (4mm × 4mm) | Yes |
| **DC2290A-C** | LTC2386-18 | 18-bit | 10 Msps | 24-Lead QFN (4mm × 4mm) | Yes |
| **DC2290A-D** | LTC2386-16 | 16-bit | 10 Msps | 24-Lead QFN (4mm × 4mm) | Yes |
| **DC2290A-E** | LTC2385-18 | 18-bit | 5 Msps | 24-Lead QFN (4mm × 4mm) | Yes |
| **DC2290A-F** | LTC2385-16 | 16-bit | 5 Msps | 24-Lead QFN (4mm × 4mm) | Yes |

**Note**: All LTC238x ADCs are pin-compatible, enabling a single FMC PCB design to replace all 6 DC2290A boards through component substitution.

### Key Components

**ADC and Analog Front-End**:
- **ADC**: LTC2387/2386/2385 (18-bit or 16-bit, 5/10/15 Msps)
- **ADC Driver**: ADA4945-1 (low noise, low distortion differential amplifier)
- **Reference**: LTC6655-5.0 (5.0V ultra-low noise reference, 0.25 ppm typ)
- **Anti-Alias Filter**: 5th-order Butterworth, programmable bandwidth

**Clock and Timing**:
- **Encode Clock**: Low jitter oscillator or external clock input
- **Clock Buffer**: LTC6957 (ultralow jitter clock fanout buffer)

**Power Management**:
- **Analog Supply**: LT3045 (ultralow noise LDO for ±5V analog)
- **Digital Supply**: ADP7104 (1.8V digital LDO)
- **DC-DC Converter**: LTM8002 (Silent Switcher for efficiency)

**Digital Interface**:
- **LVDS Buffers**: Integrated in LTC238x (no external buffers needed)
- **LVDS Termination**: 100Ω differential on FMC side

## Design Considerations

### Analog Signal Path

1. **Input Conditioning**
   - Anti-aliasing filter (5th order Butterworth)
   - Common-mode rejection
   - ESD protection (TVS diodes)
   - Input impedance: 1 MΩ || 10 pF

2. **ADC Driver Stage**
   - Fully differential amplifier (ADA4940)
   - Gain: 0.5x to 2x (configurable via resistors)
   - Bandwidth: >50 MHz
   - SNR: >90 dB

3. **Reference System**
   - Buffered 5V reference
   - Low noise (<3 µVpp, 0.1 Hz to 10 Hz)
   - Temperature coefficient: <3 ppm/°C
   - Reference bypass (0.1µF + 10µF ceramic)

### Digital Interface

1. **LVDS Signals**
   - Trace impedance: 100Ω differential
   - Length matching: ±5 mils within pairs, ±50 mils between pairs
   - Via minimization on high-speed traces
   - Ground stitching vias along trace routes

2. **Clock Distribution**
   - Dedicated clock routing layer
   - Star topology from clock source
   - Jitter budget: <1 ps RMS
   - EMI shielding with ground pours

### Power Integrity

1. **Power Planes**
   - Separate analog and digital ground regions
   - Single-point connection at FMC connector
   - Ferrite bead isolation for sensitive rails
   - Bulk capacitance: 100µF per power rail

2. **Decoupling Strategy**
   - 0.1µF ceramic at every IC power pin
   - 10µF tantalum for bulk capacitance
   - Minimize loop inductance with vias

## Manufacturing Notes

### Assembly

- **SMT Components**: Reflow profile for lead-free solder
- **Critical Components**: X-ray inspection for ADC and BGA devices
- **Analog Section**: Hand-solder option for critical passive components
- **Test Points**: Provided for all power rails and critical signals

### Testing

1. **Bare Board Test**
   - Continuity testing
   - Impedance verification (TDR)
   - Insulation resistance

2. **Functional Test**
   - Power-up sequence verification
   - Digital interface loopback
   - ADC data capture
   - Performance characterization

## Variant Configuration

### Hardware Options

The board supports variants through:

1. **Component Depopulation**: DNP (Do Not Populate) options for unused features
2. **ADC Selection**: Different ADC part numbers (same footprint)
3. **Gain Settings**: Resistor value changes for amplifier gain
4. **Filter Bandwidth**: Capacitor values for anti-aliasing filter

### Configuration Table

| Option | Config A-B (15 Msps) | Config C-D (10 Msps) | Config E-F (5 Msps) |
|--------|---------------------|---------------------|-------------------|
| ADC Part | High-speed variant | Mid-speed variant | Low-power variant |
| Filter BW | 20 MHz | 15 MHz | 10 MHz |
| Driver Gain | 1.0x | 1.0x | 0.5x |
| Clock Freq | 15 MHz | 10 MHz | 5 MHz |

## Getting Started

### Design Files

1. Open schematic in OrCAD/Altium
2. Review component selection for target variant
3. Verify FMC pinout against Zed Board constraints
4. Run DRC (Design Rule Check)

### PCB Layout

1. Import netlist from schematic
2. Place FMC connector and critical components
3. Route high-speed LVDS pairs first
4. Complete power distribution
5. Add ground stitching and thermal vias
6. Run DRC and export Gerbers

### Order PCB

1. Export Gerber files (RS-274X format)
2. Generate drill files (Excellon format)
3. Create assembly drawings
4. Send to PCB manufacturer (recommended: 6-layer capability)

## Resources

- [FMC Standard (VITA 57.1)](https://www.vita.com)
- [Zed Board Schematic](https://www.xilinx.com/products/boards-and-kits/zedboard.html)
- [ADI PCB Design Guide](https://www.analog.com/en/design-center/landing-pages/001/printed-circuit-board-design-resources.html)
- [High-Speed Digital Design Resources](https://www.analog.com)

## Support

For hardware design questions:
- Open an issue with the `hardware` label
- Contact the hardware team
- Refer to CN0577 reference schematics

## Status

| Item | Status | Notes |
|------|--------|-------|
| Schematic Design | In Progress | Block diagram complete |
| Component Selection | In Progress | ADC parts verified |
| PCB Layout | Not Started | Awaiting schematic finalization |
| Prototype Build | Not Started | Q2 2026 target |
| Testing | Not Started | Q3 2026 target |
