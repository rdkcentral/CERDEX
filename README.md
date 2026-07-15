# CERDEX — User Guide

## Table of Contents

1. [What is CERDEX?](#1-what-is-cerdex)
2. [Before You Begin](#2-before-you-begin)
3. [Installation](#3-installation)
4. [Post-Install: Updating Config Files](#4-post-install-updating-config-files)
5. [Step 1 — Run Setup](#5-step-1--run-setup)
6. [Step 2 — Run Tests](#6-step-2--run-tests)
7. [Step 3 — View the Report](#7-step-3--view-the-report)
8. [Common Run Scenarios](#8-common-run-scenarios)
9. [Understanding Results](#9-understanding-results)
10. [Uninstalling TDK](#10-uninstalling-tdk)
11. [Troubleshooting](#11-troubleshooting)

---

## 1. What is CERDEX?

CERDEX is a command-line tool that lets you run automated tests against RDK (Reference Design Kit) devices without needing the full TDK TestManager UI.

You install it once, point it at your device, pick the tests you want to run, and get a report.

```
Install ──► Configure device ──► tdk setup ──► tdk run ──► tdk report
```

---

## 2. Before You Begin

### What you need

| Item | Details |
|------|---------|
| A Linux machine | Ubuntu 20.04 or later recommended |
| A `tdk.deb` file | To be updated|
| A device config file | A device specific `.config` file for your device for certification tests |
| Module config files | Variable files for the test modules you want to run |

### Install system dependencies

```bash
sudo apt update
sudo apt install -y python3 python3-venv python3-pip git curl
```

### Verify you can reach GitHub

```bash
curl -I https://github.com
```

You should see `200` response (for example, `HTTP/2 200` or `HTTP/1.1 200`). If you get an error, the installer will not be able to download required components.

---

## 3. Installation

### Install the latest version

```bash
sudo dpkg -i tdk.deb
```

### Install a specific version

If you need a particular TDK milestone version, set `TDK_VERSION` before installing:

```bash
sudo TDK_VERSION=M149 dpkg -i tdk.deb
```

### What the installer does

The installer will clone the test framework and fetch configuration files from GitHub. You will see output like this:

```
ℹ️  No version specified — fetching latest tag from tdk-core...
ℹ️  Latest tag found: TDK_M149
ℹ️  Using cerdex tag: TDK_M149

📦 Cloning tdk-core at tag TDK_M149...
   Removing existing /opt/tdkv_repo
   ✅ tdk-core cloned → /opt/tdkv_repo

📥 Fetching config files from cerdex (TDK_M149)...
   ✅ requirements.txt
   ✅ tdk_modules.json
   Fetching test suites...
   ✅ suites/RDK8_RVS_Suite.xml
   ✅ suites/RDK8_PVS_Suite.xml

🐍 Setting up Python virtual environment...

✅ TDK installation complete! (tdk-core: TDK_M149  |  configs: TDK_M149)

Next steps:
  1. Update config files in: /opt/tdkv_repo/framework/fileStore/
  2. Run: tdk setup
  3. Run: tdk run
```

### Verify the installation

```bash
tdk --version
```

Expected output:

```
tdk M149
```

### Fix dependency errors (if any)

If the install fails with dependency errors, run:

```bash
sudo apt -f install -y
sudo dpkg -i tdk.deb
```

---

## 4. Post-Install: Updating Config Files

Before you can run any tests, you must copy your device-specific and module-specific configuration files into the framework's file store.

### Where config files go

All config files belong in:

```
/opt/tdkv_repo/framework/fileStore/
```

### Device config file

Your device config file (e.g., `mydevice.config`) must go into the `tdkvRDKServiceConfig/` subfolder:

```bash
sudo cp mydevice.config /opt/tdkv_repo/framework/fileStore/tdkvRDKServiceConfig/
```

When `tdk setup` asks for `DeviceConfigFile name`, enter just the filename without the path — e.g., `mydevice.config`.

### Module config files

Depending on which test module you plan to run, copy the relevant variable files:

| Module | Required files |
|--------|---------------|
| `rdkv_performance` | `BrowserPerformanceVariables.py`, `MediaValidationVariables.py`, `PerformanceTestVariables.py` |
| `rdkv_media` | `MediaValidationVariables.py` |
| `rdkv_stability` | `StabilityTestVariables.py` |
| `rdkservices` | No extra files needed |

**Example** — setting up for the `rdkv_performance` module:

```bash
sudo cp BrowserPerformanceVariables.py /opt/tdkv_repo/framework/fileStore/
sudo cp MediaValidationVariables.py  /opt/tdkv_repo/framework/fileStore/
sudo cp PerformanceTestVariables.py    /opt/tdkv_repo/framework/fileStore/
```

---

### `rdkv_media` — Additional setup

The `rdkv_media` module uses a built-in media server (started automatically by `tdk run`) that serves test streams and Lightning app bundles directly from the `/opt/tdkv_repo` directory. You must place the required files in the correct locations before running.

#### Step A — Copy test streams

Extract the stream archive and copy the streams to:

```
/opt/tdkv_repo/streams/
```

The media server exposes these files at `http://<HOST_IP>:8080/streams/`.

#### Step B — Copy Lightning app build folders

The `TDK_LightningApps_RDK8` directory contains four apps, each with a `build/` subfolder. Copy the `build/` folder of each app into `/opt/tdkv_repo/framework/fileStore/lightning-apps/<appname>/`:

The media server exposes these at `http://<HOST_IP>:8080/rdk-test-tool/fileStore/lightning-apps/`.

#### Step C — Configure `MediaValidationVariables.py`

Open `MediaValidationVariables.py` and update the following variables. Replace `<HOST_IP>` with the IP address of your test machine (the machine running `tdk`):

| Variable | Required value |
|----------|---------------|
| `lightning_apps_loc` | `"http://<HOST_IP>:8080/rdk-test-tool/fileStore/lightning-apps/"` |
| `test_streams_base_path` | `"http://<HOST_IP>:8080/streams/"` |


Then copy the file to the framework file store:

```bash
sudo cp MediaValidationVariables.py /opt/tdkv_repo/framework/fileStore/
```

> **Note:** The media server starts automatically when you run `tdk run` with the `rdkv_media` module. No manual server setup is required.

---

## 5. Step 1 — Run Setup

Before running tests, you must run `tdk setup` from the directory where you want your logs and reports to be saved. This creates a small config file in that directory that records your device details.

### Choose a working directory

```bash
mkdir ~/tdk_tests
cd ~/tdk_tests
```

### Run setup

```bash
tdk setup
```

You will be prompted for three values:

```
Device IP address: 192.168.1.100
Device port [9998]: 
DeviceConfigFile name: mydevice.config
```

- Press **Enter** to accept the default port (`9998`).
- For `DeviceConfigFile name`, enter the filename of the `.config` file you copied in Step 4 (just the name, not the full path).

### What setup creates

A file named `tdk_runner_config.py` in your current directory:

```python
deviceIP="192.168.1.100"
Port="9998"
deviceConfFileName="mydevice.config"
```

### Setup with command-line options (non-interactive)

You can skip the prompts by passing values directly:

```bash
tdk setup --ip 192.168.1.100 --port 9998 --conf-name mydevice.config
```

### Changing device details later

If you switch to a different device, re-run setup with `--overwrite`:

```bash
tdk setup --ip 192.168.2.50 --overwrite
```

---

## 6. Step 2 — Run Tests

Make sure you are in the same directory where `tdk_runner_config.py` was created.

```bash
cd ~/tdk_tests
sudo tdk run
```

### Interactive mode (default)

Running `sudo tdk run` without arguments launches an interactive menu.

**Step 1: Choose between a module or a suite**

```
1. Select module and scripts
2. Run a test suite

❓ Choose execution mode (1 or 2): 
```

**Step 2: Select a module**

```
Available modules:
  1. rdkservices
  2. rdkv_performance
  3. rdkv_media

Select module [1-3]: 1
```

**Step 3: Select scripts**

```
Available scripts (rdkv_performance):
 1. RDKV_CERT_AVS_AV_Input.py
 2. RDKV_CERT_AVS_Activity_Monitor.py
 3. RDKV_CERT_AVS_AppManager.py
 4. RDKV_CERT_AVS_AppStorageManager.py
  ... (20 shown, press Enter for next page)

Select scripts [1 2 3 or 'all']: 1 2
```

**Step 4: Confirm and run**

```
Running 2 script(s) against 192.168.1.100:9998...

[1/2] RDKV_CERT_AVS_AV_Input.py
  ✅ PASS

[2/2] RDKV_CERT_AVS_Activity_Monitor.py
  ❌ FAIL
```

**Step 5: Execution summary**

```
----------------------------------------
🟢 EXECUTION SUMMARY
----------------------------------------

| ------------- | ------------------------------ | ------- |
| TESTCASE ID   | TESTCASE NAME                  | STATUS  |
| ------------- | ------------------------------ | ------- |
| 01            | RDKV_CERT_AVS_AV_Input         | SUCCESS |
| 02            | RDKV_CERT_AVS_Activity_Monitor | FAILURE |
| ------------- | ------------------------------ | ------- |

Logs saved to:     Logs/rdkservices/
Report saved to:   CSV_reports/rdkservices/TDK_ExecutionResult_4521.xlsx
```

---

## 7. Step 3 — View the Report

After a test run, launch the interactive HTML report viewer:

```bash
tdk report
```

This starts a local web server and opens the report in your browser automatically:

```
Serving report at http://192.168.1.10:8899
Opening browser...
```

The report viewer shows:

- A summary bar with **total / passed / failed** counts
- Per-script tabs
- Search and filter by test status
- Expandable log output for each test case
<img width="1358" height="636" alt="image" src="https://github.com/user-attachments/assets/190d6fd9-6625-48b5-8f38-6f85b4e6191b" />


### View a specific report file

```bash
tdk report --file CSV_reports/rdkv_performance/TDK_ExecutionResult_4521.xlsx
```

### Use a different port (if 8899 is taken)

```bash
tdk report --port 9000
```

---

## 8. Common Run Scenarios

### Run a single specific script

```bash
sudo tdk run --module rdkv_performance --script RDKV_CERT_PVS_AppManager_TimeTo_Install_App.py
```

### Run all scripts in a module (skip interactive selection)

```bash
sudo tdk run --module rdkservices --all
```

### Run a predefined test suite

Suites are curated lists of scripts across modules. To see what suites are available:

```bash
sudo tdk run --list-suites
```

Sample output:

```
Available suites:
  • <suite_1> — <description>
  • <suite_2> — <description>
```

Run a suite:

```bash
sudo tdk run --suite RDK8_PVS_Suite
```

### Run multiple specific scripts

```bash
sudo tdk run --module rdkv_performance \
    --script RDKV_CERT_PVS_AppManager_TimeTo_Install_App.py \
    --script RDKV_CERT_PVS_Apps_TimeTo_Video_PlayPause_4K_MKV.py
```

### Override device details for a single run

Useful if you want to test against a different device without re-running setup:

```bash
sudo tdk run --ip 192.168.2.50 --port 9998
```

### Skip confirmation prompts (automated / CI use)

```bash
sudo tdk run --module rdkservices --all --assume-yes
```

---

## 9. Understanding Results

### Output files

Every test run produces two outputs in your working directory:

```
~/tdk_tests/
├── Logs/
│   └── rdkv_performance/
│       └── RDKV_CERT_PVS_Apps_TimeTo_Video_PlayPause_4K_MKV_4521.log   ← full execution log
└── CSV_reports/
    └── rdkv_performance/
        └── TDK_ExecutionResult_4521.xlsx        ← summary report
```

The number `4521` is a unique run ID assigned to that execution.

## 10. Uninstalling TDK

To completely remove CERDEX and all its files:

```bash
sudo apt purge -y tdk
```

This removes:
- The `tdk` command
- The cloned test framework (`/opt/tdkv_repo`)
- The Python virtual environment (`/opt/TDK_ENV`)
- All fetched config files (`/opt/cerdex`)

> **Note:** Your working directory (`~/tdk_tests/`) and its logs/reports are **not** removed.

---

## 11. Troubleshooting

### "tdk: command not found"

The package may not have installed correctly.

```bash
# Check if tdk is installed
dpkg -l | grep tdk

# Reinstall
sudo dpkg -i tdk.deb
```

### "tdk --version" shows "unknown"

The test framework was not cloned properly during install.

```bash
# Check if the directory exists
ls /opt/tdkv_repo

# Fix: reinstall
sudo dpkg -i tdk.deb
```

### "Setup not completed" when running `tdk run`

You are either in the wrong directory, or you have not run `tdk setup` yet.

```bash
# Check if the config file exists in your current directory
ls tdk_runner_config.py

# If not, run setup
tdk setup
```

### "Setup file is incomplete"

The `tdk_runner_config.py` is missing required values. Re-run setup with `--overwrite`:

```bash
tdk setup --overwrite
```

### Config file validation fails

Example error:

```
❌ Missing: BrowserPerformanceVariables.py
ℹ️  Please update them in: /opt/tdkv_repo/framework/fileStore/
```

Copy the missing file to the correct location and retry:

```bash
sudo cp BrowserPerformanceVariables.py /opt/tdkv_repo/framework/fileStore/
sudo tdk run
```

### Install fails: "Could not determine latest tdk-core tag"

Your machine cannot reach GitHub. Check your network connection:

```bash
curl -I https://github.com
```

If you know which version you need, pin it explicitly:

```bash
sudo TDK_VERSION=M149 dpkg -i tdk.deb
```

### Install fails: dependency errors

```bash
sudo apt -f install -y
sudo dpkg -i tdk.deb
```

### pip install fails during postinst

Manually install the Python dependencies:

```bash
/opt/TDK_ENV/bin/pip install -r /opt/cerdex/requirements.txt
```

If that does not work, purge and reinstall:

```bash
sudo dpkg --purge tdk && sudo dpkg -i tdk.deb
```

### Report viewer does not open

Try running without auto-open and access the URL manually:

```bash
tdk report --no-open
# Open http://localhost:8899 in your browser
```

If port 8899 is in use:

```bash
tdk report --port 9000
# Open http://localhost:9000 in your browser
```

---

## Quick Reference

```bash
# Install
sudo dpkg -i tdk.deb
sudo TDK_VERSION=M149 dpkg -i tdk.deb     # specific version

# Check version
tdk --version

# Setup (run once per working directory)
tdk setup
tdk setup --ip 192.168.1.100 --port 9998 --conf-name mydevice.config

# Run tests (interactive)
sudo tdk run

# Run tests (non-interactive)
sudo tdk run --module rdkv_performance --script RDKV_PERF_BrowserMemory.py
sudo tdk run --module rdkservices --all
sudo tdk run --suite RDK8_PVS_Suite

# List available suites
sudo tdk run --list-suites

# View report
tdk report
tdk report --file CSV_reports/rdkv_performance/TDK_ExecutionResult_4521.xlsx

# Uninstall
sudo apt purge -y tdk
```
