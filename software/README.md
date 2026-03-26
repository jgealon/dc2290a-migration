# CN0577 FMC Software

Software stack for the CN0577 FMC card on Zed Board, including Linux drivers, Python API, and example applications.

## Overview

The software provides a complete stack from kernel drivers to high-level Python APIs for controlling the CN0577 ADC system and acquiring data.

### Software Architecture

```
┌──────────────────────────────────────────────────────┐
│          User Applications                           │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────┐  │
│  │   Python     │  │   MATLAB     │  │    Web    │  │
│  │     API      │  │   Scripts    │  │ Interface │  │
│  └──────────────┘  └──────────────┘  └───────────┘  │
├──────────────────────────────────────────────────────┤
│               LibIIO Client Library                  │
├──────────────────────────────────────────────────────┤
│          IIO Daemon (iiod) - Network Layer           │
├──────────────────────────────────────────────────────┤
│        Linux Industrial I/O (IIO) Framework          │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────┐  │
│  │    IIO       │  │   Trigger    │  │  Buffer   │  │
│  │   Device     │  │   Support    │  │  Management│  │
│  └──────────────┘  └──────────────┘  └───────────┘  │
├──────────────────────────────────────────────────────┤
│         CN0577 IIO Kernel Driver (cn0577.ko)         │
├──────────────────────────────────────────────────────┤
│                  Hardware (FPGA)                     │
└──────────────────────────────────────────────────────┘
```

## Directory Structure

```
software/
├── linux/
│   ├── drivers/
│   │   ├── iio/                # IIO driver source
│   │   │   └── adc/
│   │   │       └── cn0577.c    # Main driver
│   │   └── Makefile
│   ├── devicetree/
│   │   └── cn0577-fmc.dtsi     # Device tree overlay
│   └── patches/                # Kernel patches
├── python/
│   ├── cn0577_fmc/
│   │   ├── __init__.py
│   │   ├── adc.py              # Main ADC class
│   │   ├── calibration.py      # Calibration routines
│   │   ├── analysis.py         # Signal analysis tools
│   │   └── plot.py             # Plotting utilities
│   ├── examples/
│   │   ├── basic_capture.py
│   │   ├── continuous_streaming.py
│   │   ├── fft_analysis.py
│   │   └── calibration.py
│   ├── setup.py
│   └── README.md
├── firmware/
│   ├── bare-metal/             # Standalone applications
│   ├── fsbl/                   # First Stage Boot Loader
│   └── u-boot/                 # U-Boot configuration
├── web/
│   ├── frontend/               # React web interface
│   ├── backend/                # Flask API server
│   └── README.md
├── matlab/
│   ├── cn0577_toolbox/         # MATLAB toolbox
│   └── examples/               # MATLAB examples
└── README.md                   # This file
```

## Linux Kernel Driver

### IIO Driver Architecture

The CN0577 driver implements the Linux Industrial I/O (IIO) framework:

**Key Features:**
- IIO device registration
- Channel configuration (16/18-bit modes)
- Buffered data capture with DMA
- Triggered sampling
- Sysfs attribute interface
- Calibration data storage

### Driver Files

**cn0577.c** - Main driver implementation
```c
static const struct iio_info cn0577_iio_info = {
    .read_raw = cn0577_read_raw,
    .write_raw = cn0577_write_raw,
    .attrs = &cn0577_attribute_group,
};

static int cn0577_probe(struct platform_device *pdev)
{
    struct iio_dev *indio_dev;
    struct cn0577_state *st;
    // ... driver initialization
}
```

### Building the Driver

#### As Kernel Module

```bash
cd software/linux/drivers/iio/adc/

# Set kernel source path
export KERNEL_SRC=/path/to/linux-xlnx

# Build module
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf-

# Install on target
scp cn0577.ko root@zedboard:/lib/modules/$(uname -r)/kernel/drivers/iio/adc/
```

#### Built into Kernel

```bash
# Copy driver to kernel source
cp cn0577.c $KERNEL_SRC/drivers/iio/adc/

# Add to Kconfig
echo "config ADI_CN0577" >> $KERNEL_SRC/drivers/iio/adc/Kconfig
echo "    tristate \"Analog Devices CN0577 ADC driver\"" >> ...

# Add to Makefile
echo "obj-$(CONFIG_ADI_CN0577) += cn0577.o" >> $KERNEL_SRC/drivers/iio/adc/Makefile

# Configure and build kernel
cd $KERNEL_SRC
make ARCH=arm menuconfig  # Enable ADI_CN0577
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- zImage modules
```

### Device Tree Integration

**cn0577-fmc.dtsi:**
```dts
/ {
    cn0577_adc: cn0577@43c00000 {
        compatible = "adi,cn0577-adc-1.0";
        reg = <0x43c00000 0x10000>;
        interrupts = <0 29 IRQ_TYPE_LEVEL_HIGH>;
        interrupt-parent = <&intc>;

        clocks = <&clkc 15>;
        clock-names = "adc_clk";

        adi,adc-variant = <0>;  /* 0-5 for Config A-F */
        adi,data-width = <18>;
        adi,sample-rate = <15000000>;
    };
};
```

### Sysfs Interface

After driver loads, sysfs entries appear at:
```
/sys/bus/iio/devices/iio:device0/
├── in_voltage0_raw              # Read current ADC value
├── in_voltage0_scale            # Voltage scale factor
├── in_voltage0_offset           # Offset calibration
├── sampling_frequency           # Sample rate
├── buffer/
│   ├── enable                   # Enable buffered capture
│   ├── length                   # Buffer size
│   └── data_available           # Bytes available
└── scan_elements/
    └── in_voltage0_en           # Enable channel
```

### Usage Example

```bash
# Read single sample
cat /sys/bus/iio/devices/iio:device0/in_voltage0_raw

# Set sample rate
echo 10000000 > /sys/bus/iio/devices/iio:device0/sampling_frequency

# Enable buffered capture
echo 1 > /sys/bus/iio/devices/iio:device0/scan_elements/in_voltage0_en
echo 16384 > /sys/bus/iio/devices/iio:device0/buffer/length
echo 1 > /sys/bus/iio/devices/iio:device0/buffer/enable

# Read data
cat /dev/iio:device0 > capture.bin
```

## Python API

### Installation

```bash
cd software/python/

# Install dependencies
pip install numpy scipy matplotlib pylibiio

# Install CN0577 package
pip install -e .

# Or install from PyPI (when published)
pip install cn0577-fmc
```

### Quick Start

```python
from cn0577_fmc import CN0577

# Initialize ADC
adc = CN0577(uri='ip:192.168.1.100')  # Network connection
# OR
adc = CN0577(uri='local:')             # Local IIO

# Configure for variant A (18-bit, 15 Msps)
adc.set_variant('config-a')
adc.sample_rate = 15e6

# Capture data
data = adc.capture(num_samples=16384)

# Analyze
adc.plot_time_domain(data)
adc.plot_fft(data)

# Print stats
print(f"Mean: {data.mean():.2f}")
print(f"Std Dev: {data.std():.2f}")
print(f"Peak-to-Peak: {data.ptp():.2f}")
```

### API Reference

#### CN0577 Class

```python
class CN0577:
    def __init__(self, uri='local:', device_name='cn0577'):
        """
        Initialize CN0577 device

        Args:
            uri: IIO context URI ('local:', 'ip:hostname', 'usb:')
            device_name: Device name in IIO
        """

    def set_variant(self, variant):
        """Set ADC variant configuration (config-a to config-f)"""

    @property
    def sample_rate(self):
        """Get/set sample rate in Hz"""

    def capture(self, num_samples, timeout_ms=5000):
        """
        Capture ADC samples

        Args:
            num_samples: Number of samples to capture
            timeout_ms: Timeout in milliseconds

        Returns:
            numpy array of samples
        """

    def continuous_capture(self, callback, buffer_size=8192):
        """
        Start continuous capture with callback

        Args:
            callback: Function called with each buffer
            buffer_size: Size of each buffer
        """

    def calibrate(self, mode='auto'):
        """
        Perform calibration

        Args:
            mode: 'auto', 'offset', 'gain'
        """

    def compute_fft(self, data, window='hann'):
        """Compute FFT with windowing"""

    def plot_time_domain(self, data):
        """Plot time-domain waveform"""

    def plot_fft(self, data, show_harmonics=True):
        """Plot frequency spectrum"""
```

### Examples

#### Example 1: Basic Capture

```python
from cn0577_fmc import CN0577
import numpy as np

adc = CN0577()
adc.set_variant('config-a')
adc.sample_rate = 15e6

# Capture one second of data
data = adc.capture(num_samples=int(15e6))

# Calculate statistics
print(f"Mean: {np.mean(data)}")
print(f"RMS: {np.sqrt(np.mean(data**2))}")
```

#### Example 2: FFT Analysis

```python
from cn0577_fmc import CN0577
import matplotlib.pyplot as plt

adc = CN0577()
adc.sample_rate = 15e6

# Capture data
data = adc.capture(16384)

# Compute FFT
freqs, spectrum = adc.compute_fft(data, window='blackman')

# Plot
plt.figure(figsize=(12, 6))
plt.plot(freqs/1e6, 20*np.log10(spectrum))
plt.xlabel('Frequency (MHz)')
plt.ylabel('Magnitude (dBFS)')
plt.title('CN0577 FFT Analysis')
plt.grid(True)
plt.show()

# Calculate ENOB, SFDR
enob = adc.calculate_enob(spectrum)
sfdr = adc.calculate_sfdr(spectrum)
print(f"ENOB: {enob:.2f} bits")
print(f"SFDR: {sfdr:.2f} dBc")
```

#### Example 3: Continuous Streaming

```python
from cn0577_fmc import CN0577
import time

def process_data(data):
    """Callback function for each buffer"""
    mean = data.mean()
    rms = np.sqrt((data**2).mean())
    print(f"Mean: {mean:.2f}, RMS: {rms:.2f}")

adc = CN0577()
adc.sample_rate = 10e6

# Start continuous capture
adc.continuous_capture(callback=process_data, buffer_size=8192)

# Run for 10 seconds
time.sleep(10)

# Stop
adc.stop()
```

#### Example 4: Multi-Variant Testing

```python
from cn0577_fmc import CN0577

variants = ['config-a', 'config-b', 'config-c',
            'config-d', 'config-e', 'config-f']

adc = CN0577()

for variant in variants:
    print(f"\nTesting {variant}...")
    adc.set_variant(variant)

    # Capture and analyze
    data = adc.capture(16384)
    freqs, spectrum = adc.compute_fft(data)

    enob = adc.calculate_enob(spectrum)
    sfdr = adc.calculate_sfdr(spectrum)

    print(f"  ENOB: {enob:.2f} bits")
    print(f"  SFDR: {sfdr:.2f} dBc")
```

## MATLAB Integration

### Installation

```matlab
% Add toolbox to path
addpath('software/matlab/cn0577_toolbox');

% Initialize ADC
adc = cn0577('ip', '192.168.1.100');
```

### MATLAB Examples

```matlab
% Configure ADC
adc.setVariant('config-a');
adc.setSampleRate(15e6);

% Capture data
data = adc.capture(16384);

% Plot
figure;
subplot(2,1,1);
plot(data);
title('Time Domain');
xlabel('Sample');
ylabel('ADC Code');

subplot(2,1,2);
[f, P] = adc.computeFFT(data);
plot(f/1e6, 20*log10(P));
title('Frequency Spectrum');
xlabel('Frequency (MHz)');
ylabel('Magnitude (dBFS)');
```

## Web Interface

### Running the Web Interface

```bash
cd software/web/

# Install dependencies
npm install  # Frontend
pip install flask flask-cors  # Backend

# Start backend
cd backend/
python app.py

# Start frontend (separate terminal)
cd frontend/
npm start

# Access at http://localhost:3000
```

### Features

- Real-time waveform display
- FFT spectrum analyzer
- Variant configuration
- Data export (CSV, MAT, HDF5)
- Calibration wizard
- Performance metrics

## Firmware (Bare-Metal)

### Building Bare-Metal Applications

```bash
cd software/firmware/bare-metal/

# Source Vitis settings
source /opt/Xilinx/Vitis/2022.2/settings64.sh

# Create platform
xsct create_platform.tcl

# Build application
cd app/
make

# Load via JTAG
xsct download.tcl
```

### Example: Simple ADC Read

```c
#include "xil_printf.h"
#include "cn0577_ll.h"  // Low-level hardware API

int main() {
    uint32_t adc_value;

    // Initialize ADC
    cn0577_init();
    cn0577_set_sample_rate(15000000);

    while (1) {
        // Trigger conversion
        cn0577_trigger();

        // Read result
        adc_value = cn0577_read();

        xil_printf("ADC: %d\r\n", adc_value);

        usleep(100000);  // 100ms delay
    }

    return 0;
}
```

## Testing

### Unit Tests

```bash
cd software/python/

# Run tests
pytest tests/

# With coverage
pytest --cov=cn0577_fmc tests/
```

### Integration Tests

```bash
# Requires hardware connected
pytest tests/integration/ --hardware
```

## Performance Optimization

### DMA Transfer Optimization

- Use large buffers (8K-16K samples)
- Enable cache coherency
- Use high-priority DMA channels

### Python Performance

```python
# Use numpy operations (vectorized)
data_filtered = np.convolve(data, kernel, mode='valid')  # Fast

# Avoid loops
# for i in range(len(data)):  # Slow
#     data_filtered[i] = ...
```

## Troubleshooting

### Driver Not Loading

```bash
# Check dmesg
dmesg | grep cn0577

# Verify device tree
cat /proc/device-tree/cn0577@43c00000/compatible

# Load manually
insmod /lib/modules/$(uname -r)/extra/cn0577.ko
```

### IIO Device Not Appearing

```bash
# List IIO devices
ls /sys/bus/iio/devices/

# Check IIO info
iio_info

# Verify permissions
sudo chmod 666 /dev/iio:device0
```

### Python Import Errors

```bash
# Reinstall
pip uninstall cn0577-fmc
pip install -e .

# Check installation
python -c "import cn0577_fmc; print(cn0577_fmc.__version__)"
```

## Contributing

See [CONTRIBUTING.md](../../CONTRIBUTING.md) for guidelines.

### Code Style

- **Python**: PEP 8, use black formatter
- **C**: Linux kernel coding style
- **MATLAB**: MATLAB style guidelines

## Documentation

- [Driver API Reference](docs/driver-api.md)
- [Python API Reference](docs/python-api.md)
- [MATLAB API Reference](docs/matlab-api.md)
- [Web Interface Guide](web/README.md)

## Resources

- [Linux IIO Framework](https://www.kernel.org/doc/html/latest/driver-api/iio/)
- [LibIIO Documentation](https://analogdevicesinc.github.io/libiio/)
- [PyIIO Python Bindings](https://github.com/analogdevicesinc/pyadi-iio)

## Status

| Component | Status | Notes |
|-----------|--------|-------|
| IIO Driver | In Progress | Basic functionality complete |
| Python API | In Progress | Core features implemented |
| MATLAB Toolbox | Planned | Q3 2026 |
| Web Interface | In Progress | Frontend 80% complete |
| Bare-Metal Firmware | Complete | Example applications working |
| Documentation | In Progress | API docs need completion |
