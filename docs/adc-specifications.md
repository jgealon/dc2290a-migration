# LTC238x ADC Specifications

Detailed specifications for the LTC2387/2386/2385 ADC family used in CN0577 FMC design.

## ADC Family Overview

The LTC238x is a family of low noise, high-speed, 16-bit/18-bit successive approximation register (SAR) ADCs from Analog Devices. These ADCs feature a 6-lane LVDS digital interface optimized for high throughput and low latency data acquisition.

## Variant Comparison

| Feature | LTC2387 | LTC2386 | LTC2385 |
|---------|---------|---------|---------|
| **Sample Rate** | 15 Msps | 10 Msps | 5 Msps |
| **Resolution** | 16-bit or 18-bit | 16-bit or 18-bit | 16-bit or 18-bit |
| **SNR (typ)** | 91.5 dB | 91.5 dB | 91.5 dB |
| **SFDR (typ)** | 108 dB | 108 dB | 108 dB |
| **THD (typ)** | -108 dB | -108 dB | -108 dB |
| **Input BW** | 300 MHz | 300 MHz | 300 MHz |
| **Power** | 170 mW | 140 mW | 115 mW |
| **Part Numbers** | LTC2387-16/18 | LTC2386-16/18 | LTC2385-16/18 |

## Detailed Specifications

### LTC2387-18 (18-bit, 15 Msps)

**Performance**:
- Resolution: 18 bits (262,144 codes)
- Sample Rate: 15 Msps (max)
- SNR: 91.5 dB (typ) @ 1 MHz input
- ENOB: 16.5 bits (typ)
- SFDR: 108 dB (typ) @ 1 MHz input
- THD: -108 dB (typ) @ 1 MHz input
- IMD: -106 dB (typ)

**Analog Input**:
- Input Type: Fully differential
- Input Range: ±VREF (programmable)
- Common Mode: VCM = VDD/2
- Input Impedance: 10 kΩ || 5 pF
- Input Bandwidth (-3 dB): 300 MHz

**Digital Interface**:
- Interface Type: 6-lane LVDS
- Data Format: Offset binary or two's complement
- Output Coding: 18-bit
- DCO Frequency: 90 MHz (6× oversample)
- Logic Levels: LVDS (ANSI/TIA/EIA-644)

**Timing**:
- Conversion Time: 66.7 ns (at 15 Msps)
- Acquisition Time: Instantaneous (SAR architecture)
- Pipeline Delay: 0 samples (no latency)
- CNV Pulse Width: 40 ns (min)

**Power**:
- Supply Voltage:
  - VDD (analog): 5.0V ±5%
  - OVDD (digital): 1.8V to 3.3V
- Power Dissipation: 170 mW (typ) @ 15 Msps
- Power-Down Current: < 1 µA

### LTC2387-16 (16-bit, 15 Msps)

Same as LTC2387-18 except:
- Resolution: 16 bits (65,536 codes)
- ENOB: 15.5 bits (typ)
- Output data: 16 bits (2 LSBs unused in 6-lane LVDS)

### LTC2386-18 (18-bit, 10 Msps)

**Performance**:
- Sample Rate: 10 Msps (max)
- SNR: 91.5 dB (typ)
- ENOB: 16.5 bits (typ)
- SFDR: 108 dB (typ)
- THD: -108 dB (typ)

**Digital Interface**:
- DCO Frequency: 60 MHz (6× oversample @ 10 Msps)

**Timing**:
- Conversion Time: 100 ns
- CNV Pulse Width: 40 ns (min)

**Power**:
- Power Dissipation: 140 mW (typ) @ 10 Msps

### LTC2386-16 (16-bit, 10 Msps)

Same as LTC2386-18 except:
- Resolution: 16 bits
- ENOB: 15.5 bits (typ)

### LTC2385-18 (18-bit, 5 Msps)

**Performance**:
- Sample Rate: 5 Msps (max)
- SNR: 91.5 dB (typ)
- ENOB: 16.5 bits (typ)
- SFDR: 108 dB (typ)
- THD: -108 dB (typ)

**Digital Interface**:
- DCO Frequency: 30 MHz (6× oversample @ 5 Msps)

**Timing**:
- Conversion Time: 200 ns
- CNV Pulse Width: 40 ns (min)

**Power**:
- Power Dissipation: 115 mW (typ) @ 5 Msps

### LTC2385-16 (16-bit, 5 Msps)

Same as LTC2385-18 except:
- Resolution: 16 bits
- ENOB: 15.5 bits (typ)

## Interface Details

### LVDS Digital Output

All LTC238x variants use the same 6-lane LVDS interface:

| Signal | Type | Description |
|--------|------|-------------|
| D0A, D0B | LVDS Output | Data lane 0 (bits 17,16 for 18-bit; bits 15,14 for 16-bit) |
| D1A, D1B | LVDS Output | Data lane 1 (bits 15,14 / 13,12) |
| D2A, D2B | LVDS Output | Data lane 2 (bits 13,12 / 11,10) |
| D3A, D3B* | LVDS Output | Data lane 3 (bits 11,10 / 9,8) |
| D4A, D4B* | LVDS Output | Data lane 4 (bits 9,8 / 7,6) |
| D5A, D5B* | LVDS Output | Data lane 5 (bits 7,6 / 5,4) |
| D6A, D6B* | LVDS Output | Data lane 6 (bits 5,4 / 3,2) |
| D7A, D7B* | LVDS Output | Data lane 7 (bits 3,2 / 1,0) |
| DCO | LVDS Output | Data clock output (6× sample rate) |
| FCO | LVDS Output | Frame clock output (sample rate) |

*Note: For 18-bit mode, 6 lanes used; for 16-bit, some lanes carry fewer bits.

### SPI Configuration Interface

| Signal | Direction | Description |
|--------|-----------|-------------|
| SCK | Input | SPI clock (max 50 MHz) |
| SDI | Input | SPI data input |
| SDO | Output | SPI data output |
| CS# | Input | Chip select (active low) |

### Control Signals

| Signal | Direction | Description |
|--------|-----------|-------------|
| CNV+ / CNV- | Input (Differential) | Convert command (rising edge triggers conversion) |
| CLKOUT+ / CLKOUT-* | Output | Optional clock output |
| PD | Input | Power down (active high) |

*Some variants only

## Typical Performance Curves

### SNR vs. Input Frequency

| Input Freq | LTC2387-18 | LTC2386-18 | LTC2385-18 |
|------------|------------|------------|------------|
| 100 kHz | 91.5 dB | 91.5 dB | 91.5 dB |
| 1 MHz | 91.5 dB | 91.5 dB | 91.5 dB |
| 10 MHz | 91.0 dB | 91.0 dB | 91.0 dB |
| 50 MHz | 89.5 dB | 89.5 dB | 89.5 dB |
| 100 MHz | 87.5 dB | 87.5 dB | 87.5 dB |

### SFDR vs. Input Frequency

| Input Freq | LTC2387-18 | LTC2386-18 | LTC2385-18 |
|------------|------------|------------|------------|
| 100 kHz | 110 dB | 110 dB | 110 dB |
| 1 MHz | 108 dB | 108 dB | 108 dB |
| 10 MHz | 108 dB | 108 dB | 108 dB |
| 50 MHz | 105 dB | 105 dB | 105 dB |
| 100 MHz | 102 dB | 102 dB | 102 dB |

### Power vs. Sample Rate

| Sample Rate | LTC2387 | LTC2386 | LTC2385 |
|-------------|---------|---------|---------|
| 5 Msps | 115 mW | 115 mW | 115 mW |
| 10 Msps | 140 mW | 140 mW | N/A |
| 15 Msps | 170 mW | N/A | N/A |

## Temperature Performance

All variants maintain specifications over:
- Operating Range: -40°C to +125°C
- Typical drift: <1 ppm/°C (with LTC6655 reference)

## Package Information

- **Package**: 24-Lead QFN (4mm × 4mm × 0.75mm)
- **Thermal Pad**: Exposed pad for heat dissipation
- **Pin Pitch**: 0.5 mm
- **RoHS Compliant**: Yes
- **MSL Rating**: MSL 3 (per JEDEC J-STD-020)

## Ordering Information

| Part Number | Resolution | Sample Rate | Temperature Range | Package |
|-------------|------------|-------------|-------------------|---------|
| LTC2387IUP-18#PBF | 18-bit | 15 Msps | -40°C to +125°C | 24-QFN |
| LTC2387IUP-16#PBF | 16-bit | 15 Msps | -40°C to +125°C | 24-QFN |
| LTC2386IUP-18#PBF | 18-bit | 10 Msps | -40°C to +125°C | 24-QFN |
| LTC2386IUP-16#PBF | 16-bit | 10 Msps | -40°C to +125°C | 24-QFN |
| LTC2385IUP-18#PBF | 18-bit | 5 Msps | -40°C to +125°C | 24-QFN |
| LTC2385IUP-16#PBF | 16-bit | 5 Msps | -40°C to +125°C | 24-QFN |

## Application Notes

### Recommended Operating Conditions

- **Supply Voltage**: VDD = 5.0V ±5%
- **Digital Supply**: OVDD = 1.8V to 3.3V
- **Reference**: External 5.0V (LTC6655-5.0)
- **Input Range**: ±4.096V typical (with 5V reference)
- **Common Mode**: 2.5V (VDD/2)

### Layout Considerations

1. **Power Planes**: Separate analog and digital ground planes
2. **Bypassing**: 0.1µF + 10µF at VDD, OVDD pins
3. **Reference**: Low-noise reference with 0.1µF + 10µF bypass
4. **LVDS**: 100Ω differential impedance, length-matched pairs
5. **Clock**: Low-jitter encode clock (<1 ps RMS)

### Performance Optimization Tips

1. Use ultra-low noise reference (LTC6655)
2. Minimize jitter on encode clock
3. Use differential input drive (ADA4945-1)
4. Proper LVDS termination (100Ω differential)
5. Shield analog input from digital noise

## CN0577 Specific Configuration

In the CN0577 FMC design:
- **Reference**: LTC6655-5.0 (5.0V, 0.25 ppm noise)
- **Driver**: ADA4945-1 (low noise, low distortion)
- **Anti-Alias Filter**: 5th-order Butterworth
- **Encode Clock**: Low-jitter TCXO or crystal oscillator
- **LVDS Receiver**: Xilinx Zynq ISERDES (HR I/O banks)

## Resources

- [LTC2387 Product Page](https://www.analog.com/ltc2387)
- [LTC2386 Product Page](https://www.analog.com/ltc2386)
- [LTC2385 Product Page](https://www.analog.com/ltc2385)
- [LTC2387 Datasheet PDF](https://www.analog.com/media/en/technical-documentation/data-sheets/ltc2387-18.pdf)
- [CN0577 Reference Design](https://www.analog.com/CN0577)
- [ADI HDL AXI_LTC2387 IP](https://analogdevicesinc.github.io/hdl/library/axi_ltc2387/)

## Revision History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2026-03-26 | Initial specifications document |
