# Getting Started with CN0577 FMC

Quick start guide for setting up and using the CN0577 FMC card with Zed Board.

## Prerequisites

### Hardware Required

- **Zed Board** (Xilinx Zynq-7000 development board)
- **CN0577 FMC Card** (with desired ADC variant populated)
- **12V Power Supply** (2A minimum) for Zed Board
- **microSD Card** (8GB minimum) with ADI Kuiper Linux image
- **USB Cable** (Type-A to Micro-B) for UART console
- **Ethernet Cable** (for network connectivity)
- **Signal Generator** (for ADC input stimulus)
- **SMA Cables** (for analog connections)

### Software Required

- **Vivado** 2022.2 or later (for FPGA programming)
- **ADI Kuiper Linux** (ADI's Linux distribution for evaluation boards)
- **Python** 3.8 or later
- **LibIIO** library
- **Terminal emulator** (PuTTY, Tera Term, or screen)

### Optional Equipment

- Oscilloscope (for signal verification)
- Logic analyzer (for digital debugging)
- Multimeter (for power rail verification)

## Hardware Setup

### Step 1: Prepare the Zed Board

1. **Check Board Revision**
   - Verify you have Zed Board Rev C or later
   - Check for FMC LPC connector (J44)

2. **Configure Boot Mode**
   - Set boot mode jumpers (JP7-JP11) for SD card boot:
     - JP7: 1-2 (GND)
     - JP8: 2-3 (GND)
     - JP9: 2-3 (GND)
     - JP10: 2-3 (GND)
     - JP11: 2-3 (GND)

3. **Insert microSD Card**
   - Download pre-built ADI Kuiper Linux image from releases page
   - Write image to microSD card (use Win32DiskImager or dd)
   - Insert card into Zed Board SD slot (J12)

### Step 2: Install FMC Card

```
⚠️ CAUTION: Power off Zed Board before installing FMC card!
```

1. **Align FMC Card**
   - Orient FMC card so connector aligns with J44 on Zed Board
   - Note the alignment key on one side of connector

2. **Install Card**
   - Carefully align the 160-pin FMC connector
   - Gently press card into connector (do not force)
   - You should feel it seat firmly

3. **Secure with Standoffs**
   - Install M3 standoffs at the four mounting holes
   - Tighten screws finger-tight (do not overtighten)

4. **Verify Installation**
   - Check that card is level and fully seated
   - Ensure no pins are bent
   - Verify standoffs are secure

### Step 3: Connect Peripherals

1. **Console UART**
   - Connect USB cable from PC to J14 (USB-UART)
   - Install Digilent drivers if needed (Windows)
   - Open serial terminal:
     - Baud: 115200
     - Data bits: 8
     - Parity: None
     - Stop bits: 1
     - Flow control: None

2. **Ethernet**
   - Connect Ethernet cable to J20
   - Connect to network with DHCP server
   - Or use direct connection with static IP

3. **Power**
   - Connect 12V power supply to J18
   - Do not turn on yet!

### Step 4: Connect Analog Signals

1. **Analog Input**
   - Connect signal generator to J2 on FMC card (AIN_P/N)
   - Use SMA cable
   - For differential input, connect both + and - signals
   - For single-ended, connect signal to + and ground - terminal

2. **Reference Output** (Optional)
   - If monitoring reference voltage, connect scope to J3

3. **External Clock** (Optional)
   - If using external clock, connect to J4
   - Otherwise, onboard oscillator will be used

## First Power-On

### Step 1: Initial Power-Up

1. **Double-check Connections**
   - FMC card properly seated
   - SD card inserted
   - Console connected
   - Power supply connected but OFF

2. **Power On**
   - Have console terminal open and connected
   - Turn on 12V power supply
   - You should see:
     - Power LED (D14) illuminate on Zed Board
     - Done LED (D3) illuminate after ~10 seconds
     - Boot messages in console

3. **Monitor Boot Process**
   ```
   Xilinx First Stage Boot Loader
   Release 2022.2
   ...
   [OK] Started Analog Devices IIO Daemon
   ...
   Analog Devices Kuiper Linux zedboard /dev/ttyPS0

   zedboard login:
   ```

4. **Login**
   - Username: `analog` (or `root`)
   - Password: `analog` (or as configured)

### Step 2: Verify System

1. **Check Kernel Version**
   ```bash
   uname -a
   # Should show: Linux zedboard 5.15.x...
   ```

2. **Check IIO Devices**
   ```bash
   ls /sys/bus/iio/devices/
   # Should show: iio:device0
   ```

3. **Verify CN0577 Driver**
   ```bash
   dmesg | grep cn0577
   # Should show driver loading messages
   ```

4. **Check Network**
   ```bash
   ifconfig eth0
   # Should show IP address
   ping 8.8.8.8
   # Should get responses
   ```

## First Data Capture

### Using Command Line

1. **Check IIO Device**
   ```bash
   cd /sys/bus/iio/devices/iio:device0/
   cat name
   # Should output: cn0577
   ```

2. **Read Single Sample**
   ```bash
   cat in_voltage0_raw
   # Outputs: raw ADC value (e.g., 32768)
   ```

3. **Calculate Voltage**
   ```bash
   # Read scale factor
   cat in_voltage0_scale
   # Multiply: voltage = raw * scale
   ```

4. **Buffered Capture**
   ```bash
   # Enable channel
   echo 1 > scan_elements/in_voltage0_en

   # Set buffer size
   echo 16384 > buffer/length

   # Enable buffer
   echo 1 > buffer/enable

   # Read data
   cat /dev/iio:device0 > /tmp/capture.bin

   # Disable buffer
   echo 0 > buffer/enable
   ```

### Using Python

1. **Install Python Package** (from PC over network)
   ```bash
   # On development PC
   cd software/python/
   scp -r cn0577_fmc root@<zedboard-ip>:/tmp/

   # On Zed Board
   ssh root@<zedboard-ip>
   cd /tmp/cn0577_fmc
   pip3 install .
   ```

2. **Run Basic Capture Script**
   ```python
   from cn0577_fmc import CN0577
   import matplotlib.pyplot as plt

   # Initialize (use local IIO)
   adc = CN0577(uri='local:')

   # Configure for Config A (18-bit, 15 Msps)
   adc.set_variant('config-a')
   adc.sample_rate = 15e6

   # Capture data
   print("Capturing data...")
   data = adc.capture(num_samples=16384)

   # Display statistics
   print(f"Samples captured: {len(data)}")
   print(f"Mean: {data.mean():.2f}")
   print(f"Std Dev: {data.std():.2f}")
   print(f"Min: {data.min()}")
   print(f"Max: {data.max()}")

   # Plot (save to file, display on PC)
   plt.figure(figsize=(12, 4))
   plt.plot(data[:1000])  # Plot first 1000 samples
   plt.xlabel('Sample')
   plt.ylabel('ADC Code')
   plt.title('CN0577 Data Capture')
   plt.grid(True)
   plt.savefig('/tmp/capture.png')
   ```

3. **Copy Plot to PC**
   ```bash
   # From PC
   scp root@<zedboard-ip>:/tmp/capture.png .
   ```

### Using LibIIO from PC

1. **Install LibIIO on PC**
   ```bash
   # Ubuntu/Debian
   sudo apt-get install libiio-utils python3-libiio

   # Windows: Download from analog.com/libiio
   ```

2. **Test Network Connection**
   ```bash
   iio_info -n <zedboard-ip>
   # Should list IIO devices
   ```

3. **Capture from PC**
   ```bash
   iio_readdev -n <zedboard-ip> -b 16384 cn0577 voltage0 > capture.bin
   ```

4. **Plot Data**
   ```python
   import numpy as np
   import matplotlib.pyplot as plt

   # Read binary data (assumes 16-bit samples)
   data = np.fromfile('capture.bin', dtype=np.int16)

   plt.plot(data[:1000])
   plt.show()
   ```

## Testing Signal Path

### Test 1: DC Input

1. **Apply Known DC Voltage**
   - Set signal generator to 1.0 VDC
   - Connect to ADC input

2. **Read Value**
   ```bash
   cat /sys/bus/iio/devices/iio:device0/in_voltage0_raw
   ```

3. **Verify**
   - Calculate expected code: `code = (V_in / V_ref) * 2^bits`
   - Compare with measured value
   - Should be within ±1 LSB

### Test 2: AC Sine Wave

1. **Apply Sine Wave**
   - Frequency: 1 kHz
   - Amplitude: 1.0 Vpp
   - Offset: 2.5 VDC (mid-scale)

2. **Capture Data**
   ```python
   from cn0577_fmc import CN0577
   adc = CN0577(uri='local:')
   adc.sample_rate = 100e3  # 100 kHz
   data = adc.capture(10000)  # 10k samples = 100ms
   ```

3. **Verify Frequency**
   ```python
   # Should see ~10 cycles in the data
   from scipy import signal
   freqs, psd = signal.welch(data, fs=100e3)
   peak_freq = freqs[psd.argmax()]
   print(f"Detected frequency: {peak_freq:.1f} Hz")
   # Should be ~1000 Hz
   ```

### Test 3: FFT Analysis

1. **Apply 1 MHz Sine Wave**
   - Use high-quality signal generator
   - Amplitude: -1 dBFS
   - Low distortion (<-80 dB THD)

2. **Capture at Full Rate**
   ```python
   adc.sample_rate = 15e6
   data = adc.capture(16384)
   ```

3. **Perform FFT**
   ```python
   freqs, spectrum = adc.compute_fft(data, window='blackman')
   adc.plot_fft(data, show_harmonics=True)

   # Calculate performance metrics
   enob = adc.calculate_enob(spectrum)
   sfdr = adc.calculate_sfdr(spectrum)
   snr = adc.calculate_snr(spectrum)

   print(f"ENOB: {enob:.2f} bits")
   print(f"SFDR: {sfdr:.2f} dBc")
   print(f"SNR: {snr:.2f} dB")
   ```

## Troubleshooting

### Problem: No Boot Messages

**Symptoms**: No output on console after power-on

**Solutions**:
- Check UART connection and settings (115200 baud)
- Verify boot jumpers are correct for SD card
- Check SD card is properly inserted
- Try re-writing SD card image
- Verify 12V power supply is working

### Problem: IIO Device Not Found

**Symptoms**: `/sys/bus/iio/devices/` is empty

**Solutions**:
```bash
# Check if driver is loaded
lsmod | grep cn0577

# Load manually if needed
modprobe cn0577

# Check dmesg for errors
dmesg | grep -i error

# Verify device tree
ls /proc/device-tree/cn0577*
```

### Problem: Data Looks Wrong

**Symptoms**: Data appears noisy or stuck at one value

**Solutions**:
- Check analog input connections
- Verify input signal is within ADC range (0-5V)
- Check reference voltage output (should be 5.0V)
- Verify FPGA bitstream is loaded correctly
- Check for loose connections

### Problem: Network Not Working

**Symptoms**: Cannot ping or connect via network

**Solutions**:
```bash
# Check Ethernet link
ifconfig eth0
ethtool eth0  # Check link status

# Set static IP if DHCP not working
ifconfig eth0 192.168.1.100 netmask 255.255.255.0
route add default gw 192.168.1.1
```

### Problem: Poor Performance

**Symptoms**: ENOB/SFDR lower than expected

**Solutions**:
- Verify signal generator quality (low distortion)
- Check for ground loops
- Verify ADC reference voltage (5.0V ±0.1%)
- Perform calibration: `python /usr/bin/cn0577_calibrate.py`
- Check for EMI sources near FMC card

## Next Steps

### Learn More

- [Hardware Design Details](../hardware/README.md)
- [FPGA Architecture](../fpga/README.md)
- [Software API Reference](../software/README.md)
- [Python Examples](../software/python/examples/)

### Advanced Topics

- [Multi-Channel Operation](advanced/multi-channel.md)
- [Real-Time Processing](advanced/realtime.md)
- [Custom FPGA Design](advanced/fpga-customization.md)
- [Calibration Procedures](advanced/calibration.md)

### Get Help

- Open an issue on GitHub
- Check documentation at [link]
- Contact support team

## Appendix

### Useful Commands

```bash
# System info
cat /proc/cpuinfo
cat /proc/meminfo
cat /proc/version

# IIO device info
iio_info

# Read all attributes
iio_attr -a

# Monitor data rate
watch -n 1 'cat /sys/bus/iio/devices/iio:device0/buffer/data_available'

# Check FPGA temperature
cat /sys/class/hwmon/hwmon0/temp1_input
```

### Pin Reference

See [Hardware README](../hardware/README.md#fmc-pin-assignment) for complete FMC pinout.

### Performance Specifications

See [Technical Specifications](technical-specs.md) for expected performance of each variant.
