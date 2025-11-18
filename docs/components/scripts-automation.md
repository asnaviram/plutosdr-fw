# Scripts and Automation Component Documentation

## Overview

The PlutoSDR project includes comprehensive automation infrastructure with shell scripts, Python utilities, and TCL scripts for build automation, deployment, and CI/CD integration.

## Shell Scripts

### 1. Environment Setup (setup_env.sh)

**Location**: Root directory (48 lines)

**Purpose**: Automatic environment detection and configuration

```bash
Usage: source setup_env.sh

Function:
├── Detect Vivado installation path
│   └── Searches /opt/Xilinx for 2023.2 (or specified version)
│
├── Configure cross-compiler path
│   ├── Detects arm-linux-gnueabihf toolchain
│   ├── Validates GCC version
│   └── Sets CROSS_COMPILE environment variable
│
├── Setup PATH for build tools
│   └── Adds buildroot/output/host/bin to PATH
│
└── Export recommended commands for user
    └── Display: export VIVADO_SETTINGS=...
```

**Output Example**:
```bash
export VIVADO_SETTINGS=/opt/Xilinx/Vivado/2023.2/settings64.sh
export CROSS_COMPILE=arm-linux-gnueabihf-
export PATH=...
```

### 2. U-Boot Environment Extraction (get_default_envs.sh)

**Location**: `scripts/get_default_envs.sh` (34 lines)

**Purpose**: Extract default U-Boot environment variables from compiled binary

```bash
Usage: scripts/get_default_envs.sh > build/uboot-env.txt

Process:
1. Locate env_common.o in u-boot build artifacts
2. Extract .rodata.default_environment section using objcopy
3. Convert binary to null-terminated strings
4. Convert null terminators to newlines
5. Sort alphabetically for consistent output

Binary Section Layout:
┌─────────────────────────────┐
│ .rodata.default_environment │
│ (Null-terminated strings)   │
│                             │
│ "varname=value\0"          │
│ "varname2=value2\0"        │
│ ...                         │
└─────────────────────────────┘
```

**Output Format**:
```
bootargs=console=ttyPS0,115200 root=/dev/mtdblock3
bootcmd=run boot_config
ethaddr=00:AD:DE:AD:BE:EF
ipaddr=192.168.2.1
...
```

### 3. License Documentation Generator (legal_info_html.sh)

**Location**: `scripts/legal_info_html.sh` (368 lines)

**Purpose**: Generate comprehensive HTML license documentation for firmware

```bash
Usage: scripts/legal_info_html.sh "PlutoSDR" "buildroot/board/pluto/VERSIONS"

Functions:
├── html_header()
│   └── Generate HTML5 boilerplate with CSS styling
│
├── html_h1() / html_h2() / html_p()
│   └── Generate HTML semantic elements
│
├── html_pre_file()
│   └── Sanitize text files for HTML display
│   │   • HTML-escape special characters
│   │   • Preserve whitespace and formatting
│   │   └── Wrap in <pre> tags
│
├── package_table_items()
│   └── Generate sortable table of packages
│       • Name, Version, License, URL
│       • Click-through to full license text
│       └── Vendor information
│
├── markdown_to_html()
│   └── Convert LICENSE.md to HTML
│       • Process headers (# → <h1>)
│       • Convert links [text](url) → <a href>
│       • Handle images ![alt](url) → <img>
│       └── Validate URLs with curl
│
├── written_offer_clause()
│   └── Add GPL written offer for source code
│
└── generate_license_archive()
    └── Package all license files and COPYING text
```

**Output Structure**:
```html
build/LICENSE.html
├── Document Header
│   ├── Title: "PlutoSDR License Information"
│   ├── Generation Date
│   └── Version Information
│
├── Summary Section
│   └── Quick license overview
│
├── Component List
│   └── Sortable table with:
│       • Package names & versions
│       • License types
│       • Download URLs
│       • License text links
│
├── Detailed License Texts
│   ├── GPL v2 (Kernel, etc.)
│   ├── MIT (Various components)
│   ├── BSD (Libraries)
│   └── ...
│
└── Compliance Information
    ├── Written offer for source code
    ├── How to obtain sources
    └── Distribution media info
```

### 4. Firmware Download and Test (download_and_test.sh)

**Location**: Root directory (35 lines)

**Purpose**: Automated firmware deployment and testing via DFU

```bash
Usage: ./download_and_test.sh

Process Flow:
1. Verify firmware file exists
   └─ Check: build/pluto.dfu

2. Establish SSH connection to device
   ├─ Target IP: 192.168.2.1
   ├─ Username: root
   ├─ Password: analog (via sshpass)
   └─ Command: device_reboot ram

3. Wait for DFU device to appear
   ├─ Monitor USB enumeration
   ├─ Timeout: 10 seconds
   └─ Look for: 0456:b674 (DFU mode)

4. Download firmware via DFU
   ├─ Command: dfu-util -R
   ├─ USB IDs: 0456:b673 → 0456:b674
   └─ Action: Reset device after completion

Success Indicators:
• Device enumerates as DFU
• dfu-util reports successful transfer
• Device restarts automatically
```

**Dependencies**: `sshpass`, `dfu-util`, SSH connectivity

## Python Scripts

### CI/CD Artifact Upload (upload_to_artifactory.py)

**Location**: `CI/upload_to_artifactory.py` (141 lines)

**Purpose**: Upload build artifacts to internal Artifactory repository

```python
Usage: ./upload_to_artifactory.py \
    --base_path="https://artifactory.example.com/artifactory" \
    --server_path="hdl/master/2024_01_15/pluto" \
    --local_path="build/plutosdr-fw-v0.39.zip" \
    --properties="git_sha=abc123;git_commit_date=2024-01-15" \
    --token="$API_TOKEN"

Features:
├── Command-line Argument Parsing
│   ├── --base_path: Artifactory URL
│   ├── --server_path: Target folder
│   ├── --local_path: Source file/directory
│   ├── --properties: Metadata key=value pairs
│   ├── --token: API authentication
│   └── --no_rel_path: Flatten directory structure
│
├── Allowed Server Paths (validation)
│   ├── hdl, linux, linux_rpi
│   ├── arm_trusted_firmware, boot_partition
│   ├── rootfs, u-boot
│   ├── HighSpeedConverterToolbox, TransceiverToolbox
│   ├── SD_card_image, m2k_and_pluto
│   └── Custom paths (if pre-approved)
│
├── File Upload
│   ├── Single file upload
│   ├── Recursive directory handling
│   ├── Preserve directory structure
│   └── Parallel uploads for speed
│
├── Metadata Handling
│   ├── Property attachment to files
│   ├── Property propagation to parent dirs
│   ├── Custom metadata: git_sha, git_commit_date, etc.
│   └── Transitive property assignment
│
└── Authentication
    ├── API Token from environment (API_TOKEN)
    ├── Or via command-line argument
    └── JFrog-compatible HTTP headers

Example Properties:
• git_sha=928ggraf93                    (Git commit SHA)
• git_commit_date=2024-01-15           (Commit date)
• build_status=success                  (Build result)
• build_number=12345                    (CI/CD build ID)
• arch=arm                              (Target architecture)
```

**Authentication Flow**:
```
1. Read API_TOKEN from environment
2. Construct HTTP headers for JFrog API
3. Setup SSL verification
4. Add properties as query parameters
5. POST file to Artifactory endpoint
6. Handle authentication errors
```

## TCL Scripts

### JTAG Bootstrap Scripts

#### run.tcl (XMD-based)

**Location**: `scripts/run.tcl` (18 lines)

**Purpose**: JTAG bootstrap U-Boot using Xilinx Command Line Tool (xmd)

```tcl
Usage: xmd -tcl run.tcl

Commands:
┌─────────────────────────────────────┐
│ connect arm hw                      │
│ # Connect to ARM processor via JTAG │
└─────────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│ stop                                │
│ # Stop execution (halt)             │
└─────────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│ xreset 64                           │
│ # Reset PS7 (processor system)      │
│ # 64 = Reset width                  │
└─────────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│ source ps7_init.tcl                 │
│ # Load Vivado-generated init file   │
└─────────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│ ps7_init                            │
│ # Execute PS7 initialization        │
│ (Memory controller, clocks, etc.)   │
└─────────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│ ps7_post_config                     │
│ # Post-initialization configuration │
└─────────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│ dow u-boot.elf                      │
│ # Download u-boot ELF to RAM        │
│ # Address: 0x04000000 (default)     │
└─────────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│ run                                 │
│ # Start execution                   │
│ # U-Boot runs, can interact via     │
│ # serial console or Telnet          │
└─────────────────────────────────────┘
```

**Use Cases**:
- Initial SPI flash programming
- System debugging and verification
- Bootloader recovery
- Hardware validation

#### run-xsdb.tcl (XSDB-based, Recommended)

**Location**: `scripts/run-xsdb.tcl` (18 lines)

**Purpose**: JTAG bootstrap using Xilinx System Debug Tool (XSDB)

```tcl
Usage: xsdb run-xsdb.tcl

Features (vs XMD):
├── Newer Vivado support
├── Better integration with 2023.x releases
├── Improved performance
└── More robust error handling

Commands:
┌─────────────────────────────────────┐
│ connect                             │
│ # Connect to XSDB server            │
└─────────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│ target 2                            │
│ # Select target 2 (ARM processor)   │
│ # Target 1: JTAG, Target 2: ARM     │
└─────────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│ rst                                 │
│ # Reset target (PS7 and fabric)     │
└─────────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│ source ps7_init.tcl                 │
│ # Load initialization script        │
└─────────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│ ps7_init                            │
│ # Initialize processing system      │
│ # Clocks, DDR3, IO banks            │
└─────────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│ ps7_post_config                     │
│ # Post-configuration setup          │
└─────────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│ dow u-boot.elf                      │
│ # Download u-boot to RAM            │
└─────────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│ con                                 │
│ # Continue execution                │
│ # U-Boot starts running             │
└─────────────────────────────────────┘
```

### FSBL Project Generation (create_fsbl_project.tcl)

**Location**: `scripts/create_fsbl_project.tcl` (8 lines)

**Purpose**: Automated First-Stage Bootloader generation via XSCT

```tcl
Usage: xsct scripts/create_fsbl_project.tcl

Process:
┌────────────────────────────────────────────┐
│ 1. Load Hardware Description               │
│    • Input: build/system_top.xsa           │
│    • Extract: FPGA design and PS config    │
└────────────────────────────────────────────┘
               │
               ▼
┌────────────────────────────────────────────┐
│ 2. Extract Processor Information           │
│    • Find: CPU IP name (e.g., ps7_0)      │
│    • Type: ARM Cortex-A9                   │
│    • Validate: Single processor expected   │
└────────────────────────────────────────────┘
               │
               ▼
┌────────────────────────────────────────────┐
│ 3. Create SDK Workspace                    │
│    • Directory: build/sdk/                 │
│    • Purpose: Software development area    │
│    • Initialize: HSI (Hardware SW I/F)     │
└────────────────────────────────────────────┘
               │
               ▼
┌────────────────────────────────────────────┐
│ 4. Create FSBL Application                 │
│    • Template: "Zynq FSBL"                 │
│    • Language: C                           │
│    • Board: Zynq-7000                      │
│    • Features: Standard initialization     │
└────────────────────────────────────────────┘
               │
               ▼
┌────────────────────────────────────────────┐
│ 5. Configure Build Settings                │
│    • Build Type: Release (optimized)       │
│    • Optimization: -O2                     │
│    • Debug Info: Stripped                  │
│    • Output Size: ~50 KB                   │
└────────────────────────────────────────────┘
               │
               ▼
┌────────────────────────────────────────────┐
│ 6. Compile FSBL                            │
│    • Use cross-compiler                    │
│    • Link against PSL (PS library)         │
│    • Generate: fsbl.elf                    │
└────────────────────────────────────────────┘
               │
               ▼
┌────────────────────────────────────────────┐
│ Output: build/sdk/fsbl/Release/fsbl.elf    │
│                                            │
│ Next Step: bootgen (combine with u-boot)  │
└────────────────────────────────────────────┘
```

## CI/CD Configuration Files

### Status Rule Pattern Matching

**Location**: `CI/status_rule` (19 lines)

**Purpose**: Regex patterns for CI/CD build status detection

```
Pattern Categories:

[WARNING] - Non-critical issues
├── Warning
├── "cannot open"
├── "cannot create"
├── "cannot stat"
├── "doesn't exist"
├── "no such file or directory"
├── "Permission denied"
├── Artifactory connection timeout
└── Unknown non-specific errors

[ERROR] - Critical build failures
├── Error (keyword)
├── "no rule to make target" (Makefile error)
├── "command not found" (missing tool)
├── Compilation failures
└── Linker errors

Usage in CI/CD:
├── Parse build logs with regex
├── Extract warnings vs errors
├── Aggregate build health status
├── Pass/fail determination
└── Report generation
```

## Build Automation Workflow

```
Build Triggered (Manual or Webhook)
    │
    ▼
source setup_env.sh
    │
    ├─ Detect Vivado path
    ├─ Setup cross-compiler
    └─ Configure environment
    │
    ▼
make TOOLCHAIN
    │
    ├─ Build cross-compiler
    └─ Setup buildroot
    │
    ▼
make all
    │
    ├─ Build U-Boot, Linux, FPGA, etc.
    ├─ Run get_default_envs.sh
    ├─ Generate legal_info_html.sh
    └─ Package firmware
    │
    ▼
CI/status_rule (log parsing)
    │
    ├─ Extract warnings/errors
    └─ Determine build health
    │
    ▼
upload_to_artifactory.py
    │
    ├─ Upload build artifacts
    ├─ Attach metadata (git SHA, date)
    └─ Publish to repository
    │
    ▼
Build Complete
    ├─ Success: Distribute artifacts
    ├─ Failure: Alert developers
    └─ Log: Save build logs
```

## Deployment Workflows

### Via DFU (Device Firmware Update)

```
download_and_test.sh sequence:

1. SSH to device (192.168.2.1)
2. Reboot to RAM (bootm 0x3000000)
3. Wait for DFU enumeration (~10s)
4. dfu-util upload:
   • Download pluto.dfu
   • Write to mtd3 (QSPI)
   • Reset device
5. Device restarts with new firmware
```

### Via USB Mass Storage Device

```
Manual MSD update:

1. Connect PlutoSDR to PC
2. Device appears as USB storage
3. Copy pluto.frm to device
4. Copy boot.frm (optional)
5. Device auto-detects and reflashes
6. Device restarts with new firmware
```

## Related Documentation

- [Build System](./build-system.md) - Build infrastructure
- [Build Process](../build-system/build-process.md) - Step-by-step guide
- [Deployment Methods](../build-system/deployment.md) - Firmware update methods
