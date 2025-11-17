# PlutoSDR Firmware - Build Flow

## Overview

This document provides a comprehensive analysis of the complete build flow for PlutoSDR firmware, from source code to flashable firmware images.

## Complete Build Flow Diagram

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                         PLUTOSDR FIRMWARE BUILD FLOW                         │
└──────────────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────────────┐
│  PHASE 0: Prerequisites and Environment Setup                             │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  User Action:                                                              │
│    $ export VIVADO_SETTINGS=/opt/Xilinx/Vivado/2023.2/settings64.sh       │
│    $ export CROSS_COMPILE=arm-linux-gnueabihf-                             │
│    $ export TARGET=pluto  (or sidekiqz2)                                   │
│    $ git clone --recursive https://github.com/analogdevicesinc/plutosdr-fw│
│    $ cd plutosdr-fw                                                        │
│    $ make                                                                  │
│                                                                            │
│  Makefile Initialization:                                                  │
│  ┌────────────────────────────────────────────────────────────┐           │
│  │ • Detect VIVADO_VERSION (default: 2023.2)                  │           │
│  │ • Set NCORES from /proc/cpuinfo                            │           │
│  │ • Include scripts/$(TARGET).mk                             │           │
│  │ • Check for Vivado installation → HAVE_VIVADO             │           │
│  │ • Check for dfu-suffix → determines .dfu targets          │           │
│  │ • Set TOOLS_PATH with buildroot toolchain                 │           │
│  └────────────────────────────────────────────────────────────┘           │
│                                                                            │
│  Output: Build environment configured                                      │
│  Duration: <1 second                                                       │
└────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌────────────────────────────────────────────────────────────────────────────┐
│  PHASE 1: Toolchain Bootstrap                                             │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  Target: TOOLCHAIN                                                         │
│  Makefile Line: 61-63                                                      │
│                                                                            │
│  Build Steps:                                                              │
│  ┌────────────────────────────────────────────────────────────┐           │
│  │ 1. make -C buildroot zynq_$(TARGET)_defconfig              │           │
│  │    • Load buildroot/configs/zynq_pluto_defconfig           │           │
│  │    • Configure for ARM Cortex-A9                           │           │
│  │    • Set external Linaro toolchain                         │           │
│  │                                                             │           │
│  │ 2. make -C buildroot toolchain                             │           │
│  │    • Download Linaro GCC 7.3-2018.05                       │           │
│  │      URL: releases.linaro.org/...arm-linux-gnueabihf       │           │
│  │    • Extract to buildroot/output/host/                     │           │
│  │    • Verify toolchain components:                          │           │
│  │      ✓ arm-linux-gnueabihf-gcc                             │           │
│  │      ✓ arm-linux-gnueabihf-g++                             │           │
│  │      ✓ arm-linux-gnueabihf-ld                              │           │
│  │      ✓ arm-linux-gnueabihf-objcopy                         │           │
│  │      ✓ arm-linux-gnueabihf-strip                           │           │
│  └────────────────────────────────────────────────────────────┘           │
│                                                                            │
│  Artifacts Created:                                                        │
│    buildroot/output/host/bin/arm-linux-gnueabihf-*                        │
│                                                                            │
│  Duration: ~5 minutes (download + extract)                                │
│  Parallel: N/A (sequential download)                                      │
└────────────────────────────────────────────────────────────────────────────┘
                                    │
         ┌──────────────────────────┼──────────────────────────┐
         │                          │                          │
         ▼                          ▼                          ▼
┌─────────────────┐      ┌─────────────────┐      ┌─────────────────┐
│  PHASE 2A:      │      │  PHASE 2B:      │      │  PHASE 2C:      │
│  U-Boot Build   │      │  Linux Kernel   │      │  HDL/FPGA       │
│                 │      │  Build          │      │  Build          │
└─────────────────┘      └─────────────────┘      └─────────────────┘
         │                          │                          │
         │                          │                          │
         ▼                          ▼                          ▼

┌────────────────────────────────────────────────────────────────────────────┐
│  PHASE 2A: U-Boot Bootloader Build                                        │
├────────────────────────────────────────────────────────────────────────────┤
│  Target: u-boot-xlnx/u-boot                                                │
│  Makefile Lines: 70-72                                                     │
│  Dependencies: TOOLCHAIN                                                   │
│                                                                            │
│  Build Steps:                                                              │
│  ┌────────────────────────────────────────────────────────────┐           │
│  │ 1. cd u-boot-xlnx                                           │           │
│  │ 2. make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf-        │           │
│  │         zynq_$(TARGET)_defconfig                            │           │
│  │    • Load configs/zynq_pluto_defconfig                      │           │
│  │    • Configure: FIT support, DFU, USB gadget               │           │
│  │                                                             │           │
│  │ 3. make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf-        │           │
│  │         UBOOTVERSION="PlutoSDR $(VERSION)"                  │           │
│  │    • Compile U-Boot ELF binary                              │           │
│  │    • Build tools/mkimage (FIT image generator)              │           │
│  │    • Build tools/mkenvimage (environment generator)         │           │
│  │    • Parallel build (-j not used)                           │           │
│  └────────────────────────────────────────────────────────────┘           │
│                                                                            │
│  Post-Build: Extract Default Environment                                  │
│  ┌────────────────────────────────────────────────────────────┐           │
│  │ scripts/get_default_envs.sh > build/uboot-env.txt          │           │
│  │  • Find env_common.o                                        │           │
│  │  • Extract .rodata.default_environment section             │           │
│  │  • Convert to text format                                  │           │
│  │  • Sort variables                                           │           │
│  │                                                             │           │
│  │ u-boot-xlnx/tools/mkenvimage -s 0x20000 \                  │           │
│  │         -o build/uboot-env.bin build/uboot-env.txt         │           │
│  │  • Create 128KB binary environment                         │           │
│  └────────────────────────────────────────────────────────────┘           │
│                                                                            │
│  Artifacts Created:                                                        │
│    ✓ u-boot-xlnx/u-boot (ELF binary)                                       │
│    ✓ u-boot-xlnx/tools/mkimage                                             │
│    ✓ u-boot-xlnx/tools/mkenvimage                                          │
│    ✓ build/u-boot.elf                                                      │
│    ✓ build/uboot-env.txt                                                   │
│    ✓ build/uboot-env.bin                                                   │
│                                                                            │
│  Duration: ~3 minutes                                                      │
│  Size: u-boot.elf (~761KB), uboot-env.bin (128KB)                         │
└────────────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────────────┐
│  PHASE 2B: Linux Kernel Build                                             │
├────────────────────────────────────────────────────────────────────────────┤
│  Target: linux/arch/arm/boot/zImage                                        │
│  Makefile Lines: 89-96                                                     │
│  Dependencies: TOOLCHAIN                                                   │
│                                                                            │
│  Build Steps:                                                              │
│  ┌────────────────────────────────────────────────────────────┐           │
│  │ 1. cd linux                                                 │           │
│  │ 2. make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf-        │           │
│  │         zynq_$(TARGET)_defconfig                            │           │
│  │    • Load arch/arm/configs/zynq_pluto_defconfig             │           │
│  │    • Configure kernel features:                             │           │
│  │      - IIO subsystem                                        │           │
│  │      - AD9361 driver                                        │           │
│  │      - USB gadget                                           │           │
│  │      - AXI DMA                                              │           │
│  │      - Device tree support                                  │           │
│  │                                                             │           │
│  │ 3. make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf-        │           │
│  │         -j $(NCORES) zImage UIMAGE_LOADADDR=0x8000         │           │
│  │    • Parallel compilation (uses all CPU cores)              │           │
│  │    • Compile kernel modules                                 │           │
│  │    • Link kernel image                                      │           │
│  │    • Compress to zImage format                              │           │
│  └────────────────────────────────────────────────────────────┘           │
│                                                                            │
│  Device Tree Build:                                                        │
│  ┌────────────────────────────────────────────────────────────┐           │
│  │ For each DTB in TARGET_DTS_FILES:                          │           │
│  │   make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf-         │           │
│  │        -j $(NCORES) DTC_FLAGS=-@ $(DTB_NAME).dtb           │           │
│  │                                                             │           │
│  │ PlutoSDR Device Trees:                                      │           │
│  │   • zynq-pluto-sdr.dtb (Rev A)                             │           │
│  │   • zynq-pluto-sdr-revb.dtb (Rev B)                        │           │
│  │   • zynq-pluto-sdr-revc.dtb (Rev C)                        │           │
│  │                                                             │           │
│  │ Post-processing (for each DTB):                             │           │
│  │   dtc -I dtb -O dts input.dtb | \                          │           │
│  │   sed 's/axi {/amba {/g' | \                               │           │
│  │   dtc -I dts -O dtb -o build/output.dtb                    │           │
│  │   • Convert axi → amba for compatibility                   │           │
│  └────────────────────────────────────────────────────────────┘           │
│                                                                            │
│  Artifacts Created:                                                        │
│    ✓ linux/arch/arm/boot/zImage                                            │
│    ✓ linux/arch/arm/boot/dts/*.dtb                                         │
│    ✓ build/zImage                                                          │
│    ✓ build/zynq-pluto-sdr*.dtb                                             │
│                                                                            │
│  Duration: ~15 minutes (depends on NCORES)                                │
│  Size: zImage (~4.1MB), DTB (~22-23KB each)                               │
└────────────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────────────┐
│  PHASE 2C: HDL/FPGA Build                                                 │
├────────────────────────────────────────────────────────────────────────────┤
│  Target: build/system_top.xsa                                              │
│  Makefile Lines: 157-167                                                   │
│  Dependencies: Vivado (optional)                                           │
│                                                                            │
│  Build Path A: WITH VIVADO                                                │
│  ┌────────────────────────────────────────────────────────────┐           │
│  │ 1. source $VIVADO_SETTINGS                                 │           │
│  │    • Load Vivado environment                                │           │
│  │                                                             │           │
│  │ 2. make -C hdl/projects/$(TARGET)                          │           │
│  │    • Open Vivado project                                    │           │
│  │    • Synthesize HDL design:                                 │           │
│  │      - AXI DMA controllers                                  │           │
│  │      - ADC/DAC cores                                        │           │
│  │      - AXI interconnects                                    │           │
│  │      - GPIO/I2C peripherals                                 │           │
│  │    • Implement design (place & route)                       │           │
│  │    • Generate bitstream                                     │           │
│  │    • Export hardware (XSA file)                             │           │
│  │      Contains:                                              │           │
│  │        - system_top.bit (bitstream)                         │           │
│  │        - ps7_init.c/h/tcl                                   │           │
│  │        - Hardware metadata                                  │           │
│  │                                                             │           │
│  │ 3. cp hdl/projects/$(TARGET)/$(TARGET).sdk/system_top.xsa \│           │
│  │       build/system_top.xsa                                  │           │
│  └────────────────────────────────────────────────────────────┘           │
│                                                                            │
│  Build Path B: WITHOUT VIVADO (Pre-built)                                 │
│  ┌────────────────────────────────────────────────────────────┐           │
│  │ 1. wget ${XSA_URL} -O build/system_top.xsa                 │           │
│  │    URL: github.com/analogdevicesinc/plutosdr-fw/releases/  │           │
│  │         download/${LATEST_TAG}/system_top.xsa              │           │
│  │    • Download pre-built XSA from GitHub releases           │           │
│  │                                                             │           │
│  │ 2. unzip -l build/system_top.xsa system_top.bit            │           │
│  │    • Extract bitstream only                                 │           │
│  └────────────────────────────────────────────────────────────┘           │
│                                                                            │
│  FSBL Generation (if Vivado available):                                   │
│  ┌────────────────────────────────────────────────────────────┐           │
│  │ xsct scripts/create_fsbl_project.tcl                        │           │
│  │  • Open system_top.xsa in HSI                               │           │
│  │  • Detect ARM processor                                     │           │
│  │  • Create FSBL application                                  │           │
│  │  • Build in Release mode                                    │           │
│  │  • Output: build/sdk/fsbl/Release/fsbl.elf                 │           │
│  └────────────────────────────────────────────────────────────┘           │
│                                                                            │
│  Artifacts Created:                                                        │
│    ✓ build/system_top.xsa (~716KB)                                         │
│    ✓ build/system_top.bit (~943KB)                                         │
│    ✓ build/ps7_init.tcl/c/h                                                │
│    ✓ build/sdk/fsbl/Release/fsbl.elf (if Vivado available)                │
│                                                                            │
│  Duration:                                                                 │
│    With Vivado: 1-2 hours (FPGA synthesis)                                │
│    Without Vivado: ~30 seconds (download)                                 │
└────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌────────────────────────────────────────────────────────────────────────────┐
│  PHASE 3: Buildroot Root Filesystem                                       │
├────────────────────────────────────────────────────────────────────────────┤
│  Target: buildroot/output/images/rootfs.cpio.gz                           │
│  Makefile Lines: 112-124                                                   │
│  Dependencies: TOOLCHAIN, $(TARGET_DTS_FILES), build/system_top.bit       │
│                                                                            │
│  Pre-Build: Version Information Generation                                │
│  ┌────────────────────────────────────────────────────────────┐           │
│  │ Create buildroot/board/$(TARGET)/VERSIONS:                 │           │
│  │   firmware:   v0.39                                         │           │
│  │   hdl:        hdl_2018_r1-XXX-gYYYY                        │           │
│  │   linux:      2018_R1-XXX-gYYYY                            │           │
│  │   u-boot:     pluto-XXX-gYYYY                              │           │
│  │   buildroot:  master-XXX-gYYYY                             │           │
│  └────────────────────────────────────────────────────────────┘           │
│                                                                            │
│  Build Steps:                                                              │
│  ┌────────────────────────────────────────────────────────────┐           │
│  │ 1. make -C buildroot zynq_$(TARGET)_defconfig              │           │
│  │    • Load buildroot/configs/zynq_pluto_defconfig           │           │
│  │                                                             │           │
│  │ 2. make -C buildroot legal-info (unless SKIP_LEGAL=1)      │           │
│  │    • Generate buildroot/output/legal-info/                 │           │
│  │    • Create manifest.csv with all packages                 │           │
│  │    • Copy license files                                     │           │
│  │                                                             │           │
│  │ 3. make -C buildroot \                                      │           │
│  │         BUSYBOX_CONFIG_FILE=$(CURDIR)/buildroot/board/     │           │
│  │         $(TARGET)/busybox-1.25.0.config all                │           │
│  │    • Build all selected packages:                           │           │
│  │      ┌─────────────────────────────────────────┐           │           │
│  │      │ Core System:                             │           │
│  │      │  - BusyBox (configured utilities)        │           │
│  │      │  - Init scripts                           │           │
│  │      │  - Base libraries (glibc)                 │           │
│  │      │                                           │           │
│  │      │ Network:                                  │           │
│  │      │  - Dropbear (SSH server)                  │           │
│  │      │  - Avahi (mDNS)                           │           │
│  │      │  - wpa_supplicant                         │           │
│  │      │  - iw (wireless tools)                    │           │
│  │      │                                           │           │
│  │      │ IIO Stack:                                │           │
│  │      │  - libiio                                 │           │
│  │      │  - libad9361-iio                          │           │
│  │      │  - ad936x_ref_cal                         │           │
│  │      │  - iiod (IIO daemon)                      │           │
│  │      │                                           │           │
│  │      │ Utilities:                                │           │
│  │      │  - mtd-utils                              │           │
│  │      │  - u-boot-tools                           │           │
│  │      │  - libgpiod                               │           │
│  │      └─────────────────────────────────────────┘           │           │
│  │                                                             │           │
│  │ 4. Post-build script execution:                             │           │
│  │    board/$(TARGET)/post-build.sh                           │           │
│  │    • Install init scripts to /etc/init.d/                  │           │
│  │    • Install utility scripts                                │           │
│  │    • Create directory structure                             │           │
│  │    • Copy web interface files                               │           │
│  │    • Generate version information                           │           │
│  │                                                             │           │
│  │ 5. Generate root filesystem:                                │           │
│  │    • Pack into cpio archive                                 │           │
│  │    • Compress with gzip                                     │           │
│  │    • Output: rootfs.cpio.gz                                 │           │
│  └────────────────────────────────────────────────────────────┘           │
│                                                                            │
│  Legal Information Processing:                                            │
│  ┌────────────────────────────────────────────────────────────┐           │
│  │ scripts/legal_info_html.sh "PlutoSDR" \                     │           │
│  │     buildroot/board/pluto/VERSIONS                          │           │
│  │  • Parse manifest.csv                                       │           │
│  │  • Generate HTML license report                             │           │
│  │  • Include GPL written offer                                │           │
│  │  • Output: build/LICENSE.html                               │           │
│  └────────────────────────────────────────────────────────────┘           │
│                                                                            │
│  Artifacts Created:                                                        │
│    ✓ buildroot/output/images/rootfs.cpio.gz (~5.3MB)                      │
│    ✓ build/rootfs.cpio.gz                                                  │
│    ✓ build/LICENSE.html                                                    │
│    ✓ buildroot/output/legal-info/ (directory)                             │
│                                                                            │
│  Duration: ~30 minutes (package compilation)                              │
└────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌────────────────────────────────────────────────────────────────────────────┐
│  PHASE 4: Boot Image Generation                                           │
├────────────────────────────────────────────────────────────────────────────┤
│  Target: build/boot.bin                                                    │
│  Makefile Lines: 137-147                                                   │
│  Dependencies: build/sdk/fsbl/Release/fsbl.elf, build/u-boot.elf          │
│                                                                            │
│  Build Steps:                                                              │
│  ┌────────────────────────────────────────────────────────────┐           │
│  │ 1. Generate boot.bif (Boot Image Format file):             │           │
│  │    ┌────────────────────────────────────────┐              │           │
│  │    │ img:                                    │              │           │
│  │    │ {                                       │              │           │
│  │    │   [bootloader]build/sdk/fsbl/Release/  │              │           │
│  │    │               fsbl.elf                  │              │           │
│  │    │   build/u-boot.elf                      │              │           │
│  │    │ }                                       │              │           │
│  │    └────────────────────────────────────────┘              │           │
│  │                                                             │           │
│  │ 2. bootgen -image build/boot.bif -w -o build/boot.bin      │           │
│  │    • Combine FSBL + U-Boot                                  │           │
│  │    • Add Zynq boot header                                   │           │
│  │    • Generate authentication fields                         │           │
│  │    • Create bootable image                                  │           │
│  └────────────────────────────────────────────────────────────┘           │
│                                                                            │
│  Boot Firmware Packaging:                                                 │
│  ┌────────────────────────────────────────────────────────────┐           │
│  │ build/boot.frm:                                             │           │
│  │   cat build/boot.bin \                                      │           │
│  │       build/uboot-env.bin \                                 │           │
│  │       scripts/target_mtd_info.key | \                       │           │
│  │   tee build/boot.frm | md5sum | cut -d ' ' -f1 >> \        │           │
│  │       build/boot.frm                                        │           │
│  │   • Concatenate: boot.bin + env + mtd_info                  │           │
│  │   • Append MD5 checksum                                     │           │
│  │   • Format for USB MSD update                               │           │
│  │                                                             │           │
│  │ build/boot.dfu:                                             │           │
│  │   cp build/boot.bin build/boot.bin.tmp                      │           │
│  │   dfu-suffix -a build/boot.bin.tmp \                        │           │
│  │              -v $(DEVICE_VID) -p $(DEVICE_PID)             │           │
│  │   mv build/boot.bin.tmp build/boot.dfu                      │           │
│  │   • Add DFU suffix with USB VID/PID                         │           │
│  │   • Format for DFU mode update                              │           │
│  └────────────────────────────────────────────────────────────┘           │
│                                                                            │
│  Artifacts Created:                                                        │
│    ✓ build/boot.bin (~572KB)                                               │
│    ✓ build/boot.frm (boot.bin + env + mtd + MD5)                          │
│    ✓ build/boot.dfu (boot.bin + DFU suffix)                                │
│    ✓ build/uboot-env.dfu (uboot-env.bin + DFU suffix)                     │
│                                                                            │
│  Duration: ~10 seconds                                                     │
└────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌────────────────────────────────────────────────────────────────────────────┐
│  PHASE 5: FIT Image Creation                                              │
├────────────────────────────────────────────────────────────────────────────┤
│  Target: build/$(TARGET).itb                                               │
│  Makefile Line: 130                                                        │
│  Dependencies: build/zImage, build/rootfs.cpio.gz, DTBs, system_top.bit   │
│                                                                            │
│  Build Steps:                                                              │
│  ┌────────────────────────────────────────────────────────────┐           │
│  │ u-boot-xlnx/tools/mkimage -f scripts/$(TARGET).its \       │           │
│  │         build/$(TARGET).itb                                 │           │
│  │                                                             │           │
│  │ FIT Image Structure (from pluto.its):                       │           │
│  │ ┌───────────────────────────────────────────────┐          │           │
│  │ │ Images:                                        │          │           │
│  │ │   fdt@1:          zynq-pluto-sdr.dtb          │          │           │
│  │ │   fdt@2:          zynq-pluto-sdr-revb.dtb     │          │           │
│  │ │   fdt@3:          zynq-pluto-sdr-revc.dtb     │          │           │
│  │ │   fpga@1:         system_top.bit              │          │           │
│  │ │                   load=0xF000000               │          │           │
│  │ │   linux_kernel@1: zImage                       │          │           │
│  │ │                   load=0x8000                  │          │           │
│  │ │                   entry=0x8000                 │          │           │
│  │ │   ramdisk@1:      rootfs.cpio.gz               │          │           │
│  │ │                                                │          │           │
│  │ │ Configurations:                                │          │           │
│  │ │   config@0:  Rev A (default)                   │          │           │
│  │ │   config@1-7,9-10: Rev B                       │          │           │
│  │ │   config@8:  Rev C                             │          │           │
│  │ │                                                │          │           │
│  │ │ Each config specifies:                         │          │           │
│  │ │   - Device tree to use                         │          │           │
│  │ │   - Kernel image                               │          │           │
│  │ │   - Ramdisk                                    │          │           │
│  │ │   - FPGA bitstream (loadables)                 │          │           │
│  │ └───────────────────────────────────────────────┘          │           │
│  │                                                             │           │
│  │ mkimage Processing:                                         │           │
│  │ • Parse .its file                                           │           │
│  │ • Read all binary components                                │           │
│  │ • Calculate MD5 hashes                                      │           │
│  │ • Create FDT (Flattened Device Tree) structure              │           │
│  │ • Package everything into single .itb file                  │           │
│  └────────────────────────────────────────────────────────────┘           │
│                                                                            │
│  Main Firmware Packaging:                                                 │
│  ┌────────────────────────────────────────────────────────────┐           │
│  │ build/$(TARGET).frm:                                        │           │
│  │   cat build/$(TARGET).itb | \                               │           │
│  │   tee build/$(TARGET).frm | \                               │           │
│  │   md5sum | cut -d ' ' -f1 >> build/$(TARGET).frm          │           │
│  │   • Append MD5 checksum to ITB                              │           │
│  │   • Format for USB MSD update                               │           │
│  │                                                             │           │
│  │ build/$(TARGET).dfu:                                        │           │
│  │   cp build/$(TARGET).itb build/$(TARGET).itb.tmp           │           │
│  │   dfu-suffix -a build/$(TARGET).itb.tmp \                   │           │
│  │              -v $(DEVICE_VID) -p $(DEVICE_PID)             │           │
│  │   mv build/$(TARGET).itb.tmp build/$(TARGET).dfu           │           │
│  │   • Add DFU suffix                                          │           │
│  │   • Format for DFU mode update                              │           │
│  └────────────────────────────────────────────────────────────┘           │
│                                                                            │
│  Artifacts Created:                                                        │
│    ✓ build/pluto.itb (~11MB)                                               │
│    ✓ build/pluto.frm (ITB + MD5)                                           │
│    ✓ build/pluto.dfu (ITB + DFU suffix)                                    │
│                                                                            │
│  Duration: ~5 seconds                                                      │
└────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌────────────────────────────────────────────────────────────────────────────┐
│  PHASE 6: Final Packaging                                                 │
├────────────────────────────────────────────────────────────────────────────┤
│  Target: build/plutosdr-fw-$(VERSION).zip                                  │
│  Makefile Lines: 209-218                                                   │
│                                                                            │
│  Packaging Steps:                                                          │
│  ┌────────────────────────────────────────────────────────────┐           │
│  │ 1. Create release ZIP:                                      │           │
│  │    zip -j build/$(ZIP_ARCHIVE_PREFIX)-fw-$(VERSION).zip \  │           │
│  │        build/pluto.frm \                                    │           │
│  │        build/pluto.dfu \                                    │           │
│  │        build/boot.frm \                                     │           │
│  │        build/boot.dfu \                                     │           │
│  │        build/uboot-env.dfu                                  │           │
│  │                                                             │           │
│  │ 2. Create JTAG bootstrap ZIP (if Vivado available):        │           │
│  │    strip build/u-boot.elf -o build/u-boot.elf.strip        │           │
│  │    zip -j build/$(ZIP_ARCHIVE_PREFIX)-jtag-bootstrap-\     │           │
│  │        $(VERSION).zip \                                     │           │
│  │        build/u-boot.elf.strip \                             │           │
│  │        build/ps7_init.tcl \                                 │           │
│  │        build/system_top.bit \                               │           │
│  │        scripts/run.tcl \                                    │           │
│  │        scripts/run-xsdb.tcl                                 │           │
│  │                                                             │           │
│  │ 3. Package legal information:                               │           │
│  │    tar czf build/legal-info-$(VERSION).tar.gz \            │           │
│  │        -C buildroot/output legal-info                       │           │
│  └────────────────────────────────────────────────────────────┘           │
│                                                                            │
│  Final Artifacts:                                                          │
│  ┌────────────────────────────────────────────────────────────┐           │
│  │ build/                                                      │           │
│  │ ├── pluto.frm                      (~11MB)                 │           │
│  │ ├── pluto.dfu                      (~11MB)                 │           │
│  │ ├── boot.frm                       (~572KB)                │           │
│  │ ├── boot.dfu                       (~443KB)                │           │
│  │ ├── uboot-env.dfu                  (~129KB)                │           │
│  │ ├── plutosdr-fw-v0.39.zip          (~23MB)                 │           │
│  │ ├── plutosdr-jtag-bootstrap-v0.39.zip (~2MB)              │           │
│  │ └── legal-info-v0.39.tar.gz        (~475MB)               │           │
│  └────────────────────────────────────────────────────────────┘           │
│                                                                            │
│  Duration: ~10 seconds                                                     │
└────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
                          ┌─────────────────┐
                          │  BUILD COMPLETE │
                          └─────────────────┘
```

## Build Time Summary

```
┌──────────────────────────────────────────────────────────────┐
│               Total Build Time Breakdown                     │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  WITH VIVADO (FPGA synthesis):                              │
│  ═══════════════════════════════════════════════════════════│
│  Toolchain Bootstrap    [█████] ~5 minutes                  │
│  U-Boot Build           [███] ~3 minutes                    │
│  Linux Kernel Build     [███████████████] ~15 minutes       │
│  FPGA Synthesis         [████████████████████████████████]  │
│                         ~60-120 minutes                      │
│  Buildroot              [███████████████] ~30 minutes       │
│  Packaging              [█] ~1 minute                        │
│  ─────────────────────────────────────────────────────────  │
│  TOTAL: ~2-2.5 hours                                         │
│                                                              │
│  WITHOUT VIVADO (pre-built XSA):                            │
│  ═══════════════════════════════════════════════════════════│
│  Toolchain Bootstrap    [█████] ~5 minutes                  │
│  U-Boot Build           [███] ~3 minutes                    │
│  Linux Kernel Build     [███████████████] ~15 minutes       │
│  XSA Download           [█] ~30 seconds                     │
│  Buildroot              [███████████████] ~30 minutes       │
│  Packaging              [█] ~1 minute                        │
│  ─────────────────────────────────────────────────────────  │
│  TOTAL: ~55 minutes                                          │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

## Parallel Build Optimization

The build system uses parallel compilation where possible:

```
┌──────────────────────────────────────────────────────────────┐
│         Parallel vs Sequential Build Phases                  │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  SEQUENTIAL PHASES:                                          │
│  (Must run in order)                                         │
│  ┌──────────────┐                                            │
│  │ Toolchain    │ Cannot parallelize                        │
│  └──────┬───────┘                                            │
│         │                                                    │
│         ▼                                                    │
│  ┌──────────────────────────────────────┐                   │
│  │ U-Boot │ Linux │ FPGA  │ Buildroot  │ Can run parallel  │
│  └──────────────────────────────────────┘ (independent)     │
│         │                                                    │
│         ▼                                                    │
│  ┌──────────────┐                                            │
│  │ FIT Image    │ Requires all inputs                       │
│  └──────┬───────┘                                            │
│         │                                                    │
│         ▼                                                    │
│  ┌──────────────┐                                            │
│  │ Packaging    │ Final step                                │
│  └──────────────┘                                            │
│                                                              │
│  INTERNAL PARALLELISM:                                       │
│  • Linux kernel: make -j $(NCORES)                           │
│  • Buildroot: make -j $(NCORES)                              │
│  • U-Boot: Sequential (not parallelized in Makefile)         │
│  • FPGA: Vivado internal parallelism                         │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

## Incremental Build Support

```
Makefile Target Dependencies:
═════════════════════════════

build/pluto.frm:
  ├─ build/pluto.itb
  │   ├─ build/zImage (rebuilds if linux sources change)
  │   ├─ build/rootfs.cpio.gz (rebuilds if buildroot changes)
  │   ├─ build/*.dtb (rebuilds if device tree changes)
  │   └─ build/system_top.bit (rebuilds if HDL changes)
  ├─ u-boot-xlnx/tools/mkimage (rebuilds if u-boot changes)
  └─ scripts/pluto.its (rebuilds if ITS file changes)

Make automatically detects which components need rebuilding based on
file timestamps. Only modified components are recompiled.
```

## Build Artifacts Size Analysis

```
┌──────────────────────────────────────────────────────────────┐
│                  Component Size Breakdown                    │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  FPGA Bitstream      [████████] 943 KB (8.5%)               │
│  Linux Kernel        [████████████████████] 4.1 MB (37%)    │
│  Root Filesystem     [██████████████████████████] 5.3 MB    │
│                      (48%)                                   │
│  Device Trees        [█] 69 KB (3 files) (0.6%)             │
│  U-Boot              [███████] 761 KB (6.9%)                │
│  ────────────────────────────────────────────────────────   │
│  Total FIT Image: ~11 MB                                     │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

## Related Documentation

- [Makefile System](01-makefile-system.md)
- [Toolchain](02-toolchain.md)
- [Firmware Packaging](04-firmware-packaging.md)
- [Components Overview](../components/)
