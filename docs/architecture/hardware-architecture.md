# Hardware Architecture

## Xilinx Zynq-7010 Overview

The PlutoSDR is built on the Xilinx Zynq-7010, a hybrid FPGA+ARM device combining:
- Dual ARM Cortex-A9 processors in the Processing System (PS)
- FPGA fabric in the Programmable Logic (PL)
- Tight integration between PS and PL via AXI interconnect

```
┌──────────────────────────────────────────────────────────┐
│            Xilinx Zynq-7010 (xc7z010clg225-1)           │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  ┌─────────────────────────────────────────────────┐   │
│  │       ARM Processing System (PS)                │   │
│  │                                                 │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────────┐ │   │
│  │  │ Cortex   │  │ Cortex   │  │   Cache &    │ │   │
│  │  │  A9 #0   │  │  A9 #1   │  │   Memory     │ │   │
│  │  │ 667 MHz  │  │ 667 MHz  │  │  Controller  │ │   │
│  │  └────┬─────┘  └────┬─────┘  └──────────────┘ │   │
│  │       │             │                          │   │
│  │  ┌────┴─────────────┴────────────────────────┐ │   │
│  │  │       L2 Cache (512 KB Shared)            │ │   │
│  │  └────┬──────────────────────────────────────┘ │   │
│  │       │                                       │   │
│  │  ┌────┴──────────────────────────────────────┐ │   │
│  │  │   DDR3 Memory Controller (512 MB)         │ │   │
│  │  └────┬──────────────────────────────────────┘ │   │
│  │       │                                       │   │
│  │  ┌────┴────┬─────┬──────┬────┬──────┬──────┐│   │
│  │  │QSPI │I2C │ SPI │ UART│GPIO│USB  │Others││   │
│  │  │Flash│    │     │     │    │ OTG │      ││   │
│  │  └─────┴─────┴──────┴────┴──────┴──────┘│   │
│  │                                          │   │
│  └──────────────────────────────────────────┘   │
│                     │                           │
│                     │ AXI Master/Slave          │
│                     ▼                           │
│  ┌──────────────────────────────────────────┐  │
│  │    FPGA Fabric / Programmable Logic      │  │
│  │                                          │  │
│  │  ┌────────────────────────────────────┐ │  │
│  │  │    AXI Interconnect                │ │  │
│  │  │  (Controls IP communication)       │ │  │
│  │  └────────┬───────────────────────────┘ │  │
│  │           │                             │  │
│  │  ┌────────┴──────────────────────────┐ │  │
│  │  │   AD9361 RF Transceiver Interface │ │  │
│  │  │                                  │ │  │
│  │  │  ┌───────────┐   ┌─────────────┐ │ │  │
│  │  │  │    RX     │   │     TX      │ │ │  │
│  │  │  │  Channel  │   │   Channel   │ │ │  │
│  │  │  └─────┬─────┘   └──────┬──────┘ │ │  │
│  │  │        │                │        │ │  │
│  │  │  ┌─────▼────────────────▼─────┐ │ │  │
│  │  │  │  FIR Decimator (RX)        │ │ │  │
│  │  │  │  FIR Interpolator (TX)     │ │ │  │
│  │  │  └─────┬────────────────┬─────┘ │ │  │
│  │  │        │                │       │ │  │
│  │  │  ┌─────▼────────────────▼─────┐ │ │  │
│  │  │  │   DMA Controllers           │ │ │  │
│  │  │  │  (HP1: ADC, HP2: DAC)      │ │ │  │
│  │  │  └─────┬────────────────┬─────┘ │ │  │
│  │  │        │                │       │ │  │
│  │  │  ┌─────▼────────────────▼─────┐ │ │  │
│  │  │  │   Clock Generators          │ │ │  │
│  │  │  │  FPGA0: 100 MHz             │ │ │  │
│  │  │  │  FPGA1: 200 MHz             │ │ │  │
│  │  │  └─────────────────────────────┘ │ │  │
│  │  │                                  │ │  │
│  │  └──────────────────────────────────┘ │  │
│  │                                        │  │
│  └────────────────────────────────────────┘  │
│                                              │
└──────────────────────────────────────────────┘
```

## Memory Hierarchy

### System Memory Organization

```
┌───────────────────────────────────────────┐
│        DDR3 SDRAM (512 MB)               │
│  Address: 0x00000000 - 0x1FFFFFFF        │
├───────────────────────────────────────────┤
│  │                                       │
│  ├─ 0x00000000: Linux Kernel             │
│  │                                       │
│  ├─ 0x04000000: Device Tree              │
│  │                                       │
│  ├─ 0x08000000: Root Filesystem          │
│  │                                       │
│  ├─ 0x10000000: IIO Driver Memory        │
│  │                                       │
│  ├─ 0x14000000: User Application Space   │
│  │                                       │
│  └─ 0x1FFFFFFF: (End of DDR3)            │
│                                         │
└───────────────────────────────────────────┘

┌───────────────────────────────────────────┐
│     QSPI Flash Memory (32 MB)            │
│  Address: 0x00000000 - 0x01FFFFFF        │
├───────────────────────────────────────────┤
│  Partition  │ Start   │ End     │ Size   │
│─────────────┼─────────┼─────────┼────────│
│ mtd0        │ 0x00000 │ 0x100FF │ 1 MB   │
│ (FSBL+U-B)  │         │         │        │
├─────────────┼─────────┼─────────┼────────┤
│ mtd1        │ 0x10000 │ 0x1FFFF │ 128 KB │
│ (U-Boot Env)│         │         │        │
├─────────────┼─────────┼─────────┼────────┤
│ mtd2        │ 0x20000 │ 0x1FFFFF│ 896 KB │
│ (NVMFS)     │         │         │        │
├─────────────┼─────────┼─────────┼────────┤
│ mtd3        │ 0x200000│ 0x1FFFFFF 30 MB  │
│ (Linux+RFS) │         │         │        │
└─────────────┴─────────┴─────────┴────────┘
```

## AXI Interconnect Architecture

```
┌─────────────────────────────────────────────────────────────┐
│               AXI Interconnect (FPGA Fabric)               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   ┌─────────────────────────────────────────────────────┐  │
│   │          ARM Cortex-A9 (AXI Master)                │  │
│   │  (CPU, accessing IP cores via L1 interconnect)     │  │
│   └────────────────┬────────────────────────────────────┘  │
│                    │                                       │
│   ┌────────────────▼────────────────────────────────────┐  │
│   │        Central Interconnect Matrix                  │  │
│   │  (Routes AXI transactions between masters/slaves)  │  │
│   └──┬─────┬─────┬──────┬────────┬──────────────────────┘  │
│      │     │     │      │        │                        │
│      ▼     ▼     ▼      ▼        ▼                        │
│   ┌────┐┌───┐┌─────┐┌──────┐┌────────────────────────┐  │
│   │AD9 ││FIR││DMA  ││CLK   ││    Other IP Cores    │  │
│   │361 ││   ││Ctrls││Gen   ││  (GPIO, Interrupts)  │  │
│   └────┘└───┘└─────┘└──────┘└────────────────────────┘  │
│                                                             │
│   ┌────────────────────────────────────────────────────┐  │
│   │        DDR3 Memory Port (AXI Slave)               │  │
│   │  HP0: ARM processor (read/write)                   │  │
│   │  HP1: ADC DMA engine (write only)                 │  │
│   │  HP2: DAC DMA engine (read only)                  │  │
│   └────────────────────────────────────────────────────┘  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## Clock Distribution

```
┌───────────────────────────────────────────────────────────┐
│             Clock Distribution Network                    │
├───────────────────────────────────────────────────────────┤
│                                                           │
│   ┌─────────────────────────────────────────┐           │
│   │    ARM Processing System Clock (PS)     │           │
│   │          (Fixed 667 MHz)                │           │
│   └────┬────────────────────────────────────┘           │
│        │                                                │
│   ┌────▼──────────────────────────────────────────┐    │
│   │        Clock Generator (util_clkgen)          │    │
│   │     (Fractional-N Frequency Synthesis)        │    │
│   └────┬──────┬──────────────────────────────────┘    │
│        │      │                                       │
│        ▼      ▼                                       │
│   ┌────────┐┌──────────┐                             │
│   │FPGA0   ││ FPGA1    │                             │
│   │100 MHz ││ 200 MHz  │                             │
│   └───┬────┘└───┬──────┘                             │
│       │         │                                    │
│       ▼         ▼                                    │
│   ┌──────────────────────────────────────────────┐  │
│   │  RF Transceiver Clock (AD9361)               │  │
│   │  Derived from FPGA0, ~61.5 MHz               │  │
│   │  (Synchronized to AD9361 clock input)        │  │
│   └──────────────────────────────────────────────┘  │
│                                                      │
│   Clock Domain Crossings:                           │
│   • FPGA0 ↔ FPGA1: Via async FIFOs               │
│   • FPGA1 ↔ RF Clock: CDC (Clock Domain Crossing)│
│   • PS Clock ↔ FPGA0: Via PS7 MMR control        │
│                                                     │
└───────────────────────────────────────────────────────┘
```

## RF Transceiver Interface

### AD9361 Pin Mapping

```
AD9361 (Analog Devices RF Transceiver)
│
├─ RX Data Path (12-bit parallel)
│  └─ Pins: L12, M13, M14, M15, N12, N13, N14, N15, P12, P13, P14, P15
│     Signal: rx_d[0:11] @ ~61.5 MHz
│
├─ RX Control Signals
│  ├─ rx_clk (L12)  - Input clock
│  └─ rx_frame (N13) - Frame synchronization
│
├─ TX Data Path (12-bit parallel)
│  └─ Pins: R7, T7, T8, T9, T10, T14, T15, U9, U10, U13, U14, U15
│     Signal: tx_d[0:11] @ ~61.5 MHz
│
├─ TX Control Signals
│  ├─ tx_clk (P10) - Output clock
│  └─ tx_frame (L14) - Frame synchronization
│
├─ GPIO Control Lines (4-bit)
│  ├─ gpio_ctl[0] → Control line 0
│  ├─ gpio_ctl[1] → Control line 1
│  ├─ gpio_ctl[2] → Control line 2
│  └─ gpio_ctl[3] → Control line 3
│
├─ GPIO Status Lines (8-bit)
│  └─ gpio_status[0:7] → Transceiver status bits
│
├─ Power Control
│  ├─ gpio_resetb → Active-low reset (P9)
│  └─ gpio_en_agc → Enable Automatic Gain Control (L13)
│
└─ Configuration Interfaces
   ├─ SPI Interface
   │  ├─ spi_clk (E11) - Clock
   │  ├─ spi_mosi (E13) - Master out, slave in
   │  ├─ spi_miso (F12) - Master in, slave out
   │  └─ spi_csn (E12) - Chip select (active low)
   │
   └─ I2C Interface
      ├─ iic_scl (M14) - Serial clock (with pullup)
      └─ iic_sda (N14) - Serial data (with pullup)
```

### AD9361 Tuning Ranges

```
┌────────────────────────────────────────────────────┐
│        AD9361 RF Transceiver Specifications        │
├────────────────────────────────────────────────────┤
│                                                    │
│  Frequency Range:      70 MHz - 6 GHz             │
│                                                    │
│  RX Specifications:                                │
│  • Gain Range:         -12 dB to +73 dB           │
│  • Noise Figure:       ~3 dB @ 2.4 GHz            │
│  • Input Impedance:    50 Ω                       │
│  • Filter Types:       Automatic, lowpass         │
│                                                    │
│  TX Specifications:                                │
│  • Power Range:        -20 dBm to +10 dBm         │
│  • Output Impedance:   50 Ω                       │
│  • Efficiency:         High power with linearity  │
│                                                    │
│  Bandwidth Modes:                                  │
│  • Minimum:            200 kHz                     │
│  • Maximum:            56 MHz                      │
│  • Programmable:       56 MHz to 200 kHz           │
│                                                    │
│  Sample Rates:                                     │
│  • Minimum:            2.084 MSps                  │
│  • Maximum:            61.44 MSps                  │
│                                                    │
│  ADC/DAC Resolution:   12-bit dual-channel        │
│                                                    │
│  Operating Modes:                                  │
│  • Full-Duplex (TDD)   Time-division duplex       │
│  • Half-Duplex         Switchable RX/TX            │
│                                                    │
└────────────────────────────────────────────────────┘
```

## DDR3 Memory Organization

```
Task Memory Layout:
┌─────────────────────────────┐
│ Kernel Space (High Addresses)
│ 0xFFFFFFFF                  │
├─────────────────────────────┤
│ Kernel Code & Data          │
│ (Loaded at boot)            │
├─────────────────────────────┤
│ (Reserved/Swapable Space)   │
├─────────────────────────────┤
│ User Application Space      │
│ (IIO, userspace drivers)    │
├─────────────────────────────┤
│ DMA Buffers                 │
│ (RX/TX ring buffers)        │
├─────────────────────────────┤
│ Root Filesystem             │
│ (CPIO archive loaded)       │
├─────────────────────────────┤
│ Device Tree Blob            │
│ (Hardware description)      │
├─────────────────────────────┤
│ Linux Kernel (zImage)       │
│ Decompressed at boot        │
├─────────────────────────────┤
│ 0x00000000                  │
│ (Low Addresses)             │
└─────────────────────────────┘
```

## Power Supply and Management

```
┌─────────────────────────────────────────────┐
│         Power Domains (Zynq-7010)          │
├─────────────────────────────────────────────┤
│                                             │
│  VCCINT (1.0V)                              │
│  • Zynq core logic (PS + PL)               │
│  • Requires low ripple, high quality       │
│                                             │
│  VCCAUX (1.8V)                              │
│  • I/O banks, PLL supply                   │
│  • Analog circuits                         │
│                                             │
│  VCCO (3.3V or custom)                      │
│  • I/O bank standards                      │
│  • PlutoSDR uses LVCMOS18 and SSTL15      │
│                                             │
│  VREF (1.2V)                                │
│  • DDR3 reference voltage                  │
│  • Termination reference                   │
│                                             │
│  AD9361 Supplies:                           │
│  • 1.3V, 2.5V, 3.3V (multiple rails)      │
│  • RF supply isolation required            │
│                                             │
└─────────────────────────────────────────────┘
```

## Related Documentation

- [System Overview](./system-overview.md) - High-level architecture
- [FPGA Design](./fpga-design.md) - FPGA implementation details
- [Boot Sequence](../api-reference/boot-sequence.md) - Hardware initialization
