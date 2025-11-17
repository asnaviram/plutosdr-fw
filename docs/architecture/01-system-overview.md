# PlutoSDR Firmware - System Overview

## Introduction

The PlutoSDR firmware is a complete embedded Linux system for Software-Defined Radio (SDR) platforms built on the Xilinx Zynq-7000 SoC. This document provides a high-level architectural overview of the entire system.

## System Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        PLUTOSDR FIRMWARE SYSTEM                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌────────────────────────┐         ┌───────────────────────────────┐  │
│  │   HOST COMPUTER        │         │     PLUTOSDR DEVICE          │  │
│  │                        │         │                               │  │
│  │  ┌──────────────────┐  │   USB   │  ┌────────────────────────┐  │  │
│  │  │  Linux/Windows   │  │◄────────┼─►│   USB Composite Device │  │  │
│  │  │  MacOS           │  │         │  │  - Network (RNDIS/NCM) │  │  │
│  │  └──────────────────┘  │         │  │  - Mass Storage        │  │  │
│  │                        │         │  │  - Serial Console      │  │  │
│  │  ┌──────────────────┐  │         │  │  - IIO Control         │  │  │
│  │  │  IIO Libraries   │  │         │  └────────────────────────┘  │  │
│  │  │  - libiio        │  │         │              │               │  │
│  │  │  - libad9361     │  │         │              ▼               │  │
│  │  └──────────────────┘  │         │  ┌────────────────────────┐  │  │
│  │                        │         │  │    LINUX KERNEL        │  │  │
│  └────────────────────────┘         │  │    (4.9.0 LTS)         │  │  │
│                                      │  │                        │  │  │
│                                      │  │  ┌──────────────────┐  │  │  │
│                                      │  │  │  IIO Subsystem   │  │  │  │
│                                      │  │  │  - AD9361 Driver │  │  │  │
│                                      │  │  │  - AXI DMA       │  │  │  │
│                                      │  │  └──────────────────┘  │  │  │
│                                      │  └────────────────────────┘  │  │
│                                      │              │               │  │
│                                      │              ▼               │  │
│                                      │  ┌────────────────────────┐  │  │
│                                      │  │   ZYNQ-7000 SOC        │  │  │
│                                      │  │                        │  │  │
│                                      │  │  ┌─────────────────┐   │  │  │
│                                      │  │  │ ARM Cortex-A9   │   │  │  │
│                                      │  │  │ Dual Core (PS)  │   │  │  │
│                                      │  │  └─────────────────┘   │  │  │
│                                      │  │          │  ▲          │  │  │
│                                      │  │          ▼  │          │  │  │
│                                      │  │  ┌─────────────────┐   │  │  │
│                                      │  │  │  AXI/AMBA Bus   │   │  │  │
│                                      │  │  └─────────────────┘   │  │  │
│                                      │  │          │  ▲          │  │  │
│                                      │  │          ▼  │          │  │  │
│                                      │  │  ┌─────────────────┐   │  │  │
│                                      │  │  │ FPGA Fabric     │   │  │  │
│                                      │  │  │ (PL)            │   │  │  │
│                                      │  │  │ - ADC Core      │   │  │  │
│                                      │  │  │ - DAC Core      │   │  │  │
│                                      │  │  │ - DMA Engines   │   │  │  │
│                                      │  │  └─────────────────┘   │  │  │
│                                      │  └────────────────────────┘  │  │
│                                      │              │               │  │
│                                      │              ▼               │  │
│                                      │  ┌────────────────────────┐  │  │
│                                      │  │   AD9363/AD9364        │  │  │
│                                      │  │   RF Transceiver       │  │  │
│                                      │  │   70MHz - 6GHz         │  │  │
│                                      │  └────────────────────────┘  │  │
│                                      └───────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
```

## Major Components

### 1. Hardware Platform

**Xilinx Zynq-7000 SoC (Z7010)**
- **Processing System (PS)**: Dual-core ARM Cortex-A9 @ 667 MHz
- **Programmable Logic (PL)**: FPGA fabric for custom hardware acceleration
- **Memory**: 512MB DDR3 RAM
- **Storage**: 32MB QSPI Flash

**RF Frontend**
- **PlutoSDR**: AD9363 (1x1 transceiver, 325-3800 MHz)
- **SidekiqZ2**: AD9364 (1x1 transceiver, 70 MHz - 6 GHz)

### 2. Software Stack

```
┌─────────────────────────────────────────────────────┐
│                 Application Layer                   │
│  - Web Interface                                    │
│  - Configuration Tools                              │
│  - Calibration Utilities                            │
└─────────────────────────────────────────────────────┘
                        │
┌─────────────────────────────────────────────────────┐
│              User Space Services                    │
│  - SSH Server (Dropbear)                            │
│  - IIO Daemon (iiod)                                │
│  - mDNS/DNS-SD (Avahi)                              │
│  - USB Gadget Manager                               │
└─────────────────────────────────────────────────────┘
                        │
┌─────────────────────────────────────────────────────┐
│           Linux Kernel (4.9.0 LTS)                  │
│  - IIO Framework                                    │
│  - AD9361/AD9364 Driver                             │
│  - USB Gadget Framework                             │
│  - AXI DMA Driver                                   │
│  - Device Tree Support                              │
└─────────────────────────────────────────────────────┘
                        │
┌─────────────────────────────────────────────────────┐
│              U-Boot Bootloader                      │
│  - FIT Image Support                                │
│  - DFU Mode                                         │
│  - FPGA Loading                                     │
└─────────────────────────────────────────────────────┘
                        │
┌─────────────────────────────────────────────────────┐
│         First Stage Boot Loader (FSBL)              │
│  - PS7 Initialization                               │
│  - DDR Training                                     │
│  - Clock Configuration                              │
└─────────────────────────────────────────────────────┘
```

### 3. Build System Components

The firmware is built from four main Git submodules:

1. **HDL** (hdl_2018_r1 branch)
   - FPGA designs for PlutoSDR and SidekiqZ2
   - Vivado project files
   - Bitstream generation

2. **Linux** (2018_R1 branch)
   - Custom kernel with ADI drivers
   - Device tree sources
   - IIO subsystem enhancements

3. **U-Boot-XLNX** (pluto branch)
   - Xilinx U-Boot fork
   - PlutoSDR-specific patches
   - FIT image support

4. **Buildroot**
   - Root filesystem generation
   - Package management
   - Cross-compilation toolchain

## Data Flow Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                         RF SIGNAL PATH                           │
└──────────────────────────────────────────────────────────────────┘

  Antenna                AD9363/4              FPGA                DDR3
    │                    Transceiver           Fabric              Memory
    │                                                                 │
    ▼                                                                 │
  ┌───┐                 ┌─────────┐                                  │
  │RF │──────────RX────►│ ADC     │                                  │
  │   │                 │  Path   │                                  │
  │   │                 └─────────┘                                  │
  │   │                      │                                       │
  │   │                      │ Digital I/Q                           │
  │   │                      ▼                                       │
  │   │                 ┌─────────────────┐                          │
  │   │                 │  ADC Core       │                          │
  │   │                 │  - Decimation   │                          │
  │   │                 │  - Filtering    │                          │
  │   │                 └─────────────────┘                          │
  │   │                      │                                       │
  │   │                      │ AXI Stream                            │
  │   │                      ▼                                       │
  │   │                 ┌─────────────────┐        AXI Bus          │
  │   │                 │  RX DMA Engine  │◄────────────────────────┤
  │   │                 │  (axi-dmac)     │                         │
  │   │                 └─────────────────┘                         │
  │   │                      │                                      │
  │   │                      │ Writes RX samples                    │
  │   │                      ▼                                      │
  │   │                 ┌─────────────────────────────────────────┐│
  │   │                 │         DDR3 Memory Buffer              ││
  │   │                 │  (Accessed via IIO buffer interface)    ││
  │   │                 └─────────────────────────────────────────┘│
  │   │                      ▲                                      │
  │   │                      │ Reads TX samples                     │
  │   │                      │                                      │
  │   │                 ┌─────────────────┐                         │
  │   │                 │  TX DMA Engine  │◄────────────────────────┘
  │   │                 │  (axi-dmac)     │
  │   │                 └─────────────────┘
  │   │                      │
  │   │                      │ AXI Stream
  │   │                      ▼
  │   │                 ┌─────────────────┐
  │   │                 │  DAC Core       │
  │   │                 │  - Interpolation│
  │   │                 │  - Filtering    │
  │   │                 └─────────────────┘
  │   │                      │
  │   │                      │ Digital I/Q
  │   │                      ▼
  │   │                 ┌─────────┐
  │   │◄───────TX───────│ DAC     │
  │   │                 │  Path   │
  └───┘                 └─────────┘
```

## Flash Memory Organization

```
┌─────────────────────────────────────────────┐
│      32MB QSPI Flash Memory Layout          │
├─────────────────────────────────────────────┤
│                                              │
│  0x00000000 ┌──────────────────────────┐    │
│             │  mtd0: boot              │    │
│             │  FSBL + U-Boot           │    │
│             │  Size: 1MB               │    │
│  0x00100000 ├──────────────────────────┤    │
│             │  mtd1: uboot-env         │    │
│             │  U-Boot Environment      │    │
│             │  Size: 128KB             │    │
│  0x00120000 ├──────────────────────────┤    │
│             │  mtd2: nvmfs             │    │
│             │  Persistent Storage      │    │
│             │  (JFFS2 filesystem)      │    │
│             │  Size: 896KB             │    │
│  0x00200000 ├──────────────────────────┤    │
│             │  mtd3: linux             │    │
│             │  FIT Image:              │    │
│             │  - FPGA Bitstream        │    │
│             │  - Linux Kernel          │    │
│             │  - Device Trees          │    │
│             │  - Root Filesystem       │    │
│             │  Size: 30MB              │    │
│  0x02000000 └──────────────────────────┘    │
│                                              │
└─────────────────────────────────────────────┘
```

## Update Mechanisms

The firmware supports three update methods:

### 1. USB Mass Storage Device (MSD) Update
```
User Action              PlutoSDR Device
    │                          │
    │  Copy pluto.frm          │
    ├─────────────────────────►│
    │  to MSD drive            │
    │                          │
    │                          ▼
    │                    ┌──────────┐
    │                    │update.sh │
    │                    │daemon    │
    │                    └──────────┘
    │                          │
    │                          │ Validates MD5
    │                          │ Checks magic string
    │                          │
    │                          ▼
    │                    ┌──────────┐
    │                    │Flash to  │
    │                    │mtd3      │
    │                    └──────────┘
    │                          │
    │                          ▼
    │                    ┌──────────┐
    │◄───────────────────│ Reboot   │
    │  Device reboots    └──────────┘
```

### 2. DFU (Device Firmware Update) Mode
```
Host Computer            PlutoSDR (DFU Mode)
    │                          │
    │  dfu-util command        │
    ├─────────────────────────►│
    │  -D pluto.dfu            │
    │                          │
    │                          ▼
    │                    ┌──────────┐
    │                    │U-Boot DFU│
    │                    │Handler   │
    │                    └──────────┘
    │                          │
    │  USB Transfer            │
    │◄────────────────────────►│
    │  (firmware blocks)       │
    │                          │
    │                          ▼
    │                    ┌──────────┐
    │                    │Write to  │
    │                    │Flash     │
    │                    └──────────┘
```

### 3. JTAG Programming
```
Xilinx Tools             PlutoSDR Hardware
    │                          │
    │  XSDB/XMD                │
    ├─────────────────────────►│
    │  + ps7_init.tcl          │
    │  + u-boot.elf            │
    │                          │
    │                          ▼
    │                    ┌──────────┐
    │                    │JTAG      │
    │                    │Interface │
    │                    └──────────┘
    │                          │
    │                          ▼
    │                    ┌──────────┐
    │                    │Load to   │
    │                    │DDR RAM   │
    │                    └──────────┘
```

## Build Artifact Flow

```
Source Code Repositories
┌────────┬────────┬─────────┬──────────┐
│  HDL   │ Linux  │ U-Boot  │Buildroot │
└───┬────┴────┬───┴────┬────┴─────┬────┘
    │         │        │          │
    │         │        │          │
    ▼         ▼        ▼          ▼
┌────────────────────────────────────┐
│         Make Build System          │
│     (Orchestrated by Makefile)     │
└────────────────┬───────────────────┘
                 │
     ┌───────────┼───────────┐
     │           │           │
     ▼           ▼           ▼
┌─────────┐ ┌─────────┐ ┌─────────┐
│ FPGA    │ │ Linux   │ │ Root    │
│Bitstream│ │ zImage  │ │Filesystem│
│  .bit   │ │         │ │  .cpio  │
└────┬────┘ └────┬────┘ └────┬────┘
     │           │           │
     │           │           │
     └───────────┼───────────┘
                 │
                 │ U-Boot mkimage
                 │ (FIT format)
                 ▼
         ┌───────────────┐
         │  pluto.itb    │
         │ (FIT Image)   │
         └───────┬───────┘
                 │
         ┌───────┴───────┐
         │               │
         ▼               ▼
   ┌─────────┐     ┌─────────┐
   │pluto.frm│     │pluto.dfu│
   │(+MD5)   │     │(+DFU    │
   │         │     │ suffix) │
   └─────────┘     └─────────┘
```

## Key Technologies

- **SoC**: Xilinx Zynq-7000 (ARM + FPGA)
- **RF**: Analog Devices AD9363/AD9364 Transceivers
- **OS**: Linux 4.9.0 LTS
- **Bootloader**: U-Boot (Xilinx fork)
- **Build System**: GNU Make + Buildroot
- **FPGA Tools**: Xilinx Vivado 2023.2
- **Toolchain**: Linaro GCC 7.3-2018.05
- **IIO Framework**: Industrial I/O for SDR control

## Target Platforms

### PlutoSDR (ADALM-PLUTO)
- USB VID:PID = 0x0456:0xb673
- Frequency Range: 325 - 3800 MHz
- Hardware Revisions: A, B, C

### SidekiqZ2
- USB VID:PID = 0x2fa2:0x5a02
- Frequency Range: 70 MHz - 6 GHz
- Hardware Revision: B

## Documentation Map

For detailed information on specific topics, refer to:

- **Architecture**: Detailed hardware and software architecture
- **Components**: In-depth component analysis
- **Build System**: Build process and toolchain details
- **Workflows**: Development, testing, and release procedures
- **Hardware**: Platform-specific hardware documentation
- **Scripts**: Build and utility script reference
- **CI/CD**: Continuous integration and deployment
