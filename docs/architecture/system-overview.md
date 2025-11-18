# PlutoSDR System Overview

## Executive Summary

PlutoSDR is an embedded Software Defined Radio (SDR) platform built on the Xilinx Zynq-7010 FPGA/ARM hybrid device. It combines dual ARM Cortex-A9 processors with FPGA fabric to create a compact, portable RF transceiver supporting frequencies from 70 MHz to 6 GHz.

## System Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                         USB Host Interface                       │
│  (DFU Mode: Firmware Updates | SDR Mode: Normal Operation)      │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
              ┌──────────────────────────────┐
              │  Zynq-7010 Processing Unit   │
              │  (ARM Cortex-A9 × 2)        │
              │  512 MB DDR3 QSPI Flash     │
              └──────────────┬───────────────┘
                             │
            ┌────────────────┼────────────────┐
            │                │                │
            ▼                ▼                ▼
      ┌─────────────┐ ┌─────────────┐ ┌──────────────┐
      │  FPGA       │ │   U-Boot    │ │   Linux      │
      │  Fabric     │ │ Bootloader  │ │   Kernel     │
      │  (HDL)      │ │             │ │   + libiio   │
      └──────┬──────┘ └─────────────┘ └──────┬───────┘
             │                               │
      ┌──────┴───────────────────────────────┘
      │
      ▼
┌──────────────────────────────────────────────────┐
│        AD9361 RF Transceiver (Analog Devices)    │
│  • 70 MHz - 6 GHz tuning range                   │
│  • Dual RX/TX channels, 12-bit resolution        │
│  • 200 kHz - 56 MHz bandwidth                    │
│  • SPI interface for configuration               │
└──────────────┬───────────────────────────────────┘
               │
       ┌───────┴────────┐
       ▼                ▼
    ┌─────────┐    ┌─────────┐
    │ RX Path │    │ TX Path │
    │ (Input) │    │ (Output)│
    └─────────┘    └─────────┘
```

## Layered Architecture

```
┌─────────────────────────────────────────────────────────┐
│                   Application Layer                      │
│  (User Applications via IIO + libiio)                   │
└─────────────────────────────────────────────────────────┘
                         │
┌─────────────────────────────────────────────────────────┐
│                 Linux Kernel + Device Drivers            │
│  • AD9361 IIO ADC/DAC drivers                           │
│  • AXI DMA controllers                                  │
│  • Clock and GPIO management                           │
└─────────────────────────────────────────────────────────┘
                         │
┌─────────────────────────────────────────────────────────┐
│           FPGA Fabric + Memory Hierarchy                 │
│  • RF Signal Processing (FIR filters, decimation)       │
│  • DMA Engines for data movement                        │
│  • AXI Interconnect for IP communication               │
└─────────────────────────────────────────────────────────┘
                         │
┌─────────────────────────────────────────────────────────┐
│          Zynq ARM Processing System (PS)                 │
│  • DDR3 Memory (512 MB)                                │
│  • QSPI Flash (32 MB)                                  │
│  • GPIO, I2C, SPI, UART, USB controllers               │
└─────────────────────────────────────────────────────────┘
```

## Hardware Variants

### PlutoSDR

**USB Device IDs:**
- DFU Mode: 0x0456:0xb674
- SDR Mode: 0x0456:0xb673

**PCB Revisions:**
- Revision A: Original design
- Revision B: Enhanced design
- Revision C: Latest design

**Default Access:**
- IP Address: 192.168.2.1
- SSH User: root
- SSH Password: analog

### SidekiqZ2

**USB Device IDs:**
- DFU Mode: 0x2fa2:0x5a32
- SDR Mode: 0x2fa2:0x5a02

**Variants:**
- Revision B only

## Boot Flow Sequence

```
Step 1: Power-on Reset
        │
        ▼
Step 2: QSPI Flash Boot
        │
        ├─ FSBL (First-Stage Bootloader) from Flash
        │  └─ Initializes Zynq PS7 memory controller
        │
        ├─ U-Boot (Second-Stage Bootloader) from Flash
        │  └─ Loads environment variables
        │
        └─ FIT Image Assembly from Flash
           ├─ FPGA Bitstream
           ├─ Device Tree Blob
           ├─ Linux Kernel (zImage)
           └─ Root Filesystem (CPIO)
        │
        ▼
Step 3: FPGA Configuration
        │
        └─ Program system_top.bit to FPGA fabric
           ├─ Initialize AD9361 RF transceiver
           ├─ Setup DMA engines
           └─ Configure clock domains
        │
        ▼
Step 4: Linux Kernel Boot
        │
        ├─ Kernel initialization with device tree
        ├─ Load device drivers
        ├─ Mount root filesystem
        └─ Start init system
        │
        ▼
Step 5: User Applications Ready
        │
        └─ Device accessible via USB IIO interface
```

## Data Flow Paths

### RX (Receive/Input) Path

```
AD9361 RF Frontend
    │ (12-bit parallel I/Q data @ 61.5 MHz)
    ▼
axi_ad9361_rx
    │ (RX control and channel selection)
    ▼
axi_ad9361_rx_channel
    │ (Per-channel signal processing)
    ▼
util_fir_dec (FIR Decimator)
    │ (Sample rate reduction filter)
    ▼
axi_dmac (DMA Controller)
    │ (Memory transfer engine)
    ▼
DDR3 System Memory
    │
    ▼
Linux Kernel + User Application
```

### TX (Transmit/Output) Path

```
User Application
    │
    ▼
DDR3 System Memory
    │
    ▼
axi_dmac (DMA Controller)
    │ (Memory transfer engine)
    ▼
util_fir_int (FIR Interpolator)
    │ (Sample rate increase filter)
    ▼
axi_ad9361_tx_channel
    │ (Per-channel signal processing)
    ▼
axi_ad9361_tx
    │ (TX control and channel selection)
    ▼
AD9361 RF Frontend
    │ (12-bit parallel I/Q data @ 61.5 MHz)
    ▼
RF Output Connector
```

## Key Statistics

| Component | Specification |
|-----------|---------------|
| **FPGA Device** | Xilinx Zynq-7010 (xc7z010clg225-1) |
| **ARM Processors** | Dual Cortex-A9 @ 667 MHz |
| **FPGA LUTs** | 28,000 |
| **Block RAMs** | 240 |
| **DSP48E1 Slices** | 80 |
| **Memory** | 512 MB DDR3 |
| **Flash** | 32 MB QSPI |
| **RF Transceiver** | AD9361 |
| **Tuning Range** | 70 MHz - 6 GHz |
| **Bandwidth** | 200 kHz - 56 MHz |
| **Sample Rates** | 2.084 - 61.44 MSps |
| **USB Interface** | 2.0 High-Speed |

## Related Documentation

- [Hardware Architecture](./hardware-architecture.md) - Detailed hardware design
- [FPGA Design](./fpga-design.md) - FPGA architecture and signal processing
- [Build Process](../build-system/build-process.md) - How firmware is built
- [Boot Sequence](../api-reference/boot-sequence.md) - Detailed boot process
