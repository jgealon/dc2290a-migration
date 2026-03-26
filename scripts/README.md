# DC2290A-x Migration Scripts

Automation and validation scripts for the DC2290A-x board migration project.

## Available Scripts

### Validation Scripts

**validate.sh** (Coming Soon)
- Validates board detection and connectivity
- Performs basic data capture tests
- Generates validation report

**test-all-variants.sh** (Coming Soon)
- Automated testing across all 6 board variants
- Performance benchmarking
- Generates comparison reports

### Migration Scripts

**migrate-software.sh** (Coming Soon)
- Automated installation of ADI Evaluation Suite
- Driver updates and configuration
- Legacy software backup and removal

**calibrate-adc.sh** (Coming Soon)
- Automated ADC calibration procedure
- Offset and gain adjustment
- Calibration data logging

### Utility Scripts

**check-environment.sh** (Coming Soon)
- Verifies system prerequisites
- Checks for required software packages
- Validates hardware connections

**generate-report.sh** (Coming Soon)
- Generates migration status report
- Compiles test results
- Creates summary documentation

## Script Usage

### Prerequisites

All scripts require:
- Bash 4.0 or higher
- DC718 communication board connected
- ADI Evaluation Software installed
- Appropriate board variant connected

### Running Scripts

```bash
# Make scripts executable
chmod +x scripts/*.sh

# Run validation
./scripts/validate.sh

# Run full test suite
./scripts/test-all-variants.sh DC2290A-A

# Check environment
./scripts/check-environment.sh
```

### Script Exit Codes

- `0` - Success
- `1` - General error
- `2` - Missing prerequisites
- `3` - Hardware not detected
- `4` - Test failure

## Development Guidelines

### Adding New Scripts

1. Use descriptive filenames with `.sh` extension
2. Include comprehensive header comments
3. Implement proper error handling
4. Use consistent exit codes
5. Add usage documentation in this README

### Script Template

```bash
#!/bin/bash
# Script: script-name.sh
# Description: Brief description of what the script does
# Usage: ./script-name.sh [options]
# Exit Codes: 0=success, 1=error

set -euo pipefail  # Exit on error, undefined variables, pipe failures

# Script implementation
```

### Testing Scripts

Before committing new scripts:
1. Test on clean environment
2. Verify error handling
3. Update documentation
4. Add example usage

## Continuous Integration

Scripts are automatically tested by GitHub Actions on:
- Every push to main branch
- Pull request submissions
- Weekly scheduled runs

## Troubleshooting

### Common Issues

**Permission Denied**
```bash
chmod +x scripts/<script-name>.sh
```

**USB Device Not Found**
```bash
# Check USB permissions
sudo usermod -a -G plugdev $USER
# Re-login for changes to take effect
```

**Missing Dependencies**
```bash
# Install required packages (Ubuntu/Debian)
sudo apt-get install libusb-1.0-0-dev
```

## Contributing

When contributing scripts:
1. Follow the coding style of existing scripts
2. Add comprehensive error handling
3. Include usage examples
4. Update this README with new scripts
5. Test thoroughly before submitting

## Script Status

| Script | Status | Description |
|--------|--------|-------------|
| validate.sh | Planned | Board validation and connectivity |
| test-all-variants.sh | Planned | Comprehensive variant testing |
| migrate-software.sh | Planned | Software migration automation |
| calibrate-adc.sh | Planned | ADC calibration automation |
| check-environment.sh | Planned | Environment validation |
| generate-report.sh | Planned | Report generation |

## Resources

- [Bash Best Practices](https://github.com/progrium/bashstyle)
- [ShellCheck - Shell Script Linter](https://www.shellcheck.net/)
- [ADI Linux Evaluation Drivers](https://github.com/analogdevicesinc)
