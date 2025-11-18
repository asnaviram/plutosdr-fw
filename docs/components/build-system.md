# Build System Component Documentation

## Overview

The PlutoSDR build system is a sophisticated multi-stage embedded Linux build orchestrated by GNU Make. It coordinates compilation of U-Boot, Linux kernel, Buildroot filesystem, and FPGA designs into complete firmware packages.

## Build System Architecture

```
┌────────────────────────────────────────────────────────────────┐
│                    Main Makefile Entry Point                   │
│  (/home/user/plutosdr-fw/Makefile - 234 lines)               │
└─────────────────────┬──────────────────────────────────────────┘
                      │
        ┌─────────────┼─────────────┐
        │             │             │
        ▼             ▼             ▼
    ┌────────────┐┌────────────┐┌────────────┐
    │Target      ││Submodule   ││Cross-      │
    │Configs     ││Management  ││compiler    │
    │(pluto.mk)  ││(u-boot,    ││Setup       │
    │(sidekiq.mk)││linux, hdl) ││(buildroot) │
    └────────────┘└────────────┘└────────────┘
        │             │             │
        └─────────────┼─────────────┘
                      │
                      ▼
    ┌──────────────────────────────────────┐
    │       Build Stage Orchestration      │
    │  (Dependencies managed by Make)      │
    └──────────────────────────────────────┘
        │
        ├─ Stage 1: Toolchain (buildroot cross-compiler)
        ├─ Stage 2: U-Boot bootloader
        ├─ Stage 3: Linux kernel + device trees
        ├─ Stage 4: Buildroot filesystem
        ├─ Stage 5: FPGA hardware (Vivado)
        ├─ Stage 6: Boot image generation
        ├─ Stage 7: Firmware packaging (FRM/DFU)
        └─ Stage 8: Distribution archives

        │
        ▼
    ┌──────────────────────────────────────┐
    │      Output Artifacts (build/)       │
    │                                      │
    │  • Firmware files (*.frm, *.dfu)     │
    │  • Bitstream (*.bit)                 │
    │  • Bootloaders (*.elf, *.bin)        │
    │  • Distribution archives (*.zip)     │
    └──────────────────────────────────────┘
```

## Key Makefile Components

### 1. Target-Specific Configuration Files

**File: `scripts/pluto.mk` (11 lines)**
```makefile
# PlutoSDR Target Configuration
HDF_URL = [Vivado hardware description file location]
TARGET_DTS_FILES = zynq-pluto-sdr.dtb zynq-pluto-sdr-revb.dtb zynq-pluto-sdr-revc.dtb
COMPLETE_NAME = PlutoSDR
ZIP_ARCHIVE_PREFIX = plutosdr
DEVICE_VID = 0x0456
DEVICE_PID = 0xb673
```

**File: `scripts/sidekiqz2.mk` (7 lines)**
```makefile
# SidekiqZ2 Target Configuration
TARGET_DTS_FILES = zynq-sidekiqz2-revb.dtb
COMPLETE_NAME = SidekiqZ2
ZIP_ARCHIVE_PREFIX = sidekiqz2
DEVICE_VID = 0x2fa2
DEVICE_PID = 0x5a02
```

### 2. Build Variables and Configuration

```makefile
# Cross-compilation settings
CROSS_COMPILE = arm-linux-gnueabihf-
VIVADO_VERSION = 2023.2
NCORES = $(shell nproc)              # Auto-detect CPU cores
TARGET = pluto                        # or sidekiqz2

# Tool paths
TOOLCHAIN = buildroot/output/host/bin
TOOLS_PATH = PATH=$(TOOLCHAIN):$$PATH
DTC_FLAGS = -@                        # Device tree overlays
```

### 3. Build Stages

#### Stage 1: Toolchain Setup
```makefile
TOOLCHAIN:
    make -C buildroot
    # Output: arm-linux-gnueabihf-gcc cross-compiler
    # Location: buildroot/output/host/bin/
```

#### Stage 2: U-Boot Compilation
```makefile
u-boot-xlnx/u-boot:
    make -C u-boot-xlnx zynq_pluto_defconfig
    make -C u-boot-xlnx -j$(NCORES)
    # Output: u-boot.elf
```

#### Stage 3: Linux Kernel
```makefile
linux/arch/arm/boot/zImage:
    make -C linux zynq_pluto_defconfig
    make -C linux -j$(NCORES) zImage
    # Output: zImage (compressed kernel)
```

#### Stage 4: Device Tree Compilation
```makefile
build/%.dtb: linux/arch/arm/boot/dts/%.dtb
    dtc -q -@ -I dtb -O dts $< | sed 's/axi {/amba {/g' | dtc -q -@ -I dts -O dtb -o $@
    # Output: Multiple device tree blobs (revA, revB, revC)
```

#### Stage 5: Root Filesystem
```makefile
build/rootfs.cpio.gz:
    make -C buildroot
    # Output: Compressed root filesystem (cpio archive)
```

#### Stage 6: FPGA Bitstream (Optional)
```makefile
build/system_top.xsa:
    make -C hdl/projects/pluto
    # Calls Vivado for FPGA synthesis and place & route
    # Output: system_top.bit (FPGA bitstream)
```

#### Stage 7: Boot Image Generation
```makefile
build/boot.bin:
    create_fsbl_project.tcl (XSCT)      # Generate FSBL
    bootgen                              # Combine FSBL + U-Boot
    # Output: boot.bin (Zynq boot image)
```

#### Stage 8: Firmware Packaging
```makefile
build/pluto.frm:                        # MSD (Mass Storage Device) format
    dfu-suffix -add -vid 0x0456 -pid 0xb673
    # Output: pluto.frm (DFU packaged firmware with MD5)

build/pluto.dfu:                        # DFU (Device Firmware Update) format
    # Output: DFU-format image
```

## Build Targets

### Main Targets

| Target | Purpose | Output |
|--------|---------|--------|
| `all` | Complete build from scratch | FRM/DFU + legal info |
| `TOOLCHAIN` | Cross-compiler only | `buildroot/output/host/bin/` |
| `build/pluto.frm` | MSD firmware | `build/pluto.frm` |
| `build/pluto.dfu` | DFU firmware | `build/pluto.dfu` |
| `jtag-bootstrap` | JTAG recovery package | ZIP archive |
| `legal-info` | License documentation | `build/LICENSE.html` |
| `zip-all` | Complete distribution | `plutosdr-fw-vX.XX.zip` |

### Deployment Targets

```makefile
# Via DFU Mode
dfu-pluto:      # Deploy pluto.dfu to QSPI mtd3
dfu-ram:        # Boot to RAM via SSH, then deploy
dfu-sf-uboot:   # Flash U-Boot (caution!)
dfu-all:        # Full deployment sequence

# Via USB Mass Storage
msd-boot:       # Copy boot.frm to MSD device
msd-pluto:      # Copy pluto.frm to MSD device
```

## Git Submodule Structure

```
plutosdr-fw/ (meta-repository)
├── u-boot-xlnx/ (submodule: Xilinx U-Boot fork)
│   ├── branch: pluto
│   └── build output: u-boot.elf
│
├── linux/ (submodule: Linux kernel)
│   ├── branch: 2018_R1
│   ├── arch/arm/boot/dts/*.dts (device tree sources)
│   └── build output: zImage, *.dtb
│
├── buildroot/ (submodule: Filesystem/toolchain)
│   ├── branch: master (ADI variant)
│   ├── output/host/bin/ (cross-compiler)
│   └── output/images/ (rootfs.cpio.gz)
│
└── hdl/ (submodule: FPGA designs)
    ├── branch: hdl_2018_r1
    ├── projects/pluto/ (system_top designs)
    └── build output: system_top.bit
```

## Build Process Flow

```
1. MAKE TOOLCHAIN
   ├─ Configure Buildroot
   ├─ Build cross-compiler (GCC 7.3)
   └─ Output: arm-linux-gnueabihf-*

2. MAKE U-BOOT
   ├─ Configure: zynq_pluto_defconfig
   ├─ Compile with CROSS_COMPILE
   ├─ Extract environment: get_default_envs.sh
   └─ Output: u-boot.elf, uboot-env.txt

3. MAKE LINUX
   ├─ Configure: zynq_pluto_defconfig
   ├─ Compile kernel: zImage
   ├─ Compile device trees (3 variants)
   │  └─ Postprocess with dtc (axi→amba conversion)
   └─ Output: zImage, *.dtb

4. MAKE BUILDROOT
   ├─ Build rootfs
   ├─ Generate legal-info (license documentation)
   └─ Output: rootfs.cpio.gz, LICENSE.html

5. MAKE VIVADO (Optional, can skip)
   ├─ Run Vivado synthesis/P&R
   ├─ Generate bitstream
   └─ Output: system_top.bit

6. MAKE FSBL
   ├─ Run XSCT: create_fsbl_project.tcl
   ├─ Compile First-Stage Bootloader
   └─ Output: fsbl.elf

7. MAKE BOOT.BIN
   ├─ Run bootgen
   ├─ Combine FSBL + U-Boot
   └─ Output: boot.bin (Zynq boot image)

8. MAKE FIRMWARE
   ├─ Create FIT image (pluto.itb)
   │  └─ Combines: FPGA + DTB + kernel + rootfs
   ├─ Package with DFU suffix
   ├─ Generate MD5 checksums
   └─ Output: pluto.frm, pluto.dfu, etc.

9. MAKE LEGAL-INFO
   ├─ Run legal_info_html.sh
   ├─ Generate comprehensive license report
   └─ Output: LICENSE.html, license files

10. MAKE ARCHIVES
    ├─ Create distribution ZIP
    ├─ Create JTAG bootstrap package
    ├─ Create sysroot tarball
    └─ Output: plutosdr-fw-vX.XX.zip
```

## Compiler and Tool Configuration

### Cross-Compilation Toolchain

```
Compiler: arm-linux-gnueabihf-gcc (Linaro 7.3-2018.05)
Architecture: ARM 32-bit EABI (hard float)
Optimization: -O2 (default in Buildroot)
Kernel Version: 2.018.R1
C Library: glibc

Default Flags:
  • CFLAGS: -O2 -mcpu=cortex-a9 -mfpu=neon -mfloat-abi=hard
  • LDFLAGS: -Wl,-rpath,/usr/lib
```

### Device Tree Compiler

```
Tool: dtc (device-tree-compiler)
Flags: -@ (enable device tree overlays)

Postprocessing:
  • axi bus → amba bus conversion
  • Maintains compatibility with Linux drivers
```

### Boot Image Tools

```
Tool: bootgen (Xilinx)
Purpose: Generate Zynq boot.bin from FSBL + U-Boot

Tool: dfu-suffix (dfu-util)
Purpose: Add DFU trailer for device firmware updates
```

## Build Output Structure

```
build/
├── TOOLCHAIN/ (buildroot output)
│   └── output/host/bin/ (cross-compiler)
├── u-boot.elf
├── zImage
├── *.dtb (multiple device tree variants)
├── rootfs.cpio.gz
├── system_top.bit (FPGA bitstream)
├── system_top.xsa (Vivado export)
├── boot.bin
├── pluto.itb (FIT image)
├── pluto.frm (MSD firmware)
├── pluto.dfu (DFU firmware)
├── uboot-env.dfu
├── boot.dfu
├── boot.frm
├── uboot-env.txt (environment variables)
├── LICENSE.html
├── sysroot-vX.XX.tar.gz (development sysroot)
├── legal-info-vX.XX.tar.gz (license files)
├── plutosdr-fw-vX.XX.zip (complete distribution)
└── plutosdr-jtag-bootstrap-vX.XX.zip (recovery)
```

## Key Build Scripts

| Script | Location | Purpose |
|--------|----------|---------|
| `setup_env.sh` | Root directory | Environment detection and setup |
| `get_default_envs.sh` | `scripts/` | U-Boot environment extraction |
| `create_fsbl_project.tcl` | `scripts/` | FSBL generation via XSCT |
| `legal_info_html.sh` | `scripts/` | License documentation generator |
| `download_and_test.sh` | Root directory | Automated DFU deployment/test |
| `upload_to_artifactory.py` | `CI/` | Artifact upload for CI/CD |

## Build Performance and Parallelization

```
Automatic Core Detection:
  NCORES = $(shell nproc)
  # Auto-detected from /proc/cpuinfo

Parallel Build Usage:
  make -j$(NCORES)              # All stages use parallelization

Example on 4-core system:
  NCORES = 4
  • U-Boot: 4 parallel compilation jobs
  • Linux: 4 parallel compilation jobs
  • Buildroot: 4 parallel compilation jobs
  • Total speedup: ~3-4x vs serial build

Estimated Build Times:
  • Full clean build: 30-45 minutes (first time)
  • Incremental rebuild: 5-10 minutes
  • FPGA rebuild only: 15-20 minutes (Vivado)
  • Toolchain build: 20-30 minutes (buildroot)
```

## Conditional Compilation

```makefile
# Skip legal info generation
make SKIP_LEGAL=1 all

# Use pre-built system.xsa (avoid Vivado)
# (Automatically downloads if XSA build unavailable)

# Custom Vivado version
make VIVADO_VERSION=2023.1 all

# Specific build target
make TARGET=sidekiqz2 all
```

## Clean Targets

```makefile
clean-build          # Remove build/ directory only
clean                # Full source tree cleanup
                     # (removes all build artifacts, keeps sources)
distclean            # Ultra-clean (removes sources too)
```

## Related Documentation

- [Build Process](../build-system/build-process.md) - Detailed build guide
- [Build Targets](../build-system/build-targets.md) - Complete target list
- [Deployment Methods](../build-system/deployment.md) - Firmware deployment
