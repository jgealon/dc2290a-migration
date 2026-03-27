# Hardware Variant Detection Using Resistor Coding

## Overview

The DC2290A FMC board uses **resistor coding** (similar to legacy DC2290A) to identify which variant (A through F) is installed. Resistors R33, R34, and R35 are either populated (0Ω) or DNI (Do Not Install) to create a unique 3-bit code for each variant.

This hardware identification allows **automatic variant detection** at boot time, eliminating the need for:
- Manual variant configuration
- EEPROM on FMC card
- SPI ID register reads
- Multiple device tree files

## Hardware Implementation

### Resistor Coding Scheme

Using 3 resistors (R33, R34, R35) connected to GPIO pins gives us 8 possible combinations (000 to 111), which is enough for 6 variants plus 2 reserved codes.

```
┌─────────────────────────────────────────────────────────┐
│                DC2290A FMC Card                          │
│                                                          │
│  ┌────────────────────────────────────────────────┐    │
│  │   Variant ID Resistors (on FMC LA pins)       │    │
│  │                                                 │    │
│  │   R33 ────┬──── FMC LA00_CC_P (ID bit 0)      │    │
│  │           │                                     │    │
│  │   R34 ────┼──── FMC LA01_CC_P (ID bit 1)      │    │
│  │           │                                     │    │
│  │   R35 ────┼──── FMC LA02_P    (ID bit 2)      │    │
│  │           │                                     │    │
│  │           └──── GND (when populated)           │    │
│  │                                                 │    │
│  │   Pull-ups on Zed Board side (10kΩ to 3.3V)   │    │
│  └────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
```

### Encoding Table

| Variant | ADC Model | R33 | R34 | R35 | Binary | Hex | Dec |
|---------|-----------|-----|-----|-----|--------|-----|-----|
| **DC2290A-A** | LTC2387-18 | DNI | DNI | DNI | 111 | 0x7 | 7 |
| **DC2290A-B** | LTC2387-16 | 0Ω  | DNI | DNI | 011 | 0x3 | 3 |
| **DC2290A-C** | LTC2386-18 | DNI | 0Ω  | DNI | 101 | 0x5 | 5 |
| **DC2290A-D** | LTC2386-16 | 0Ω  | 0Ω  | DNI | 001 | 0x1 | 1 |
| **DC2290A-E** | LTC2385-18 | DNI | DNI | 0Ω  | 110 | 0x6 | 6 |
| **DC2290A-F** | LTC2385-16 | 0Ω  | DNI | 0Ω  | 010 | 0x2 | 2 |
| _Reserved 1_ | - | DNI | 0Ω  | 0Ω  | 100 | 0x4 | 4 |
| _Reserved 2_ | - | 0Ω  | 0Ω  | 0Ω  | 000 | 0x0 | 0 |

**Logic**:
- **Resistor populated (0Ω)**: Pulls GPIO to GND → reads as **0**
- **Resistor DNI**: Pull-up keeps GPIO at 3.3V → reads as **1**

### Reading Logic

```
GPIO Value = (R35_read << 2) | (R34_read << 1) | (R33_read << 0)

Where:
  R3x_read = 0 if resistor installed (pulled to GND)
  R3x_read = 1 if resistor DNI (pulled high)
```

### Hardware BOM per Variant

| Variant | R33 Value | R34 Value | R35 Value | Notes |
|---------|-----------|-----------|-----------|-------|
| DC2290A-A | DNI | DNI | DNI | All floating (0x7) |
| DC2290A-B | 0Ω 0402 | DNI | DNI | Only R33 installed |
| DC2290A-C | DNI | 0Ω 0402 | DNI | Only R34 installed |
| DC2290A-D | 0Ω 0402 | 0Ω 0402 | DNI | R33+R34 installed |
| DC2290A-E | DNI | DNI | 0Ω 0402 | Only R35 installed |
| DC2290A-F | 0Ω 0402 | DNI | 0Ω 0402 | R33+R35 installed |

## FMC Pin Assignment

Map variant ID pins to specific FMC LPC pins that connect to Zed Board GPIOs:

| Signal | FMC Pin | Zed FPGA Pin | Zynq PS GPIO | Function |
|--------|---------|--------------|--------------|----------|
| VAR_ID[0] | LA00_CC_P | G6 | MIO[40] | Variant bit 0 (R33) |
| VAR_ID[1] | LA01_CC_P | G7 | MIO[41] | Variant bit 1 (R34) |
| VAR_ID[2] | LA02_P    | H6 | MIO[42] | Variant bit 2 (R35) |

**Alternative**: Route to PL (FPGA) GPIOs if PS MIO pins are not available.

## Software Adaptations

### 1. U-Boot: Early Detection

Detect variant in U-Boot before loading kernel/device tree:

**File**: `u-boot/board/xilinx/zynq/board.c`

```c
#include <asm/gpio.h>

#define VAR_ID_GPIO0  40  // MIO40 - R33
#define VAR_ID_GPIO1  41  // MIO41 - R34
#define VAR_ID_GPIO2  42  // MIO42 - R35

static const char *dc2290a_variants[] = {
    [0] = "invalid",      // 0x0 - Reserved
    [1] = "dc2290a-d",    // 0x1 - LTC2386-16
    [2] = "dc2290a-f",    // 0x2 - LTC2385-16
    [3] = "dc2290a-b",    // 0x3 - LTC2387-16
    [4] = "invalid",      // 0x4 - Reserved
    [5] = "dc2290a-c",    // 0x5 - LTC2386-18
    [6] = "dc2290a-e",    // 0x6 - LTC2385-18
    [7] = "dc2290a-a",    // 0x7 - LTC2387-18
};

int dc2290a_detect_variant(void)
{
    int bit0, bit1, bit2, variant_id;

    // Configure GPIOs as inputs
    gpio_request(VAR_ID_GPIO0, "var_id0");
    gpio_request(VAR_ID_GPIO1, "var_id1");
    gpio_request(VAR_ID_GPIO2, "var_id2");
    gpio_direction_input(VAR_ID_GPIO0);
    gpio_direction_input(VAR_ID_GPIO1);
    gpio_direction_input(VAR_ID_GPIO2);

    // Read GPIO values
    bit0 = gpio_get_value(VAR_ID_GPIO0);
    bit1 = gpio_get_value(VAR_ID_GPIO1);
    bit2 = gpio_get_value(VAR_ID_GPIO2);

    // Compute variant ID
    variant_id = (bit2 << 2) | (bit1 << 1) | bit0;

    printf("DC2290A Variant Detection:\n");
    printf("  GPIO bits: %d%d%d (0x%X)\n", bit2, bit1, bit0, variant_id);
    printf("  Detected variant: %s\n", dc2290a_variants[variant_id]);

    // Set environment variable for kernel
    env_set("board_variant", dc2290a_variants[variant_id]);

    return variant_id;
}

int board_late_init(void)
{
    // Detect variant and set environment
    dc2290a_detect_variant();

    // Select appropriate device tree
    char *variant = env_get("board_variant");
    if (variant) {
        char dtb_name[64];
        snprintf(dtb_name, sizeof(dtb_name),
                 "zynq-zed-adv7511-%s.dtb", variant);
        env_set("devicetree_image", dtb_name);
    }

    return 0;
}
```

**U-Boot Output**:
```
U-Boot 2022.01 (Mar 27 2026 - 10:00:00 +0000)

...
DC2290A Variant Detection:
  GPIO bits: 011 (0x3)
  Detected variant: dc2290a-b
Loading device tree: zynq-zed-adv7511-dc2290a-b.dtb
...
```

### 2. Linux Device Tree: GPIO Configuration

Define the variant ID GPIOs in the device tree:

**File**: `arch/arm/boot/dts/xilinx/zynq-zed-adv7511-dc2290a.dtsi`

```dts
/ {
    dc2290a_variant: dc2290a-variant-detect {
        compatible = "gpio-keys";
        #address-cells = <1>;
        #size-cells = <0>;

        variant-id-0 {
            label = "DC2290A Variant ID Bit 0";
            gpios = <&gpio0 40 GPIO_ACTIVE_LOW>;  // R33
            linux,code = <0>;
        };

        variant-id-1 {
            label = "DC2290A Variant ID Bit 1";
            gpios = <&gpio0 41 GPIO_ACTIVE_LOW>;  // R34
            linux,code = <1>;
        };

        variant-id-2 {
            label = "DC2290A Variant ID Bit 2";
            gpios = <&gpio0 42 GPIO_ACTIVE_LOW>;  // R35
            linux,code = <2>;
        };
    };
};
```

### 3. Linux Boot Script: Variant Detection

Detect variant early in boot process:

**File**: `/etc/init.d/S00detect-variant`

```bash
#!/bin/sh
#
# DC2290A variant detection service
#

VARIANT_FILE="/etc/dc2290a_variant"
GPIO_BASE="/sys/class/gpio"

# GPIO numbers (adjust if needed)
GPIO_BIT0=40
GPIO_BIT1=41
GPIO_BIT2=42

detect_variant() {
    # Export GPIOs if not already exported
    for gpio in $GPIO_BIT0 $GPIO_BIT1 $GPIO_BIT2; do
        if [ ! -d "$GPIO_BASE/gpio$gpio" ]; then
            echo $gpio > $GPIO_BASE/export
            echo in > $GPIO_BASE/gpio$gpio/direction
        fi
    done

    # Read GPIO values
    BIT0=$(cat $GPIO_BASE/gpio$GPIO_BIT0/value)
    BIT1=$(cat $GPIO_BASE/gpio$GPIO_BIT1/value)
    BIT2=$(cat $GPIO_BASE/gpio$GPIO_BIT2/value)

    # Compute variant ID
    VARIANT_ID=$((BIT2 * 4 + BIT1 * 2 + BIT0))

    # Map to variant name
    case $VARIANT_ID in
        7) VARIANT="dc2290a-a"; ADC="LTC2387-18"; RES=18; RATE=15;;
        3) VARIANT="dc2290a-b"; ADC="LTC2387-16"; RES=16; RATE=15;;
        5) VARIANT="dc2290a-c"; ADC="LTC2386-18"; RES=18; RATE=10;;
        1) VARIANT="dc2290a-d"; ADC="LTC2386-16"; RES=16; RATE=10;;
        6) VARIANT="dc2290a-e"; ADC="LTC2385-18"; RES=18; RATE=5;;
        2) VARIANT="dc2290a-f"; ADC="LTC2385-16"; RES=16; RATE=5;;
        *) VARIANT="unknown"; ADC="unknown"; RES=0; RATE=0;;
    esac

    # Save to file
    cat > $VARIANT_FILE << EOF
VARIANT=$VARIANT
ADC_MODEL=$ADC
RESOLUTION=$RES
MAX_SAMPLE_RATE_MSPS=$RATE
VARIANT_ID=$VARIANT_ID
DETECTED_AT=$(date)
EOF

    echo "DC2290A Variant Detected: $VARIANT ($ADC, ${RES}-bit, ${RATE} Msps)"
}

start() {
    echo "Detecting DC2290A variant..."
    detect_variant
}

case "$1" in
    start)
        start
        ;;
    *)
        echo "Usage: $0 {start}"
        exit 1
esac

exit 0
```

**Make executable**:
```bash
chmod +x /etc/init.d/S00detect-variant
```

**Boot output**:
```
Starting S00detect-variant...
Detecting DC2290A variant...
DC2290A Variant Detected: dc2290a-b (LTC2387-16, 16-bit, 15 Msps)
```

### 4. Kernel Driver: Read Variant Info

Update ltc2387 driver to read variant from sysfs or device tree:

**File**: `drivers/iio/adc/ltc2387.c`

```c
#include <linux/gpio.h>
#include <linux/of_gpio.h>

// In ltc2387_state struct, add:
struct ltc2387_state {
    // ... existing fields
    int variant_gpios[3];  // GPIO numbers for variant ID
    u8 variant_id;         // Detected variant ID (0-7)
    const char *board_variant;  // "dc2290a-a" through "dc2290a-f"
};

static int ltc2387_detect_variant(struct ltc2387_state *st,
                                   struct device *dev)
{
    struct device_node *np = dev->of_node;
    int ret, i;
    u8 gpio_val[3];

    // Try to read from device tree property first
    ret = of_property_read_string(np, "adi,board-variant",
                                   &st->board_variant);
    if (ret == 0) {
        dev_info(dev, "Variant from DT: %s\n", st->board_variant);
        return 0;
    }

    // Otherwise, try GPIO detection
    for (i = 0; i < 3; i++) {
        st->variant_gpios[i] = of_get_named_gpio(np, "variant-gpios", i);
        if (!gpio_is_valid(st->variant_gpios[i])) {
            dev_warn(dev, "No variant GPIOs, using default\n");
            st->board_variant = "unknown";
            return -ENODEV;
        }

        ret = devm_gpio_request_one(dev, st->variant_gpios[i],
                                     GPIOF_IN, "variant-id");
        if (ret)
            return ret;

        gpio_val[i] = gpio_get_value(st->variant_gpios[i]);
    }

    // Compute variant ID
    st->variant_id = (gpio_val[2] << 2) | (gpio_val[1] << 1) | gpio_val[0];

    // Map to variant name
    static const char *variant_names[] = {
        [0] = "invalid",
        [1] = "dc2290a-d",
        [2] = "dc2290a-f",
        [3] = "dc2290a-b",
        [4] = "invalid",
        [5] = "dc2290a-c",
        [6] = "dc2290a-e",
        [7] = "dc2290a-a",
    };

    st->board_variant = variant_names[st->variant_id];

    dev_info(dev, "Detected variant: %s (ID=0x%X, GPIOs=%d%d%d)\n",
             st->board_variant, st->variant_id,
             gpio_val[2], gpio_val[1], gpio_val[0]);

    return 0;
}

static int ltc2387_probe(struct spi_device *spi)
{
    struct ltc2387_state *st;
    struct iio_dev *indio_dev;
    int ret;

    // ... existing probe code

    // Detect variant
    ret = ltc2387_detect_variant(st, &spi->dev);
    if (ret && ret != -ENODEV)
        return ret;

    // Configure based on variant
    if (strstr(st->board_variant, "dc2290a-")) {
        // Parse variant letter (a-f)
        char variant_letter = st->board_variant[9];

        // Set parameters based on variant
        switch (variant_letter) {
        case 'a':  // LTC2387-18
            st->chip_info = &ltc2387_18_info;
            st->max_sample_rate = 15000000;
            break;
        case 'b':  // LTC2387-16
            st->chip_info = &ltc2387_16_info;
            st->max_sample_rate = 15000000;
            break;
        case 'c':  // LTC2386-18
            st->chip_info = &ltc2386_18_info;
            st->max_sample_rate = 10000000;
            break;
        case 'd':  // LTC2386-16
            st->chip_info = &ltc2386_16_info;
            st->max_sample_rate = 10000000;
            break;
        case 'e':  // LTC2385-18
            st->chip_info = &ltc2385_18_info;
            st->max_sample_rate = 5000000;
            break;
        case 'f':  // LTC2385-16
            st->chip_info = &ltc2385_16_info;
            st->max_sample_rate = 5000000;
            break;
        }
    }

    // ... rest of probe

    return 0;
}
```

**Add sysfs attribute to expose variant**:

```c
static ssize_t ltc2387_show_variant(struct device *dev,
                                     struct device_attribute *attr,
                                     char *buf)
{
    struct iio_dev *indio_dev = dev_to_iio_dev(dev);
    struct ltc2387_state *st = iio_priv(indio_dev);

    return sprintf(buf, "%s\n", st->board_variant);
}

static IIO_DEVICE_ATTR(board_variant, S_IRUGO,
                       ltc2387_show_variant, NULL, 0);

static struct attribute *ltc2387_attributes[] = {
    &iio_dev_attr_board_variant.dev_attr.attr,
    // ... other attributes
    NULL,
};
```

**Read variant from user space**:
```bash
cat /sys/bus/iio/devices/iio:device0/board_variant
# Output: dc2290a-b
```

### 5. Python API: Automatic Variant Detection

Update the Python DC2290A class to auto-detect from hardware:

**File**: `software/python/dc2290a_fmc/dc2290a.py`

```python
import iio
import os

class DC2290A:
    """DC2290A FMC board with automatic hardware variant detection"""

    VARIANTS = {
        'dc2290a-a': {'adc': 'ltc2387-18', 'resolution': 18, 'max_rate': 15e6},
        'dc2290a-b': {'adc': 'ltc2387-16', 'resolution': 16, 'max_rate': 15e6},
        'dc2290a-c': {'adc': 'ltc2386-18', 'resolution': 18, 'max_rate': 10e6},
        'dc2290a-d': {'adc': 'ltc2386-16', 'resolution': 16, 'max_rate': 10e6},
        'dc2290a-e': {'adc': 'ltc2385-18', 'resolution': 18, 'max_rate': 5e6},
        'dc2290a-f': {'adc': 'ltc2385-16', 'resolution': 16, 'max_rate': 5e6},
    }

    def __init__(self, variant='auto', uri='local:'):
        """
        Initialize DC2290A board

        Args:
            variant: 'auto' to detect from hardware (default),
                     or explicit 'dc2290a-a' through 'dc2290a-f'
            uri: IIO context URI
        """
        self.ctx = iio.Context(uri)
        self.dev = self.ctx.find_device('ltc2387')

        if not self.dev:
            raise RuntimeError("LTC2387 IIO device not found")

        # Detect variant from hardware
        if variant == 'auto':
            variant = self._detect_variant_hardware()

        if variant not in self.VARIANTS:
            raise ValueError(f"Invalid variant: {variant}")

        self.variant = variant
        self.config = self.VARIANTS[variant]

        print(f"DC2290A initialized: {variant}")
        print(f"  ADC: {self.config['adc']}")
        print(f"  Resolution: {self.config['resolution']}-bit")
        print(f"  Max Rate: {self.config['max_rate']/1e6:.1f} Msps")

    def _detect_variant_hardware(self):
        """
        Detect variant from hardware using multiple methods

        Priority:
        1. IIO driver sysfs attribute
        2. /etc/dc2290a_variant file
        3. GPIO direct read
        """
        # Method 1: Read from IIO driver
        try:
            attr = self.dev.find_attribute('board_variant')
            if attr:
                variant = attr.value.strip()
                if variant in self.VARIANTS:
                    return variant
        except:
            pass

        # Method 2: Read from system config file
        try:
            with open('/etc/dc2290a_variant', 'r') as f:
                for line in f:
                    if line.startswith('VARIANT='):
                        variant = line.split('=')[1].strip()
                        if variant in self.VARIANTS:
                            return variant
        except:
            pass

        # Method 3: Direct GPIO read
        variant = self._detect_variant_gpio()
        if variant:
            return variant

        raise RuntimeError("Could not detect variant from hardware")

    def _detect_variant_gpio(self):
        """Read variant ID directly from GPIOs"""
        GPIO_BASE = '/sys/class/gpio'
        GPIO_BITS = [40, 41, 42]  # MIO40, MIO41, MIO42

        variant_map = {
            0: 'invalid',
            1: 'dc2290a-d',
            2: 'dc2290a-f',
            3: 'dc2290a-b',
            4: 'invalid',
            5: 'dc2290a-c',
            6: 'dc2290a-e',
            7: 'dc2290a-a',
        }

        try:
            bits = []
            for gpio in GPIO_BITS:
                gpio_path = f"{GPIO_BASE}/gpio{gpio}"

                # Export if needed
                if not os.path.exists(gpio_path):
                    with open(f"{GPIO_BASE}/export", 'w') as f:
                        f.write(str(gpio))

                # Set as input
                with open(f"{gpio_path}/direction", 'w') as f:
                    f.write('in')

                # Read value
                with open(f"{gpio_path}/value", 'r') as f:
                    bits.append(int(f.read().strip()))

            # Compute variant ID
            variant_id = (bits[2] << 2) | (bits[1] << 1) | bits[0]

            return variant_map.get(variant_id, None)

        except Exception as e:
            print(f"Warning: GPIO detection failed: {e}")
            return None

    # ... rest of the class (same as before)
```

**Usage**:

```python
from dc2290a_fmc import DC2290A

# Automatic detection (recommended)
adc = DC2290A()  # or DC2290A(variant='auto')
# Output:
#   DC2290A initialized: dc2290a-b
#     ADC: ltc2387-16
#     Resolution: 16-bit
#     Max Rate: 15.0 Msps

# Explicit variant (override detection)
adc = DC2290A(variant='dc2290a-a')

# Check detected variant
print(adc.variant)  # 'dc2290a-b'
print(adc.config)   # {'adc': 'ltc2387-16', 'resolution': 16, ...}

# Use normally
adc.sample_rate = 15e6
data = adc.capture(16384)
```

### 6. Web Interface: Display Detected Variant

Update web UI to show detected variant:

**File**: `software/web/backend/app.py`

```python
from flask import Flask, jsonify
from dc2290a_fmc import DC2290A

app = Flask(__name__)
adc = None

@app.route('/api/info')
def get_info():
    """Get board and variant information"""
    global adc

    if not adc:
        adc = DC2290A()  # Auto-detect

    info = {
        'board': 'DC2290A FMC',
        'variant': adc.variant,
        'adc_model': adc.config['adc'],
        'resolution': adc.config['resolution'],
        'max_sample_rate': adc.config['max_rate'],
        'current_sample_rate': adc.sample_rate,
    }

    return jsonify(info)
```

**Frontend display**:
```javascript
// React component
function BoardInfo() {
  const [info, setInfo] = useState(null);

  useEffect(() => {
    fetch('/api/info')
      .then(res => res.json())
      .then(data => setInfo(data));
  }, []);

  if (!info) return <div>Loading...</div>;

  return (
    <div className="board-info">
      <h2>DC2290A FMC Board</h2>
      <p><strong>Variant:</strong> {info.variant}</p>
      <p><strong>ADC:</strong> {info.adc_model}</p>
      <p><strong>Resolution:</strong> {info.resolution}-bit</p>
      <p><strong>Max Rate:</strong> {info.max_sample_rate / 1e6} Msps</p>
    </div>
  );
}
```

## Complete Boot Flow with Hardware Detection

```
┌──────────────────────────────────────────────────────┐
│                 Power On                              │
└───────────────────┬──────────────────────────────────┘
                    │
                    ▼
┌──────────────────────────────────────────────────────┐
│  U-Boot Bootloader                                    │
│  1. Read GPIO pins (R33, R34, R35)                   │
│  2. Compute variant ID (0x0 - 0x7)                   │
│  3. Set env variable: board_variant=dc2290a-X        │
│  4. Select device tree: zynq-zed-...-dc2290a-X.dtb   │
└───────────────────┬──────────────────────────────────┘
                    │
                    ▼
┌──────────────────────────────────────────────────────┐
│  Linux Kernel Boot                                    │
│  1. Load device tree with variant info               │
│  2. ltc2387 driver probes                            │
│  3. Driver reads variant from GPIOs or DT            │
│  4. Configure for detected ADC (resolution, rate)    │
│  5. Create IIO device with correct parameters        │
└───────────────────┬──────────────────────────────────┘
                    │
                    ▼
┌──────────────────────────────────────────────────────┐
│  Init Scripts (/etc/init.d/S00detect-variant)        │
│  1. Re-read variant GPIOs for verification           │
│  2. Write to /etc/dc2290a_variant                    │
│  3. Log variant to console                           │
└───────────────────┬──────────────────────────────────┘
                    │
                    ▼
┌──────────────────────────────────────────────────────┐
│  User Application (Python)                           │
│  1. Create DC2290A() instance                        │
│  2. Auto-detect from IIO attribute or config file    │
│  3. Enforce variant-specific constraints             │
│  4. User never needs to specify variant manually     │
└──────────────────────────────────────────────────────┘
```

## Verification and Testing

### Test Script

**File**: `/usr/bin/dc2290a-test-variant`

```bash
#!/bin/bash
#
# DC2290A Variant Detection Test Script
#

echo "═══════════════════════════════════════════════════"
echo "  DC2290A Variant Detection Test"
echo "═══════════════════════════════════════════════════"
echo

# Test 1: GPIO Read
echo "Test 1: GPIO Direct Read"
echo "-------------------------"
GPIO_BASE="/sys/class/gpio"
for gpio in 40 41 42; do
    if [ -d "$GPIO_BASE/gpio$gpio" ]; then
        val=$(cat $GPIO_BASE/gpio$gpio/value)
        echo "  GPIO $gpio: $val"
    else
        echo "  GPIO $gpio: NOT EXPORTED"
    fi
done
echo

# Test 2: Config File
echo "Test 2: Config File (/etc/dc2290a_variant)"
echo "-------------------------------------------"
if [ -f /etc/dc2290a_variant ]; then
    cat /etc/dc2290a_variant | grep -E "VARIANT|ADC_MODEL|RESOLUTION|MAX_SAMPLE"
else
    echo "  Config file not found"
fi
echo

# Test 3: IIO Attribute
echo "Test 3: IIO Driver Attribute"
echo "-----------------------------"
IIO_DEV="/sys/bus/iio/devices/iio:device0"
if [ -f "$IIO_DEV/board_variant" ]; then
    echo "  board_variant: $(cat $IIO_DEV/board_variant)"
else
    echo "  Attribute not found"
fi
echo

# Test 4: Python Detection
echo "Test 4: Python API Detection"
echo "-----------------------------"
python3 << 'PYEOF'
try:
    from dc2290a_fmc import DC2290A
    adc = DC2290A()
    info = adc.get_variant_info()
    print(f"  Variant: {info['variant']}")
    print(f"  ADC: {info['adc_model']}")
    print(f"  Resolution: {info['resolution']}-bit")
    print(f"  Max Rate: {info['max_sample_rate']/1e6:.1f} Msps")
except Exception as e:
    print(f"  Error: {e}")
PYEOF
echo

echo "═══════════════════════════════════════════════════"
echo "Test complete"
echo "═══════════════════════════════════════════════════"
```

**Expected output for DC2290A-B**:

```
═══════════════════════════════════════════════════════
  DC2290A Variant Detection Test
═══════════════════════════════════════════════════════

Test 1: GPIO Direct Read
-------------------------
  GPIO 40: 0
  GPIO 41: 1
  GPIO 42: 1

Test 2: Config File (/etc/dc2290a_variant)
-------------------------------------------
VARIANT=dc2290a-b
ADC_MODEL=LTC2387-16
RESOLUTION=16
MAX_SAMPLE_RATE_MSPS=15

Test 3: IIO Driver Attribute
-----------------------------
  board_variant: dc2290a-b

Test 4: Python API Detection
-----------------------------
  Variant: dc2290a-b
  ADC: ltc2387-16
  Resolution: 16-bit
  Max Rate: 15.0 Msps

═══════════════════════════════════════════════════════
Test complete
═══════════════════════════════════════════════════════
```

## Benefits of Resistor Coding Approach

### ✅ Advantages

1. **Zero Cost**: Just 0Ω resistors (or DNI)
2. **Zero External Components**: No EEPROM, no flash
3. **Manufacturing Friendly**: Simple BOM variation
4. **Instant Detection**: Read at power-on, no initialization needed
5. **Reliable**: No SPI communication, no battery/EEPROM failure
6. **Universal Image**: One SD card image for all 6 variants
7. **User Friendly**: Automatic, no manual configuration
8. **Debug Friendly**: Easy to verify with multimeter

### 🔧 Compared to Alternatives

| Method | Cost | Reliability | Speed | User Effort |
|--------|------|-------------|-------|-------------|
| **Resistor Coding** | $0 | ⭐⭐⭐⭐⭐ | Instant | None |
| SPI ID Register | $0 | ⭐⭐⭐⭐ | ~1ms | None |
| EEPROM on FMC | ~$0.50 | ⭐⭐⭐ | ~10ms | None |
| Manual Config | $0 | ⭐⭐ | - | High |
| Multiple Images | $0 | ⭐⭐⭐⭐ | - | Medium |

## Manufacturing Documentation

### Assembly Instructions

**For Manufacturing**: Populate resistors per variant:

```
DC2290A-A (LTC2387-18):
  R33: DNI
  R34: DNI
  R35: DNI
  → ID reads as 0x7

DC2290A-B (LTC2387-16):
  R33: 0Ω 0402
  R34: DNI
  R35: DNI
  → ID reads as 0x3

DC2290A-C (LTC2386-18):
  R33: DNI
  R34: 0Ω 0402
  R35: DNI
  → ID reads as 0x5

DC2290A-D (LTC2386-16):
  R33: 0Ω 0402
  R34: 0Ω 0402
  R35: DNI
  → ID reads as 0x1

DC2290A-E (LTC2385-18):
  R33: DNI
  R34: DNI
  R35: 0Ω 0402
  → ID reads as 0x6

DC2290A-F (LTC2385-16):
  R33: 0Ω 0402
  R34: DNI
  R35: 0Ω 0402
  → ID reads as 0x2
```

### Verification Test Point

Add test points TP1, TP2, TP3 next to R33, R34, R35 for post-assembly verification with multimeter:
- Populated resistor: TP reads ~0V (GND)
- DNI resistor: TP reads ~3.3V (pulled high)

## Summary

With resistor coding (R33, R34, R35), the software adaptations are:

### ✅ Required Changes

1. **U-Boot**: Add GPIO detection in board_late_init()
2. **Linux Kernel**: Update ltc2387 driver to read variant GPIOs
3. **Init Scripts**: Add S00detect-variant service
4. **Python API**: Update DC2290A class to auto-detect from hardware
5. **Device Tree**: Add variant GPIO definitions

### ✅ Benefits

- **Single SD card image** for all 6 variants
- **Automatic detection** at every layer (U-Boot, kernel, application)
- **Zero user configuration** required
- **Manufacturing friendly** (just resistor population)
- **Debug friendly** (can verify with multimeter)

### ✅ Result

User experience:
1. Insert SD card (same image for all variants)
2. Power on board
3. System automatically detects and configures for installed variant
4. Python API automatically uses correct parameters

**No manual configuration needed!**

---

**Document Version**: 1.0
**Last Updated**: 2026-03-27
**Hardware Requirement**: R33, R34, R35 resistor coding implemented on FMC PCB
