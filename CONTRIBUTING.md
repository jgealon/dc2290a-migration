# Contributing to CN0577 FMC Migration

Thank you for your interest in contributing to the CN0577 FMC migration project! This document provides guidelines and instructions for contributing.

## Table of Contents

- [Code of Conduct](#code-of-conduct)
- [Getting Started](#getting-started)
- [Development Workflow](#development-workflow)
- [Contribution Guidelines](#contribution-guidelines)
- [Testing Requirements](#testing-requirements)
- [Documentation](#documentation)
- [Review Process](#review-process)

## Code of Conduct

This project adheres to the Analog Devices Code of Conduct. By participating, you are expected to uphold this code.

## Getting Started

### Prerequisites

Before contributing, ensure you have:

1. **Hardware** (for hardware/FPGA/driver work):
   - Zed Board
   - CN0577 FMC card (or prototype)
   - Required test equipment

2. **Software Tools**:
   - Git
   - Xilinx Vivado (for FPGA)
   - Python 3.8+
   - GCC cross-compiler (for Linux drivers)

3. **Knowledge**:
   - Hardware: PCB design, analog circuits, FMC standard
   - FPGA: Verilog/VHDL, Xilinx Zynq architecture
   - Software: C (kernel drivers), Python, Linux IIO framework

### Setting Up Development Environment

```bash
# Clone the repository
git clone <repository-url>
cd dc2290a-migration

# Create a feature branch
git checkout -b feature/your-feature-name

# For Python development
cd software/python/
pip install -e .[dev]  # Install in editable mode with dev dependencies

# For FPGA development
source /opt/Xilinx/Vivado/2022.2/settings64.sh
cd fpga/vivado/
vivado -mode batch -source create_project.tcl
```

## Development Workflow

### 1. Branch Strategy

- `main` - Stable, tested code
- `develop` - Integration branch for features
- `feature/*` - New features
- `bugfix/*` - Bug fixes
- `release/*` - Release preparation

### 2. Making Changes

```bash
# Create feature branch from develop
git checkout develop
git pull origin develop
git checkout -b feature/my-new-feature

# Make changes and commit
git add <files>
git commit -m "Add: brief description of changes"

# Push to remote
git push origin feature/my-new-feature
```

### 3. Commit Message Format

Follow conventional commits:

```
<type>: <description>

[optional body]

[optional footer]
```

**Types:**
- `feat`: New feature
- `fix`: Bug fix
- `docs`: Documentation only
- `style`: Code style (formatting, missing semicolons, etc.)
- `refactor`: Code change that neither fixes bug nor adds feature
- `test`: Adding or updating tests
- `chore`: Maintenance tasks

**Examples:**
```
feat: add 18-bit ADC mode support to Python API

Add support for 18-bit precision mode in the CN0577 class.
This enables higher resolution data capture for Config A, C, and E.

Closes #123
```

```
fix: correct LVDS timing constraints in XDC file

The previous constraints were too tight for the 15 Msps mode,
causing timing failures during implementation.
```

## Contribution Guidelines

### Hardware Design

#### Schematic Changes

1. Update schematic in OrCAD/Altium
2. Export PDF schematic to `hardware/fmc_card/pdf/`
3. Update BOM in `hardware/bom/`
4. Document changes in commit message
5. Include simulation results if applicable

#### PCB Layout

1. Follow design rules in `hardware/README.md`
2. Run DRC before committing
3. Export Gerbers to `hardware/gerbers/`
4. Update stackup documentation if changed
5. Include screenshots of critical routing

#### Review Checklist

- [ ] Schematic follows ADI design guidelines
- [ ] BOM is updated and accurate
- [ ] PCB passes DRC
- [ ] Impedance-controlled traces meet spec
- [ ] Power integrity simulated and acceptable
- [ ] Thermal analysis performed
- [ ] Manufacturing files generated

### FPGA Design

#### HDL Code

```verilog
// Follow standard Verilog style
module my_module #(
    parameter DATA_WIDTH = 32
)(
    input  wire                     clk,
    input  wire                     rst_n,  // Active low reset
    input  wire [DATA_WIDTH-1:0]    data_in,
    output reg  [DATA_WIDTH-1:0]    data_out
);
    // Always use non-blocking assignments in sequential logic
    always @(posedge clk or negedge rst_n) begin
        if (!rst_n)
            data_out <= '0;
        else
            data_out <= data_in;
    end
endmodule
```

#### Coding Style

- Use lowercase with underscores for signals: `data_valid`
- Use UPPERCASE for parameters: `DATA_WIDTH`
- Comment complex logic
- Include module header with description
- Use synchronous resets where possible

#### Simulation Requirements

Every HDL module must have:

1. Testbench in `fpga/hdl/sim/`
2. Self-checking assertions
3. Coverage report >90%

```verilog
// Example testbench
`timescale 1ns/1ps

module tb_my_module;
    reg clk, rst_n;
    reg [31:0] data_in;
    wire [31:0] data_out;

    // Instantiate DUT
    my_module #(.DATA_WIDTH(32)) dut (
        .clk(clk),
        .rst_n(rst_n),
        .data_in(data_in),
        .data_out(data_out)
    );

    // Clock generation
    initial clk = 0;
    always #5 clk = ~clk;  // 100 MHz

    // Test stimulus
    initial begin
        rst_n = 0;
        data_in = 0;
        #100 rst_n = 1;

        #100 data_in = 32'hDEADBEEF;
        #10;
        assert(data_out == 32'hDEADBEEF) else $error("Data mismatch!");

        #1000 $finish;
    end
endmodule
```

#### Review Checklist

- [ ] Code follows style guidelines
- [ ] Testbench provided and passing
- [ ] Timing constraints added/updated
- [ ] Synthesis completed without errors
- [ ] Timing closed (no negative slack)
- [ ] Simulation waveforms reviewed
- [ ] Resource utilization documented

### Software Development

#### Linux Kernel Driver

Follow Linux kernel coding style (use `checkpatch.pl`):

```c
// Kernel style
static int cn0577_read_raw(struct iio_dev *indio_dev,
                          struct iio_chan_spec const *chan,
                          int *val, int *val2, long mask)
{
    struct cn0577_state *st = iio_priv(indio_dev);
    int ret;

    switch (mask) {
    case IIO_CHAN_INFO_RAW:
        ret = cn0577_read_sample(st, val);
        if (ret)
            return ret;
        return IIO_VAL_INT;
    case IIO_CHAN_INFO_SCALE:
        *val = st->vref_mv;
        *val2 = chan->scan_type.realbits;
        return IIO_VAL_FRACTIONAL_LOG2;
    default:
        return -EINVAL;
    }
}
```

**Requirements:**
- Use kernel coding style (tabs, 80 columns)
- No warnings from `sparse` or `checkpatch.pl`
- Proper error handling
- Device tree binding documentation

#### Python Code

Follow PEP 8 style (use `black` formatter):

```python
from typing import Optional, List
import numpy as np


class CN0577:
    """
    CN0577 ADC control class.

    Args:
        uri: IIO context URI
        device_name: IIO device name
    """

    def __init__(self, uri: str = "local:", device_name: str = "cn0577"):
        self._uri = uri
        self._device_name = device_name
        self._context = None
        self._initialize()

    def capture(self, num_samples: int, timeout_ms: int = 5000) -> np.ndarray:
        """
        Capture ADC samples.

        Args:
            num_samples: Number of samples to capture
            timeout_ms: Timeout in milliseconds

        Returns:
            NumPy array of captured samples

        Raises:
            TimeoutError: If capture times out
            ValueError: If num_samples is invalid
        """
        if num_samples <= 0:
            raise ValueError("num_samples must be positive")

        # Implementation...
        return np.zeros(num_samples)  # Placeholder
```

**Requirements:**
- Use `black` code formatter
- Type hints for function arguments and returns
- Docstrings for all public functions (Google style)
- Unit tests with `pytest`
- Code coverage >80%

#### Review Checklist

- [ ] Code follows language style guide
- [ ] No linter warnings
- [ ] Type hints provided (Python)
- [ ] Docstrings complete
- [ ] Unit tests provided and passing
- [ ] Code coverage adequate
- [ ] Error handling comprehensive

### Documentation

#### Code Documentation

- **HDL**: Module header comments describing functionality
- **C**: Function documentation in header files
- **Python**: Docstrings in Google format

#### User Documentation

Update relevant docs:
- `README.md` files in each directory
- User guides in `docs/`
- API reference documentation
- Examples and tutorials

#### Commit Documentation

Ensure:
- Clear commit messages
- Reference related issues (`Closes #123`)
- Describe *why*, not just *what*

## Testing Requirements

### Hardware Testing

Before submitting hardware changes:

1. **Simulation**: All circuits simulated with SPICE
2. **DRC**: Design rule check passed
3. **Review**: Peer review of schematic and layout
4. **Prototype**: If possible, build and test prototype

### FPGA Testing

Required tests:

1. **Unit Tests**: Testbench for each module
2. **Integration Tests**: Top-level simulation
3. **Timing Analysis**: Timing constraints met
4. **Hardware Tests**: Verify on actual hardware

```bash
# Run simulations
cd fpga/hdl/sim/
./run_all_tests.sh

# Check timing
cd fpga/vivado/
vivado -mode batch -source check_timing.tcl
```

### Software Testing

#### Unit Tests

```bash
# Python
cd software/python/
pytest tests/unit/ -v --cov

# Kernel driver (requires test harness)
cd software/linux/tests/
./run_kernel_tests.sh
```

#### Integration Tests

```bash
# Hardware-in-loop tests (requires board)
pytest tests/integration/ --hardware -v
```

#### Performance Tests

```bash
# Benchmark tests
pytest tests/performance/ --benchmark
```

### Test Coverage Requirements

- **Python**: >80% line coverage
- **HDL**: >90% statement coverage
- **C**: >75% branch coverage

## Review Process

### Pull Request Process

1. **Create PR**:
   - Target `develop` branch (not `main`)
   - Fill out PR template completely
   - Link related issues

2. **Automated Checks**:
   - CI/CD pipeline must pass
   - Linters and formatters must pass
   - Tests must pass

3. **Code Review**:
   - At least one maintainer approval required
   - Address all review comments
   - Re-request review after changes

4. **Merge**:
   - Squash commits if multiple small commits
   - Use descriptive merge commit message
   - Delete feature branch after merge

### PR Template

```markdown
## Description
Brief description of changes

## Type of Change
- [ ] Bug fix
- [ ] New feature
- [ ] Breaking change
- [ ] Documentation update

## Testing
- [ ] Unit tests added/updated
- [ ] Integration tests pass
- [ ] Hardware tested (if applicable)

## Checklist
- [ ] Code follows style guidelines
- [ ] Self-review completed
- [ ] Documentation updated
- [ ] No new warnings
- [ ] Tests pass locally

## Related Issues
Closes #(issue number)
```

## Getting Help

- **Questions**: Open a GitHub Discussion
- **Bugs**: Open an issue with the `bug` label
- **Features**: Open an issue with the `enhancement` label
- **Security**: Email security@analog.com (do not open public issue)

## Recognition

Contributors will be acknowledged in:
- CONTRIBUTORS.md file
- Release notes
- Project documentation

Thank you for contributing to the CN0577 FMC migration project!
