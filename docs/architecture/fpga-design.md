# FPGA Design Architecture

## Top-Level Design Structure

The PlutoSDR FPGA design is organized as a hierarchical Vivado project with the following structure:

```
projects/pluto/
├── system_top.v               # Top-level Verilog wrapper
├── system_bd.tcl              # Block design (Vivado)
├── system_project.tcl         # Build automation
├── system_constr.xdc          # Pin assignments & timing
└── Makefile                   # Build targets

Generated Files:
├── system_top.xsa             # Xilinx Scalable Architecture (exported)
├── system_top.bit             # FPGA bitstream
└── system.sdk/                # Software Development Kit
    └── system_sys_ps7_0/      # PS7 initialization files
        ├── ps7_init.tcl
        ├── ps7_init.c
        └── ps7_init.h
```

## Block Design Architecture

The FPGA design uses Vivado Block Design (BD) to interconnect IP cores via the AXI protocol:

```
┌────────────────────────────────────────────────────────────────┐
│                    Vivado Block Design (system_bd)            │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │         Zynq PS7 (ARM Processing System)                 │ │
│  │                                                          │ │
│  │  • Dual-Core ARM Cortex-A9 @ 667 MHz                   │ │
│  │  • L1 Cache: 32 KB I+D per core                        │ │
│  │  • L2 Cache: 512 KB shared                             │ │
│  │  • DDR3 Memory Controller (32-bit @ 533 MHz)           │ │
│  │  • QSPI Flash Controller                               │ │
│  │  • GPIO, I2C, SPI, UART, USB controllers               │ │
│  │  • Interrupt concatenator (16 fabric interrupts)       │ │
│  │                                                          │ │
│  └──────────────────┬───────────────────────────────────────┘ │
│                     │ (AXI Master)                           │
│  ┌──────────────────▼───────────────────────────────────────┐ │
│  │          AXI Interconnect (Central Matrix)               │ │
│  │  • Routes AXI transactions between masters/slaves       │ │
│  │  • Supports multiple concurrent transfers              │ │
│  │  • Clock domain crossing support                        │ │
│  └──┬──────┬──────┬──────┬──────┬────────────────────────────┘ │
│     │      │      │      │      │                             │
│     ▼      ▼      ▼      ▼      ▼                             │
│  ┌────┐┌────┐┌───┐┌───┐┌────┐                               │
│  │AD9 ││CLK ││FIR││DMA││INT ├─ (Other IP cores)             │
│  │361 ││GEN ││   ││CTL││DIST│                               │
│  └────┘└────┘└───┘└───┘└────┘                               │
│     │      │      │      │      │                             │
│     └──────┴──────┴──────┴──────┘                             │
│              │                                               │
│  ┌───────────▼──────────────────────────────────────────┐   │
│  │      DDR3 Memory Interface (AXI Slave)               │   │
│  │  • HP0: ARM processor (read/write)                   │   │
│  │  • HP1: ADC DMA (write only)                        │   │
│  │  • HP2: DAC DMA (read only)                         │   │
│  │  • 32-bit data @ 533 MHz effective                  │   │
│  └───────────────────────────────────────────────────────┘   │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

## AD9361 RF Transceiver Integration

```
┌─────────────────────────────────────────────────────────────┐
│         axi_ad9361 (AD9361 AXI Interface)                   │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │           RX (Receive) Path                         │   │
│  │                                                     │   │
│  │  AD9361 RF Input                                   │   │
│  │    ▼                                                │   │
│  │  axi_ad9361_rx                                     │   │
│  │  (RX control, data routing, enable logic)          │   │
│  │    ▼                                                │   │
│  │  ┌────────────────────────────────────────────┐   │   │
│  │  │  axi_ad9361_rx_channel  (× 2 channels)     │   │   │
│  │  │  • Per-channel DSP processing              │   │   │
│  │  │  • Gain & phase correction                 │   │   │
│  │  │  • Data width conversion                   │   │   │
│  │  └────────────────────┬───────────────────────┘   │   │
│  │                       │                           │   │
│  │                       ▼                           │   │
│  │  util_fir_dec (FIR Decimator Filter)             │   │
│  │  • Sample rate reduction (e.g., 61.44 → 30 MSps) │   │
│  │  • Anti-aliasing filtering                       │   │
│  │    ▼                                              │   │
│  │  DMA Write Engine (HP1)                          │   │
│  │  • Burst transfers to DDR3                       │   │
│  │    ▼                                              │   │
│  │  User Application Memory                         │   │
│  │                                                     │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │           TX (Transmit) Path                        │   │
│  │                                                     │   │
│  │  User Application Memory                          │   │
│  │    ▼                                                │   │
│  │  DMA Read Engine (HP2)                            │   │
│  │  • Burst transfers from DDR3                      │   │
│  │    ▼                                                │   │
│  │  util_fir_int (FIR Interpolator Filter)           │   │
│  │  • Sample rate increase (e.g., 30 → 61.44 MSps)  │   │
│  │  • Reconstruction filtering                       │   │
│  │    ▼                                                │   │
│  │  ┌────────────────────────────────────────────┐   │   │
│  │  │  axi_ad9361_tx_channel  (× 2 channels)     │   │   │
│  │  │  • Per-channel DSP processing              │   │   │
│  │  │  • Gain & phase correction                 │   │   │
│  │  │  • Data width conversion                   │   │   │
│  │  └────────────────────┬───────────────────────┘   │   │
│  │                       │                           │   │
│  │                       ▼                           │   │
│  │  axi_ad9361_tx                                   │   │
│  │  (TX control, data routing, enable logic)        │   │
│  │    ▼                                              │   │
│  │  AD9361 RF Output                                │   │
│  │                                                     │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │        Control & Monitoring                        │   │
│  │                                                     │   │
│  │  axi_ad9361_tdd (Time-Division Duplex Controller) │   │
│  │  • Manages RX/TX mode switching                   │   │
│  │  • Optimizes full-duplex timing                   │   │
│  │    ▼                                                │   │
│  │  Configuration Registers (AXI Slave)              │   │
│  │  • Accessible from ARM via AXI bus                │   │
│  │  • Linux IIO driver control                       │   │
│  │                                                     │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## Signal Processing Pipeline

### Complete RX Signal Path

```
RF Antenna
   │
   ▼
AD9361 RX Frontend
   │ (12-bit I/Q @ 61.44 MSps)
   ▼
┌─────────────────────────────────────┐
│ axi_ad9361_rx                       │
│ • I/Q data selection                │
│ • Mixer control                     │
│ • Interface formatting              │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│ axi_ad9361_rx_channel (IQ Path)      │
│ • Channel filtering                 │
│ • Gain adjustment                   │
│ • Impedance matching                │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│ util_fir_dec (FIR Decimator)        │
│ • Polyphase FIR structure           │
│ • Configurable decimation ratio     │
│ • Example: 61.44 MSps → 30.72 MSps │
│ • Stopband attenuation: >80 dB      │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│ axi_dmac (DMA Controller - Write)    │
│ • Burst size: 256 bytes per beat    │
│ • Continuous operation mode         │
│ • Memory address increment          │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│ DDR3 Memory (Circular Buffer)        │
│ • Typical size: 64 MB for ring buff │
│ • HP1 write port (dedicated)        │
│ • Accessed by ARM via /dev/iio:*    │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│ Linux IIO Userspace Driver          │
│ • Reads from /dev/iio:device0       │
│ • Buffers data to application       │
│ • Exports to libIIO                 │
└─────────────────────────────────────┘
               │
               ▼
User Application (Python, C, etc.)
```

### Complete TX Signal Path

```
User Application (Python, C, etc.)
   │
   ▼
┌─────────────────────────────────────┐
│ Linux IIO Userspace Driver          │
│ • Writes to /dev/iio:device0        │
│ • Buffers application data          │
│ • Manages synchronization           │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│ DDR3 Memory (Circular Buffer)        │
│ • Typical size: 64 MB for ring buff │
│ • HP2 read port (dedicated)         │
│ • Refilled by ARM                   │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│ axi_dmac (DMA Controller - Read)     │
│ • Burst size: 256 bytes per beat    │
│ • Continuous operation mode         │
│ • Memory address increment          │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│ util_fir_int (FIR Interpolator)     │
│ • Polyphase FIR structure           │
│ • Configurable interpolation ratio  │
│ • Example: 30.72 MSps → 61.44 MSps │
│ • Image attenuation: >80 dB         │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│ axi_ad9361_tx_channel (IQ Path)      │
│ • Channel filtering                 │
│ • Gain adjustment                   │
│ • Impedance matching                │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│ axi_ad9361_tx                       │
│ • I/Q data multiplexing             │
│ • Mixer control                     │
│ • Interface formatting              │
└──────────────┬──────────────────────┘
               │
               ▼
AD9361 TX Frontend
   │ (12-bit I/Q @ 61.44 MSps)
   ▼
RF Antenna
```

## Clock Domain Distribution

```
┌──────────────────────────────────────────────────────┐
│         Clock Domain Architecture                    │
├──────────────────────────────────────────────────────┤
│                                                      │
│  PS System Clock (667 MHz)                           │
│    │                                                 │
│    ├─ CPU Clock (ARM)                               │
│    ├─ DDR3 Controller (533 MHz effective)            │
│    └─ Interconnect Clock                            │
│         │                                            │
│         ▼                                            │
│    ┌──────────────────────────────────┐             │
│    │  util_clkgen (Clock Generator)    │             │
│    │  (Fractional-N PLL in FPGA)       │             │
│    └───┬──────────────┬────────────────┘             │
│        │              │                             │
│        ▼              ▼                             │
│    FPGA0 Clock     FPGA1 Clock                       │
│    100 MHz         200 MHz                           │
│    │               │                                │
│    ├─ AXI Clock    ├─ High-speed Datapath          │
│    ├─ FIR Filters  ├─ DMA Engines                   │
│    └─ Slow IPs     └─ Fast Control Logic             │
│        │               │                            │
│        └───────┬───────┘                            │
│               │                                     │
│               ▼                                     │
│    ┌──────────────────────────────────┐             │
│    │  Clock Domain Crossing (CDC)      │             │
│    │  • Async FIFO in util_axis_fifo  │             │
│    │  • Gray code synchronizers       │             │
│    │  • Double-flop synchronizers     │             │
│    └──────────────────────────────────┘             │
│               │                                     │
│               ▼                                     │
│    RF Interface Clock (~61.5 MHz)                    │
│    • Derived from FPGA0                            │
│    • Synchronized to AD9361                        │
│    │                                                │
│    ├─ RX Clock Path                                │
│    ├─ TX Clock Path                                │
│    └─ Control/Status Synchronization               │
│                                                      │
└──────────────────────────────────────────────────────┘
```

## Signal Processing Filter Specifications

### FIR Decimator (RX Path)

```
┌─────────────────────────────────────────────────┐
│     FIR Decimator (util_fir_dec)                │
├─────────────────────────────────────────────────┤
│                                                 │
│  Input Rate:  61.44 MSps (from AD9361)         │
│  Output Rate: 30.72 MSps (typically)           │
│  Decimation: 2× (configurable)                 │
│                                                 │
│  Filter Characteristics:                        │
│  • Filter Type: Polyphase FIR                  │
│  • Order: ~127 taps                            │
│  • Passband Ripple: <0.01 dB                   │
│  • Stopband Atten: >80 dB                      │
│  • Transition Band: 20% of passband             │
│                                                 │
│  Architecture:                                  │
│  ┌──────────────────────────────────────────┐ │
│  │ Input Buffer (async FIFO)                │ │
│  │   ▼                                       │ │
│  │ Decimation Filter Logic                  │ │
│  │ (Parallel taps for high throughput)      │ │
│  │   ▼                                       │ │
│  │ Output MUX (every Nth sample)            │ │
│  │   ▼                                       │ │
│  │ Output Buffer (async FIFO)               │ │
│  └──────────────────────────────────────────┘ │
│                                                 │
│  Benefits:                                      │
│  • Anti-aliasing filter (prevents aliasing)   │
│  • Reduces data rate to host                  │
│  • Lowers host processing burden              │
│  • Fixed-point arithmetic (16 or 24 bits)    │
│                                                 │
└─────────────────────────────────────────────────┘
```

### FIR Interpolator (TX Path)

```
┌─────────────────────────────────────────────────┐
│     FIR Interpolator (util_fir_int)             │
├─────────────────────────────────────────────────┤
│                                                 │
│  Input Rate:  30.72 MSps (from host)           │
│  Output Rate: 61.44 MSps (to AD9361)           │
│  Interpolation: 2× (configurable)              │
│                                                 │
│  Filter Characteristics:                        │
│  • Filter Type: Polyphase FIR                  │
│  • Order: ~127 taps                            │
│  • Passband Ripple: <0.01 dB                   │
│  • Image Attenuation: >80 dB                   │
│  • Transition Band: 20% of original band       │
│                                                 │
│  Architecture:                                  │
│  ┌──────────────────────────────────────────┐ │
│  │ Input Buffer (async FIFO)                │ │
│  │   ▼                                       │ │
│  │ Zero Insertion (add N-1 zeros)           │ │
│  │   ▼                                       │ │
│  │ Interpolation Filter Logic               │ │
│  │ (Combs for efficient computation)        │ │
│  │   ▼                                       │ │
│  │ Output Buffer (async FIFO)               │ │
│  └──────────────────────────────────────────┘ │
│                                                 │
│  Benefits:                                      │
│  • Reconstruction filter (removes images)      │
│  • Increases data rate efficiently             │
│  • Reduces host-to-RF latency                 │
│  • Fixed-point arithmetic (16 or 24 bits)    │
│                                                 │
└─────────────────────────────────────────────────┘
```

## Memory and Performance

### DMA Memory Organization

```
┌────────────────────────────────────────┐
│      RX Circular Buffer (HP1)          │
│  Typical Size: 16-64 MB                │
│                                        │
│  ┌────────────────────────────────────┐
│  │ Ring Buffer Head (write by DMA)     │
│  │                                    │
│  │ [Sample 0] [Sample 1] ... [Sample N]
│  │                                    │
│  │ Ring Buffer Tail (read by host)    │
│  └────────────────────────────────────┘
│                                        │
│  Fill Condition:                       │
│  • Host slower than RMA → Buffer fills  │
│  • DMA detects full → Generates IRQ    │
│  • Host services interrupt             │
│  • Prevents data loss (overrun)        │
│                                        │
└────────────────────────────────────────┘

┌────────────────────────────────────────┐
│      TX Circular Buffer (HP2)          │
│  Typical Size: 16-64 MB                │
│                                        │
│  ┌────────────────────────────────────┐
│  │ Ring Buffer Head (read by DMA)      │
│  │                                    │
│  │ [Sample 0] [Sample 1] ... [Sample N]
│  │                                    │
│  │ Ring Buffer Tail (write by host)   │
│  └────────────────────────────────────┘
│                                        │
│  Empty Condition:                      │
│  • Host slower than DMA → Buffer empties│
│  • DMA detects empty → Generates IRQ   │
│  • Host refills buffer                 │
│  • Prevents underrun (silence/glitch)  │
│                                        │
└────────────────────────────────────────┘
```

## FPGA Resource Utilization

```
┌─────────────────────────────────────────┐
│    Zynq-7010 Resource Utilization      │
├─────────────────────────────────────────┤
│                                         │
│  Component          │ Utilized │ Total  │
│  ────────────────────┼─────────┼────────│
│  LUT Slices         │  ~18K   │  28K   │
│  BRAM (36Kb)        │  ~200   │  240   │
│  DSP48E1 Slices     │  ~60    │  80    │
│  BUFG Clocks        │  ~4     │  32    │
│                                         │
│  Major Consumers:                       │
│  • AD9361 Interface: 30% (FIFOs, MUX)  │
│  • FIR Filters: 40% (DSP48, LUT)       │
│  • DMA Controllers: 15% (FSM, FIFOs)   │
│  • Interconnect: 15% (Routing)         │
│                                         │
│  Timing Performance:                    │
│  • Clock Period: 10 ns (100 MHz FPGA0)│
│  • Slack Available: Comfortable         │
│  • Critical Path: FPGA0 datapath       │
│  • No timing violations                │
│                                         │
└─────────────────────────────────────────┘
```

## Build and Synthesis Flow

```
Vivado Synthesis Flow:
   │
   ├─ system_top.v (Verilog)
   │  system_bd.tcl (Block Design)
   │  system_constr.xdc (Timing/Pin Constraints)
   │
   ▼
RTL Analysis
   │ (Check for syntax errors, connectivity)
   ▼
Synthesis
   │ (Convert RTL to LUTs, RAMs, DSP blocks)
   ▼
Place & Route
   │ (Place logic on FPGA, route signals)
   │ (Optimize for timing closure)
   ▼
Generate Bitstream
   │ (Create programming file)
   ▼
system_top.bit (FPGA Bitstream, ~943 KB)
   │
   └─ Packaged into pluto.itb (FIT image)
      └─ Loaded into FPGA at boot
```

## Related Documentation

- [System Overview](./system-overview.md) - High-level architecture
- [Hardware Architecture](./hardware-architecture.md) - Hardware details
- [Build Process](../build-system/build-process.md) - Firmware build
