# DC2290A-x FMC Quick Reference

Quick reference card for the DC2290A-x board migration to Zed Board project.

## DC2290A Board Variants (Legacy → FMC)

| Legacy Board | Part Number | Resolution | Rate | SNR | SFDR | Power |
|--------------|-------------|------------|------|-----|------|-------|
| **DC2290A-A** | LTC2387-18 | 18-bit | 15 Msps | 91.5 dB | 108 dB | 170 mW |
| **DC2290A-B** | LTC2387-16 | 16-bit | 15 Msps | 91.5 dB | 108 dB | 170 mW |
| **DC2290A-C** | LTC2386-18 | 18-bit | 10 Msps | 91.5 dB | 108 dB | 140 mW |
| **DC2290A-D** | LTC2386-16 | 16-bit | 10 Msps | 91.5 dB | 108 dB | 140 mW |
| **DC2290A-E** | LTC2385-18 | 18-bit | 5 Msps | 91.5 dB | 108 dB | 115 mW |
| **DC2290A-F** | LTC2385-16 | 16-bit | 5 Msps | 91.5 dB | 108 dB | 115 mW |

## Pin Summary (FMC LPC)

### LVDS Data (6-lane interface)
- **LA00_P/N**: D0A (LVDS data lane 0A)
- **LA01_P/N**: D0B (LVDS data lane 0B)
- **LA02_P/N**: D1A (LVDS data lane 1A)
- **LA03_P/N**: D1B (LVDS data lane 1B)
- **LA04_P/N**: D2A (LVDS data lane 2A)
- **LA05_P/N**: D2B (LVDS data lane 2B)
- **LA06_P/N**: DCO (Data clock output - 90/60/30 MHz)
- **LA07_P/N**: FCO (Frame clock output - 15/10/5 Msps)

### Control Signals
- **LA19_P**: CNV (Convert pulse)
- **LA20_P**: SPI_SCLK (SPI clock to ADC)
- **LA20_N**: SPI_MOSI (SPI data to ADC)
- **LA21_P**: SPI_MISO (SPI data from ADC)
- **LA21_N**: SPI_CS# (SPI chip select)

### Analog Connections (FMC SMA)
- **J2**: AIN_P/N (Differential analog input)
- **J3**: VREF_OUT (5.0V reference monitor)
- **J4**: CLK_IN (External clock input - optional)

## FPGA Resources

### ADI HDL IP Core
- **Name**: AXI_LTC2387
- **Repository**: https://github.com/analogdevicesinc/hdl
- **Path**: `hdl/library/axi_ltc2387/`
- **Documentation**: https://analogdevicesinc.github.io/hdl/library/axi_ltc2387/

### IP Configuration
```tcl
set_property -dict [list \
    CONFIG.LVDS_CMOS_N {1} \
    CONFIG.NUM_CHANNELS {1} \
    CONFIG.DATA_WIDTH {18} \
] [get_bd_cells axi_ltc2387_0]
```

### Resource Usage (Zynq-7020)
- LUTs: ~500
- FFs: ~800
- BRAMs: 0
- DSPs: 0

## Register Map

| Offset | Register | Access | Description |
|--------|----------|--------|-------------|
| 0x0000 | VERSION | RO | IP version |
| 0x0004 | ID | RO | IP ID |
| 0x0008 | SCRATCH | RW | Test register |
| 0x0010 | CONFIG | RW | Enable, mode, data width |
| 0x0014 | STATUS | RO | Locked, overflow, underrun |
| 0x0020 | CNV_CTRL | RW | CNV enable and mode |
| 0x0024 | CNV_PERIOD | RW | Sample rate divider |

### Sample Rate Configuration

With 100 MHz AXI clock:
- **15 Msps**: `CNV_PERIOD = 6`
- **10 Msps**: `CNV_PERIOD = 9`
- **5 Msps**: `CNV_PERIOD = 19`
- **1 Msps**: `CNV_PERIOD = 99`

Formula: `PERIOD = (AXI_CLK / SAMPLE_RATE) - 1`

## Software Quick Commands

### Device Tree
```dts
dc2290a_adc: axi-ltc2387@44a00000 {
    compatible = "adi,axi-ltc2387-1.0";
    reg = <0x44a00000 0x10000>;
    adi,board-variant = "dc2290a-a";  /* Specify which DC2290A board */
    adi,adc-part = "ltc2387-18";      /* ADC on that board */
};
```

### IIO Interface
```bash
# Check device
ls /sys/bus/iio/devices/iio:device0/

# Read single sample
cat /sys/bus/iio/devices/iio:device0/in_voltage0_raw

# Set sample rate
echo 15000000 > /sys/bus/iio/devices/iio:device0/sampling_frequency

# Enable buffered capture
echo 1 > /sys/bus/iio/devices/iio:device0/scan_elements/in_voltage0_en
echo 16384 > /sys/bus/iio/devices/iio:device0/buffer/length
echo 1 > /sys/bus/iio/devices/iio:device0/buffer/enable
cat /dev/iio:device0 > capture.bin
```

### Python API
```python
from dc2290a_fmc import DC2290A

# Initialize (replaces physical DC2290A board)
adc = DC2290A(uri='local:')  # Or 'ip:192.168.1.100'

# Configure variant (which DC2290A board to emulate)
adc.set_variant('dc2290a-a')  # LTC2387-18, 15 Msps
adc.sample_rate = 15e6

# Capture
data = adc.capture(16384)

# Analyze
enob = adc.calculate_enob(data)
sfdr = adc.calculate_sfdr(data)
print(f"ENOB: {enob:.2f} bits, SFDR: {sfdr:.2f} dBc")
```

### LibIIO from PC
```bash
# Install
sudo apt-get install libiio-utils python3-libiio

# Test connection
iio_info -n 192.168.1.100

# Capture data
iio_readdev -n 192.168.1.100 -b 16384 cn0577 voltage0 > data.bin
```

## Power Rails

| Rail | Voltage | Current | Source | Usage |
|------|---------|---------|--------|-------|
| VDD_ADC | 5.0V | 50 mA | LT3045 LDO | ADC analog supply |
| OVDD | 1.8V | 30 mA | ADP7104 LDO | ADC digital supply |
| VREF | 5.0V | 10 mA | LTC6655-5.0 | ADC reference |
| VDD_DRV | ±5V | 40 mA | LT3045 LDO | Amplifier supply |
| FMC_12V | 12V | 150 mA | From Zed Board | Input power |
| FMC_3V3 | 3.3V | 300 mA | From Zed Board | Logic power |

**Total Power**: ~4.2W

## Timing Specifications

### ADC Timing
| Parameter | LTC2387 | LTC2386 | LTC2385 |
|-----------|---------|---------|---------|
| Conversion Time | 66.7 ns | 100 ns | 200 ns |
| CNV Pulse Width (min) | 40 ns | 40 ns | 40 ns |
| CNV to Data Ready | <50 ns | <50 ns | <50 ns |
| DCO Frequency | 90 MHz | 60 MHz | 30 MHz |

### LVDS Setup/Hold (at FPGA)
- Setup Time: 2.0 ns (max)
- Hold Time: 0.5 ns (min)
- Use IDELAY for adjustment

## Performance Targets

| Metric | Target | Measurement |
|--------|--------|-------------|
| ENOB (18-bit) | ≥16.5 bits | FFT @ 1 MHz input |
| ENOB (16-bit) | ≥14.5 bits | FFT @ 1 MHz input |
| SFDR | ≥105 dBc | FFT @ 1 MHz input |
| SNR | ≥89 dB | FFT @ 1 MHz input |
| THD | ≤-100 dB | FFT @ 1 MHz input |
| Throughput | 15 Msps | Sustained rate |
| Latency | <10 µs | FPGA to DDR |

## Common Issues & Solutions

### Issue: DCO not locking
```bash
# Check IDELAY calibration
dmesg | grep -i idelay

# Verify clock is present
devmem 0x44a00014  # Read STATUS register, bit[0] should be 1
```

### Issue: No data capture
```bash
# Check CNV is enabled
devmem 0x44a00020  # Read CNV_CTRL, bit[0] should be 1

# Verify sample rate
devmem 0x44a00024  # Read CNV_PERIOD
```

### Issue: Data corruption
```bash
# Check for overflows
devmem 0x44a00014  # Read STATUS, check bit[1] (overflow)

# Verify DMA is configured
cat /proc/interrupts | grep dma
```

## Important Links

### Documentation
- [DC2290A Legacy Boards](https://www.analog.com) - Original evaluation boards
- [CN0577 Reference Design](https://www.analog.com/CN0577) - Design inspiration
- [LTC2387 Datasheet](https://www.analog.com/ltc2387)
- [ADI HDL Library](https://github.com/analogdevicesinc/hdl)
- [AXI_LTC2387 Docs](https://analogdevicesinc.github.io/hdl/library/axi_ltc2387/)
- [Zed Board Docs](https://www.xilinx.com/products/boards-and-kits/zedboard.html)

### Repository
- [Main README](README.md)
- [Hardware Design](hardware/README.md)
- [FPGA Design](fpga/README.md)
- [IP Integration Guide](fpga/IP_INTEGRATION.md)
- [Software Guide](software/README.md)
- [Getting Started](docs/getting-started.md)
- [ADC Specifications](docs/adc-specifications.md)
- [Project Roadmap](ROADMAP.md)

## Contact & Support

- **Repository Issues**: Open GitHub issue with appropriate label
- **ADI HDL Support**: https://github.com/analogdevicesinc/hdl/issues
- **EngineerZone**: https://ez.analog.com/
- **CN0577 Questions**: Reference design support

---

**Last Updated**: 2026-03-26
**Repository**: dc2290a-migration
**Project**: DC2290A-x Board Migration to Zed Board (FMC)
**Design Reference**: CN0577 Circuits from the Lab
