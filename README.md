# DC2290A-x Migration Repo

This repository contains workflows, documentation, and scripts to support the migration of **DC2290A-x legacy demo boards** (LTC2387 family, 6 variants). It is designed to be a living workspace where system engineers can track tasks, generate documentation, and automate testing using GitHub Actions and agentic AI workflows.

---

## 📂 Repository Structure

- **docs/**  
  Contains migration guides, comparison tables, and technical notes.  
  - `migration-checklist.md` → auto-generated checklist of migration tasks  
  - `comparison-table.md` → auto-generated table of all six board variants  

- **scripts/**  
  Automation scripts for testing and validation.  
  - Example: `adc_capture.py` for data collection  
  - Example: `fpga_interface_test.vhd` for FPGA compatibility checks  

- **data/**  
  Stores captured test results, logs, and benchmark data.  
  - Example: `results.csv`  

- **.github/workflows/**  
  GitHub Actions workflows that automate documentation and testing.  
  - `dc2290a-migration.yml` → generates migration checklist and comparison table  

---

## 🚀 Running the Migration Workflow

1. Go to your repository on GitHub.  
2. Click the **Actions** tab.  
3. Select **DC2290A-x Migration Workflow**.  
4. Click **Run workflow**.  

The workflow will:
- Generate `docs/migration-checklist.md` with migration tasks.  
- Generate `docs/comparison-table.md` with specs for all six variants.  
- Commit and push these files back into the repo automatically.  

---

## ⚙️ Node.js 24 Compatibility

GitHub Actions is deprecating Node.js 20.  
This workflow is already configured to **force Node.js 24** using:

```yaml
env:
  FORCE_JAVASCRIPT_ACTIONS_TO_NODE24: true
