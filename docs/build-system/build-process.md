# Build Process Documentation

## Prerequisites

### System Requirements

```
Minimum Specifications:
├─ CPU: Multi-core processor (4+ cores recommended)
├─ RAM: 8 GB minimum (16 GB recommended)
├─ Storage: 50 GB free space
├─ OS: Linux (Ubuntu 18.04 LTS or later recommended)
└─ Network: Internet connection (for downloading sources)

Recommended Setup:
├─ CPU: 8+ cores (faster compilation)
├─ RAM: 32 GB (parallel builds)
├─ Storage: 100 GB SSD (faster I/O)
└─ OS: Ubuntu 20.04 LTS or 22.04 LTS
```

### Required Tools

```
Essential Tools:
├─ Build system: GNU Make 4.2+
├─ Compiler: GCC, arm-linux-gnueabihf toolchain
├─ Device tree: Device tree compiler (dtc)
├─ Version control: Git 2.20+
├─ Archiving: tar, gzip
└─ Text processing: sed, awk, grep

FPGA Tools (for bitstream generation):
├─ Vivado: 2023.2 (or specified VIVADO_VERSION)
├─ Xilinx SDK/XSCT
├─ bootgen (Xilinx boot image tool)
└─ Installation path: /opt/Xilinx/Vivado/...

Device Programming:
├─ dfu-util: For DFU firmware updates
└─ sshpass: For automated SSH (optional)

Optional Tools:
├─ ccache: Compiler caching (speeds up rebuilds)
├─ jq: JSON query tool (for CI scripts)
└─ curl: URL fetching (for license validation)
```

### Dependency Installation (Ubuntu)

```bash
# Core build tools
sudo apt-get install -y build-essential git make

# Cross-compilation toolchain
sudo apt-get install -y gcc-arm-linux-gnueabihf

# Device tree compiler
sudo apt-get install -y device-tree-compiler

# Utilities
sudo apt-get install -y bc cpio squashfs-tools curl unzip

# For testing/deployment
sudo apt-get install -y dfu-util sshpass
```

## Environment Setup

### Step 1: Detect Build Environment

```bash
# Source the automatic setup script
source setup_env.sh

# Output shows detected environment:
# export VIVADO_SETTINGS=/opt/Xilinx/Vivado/2023.2/settings64.sh
# export CROSS_COMPILE=arm-linux-gnueabihf-
# export PATH=...
```

### Step 2: Apply Environment Settings

```bash
# Copy and execute the recommended exports
export VIVADO_SETTINGS=/opt/Xilinx/Vivado/2023.2/settings64.sh
export CROSS_COMPILE=arm-linux-gnueabihf-
export PATH=$PATH:...

# Or use aliases in your shell profile
echo "source /path/to/plutosdr-fw/setup_env.sh" >> ~/.bashrc
```

### Step 3: Verify Installation

```bash
# Check Vivado
source $VIVADO_SETTINGS
which vivado

# Check cross-compiler
$CROSS_COMPILE gcc --version

# Check device tree compiler
dtc --version

# Check Git submodules
git submodule status
```

## Build Process Steps

### Stage 1: Initial Setup

```bash
# Clone repository with submodules
git clone --recursive https://github.com/analogdevicesinc/plutosdr-fw.git
cd plutosdr-fw

# Update submodules to latest
git submodule update --init --recursive

# List submodule status
git submodule status

# Expected output:
#  [commit_hash] u-boot-xlnx (HEAD detached at ...)
#  [commit_hash] buildroot (...)
#  [commit_hash] linux (...)
#  [commit_hash] hdl (...)
```

### Stage 2: Toolchain Build

```bash
# Build cross-compiler (only needed once)
make TOOLCHAIN

# This takes 20-30 minutes on first run
# Progress indicators:
# ├─ Buildroot configuration
# ├─ GCC compilation
# ├─ GNU binutils
# ├─ C library (glibc)
# └─ Final toolchain setup

# Output location:
# buildroot/output/host/bin/arm-linux-gnueabihf-*

# Note: This step is cached, subsequent builds skip it
```

### Stage 3: Full Build

```bash
# Build complete firmware
make all

# Process (with automatic parallelization):
# ├─ make TOOLCHAIN (skipped if already built)
# ├─ make u-boot-xlnx
# │  ├─ Configure U-Boot
# │  ├─ Compile with CROSS_COMPILE
# │  └─ Extract environment variables
# ├─ make linux
# │  ├─ Configure kernel
# │  ├─ Compile zImage
# │  └─ Compile device trees (3 variants)
# ├─ make buildroot
# │  ├─ Configure root filesystem
# │  ├─ Compile packages
# │  └─ Generate legal info
# ├─ make hdl (FPGA synthesis - optional with fallback)
# │  ├─ Run Vivado synthesis
# │  ├─ Place & route
# │  └─ Generate bitstream
# ├─ make boot.bin (FSBL + U-Boot)
# ├─ make pluto.itb (FIT image)
# ├─ make pluto.frm (MSD firmware)
# ├─ make pluto.dfu (DFU firmware)
# └─ make zip-all (Distribution archives)

# Estimated time: 30-45 minutes first run, 5-10 minutes incremental
```

## Build Stages Detail

### U-Boot Bootloader

```bash
# Manual build command
make -C u-boot-xlnx zynq_pluto_defconfig
make -C u-boot-xlnx -j$(nproc)

# Output:
# • u-boot.elf - Second-stage bootloader
# • u-boot (ELF format, ARM executable)
# • Tools: mkimage (used for FIT images)

# Defconfig file: u-boot-xlnx/configs/zynq_pluto_defconfig
# Configuration includes:
# • DFU support (Device Firmware Update)
# • QSPI support
# • Correct load address and entry point
```

### Linux Kernel

```bash
# Manual build command
make -C linux zynq_pluto_defconfig
make -C linux -j$(nproc) zImage ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf-

# Output:
# • zImage - Compressed kernel image (~4 MB)
# • vmlinux - Uncompressed kernel (for debugging)
# • System.map - Symbol table

# Device tree compilation
make -C linux -j$(nproc) ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- dtbs

# Output device trees (3 variants):
# • arch/arm/boot/dts/zynq-pluto-sdr.dtb (RevA)
# • arch/arm/boot/dts/zynq-pluto-sdr-revb.dtb (RevB)
# • arch/arm/boot/dts/zynq-pluto-sdr-revc.dtb (RevC)

# Post-processing (via Makefile):
# • Convert 'axi' bus name to 'amba' (compatibility)
# • Decompile to DTS, modify, recompile to DTB
```

### Root Filesystem (Buildroot)

```bash
# Configuration
make -C buildroot zynq_pluto_defconfig

# Build
make -C buildroot -j$(nproc) all

# Output:
# • output/images/rootfs.cpio.gz - Compressed root filesystem
# • output/images/rootfs.tar - Uncompressed (optional)
# • output/images/ - Various generated artifacts
# • output/images/ROOT_FS_SKELETON - Filesystem structure
# • buildroot/board/zynq/pluto/VERSIONS - Version manifest

# License generation (via legal_info_html.sh):
# • scripts/legal_info_html.sh "PlutoSDR" "buildroot/board/pluto/VERSIONS"
# • Output: build/LICENSE.html (comprehensive license documentation)

# Buildroot defconfig location:
# buildroot/configs/zynq_pluto_defconfig
# Includes:
# • Kernel image source
# • Package selection
# • Library configuration
# • Init system (busybox)
```

### FPGA Bitstream (Optional)

```bash
# FPGA build (requires Vivado)
make -C hdl/projects/pluto

# Steps:
# 1. Source Vivado settings
# 2. Run vivado -mode batch
# 3. Create/open project
# 4. Add RTL sources (system_top.v, system_bd.tcl)
# 5. Synthesis → Place & Route
# 6. Generate bitstream
# 7. Export hardware (XSA)

# Output:
# • system_top.bit - FPGA bitstream (~943 KB)
# • system_top.xsa - Hardware definition
# • build/sdk/ - Software development kit

# Time estimate: 15-20 minutes on modern CPU

# Fallback mechanism:
# If Vivado not available or build fails:
# • Script can auto-download pre-built XSA
# • Uses pre-generated bitstream
# • Allows firmware builds without Vivado
```

### First-Stage Bootloader (FSBL)

```bash
# Generate FSBL from hardware definition
xsct scripts/create_fsbl_project.tcl

# Process:
# 1. Load system_top.xsa (from Vivado)
# 2. Extract PS7 configuration
# 3. Create FSBL project using Xilinx template
# 4. Configure for release build (optimized)
# 5. Compile with cross-compiler
# 6. Generate fsbl.elf

# Output:
# • build/sdk/fsbl/Release/fsbl.elf (~50 KB)
# • FSBL initializes:
#   • PS7 memory controller
#   • Clock distribution
#   • QSPI flash controller
#   • Boot device configuration

# Note: FSBL is hardware-dependent
# Must be regenerated after FPGA changes
```

### Boot Image Generation (boot.bin)

```bash
# Create boot image from FSBL and U-Boot
bootgen -image scripts/pluto.bif -o build/boot.bin

# BIF (Boot Image Format) file:
# • Specifies component ordering
# • FSBL first (bootloader startup)
# • U-Boot second (main bootloader)
# • Configuration: encryption, authentication

# Output:
# • build/boot.bin (~443 KB)
# • Zynq boot format (processor-specific)
# • Ready for QSPI Flash mtd0 partition

# Process:
# 1. FSBL loaded and executed by ROM bootloader
# 2. FSBL initializes PS7 and validates QSPI
# 3. FSBL loads U-Boot from QSPI
# 4. U-Boot transfers control to kernel
```

### Firmware Packaging

```bash
# Create Flattened Image Tree (FIT)
u-boot-xlnx/tools/mkimage -f scripts/pluto.its build/pluto.itb

# FIT image contains:
# • Device Tree Blobs (3 variants: RevA, RevB, RevC)
# • Linux kernel (zImage, compressed)
# • Root filesystem (rootfs.cpio.gz)
# • FPGA bitstream (system_top.bit)
# • Hash values for integrity check

# Output:
# • build/pluto.itb (~11 MB)
# • Bootable image for U-Boot
# • Contains all components for full boot

# Format inspection:
# u-boot-xlnx/tools/mkimage -l build/pluto.itb
# Output shows:
# Image Type: Multi-file Image (FIT)
# Data Size: ...
# Load Address: ...
# Entry Point: ...
# Component sizes and hash values
```

### DFU Firmware Creation

```bash
# Add DFU trailer to FIT image
dfu-suffix -add -vid 0x0456 -pid 0xb673 -O build/pluto.dfu < build/pluto.itb

# Process:
# 1. Read pluto.itb
# 2. Append DFU suffix (16 bytes)
#    • Device class code
#    • Subclass
#    • USB vendor ID (0x0456 - Analog Devices)
#    • USB product ID (0xb673 - PlutoSDR)
#    • Firmware version
# 3. Calculate and append CRC
# 4. Write to pluto.dfu

# Output:
# • build/pluto.dfu (~11 MB)
# • Used by dfu-util for firmware updates
# • Device recognizes this format in DFU mode

# Alternative: MSD format
# • pluto.frm = pluto.dfu with MD5 checksum
# • Used for USB Mass Storage Device firmware updates
```

### Distribution Archives

```bash
# Create distribution package
make zip-all

# Output files:
# • plutosdr-fw-vX.XX.zip
#   └─ Contains:
#      • pluto.frm / pluto.dfu (firmware)
#      • boot.frm / boot.dfu (bootloader)
#      • LICENSE.html (documentation)
#      • README.txt (instructions)
#
# • plutosdr-jtag-bootstrap-vX.XX.zip
#   └─ Contains:
#      • run.tcl, run-xsdb.tcl (JTAG scripts)
#      • u-boot.elf (for RAM bootstrap)
#      • ps7_init.tcl (hardware init)
#      • Instructions for recovery
#
# • sysroot-vX.XX.tar.gz
#   └─ Development files:
#      • Header files
#      • Pre-compiled libraries
#      • Cross-compilation tools
#
# • legal-info-vX.XX.tar.gz
#   └─ License information:
#      • COPYING files
#      • Open-source licenses
#      • License manifest
```

## Build Output Structure

```
build/
├── TOOLCHAIN/ (buildroot output)
│   └── output/
│       └── host/bin/ (arm-linux-gnueabihf-gcc, etc.)
├── u-boot.elf (Second-stage bootloader)
├── zImage (Compressed Linux kernel)
├── zynq-pluto-sdr.dtb (Device tree RevA)
├── zynq-pluto-sdr-revb.dtb (Device tree RevB)
├── zynq-pluto-sdr-revc.dtb (Device tree RevC)
├── rootfs.cpio.gz (Root filesystem)
├── system_top.bit (FPGA bitstream)
├── system_top.xsa (Vivado hardware export)
├── fsbl.elf (First-stage bootloader)
├── boot.bin (Zynq boot image)
├── pluto.itb (Flattened Image Tree)
├── pluto.frm (MSD firmware)
├── pluto.dfu (DFU firmware)
├── boot.frm / boot.dfu (Bootloader images)
├── uboot-env.dfu (U-Boot environment)
├── uboot-env.txt (Environment variables text)
├── LICENSE.html (License documentation)
├── plutosdr-fw-vX.XX.zip (Full distribution)
├── plutosdr-jtag-bootstrap-vX.XX.zip (Recovery)
├── sysroot-vX.XX.tar.gz (Development files)
└── legal-info-vX.XX.tar.gz (License archive)
```

## Clean Targets

```bash
# Clean build artifacts (keep sources)
make clean-build
# Removes: build/ directory

# Full clean (remove all generated files)
make clean
# Removes: build/, *.o, intermediate files
# Keeps: source code, git history

# Full distclean (remove sources too)
make distclean
# Removes: everything including submodules
# Use git clone --recursive to restore
```

## Build Optimization

### Parallel Building

```bash
# Automatic CPU core detection
make -j$(nproc)

# Explicit number of jobs
make -j8

# For 4-core CPU: ~3.5x faster
# For 8-core CPU: ~6x faster
# For 16-core CPU: ~10x faster

# Diminishing returns above CPU core count
```

### Incremental Builds

First build: 30-45 minutes
├─ All components built from scratch
└─ FPGA synthesis: 15-20 minutes

Incremental (no changes): <1 minute
├─ Just checks for freshness
└─ Verifies all artifacts up-to-date

Rebuild after small change: 5-10 minutes
├─ Recompiles affected components
├─ Firmware repacking
└─ FPGA rebuild (if hardware changed)

Rebuild after FPGA change: 20-30 minutes
├─ Vivado P&R: 15-20 minutes
├─ FSBL regeneration: 2-3 minutes
└─ Firmware repacking: 1-2 minutes
```

### Compiler Cache (ccache)

```bash
# Install ccache
sudo apt-get install ccache

# Enable for builds
export CROSS_COMPILE="ccache arm-linux-gnueabihf-"

# Subsequent identical builds: 70-80% faster
# Cache directory: ~/.ccache/ (default 5 GB)

# Clear cache if needed
ccache -C
```

## Troubleshooting Common Build Issues

### Issue: Missing Vivado

```
Error: Vivado not found
Solution:
1. Install Vivado 2023.2
2. Run: source setup_env.sh
3. Verify: which vivado
4. Or skip FPGA build (pre-built fallback available)
```

### Issue: Insufficient Disk Space

```
Error: No space left on device
Solution:
1. Clean previous builds: make clean
2. Remove intermediate artifacts
3. Ensure 50 GB free space
4. Use: df -h to check
```

### Issue: Timeout During Download

```
Error: Network timeout downloading submodules
Solution:
1. Retry: git submodule update --init
2. Increase timeout: export GIT_HTTP_TIMEOUT=300
3. Use SSH: git config --global url."git@github.com:".insteadOf "https://github.com/"
```

### Issue: Toolchain Version Mismatch

```
Error: arm-linux-gnueabihf-gcc: not found
Solution:
1. Build toolchain: make TOOLCHAIN
2. Verify path: which arm-linux-gnueabihf-gcc
3. Check buildroot output: ls buildroot/output/host/bin/
```

## Related Documentation

- [Build Targets](./build-targets.md) - Complete target reference
- [Deployment Methods](./deployment.md) - Firmware update methods
- [Build System](../components/build-system.md) - Build infrastructure details
