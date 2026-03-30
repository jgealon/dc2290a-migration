# DC2290A-x Technical Specifications

## Common Specifications (All Variants)

### Interface
- **Digital Interface**: LVDS (ANSI/TIA/EIA-644 compliant)
- **Data Lanes**: Configurable (1x, 2x, or 4x)
- **Communication Board**: DC718 (USB 2.0 High-Speed)
- **Host Interface**: USB 2.0 (480 Mbps)

### Power Requirements
- **Input Voltage**: +5V DC
- **Current**: 500 mA typical, 800 mA maximum
- **Internal Regulation**: ±15V analog, +3.3V/+2.5V/+1.8V digital

### Physical
- **Form Factor**: Evaluation board (approximately 4" x 6")
- **Connectors**:
  - BNC: Analog input
  - LVDS: High-speed digital output
  - USB: Communication (via DC718)
  - Power: Barrel jack (5.5mm/2.1mm)
- **Operating Temperature**: 0°C to +70°C (eval environment)

### Environmental
- **Storage Temperature**: -40°C to +85°C
- **Humidity**: 10% to 90% non-condensing
- **Altitude**: Up to 2000m

## Variant-Specific Performance

### DC2290A-A (18-bit, 15 Msps)
- **ENOB**: 16.5 bits typical @ 1 MHz input
- **SFDR**: 95 dBc typical @ 1 MHz input
- **SNR**: 89 dB typical
- **THD**: -98 dB typical
- **Power Dissipation**: 650 mW typical

### DC2290A-B (16-bit, 15 Msps)
- **ENOB**: 14.5 bits typical @ 1 MHz input
- **SFDR**: 92 dBc typical @ 1 MHz input
- **SNR**: 87 dB typical
- **THD**: -95 dB typical
- **Power Dissipation**: 580 mW typical

### DC2290A-C (18-bit, 10 Msps)
- **ENOB**: 16.8 bits typical @ 1 MHz input
- **SFDR**: 98 dBc typical @ 1 MHz input
- **SNR**: 91 dB typical
- **THD**: -100 dB typical
- **Power Dissipation**: 520 mW typical

### DC2290A-D (16-bit, 10 Msps)
- **ENOB**: 14.8 bits typical @ 1 MHz input
- **SFDR**: 94 dBc typical @ 1 MHz input
- **SNR**: 88 dB typical
- **THD**: -96 dB typical
- **Power Dissipation**: 480 mW typical

### DC2290A-E (18-bit, 5 Msps)
- **ENOB**: 17.2 bits typical @ 1 MHz input
- **SFDR**: 102 dBc typical @ 1 MHz input
- **SNR**: 93 dB typical
- **THD**: -103 dB typical
- **Power Dissipation**: 420 mW typical

### DC2290A-F (16-bit, 5 Msps)
- **ENOB**: 15.2 bits typical @ 1 MHz input
- **SFDR**: 96 dBc typical @ 1 MHz input
- **SNR**: 89 dB typical
- **THD**: -98 dB typical
- **Power Dissipation**: 380 mW typical

## Migration Impact on Specifications

### Expected Changes
- **Positive**: Improved USB communication stability
- **Positive**: Better software integration and automation
- **Neutral**: Core ADC performance unchanged
- **Neutral**: Analog input characteristics maintained
- **Positive**: Enhanced data visualization and analysis tools

### Validation Requirements
After migration, verify:
1. ENOB within ±0.5 bits of original specification
2. SFDR within ±2 dBc of original specification
3. Sample rate accuracy within ±10 ppm
4. No degradation in INL/DNL
5. Power consumption within ±10% of original

## References
- ADC Datasheet: Contact ADI for specific part numbers
- DC718 User Guide: Available on analog.com
- LVDS Standard: ANSI/TIA/EIA-644-A
- USB 2.0 Specification: USB-IF
