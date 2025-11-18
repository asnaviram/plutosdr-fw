# FPGA and HDL Components Documentation

## FPGA Design Overview

The PlutoSDR FPGA design implements the RF signal processing path, connecting the AD9361 RF transceiver to the ARM processors via AXI interconnect.

## Design Hierarchy

```
projects/pluto/ (Main Project)
│
├── system_top.v
│   └── Top-level Verilog wrapper
│       └── Instantiates system_wrapper (from Vivado)
│       └── Instantiates ad_iobuf (GPIO tristate buffers)
│
├── system_bd.tcl
│   └── Vivado Block Design (Graphical)
│       └── Defines IP connectivity via AXI
│
├── system_project.tcl
│   └── Vivado project build script
│       ├── Sources: system_top.v, constraints
│       ├── Block design: system_bd.tcl
│       ├── Synthesis, Place & Route
│       └── Generates: system_top.bit, system_top.xsa
│
├── system_constr.xdc
│   └── Xilinx Design Constraints
│       ├── Pin assignments (LOC)
│       ├── Timing constraints (clock period)
│       ├── I/O standards (LVCMOS, SSTL)
│       ├── False paths (clock domain crossings)
│       └── Voltage standards
│
└── Makefile
    └── Build automation
        ├── Sources vivado setup
        ├── Runs Vivado synthesis/P&R
        ├── Generates bitstream
        └── Exports hardware (XSA)
```

## Block Design Structure

```
Vivado Block Design Components:

┌──────────────────────────────────────────────────────────┐
│              Zynq PS7 (ARM System)                       │
│  • Dual-core Cortex-A9 @ 667 MHz                        │
│  • 512 MB DDR3 Memory Controller                        │
│  • Peripherals: QSPI, GPIO, I2C, SPI, UART, USB        │
│  • Interrupt Concatenator (16 FPGA interrupts)         │
│  • AXI Interface to FPGA Fabric                         │
└──────────┬───────────────────────────────────────────────┘
           │ (AXI Master)
           ▼
┌──────────────────────────────────────────────────────────┐
│         AXI Interconnect (Central Matrix)               │
│  • Routes transactions between PS and FPGA IPs         │
│  • Supports multiple concurrent transfers              │
│  • Clock domain crossing for independent domains       │
└─┬────────┬───────────┬────────┬────────┬────────────────┘
  │        │           │        │        │
  ▼        ▼           ▼        ▼        ▼
┌───────┐┌──────┐┌──────┐┌────┐┌─────┐
│AD9361 ││CLK   ││FIR   ││DMA ││INTR │
│AXI    ││GEN   ││FILT  ││CTL ││DIST │
└───────┘└──────┘└──────┘└────┘└─────┘
  │        │           │        │        │
  └────────┴───────────┴────────┴────────┘
           │
           ▼
┌──────────────────────────────────────────────────────────┐
│         DDR3 Memory Interface (HP Ports)                │
│  • HP0: ARM Processor (read/write)                      │
│  • HP1: ADC DMA (write only)                           │
│  • HP2: DAC DMA (read only)                            │
└──────────────────────────────────────────────────────────┘
```

## Major IP Core Components

### 1. AD9361 RF Transceiver Interface (axi_ad9361)

**Module Hierarchy**:
```
axi_ad9361 (Top-level)
├── axi_ad9361_rx (RX path master)
│   ├── axi_ad9361_rx_channel (IQ processing per channel)
│   │   ├── Data alignment
│   │   ├── Gain/phase correction
│   │   └── Width conversion
│   └── Control registers & status
│
├── axi_ad9361_tx (TX path master)
│   ├── axi_ad9361_tx_channel (IQ processing per channel)
│   │   ├── Data alignment
│   │   ├── Gain/phase correction
│   │   └── Width conversion
│   └── Control registers & status
│
├── axi_ad9361_tdd (Time-Division Duplex controller)
│   ├── RX/TX mode switching
│   ├── Timing optimization
│   └── Full-duplex support
│
└── Configuration Registers (AXI Slave)
    ├── Read/write from ARM
    ├── IIO driver access
    └── Real-time control
```

**Signal Paths**:
```
RX Path (Input):
AD9361 parallel RX data (12-bit @ 61.44 MSps)
    │
    ▼
axi_ad9361_rx
    │ (IQ multiplexing, frame synchronization)
    ▼
axi_ad9361_rx_channel
    │ (Per-channel processing)
    ▼
util_fir_dec (FIR Decimator)
    │ (Sample rate reduction, anti-aliasing filter)
    ▼
axi_dmac write engine
    │ (DMA to DDR3)
    ▼
User Application Memory

TX Path (Output):
User Application Memory
    │
    ▼
axi_dmac read engine
    │ (DMA from DDR3)
    ▼
util_fir_int (FIR Interpolator)
    │ (Sample rate increase, reconstruction filter)
    ▼
axi_ad9361_tx_channel
    │ (Per-channel processing)
    ▼
axi_ad9361_tx
    │ (IQ multiplexing, frame generation)
    ▼
AD9361 parallel TX data (12-bit @ 61.44 MSps)
```

### 2. DMA Controllers (axi_dmac)

**Two Instances**:
```
Instance 1: ADC DMA (HP1 write port)
├── Source: axi_ad9361 RX output
├── Destination: DDR3 (RX ring buffer)
├── Mode: Continuous circular buffer
├── Burst Size: 256 bytes per beat
└── Interrupts: On buffer threshold

Instance 2: DAC DMA (HP2 read port)
├── Source: DDR3 (TX ring buffer)
├── Destination: axi_ad9361 TX input
├── Mode: Continuous circular buffer
├── Burst Size: 256 bytes per beat
└── Interrupts: On buffer empty/full
```

**DMA Transaction Flow**:
```
Master (DMA)
    │
    ├─ AXI Write Address (AWADDR)
    ├─ AXI Write Data (WDATA)
    ├─ AXI Write Response (BRESP)
    ├─ AXI Read Address (ARADDR)
    └─ AXI Read Data (RDATA)
        │
        ▼
    Slave (DDR3 Memory)
        │
        ├─ Receives address
        ├─ Performs memory operation
        └─ Returns response/data
```

### 3. FIR Filter Components

#### FIR Decimator (util_fir_dec)

**Architecture**:
```
Input Stream (61.44 MSps)
    │
    ▼
┌──────────────────────────────────┐
│ Input FIFO (async clock crossing)│
└──────────────────────────────────┘
    │
    ▼
┌──────────────────────────────────┐
│ FIR Filter Taps (127 taps)       │
│ • Polyphase structure            │
│ • Parallel computation           │
│ • 16-bit coefficients            │
│ • 24-bit internal precision      │
└──────────────────────────────────┘
    │
    ▼
┌──────────────────────────────────┐
│ Decimation Logic                 │
│ • Output every Nth sample        │
│ • Typically N=2 (2× decimation)  │
└──────────────────────────────────┘
    │
    ▼
┌──────────────────────────────────┐
│ Output FIFO (async clock crossing)
└──────────────────────────────────┘
    │
    ▼
Output Stream (30.72 MSps)
```

**Filter Specifications**:
- **Filter Type**: Polyphase FIR
- **Order**: ~127 taps
- **Passband**: 0 to Fs/4
- **Stopband Attenuation**: >80 dB
- **Passband Ripple**: <0.01 dB
- **Decimation Ratio**: 2× (configurable)

#### FIR Interpolator (util_fir_int)

**Architecture**:
```
Input Stream (30.72 MSps)
    │
    ▼
┌──────────────────────────────────┐
│ Input FIFO (async clock crossing)│
└──────────────────────────────────┘
    │
    ▼
┌──────────────────────────────────┐
│ Zero Insertion                   │
│ Insert N-1 zeros between samples │
│ Upsampling by factor N           │
└──────────────────────────────────┘
    │
    ▼
┌──────────────────────────────────┐
│ FIR Interpolation Filter         │
│ • Polyphase structure            │
│ • Parallel taps                  │
│ • 16-bit coefficients            │
│ • 24-bit internal precision      │
└──────────────────────────────────┘
    │
    ▼
┌──────────────────────────────────┐
│ Output FIFO (async clock crossing)
└──────────────────────────────────┘
    │
    ▼
Output Stream (61.44 MSps)
```

**Filter Specifications**:
- **Filter Type**: Polyphase FIR
- **Order**: ~127 taps
- **Passband**: 0 to Fs/4 (original)
- **Image Attenuation**: >80 dB
- **Passband Ripple**: <0.01 dB
- **Interpolation Ratio**: 2× (configurable)

### 4. Clock Generation (util_clkgen)

**Features**:
```
PS System Clock (667 MHz)
    │
    ▼
┌──────────────────────────────────────────┐
│ util_clkgen (Clock Generator)            │
│ Fractional-N PLL in FPGA fabric          │
│                                          │
│ • Frequency synthesis                    │
│ • Multiple output clocks                 │
│ • Lock detection                         │
│ • Jitter optimization                    │
│                                          │
│ PLL Internal Structure:                  │
│ • Reference clock: 667 MHz from PS       │
│ • Phase detector                         │
│ • Charge pump                            │
│ • VCO (Voltage Controlled Oscillator)   │
│ • Feedback divider (fractional)          │
│ • Output dividers                        │
└──────────────────────────────────────────┘
    │
    ├─ FPGA0 Clock (100 MHz)
    │  └─ AXI interconnect, slow control logic
    │
    └─ FPGA1 Clock (200 MHz)
       └─ High-speed datapath (FIR, DMA)
```

## Clock Domain Crossing

```
FPGA0 (100 MHz)
    │
    │   asynchronous FIFO
    │   └─ Gray code conversion
    │   └─ Double-flop sync
    │
    ▼
FPGA1 (200 MHz)
    │
    │   asynchronous FIFO
    │   └─ Gray code conversion
    │   └─ Double-flop sync
    │
    ▼
RF Clock (~61.5 MHz)
    │
    └─ Synchronized to AD9361 clock
```

## I/O Buffers (ad_iobuf)

**Purpose**: Provide tristate GPIO control for RF transceiver signals

```
FPGA GPIO Signals
    │
    ├─ gpio_resetb (P9)       - Active-low reset
    ├─ gpio_en_agc (L13)      - AGC enable
    ├─ gpio_ctl[3:0]          - Control lines
    └─ gpio_status[7:0]       - Status inputs
        │
        ▼
┌─────────────────────────────────────┐
│ ad_iobuf (Tristate Buffer)          │
│                                     │
│ • Output enable logic               │
│ • Input/output multiplexing         │
│ • Slew rate limiting (optional)    │
│ • Protection: ESD, current limiting │
└─────────────────────────────────────┘
        │
        ▼
Physical FPGA Pins
    │
    ├─ LVCMOS18 standard
    ├─ 1.8V signaling
    └─ Connected to AD9361 GPIO lines
```

## Resource Utilization

```
Zynq-7010 Resource Budget:

Component           Utilized    Total    Usage %
───────────────────────────────────────────────
LUT Slices          ~18,000    28,000    64%
├─ AD9361 AXI: 4,500
├─ FIR Filters: 7,000
├─ DMA: 3,000
├─ Interconnect: 2,500
└─ Other: 1,000

BRAM (36Kb)         ~200       240       83%
├─ DMA buffers: 80
├─ FIR coefficients: 60
├─ Interconnect: 40
└─ Other: 20

DSP48E1 Slices      ~60        80        75%
├─ FIR filters: 40 (multiply-accumulate)
├─ DMA: 12
└─ Other: 8

BUFG Clocks         4          32        12%
├─ FPGA0 (100 MHz): 1
├─ FPGA1 (200 MHz): 1
├─ RF Clock: 1
└─ Other: 1
```

## Pin Assignment Summary

### RF Interface Pins (AD9361)

```
RX Data Path:
├─ rx_clk (L12)     [LVCMOS18]  61.5 MHz clock in
├─ rx_frame (N13)   [LVCMOS18]  Frame synchronization
└─ rx_d[0:11]       [LVCMOS18]  12-bit parallel data

TX Data Path:
├─ tx_clk (P10)     [LVCMOS18]  Clock output
├─ tx_frame (L14)   [LVCMOS18]  Frame synchronization
└─ tx_d[0:11]       [LVCMOS18]  12-bit parallel data
```

### GPIO and Control Pins

```
Control Signals:
├─ gpio_resetb (P9)     [LVCMOS18]  Active-low reset
├─ gpio_en_agc (L13)    [LVCMOS18]  AGC enable
├─ gpio_ctl[0] (F13)    [LVCMOS18]  Control 0
├─ gpio_ctl[1] (F14)    [LVCMOS18]  Control 1
├─ gpio_ctl[2] (F15)    [LVCMOS18]  Control 2
└─ gpio_ctl[3] (E15)    [LVCMOS18]  Control 3

Status Signals:
└─ gpio_status[0:7]     [LVCMOS18]  8-bit status input
```

### Memory Pins (DDR3)

```
Address Bus:
└─ A[0:13]              [SSTL15]    14-bit address

Bank Select:
└─ BA[0:2]              [SSTL15]    3-bit bank

Control Signals:
├─ CAS, RAS, WE         [SSTL15]    Column/row/write enable
├─ CS                   [SSTL15]    Chip select
├─ CKE                  [SSTL15]    Clock enable
└─ ODT                  [SSTL15]    On-die termination

Data:
├─ DQ[0:15]             [SSTL15]    16-bit data bus
└─ DQS_P, DQS_N         [DIFF_SSTL15] Differential strobes

Clock:
└─ CLK_P, CLK_N         [DIFF_SSTL15] Differential clock
```

## Timing Closure

```
Timing Constraints (system_constr.xdc):

Clock Periods:
├─ FPGA0: 10 ns (100 MHz) - Comfortable timing
├─ FPGA1: 5 ns (200 MHz) - Critical path
└─ RF Clock: ~16.27 ns (61.5 MHz) - Loose constraint

Critical Paths:
├─ FIR filter datapath (FPGA1 clock domain)
│  └─ Logic depth: ~12 levels
│  └─ Slack: Positive (~2 ns available)
│
├─ DMA burst transactions (AXI clock domain)
│  └─ Logic depth: ~8 levels
│  └─ Slack: Positive (~3 ns available)
│
└─ CDC (Clock Domain Crossing)
   └─ Handled by async FIFOs
   └─ No direct timing between domains

Timing Analysis:
• Tool: Vivado Static Timing Analyzer
• Strategy: Place & Route optimization for timing
• Result: Timing closed with positive slack
• Synthesis options: Area optimization (not speed)
```

## Build and Synthesis

### Vivado Synthesis Flow

```
RTL Sources (Verilog)
    │
    ├─ system_top.v (wrapper)
    ├─ Generated from system_bd.tcl (Block Design)
    └─ IP cores (axi_ad9361, axi_dmac, util_fir_*, etc.)
    │
    ▼
Synthesis (High-level RTL → Logic gates)
    │
    ├─ Check RTL syntax
    ├─ Elaborate design hierarchy
    ├─ Map logic to LUTs, RAMs, DSP blocks
    ├─ Optimize for area/speed tradeoffs
    └─ Generate gate-level netlist
    │
    ▼
Place & Route
    │
    ├─ Place logic blocks on FPGA grid
    ├─ Route connections between blocks
    ├─ Optimize for timing closure
    ├─ Minimize routing congestion
    └─ Generate bitstream
    │
    ▼
Generate Bitstream
    │
    └─ system_top.bit (FPGA bitstream, ~943 KB)
        │
        ├─ Encoded FPGA configuration
        ├─ CLB content
        ├─ LUT content
        ├─ RAM initialization
        └─ Routing configuration
    │
    ▼
Export Hardware (XSA)
    │
    └─ system_top.xsa (Xilinx Scalable Architecture)
        │
        ├─ Hardware definition
        ├─ IP core configurations
        ├─ Memory map
        ├─ Interrupt map
        └─ Used for bootloader generation
```

### Build Command

```makefile
make -C hdl/projects/pluto

Flow:
1. Source Vivado settings
2. Run: vivado -mode batch -source system_project.tcl
3. Vivado:
   a) Create project
   b) Add source files
   c) Source block design (system_bd.tcl)
   d) Run synthesis
   e) Run place & route
   f) Generate bitstream
   g) Export hardware (XSA)
4. Output:
   • system_top.bit (FPGA bitstream)
   • system_top.xsa (Hardware export)
   • system.sdk/ (SDK with IP cores)
```

## Related Documentation

- [FPGA Design](../architecture/fpga-design.md) - Detailed FPGA architecture
- [Hardware Architecture](../architecture/hardware-architecture.md) - System overview
- [Build Process](../build-system/build-process.md) - How FPGA is built
