# DC2290A-x Migration Roadmap

Project roadmap for migrating DC2290A-x legacy boards to Zed Board with FMC interface.

## Project Overview

**Goal**: Migrate the DC2290A evaluation board from its legacy platform to a modern Zed Board FMC design. Like the original DC2290A, the new FMC board will support all 6 ADC variants (A through F) through component population, following CN0577 reference design best practices.

**Note**: DC2290A is one board design with 6 SKUs (A/B/C/D/E/F) based on which LTC238x ADC is installed. The FMC migration maintains this same approach.

**Timeline**: Q1 2026 - Q4 2026
**Status**: Phase 1 (Planning & Design) - In Progress

## Phases

### Phase 1: Planning & Design (Q1 2026) ✅ In Progress

#### Hardware Design
- [x] Define FMC pinout and interface requirements
- [x] Select ADC parts for all 6 variants
- [x] Create block diagram
- [ ] Complete schematic capture
  - [x] Power distribution
  - [x] Analog front-end
  - [ ] Digital interface (80% complete)
  - [ ] Clock generation (90% complete)
- [ ] Generate BOM for all variants
- [ ] PCB layout planning

**Target Completion**: 2026-04-15

#### FPGA Architecture
- [x] Define overall architecture
- [x] Design HDL module hierarchy
- [ ] Implement core modules
  - [ ] ADC interface (70% complete)
  - [x] Data packer
  - [x] Trigger control
  - [x] SPI controller
- [ ] Create testbenches
- [ ] Initial simulation validation

**Target Completion**: 2026-04-30

#### Software Planning
- [x] Define software architecture
- [x] Plan IIO driver structure
- [ ] Design Python API
- [ ] Web interface mockups

**Target Completion**: 2026-04-20

### Phase 2: Implementation (Q2 2026)

#### Hardware Implementation
- [ ] Finalize schematic (Week 1-2)
- [ ] PCB layout (Week 3-6)
  - [ ] Component placement
  - [ ] High-speed routing
  - [ ] Power distribution
  - [ ] EMI/EMC considerations
- [ ] Design review (Week 7)
- [ ] Generate manufacturing files (Week 8)
- [ ] Order PCBs (Week 9)
- [ ] Order components (Week 9)

**Target Completion**: 2026-06-30

**Key Deliverables**:
- Final schematic PDF
- PCB Gerber files
- Complete BOM
- Assembly drawings

#### FPGA Implementation
- [ ] Complete HDL development (Week 1-4)
- [ ] IP integration (Week 5-6)
- [ ] Timing closure (Week 7-8)
- [ ] Generate bitstreams for all variants (Week 9)

**Target Completion**: 2026-06-30

**Key Deliverables**:
- Synthesizable HDL codebase
- Timing-closed Vivado project
- Bitstream files
- Simulation reports

#### Software Implementation
- [ ] IIO kernel driver (Week 1-6)
- [ ] Python API (Week 4-8)
- [ ] Example applications (Week 7-9)
- [ ] Documentation (Week 8-10)

**Target Completion**: 2026-06-30

**Key Deliverables**:
- Functional IIO driver
- Python library
- 10+ example applications
- API documentation

### Phase 3: Prototype & Validation (Q3 2026)

#### Hardware Bring-Up
- [ ] Receive PCBs (Week 1)
- [ ] Board assembly (Week 1-2)
  - [ ] Assemble variant A configuration
  - [ ] Initial power-up testing
  - [ ] Verify all power rails
- [ ] Debug and rework (Week 3-4)
- [ ] Assemble remaining variants (Week 5-6)

**Target Completion**: 2026-08-15

#### FPGA Integration
- [ ] Program FPGA with initial bitstream (Week 3)
- [ ] Debug LVDS interface (Week 3-4)
- [ ] Verify data capture (Week 5)
- [ ] Test all variants (Week 6-7)
- [ ] Performance optimization (Week 8)

**Target Completion**: 2026-08-31

#### Software Integration
- [ ] Load Linux on Zed Board (Week 3)
- [ ] Test driver loading (Week 4)
- [ ] Verify IIO interface (Week 5)
- [ ] Python API testing (Week 6-7)
- [ ] Web interface integration (Week 8)

**Target Completion**: 2026-08-31

### Phase 4: Characterization & Testing (Q3-Q4 2026)

#### Performance Characterization
- [ ] Measure all variants (September)
  - [ ] Config A (18-bit, 15 Msps)
  - [ ] Config B (16-bit, 15 Msps)
  - [ ] Config C (18-bit, 10 Msps)
  - [ ] Config D (16-bit, 10 Msps)
  - [ ] Config E (18-bit, 5 Msps)
  - [ ] Config F (16-bit, 5 Msps)
- [ ] FFT analysis and ENOB measurement
- [ ] DNL/INL characterization
- [ ] Temperature testing
- [ ] Long-term stability testing

**Target Completion**: 2026-09-30

**Key Metrics** (Per Variant):
- ENOB vs. input frequency
- SFDR, SNR, THD
- DNL/INL across full scale
- Performance vs. temperature (-40°C to +85°C)
- Power consumption

#### Comparison with Legacy
- [ ] Benchmark against DC2290A boards (October)
- [ ] Document performance deltas
- [ ] Identify improvement areas
- [ ] Create migration guide

**Target Completion**: 2026-10-15

#### Documentation
- [ ] Hardware design guide
- [ ] FPGA architecture document
- [ ] Software user manual
- [ ] Test reports
- [ ] Migration guide (from DC2290A)

**Target Completion**: 2026-10-31

### Phase 5: Production Release (Q4 2026)

#### Production Preparation
- [ ] Incorporate feedback from Phase 4 (November)
- [ ] Final design review
- [ ] Update documentation
- [ ] Prepare manufacturing package
- [ ] Setup quality control procedures

**Target Completion**: 2026-11-30

#### Production Build
- [ ] Order production PCBs (10-100 units)
- [ ] Order production components
- [ ] Assembly and testing
- [ ] QC testing on all units
- [ ] Package and ship

**Target Completion**: 2026-12-15

#### Software Release
- [ ] Final driver release (upstream submission)
- [ ] Python package on PyPI
- [ ] MATLAB toolbox release
- [ ] Web interface deployment
- [ ] Documentation website launch

**Target Completion**: 2026-12-20

#### Project Closeout
- [ ] Final project report
- [ ] Lessons learned document
- [ ] Archive design files
- [ ] Transition to maintenance mode

**Target Completion**: 2026-12-31

## Milestones

| Milestone | Target Date | Status |
|-----------|-------------|--------|
| M1: Design Complete | 2026-04-30 | 🟡 In Progress |
| M2: Implementation Complete | 2026-06-30 | ⚪ Not Started |
| M3: Prototype Working | 2026-08-31 | ⚪ Not Started |
| M4: Characterization Complete | 2026-10-31 | ⚪ Not Started |
| M5: Production Release | 2026-12-31 | ⚪ Not Started |

## Resource Requirements

### Team

- **Hardware Engineer**: 1 FTE (Schematic, PCB layout)
- **FPGA Engineer**: 1 FTE (HDL design, Vivado)
- **Software Engineer**: 1 FTE (Drivers, API, web)
- **Test Engineer**: 0.5 FTE (Characterization, validation)
- **Project Manager**: 0.25 FTE (Coordination, documentation)

### Equipment

- Xilinx Vivado Design Suite (licenses)
- PCB Design Tools (Altium/OrCAD)
- Test Equipment:
  - Signal generator (up to 50 MHz)
  - Oscilloscope (200 MHz, 4 channels)
  - Spectrum analyzer
  - Power supply
  - Logic analyzer
  - Temperature chamber (optional)

### Budget

| Item | Estimated Cost |
|------|----------------|
| PCB Fabrication (prototypes) | $5,000 |
| Components (prototypes) | $10,000 |
| Assembly Services | $3,000 |
| Test Equipment (if purchasing) | $50,000 |
| Software Licenses | $15,000/year |
| **Total** | **$83,000** |

## Risks & Mitigation

### Technical Risks

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| FMC pinout insufficient for all variants | Medium | High | Early pinout validation, consider FMC HPC if needed |
| LVDS timing closure issues | Medium | Medium | Conservative timing constraints, use SelectIO resources |
| ADC performance degradation | Low | High | Careful analog design, proper layout, thorough testing |
| Software driver instability | Medium | Medium | Extensive testing, upstream code review |

### Schedule Risks

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| PCB fabrication delays | Low | Medium | Order from reliable vendor, allow buffer time |
| Component supply chain issues | Medium | High | Order long-lead items early, identify alternates |
| Testing reveals major issues | Medium | High | Thorough simulation and design review upfront |
| Resource availability | Low | Medium | Cross-train team members, have backup resources |

## Success Criteria

### Technical Success

- [ ] All 6 ADC variants working on single FMC card
- [ ] Performance matches or exceeds legacy DC2290A boards
- [ ] FPGA timing closure with >10% margin
- [ ] Software stack fully functional on ADI Kuiper Linux
- [ ] Complete documentation package

### Performance Targets

| Metric | Target | Stretch Goal |
|--------|--------|--------------|
| ENOB (18-bit variants) | ≥16.5 bits | ≥17.0 bits |
| ENOB (16-bit variants) | ≥14.5 bits | ≥15.0 bits |
| SFDR | ≥92 dBc | ≥95 dBc |
| Data Throughput | ≥15 Msps | ≥20 Msps |
| Latency (FPGA to DDR) | <10 µs | <5 µs |

### Business Success

- Reduced hardware SKUs (6 boards → 1 board)
- Lower manufacturing cost per unit
- Improved user experience vs. legacy
- Positive feedback from early adopters
- Adoption by 5+ customer projects

## Communication Plan

### Internal Updates

- **Weekly**: Team standup (30 min)
- **Bi-weekly**: Technical review meeting
- **Monthly**: Stakeholder report

### External Communication

- **Monthly**: Blog post on progress
- **Quarterly**: Webinar/demo
- **At milestones**: Press releases

### Documentation Updates

- README.md updates: Weekly
- Design docs: As changes occur
- User documentation: Monthly during development
- Release notes: At each milestone

## Next Actions

### Immediate (This Week)

1. [ ] Finalize FMC pinout assignments
2. [ ] Complete ADC interface HDL module
3. [ ] Review schematic with hardware team
4. [ ] Set up GitHub Actions for automated testing

### Short-term (Next Month)

1. [ ] Complete schematic capture
2. [ ] Finish FPGA core module development
3. [ ] Begin PCB layout
4. [ ] Start IIO driver skeleton

### Medium-term (Next Quarter)

1. [ ] Complete PCB design and order
2. [ ] Complete FPGA implementation
3. [ ] Develop Python API
4. [ ] Prepare for prototype testing

## Revision History

| Version | Date | Changes | Author |
|---------|------|---------|--------|
| 1.0 | 2026-03-26 | Initial roadmap created | Project Team |

## Approval

This roadmap is a living document and will be updated as the project progresses.

**Project Sponsor**: _____________________  Date: __________

**Technical Lead**: _____________________  Date: __________

**Program Manager**: _____________________  Date: __________
