# PlutoSDR Firmware - Linux Kernel

## Overview

The PlutoSDR firmware uses a custom Linux kernel (version 4.9.0) maintained by Analog Devices Inc., featuring extensive Industrial I/O (IIO) drivers for RF transceiver control and software-defined radio applications.

## Kernel Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                    LINUX KERNEL 4.9.0 ARCHITECTURE                   │
├──────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                      USER SPACE                             │    │
│  │  ┌─────────────┐  ┌─────────────┐  ┌──────────────────┐    │    │
│  │  │ libiio      │  │ libad9361   │  │ User Applications│    │    │
│  │  │ (IIO        │  │ (AD9361     │  │ - GNU Radio      │    │    │
│  │  │  Library)   │  │  Control)   │  │ - Custom Apps    │    │    │
│  │  └──────┬──────┘  └──────┬──────┘  └────────┬─────────┘    │    │
│  │         │                │                   │              │    │
│  └─────────┼────────────────┼───────────────────┼──────────────┘    │
│            │                │                   │                   │
│  ══════════╪════════════════╪═══════════════════╪══════════════     │
│            │                │                   │                   │
│  ┌─────────┼────────────────┼───────────────────┼──────────────┐    │
│  │         ▼                ▼                   ▼              │    │
│  │   ┌──────────────────────────────────────────────────┐     │    │
│  │   │        Industrial I/O (IIO) Framework            │     │    │
│  │   │  - Device Registration                            │     │    │
│  │   │  - Buffer Management                              │     │    │
│  │   │  - Trigger Support                                │     │    │
│  │   │  - Channel Abstraction                            │     │    │
│  │   └───────────────────┬──────────────────────────────┘     │    │
│  │                       │                                    │    │
│  │        ┌──────────────┼──────────────┐                     │    │
│  │        │              │              │                     │    │
│  │        ▼              ▼              ▼                     │    │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────────┐            │    │
│  │  │ AD9361   │  │ AXI ADC  │  │   AXI DAC    │            │    │
│  │  │ Driver   │  │ Driver   │  │   Driver     │            │    │
│  │  │          │  │          │  │              │            │    │
│  │  │ -Freq Ctl│  │ -RX Path │  │   -TX Path   │            │    │
│  │  │ -Gain Ctl│  │ -DMA Buf │  │   -DMA Buf   │            │    │
│  │  │ -Calib   │  │          │  │              │            │    │
│  │  └─────┬────┘  └─────┬────┘  └──────┬───────┘            │    │
│  │        │             │               │                    │    │
│  │        │             └───────┬───────┘                    │    │
│  │        │                     │                            │    │
│  │        ▼                     ▼                            │    │
│  │  ┌──────────┐         ┌──────────────┐                   │    │
│  │  │   SPI    │         │  AXI DMA     │                   │    │
│  │  │  Driver  │         │  Engine      │                   │    │
│  │  └─────┬────┘         └──────┬───────┘                   │    │
│  │        │                     │                            │    │
│  └────────┼─────────────────────┼────────────────────────────┘    │
│           │                     │                                 │
│  ══════════════════════════════════════════════════════════       │
│           │                     │                                 │
│  ┌────────┼─────────────────────┼────────────────────────────┐    │
│  │        ▼                     ▼                            │    │
│  │  ┌──────────┐         ┌──────────────┐                   │    │
│  │  │  Zynq    │         │  FPGA Fabric │                   │    │
│  │  │  PS7 SPI │         │  AXI/AMBA Bus│                   │    │
│  │  └─────┬────┘         └──────┬───────┘                   │    │
│  │        │                     │                            │    │
│  │        ▼                     ▼                            │    │
│  │  ┌──────────┐         ┌──────────────┐                   │    │
│  │  │ AD9363/4 │         │ ADC/DAC Cores│                   │    │
│  │  │    RF    │◄────────┤ (Digital I/Q)│                   │    │
│  │  │Transceiver│         └──────────────┘                   │    │
│  │  └──────────┘                                             │    │
│  │                                                           │    │
│  │                    HARDWARE                               │    │
│  └───────────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────────────┘
```

## Kernel Version and Source

**Version:** Linux 4.9.0 "Roaring Lionus"
**Branch:** 2018_R1
**Repository:** https://github.com/analogdevicesinc/linux
**Commit:** f3da30df60047dc5a0b8fa8c640be774e0f784d9

## Key Kernel Features

### 1. Industrial I/O (IIO) Subsystem

The IIO framework is the core of PlutoSDR's RF control:

```
/sys/bus/iio/devices/
├── iio:device0 (Xilinx XADC - temperature/voltage monitoring)
├── iio:device1 (cf_ad9361_dds_core_lpc - TX DAC)
├── iio:device2 (AD9361 PHY - RF transceiver)
├── iio:device3 (cf_ad9361_lpc - RX ADC)
└── trigger0 (hrtimer trigger)

Each device exposes:
  • Channels: Individual data streams (I/Q, voltage, temperature)
  • Attributes: Configuration parameters (frequency, gain, sample rate)
  • Buffers: High-speed data transfer via DMA
  • Triggers: Synchronization mechanisms
```

### 2. AD9361 RF Transceiver Driver

**Location:** `drivers/iio/adc/ad9361.c`
**Lines of Code:** ~7000+
**License:** GPL-2.0

**Key Capabilities:**
```
┌─────────────────────────────────────────────────────────────┐
│                  AD9361 Driver Architecture                 │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  SPI Communication Layer                                    │
│  ├─ Register Read/Write                                     │
│  ├─ Multi-register Operations                               │
│  └─ Error Handling                                          │
│                                                             │
│  RF Synthesizer Control                                     │
│  ├─ RX LO Frequency (70 MHz - 6 GHz)                        │
│  ├─ TX LO Frequency (47 - 6000 MHz)                         │
│  ├─ PLL Management                                          │
│  ├─ VCO Calibration                                         │
│  └─ Integer/Fractional-N synthesis                          │
│                                                             │
│  Gain Control                                               │
│  ├─ AGC (Automatic Gain Control)                            │
│  │   ├─ Fast AGC                                            │
│  │   ├─ Slow AGC                                            │
│  │   └─ Hybrid AGC                                          │
│  ├─ MGC (Manual Gain Control)                               │
│  └─ Gain Tables (Full/Split gain modes)                     │
│                                                             │
│  Filtering and Rate Control                                 │
│  ├─ RX FIR Filter                                           │
│  ├─ TX FIR Filter                                           │
│  ├─ Decimation (RX)                                         │
│  ├─ Interpolation (TX)                                      │
│  └─ Bandwidth Configuration                                 │
│                                                             │
│  Calibration Engine                                         │
│  ├─ TX Quadrature Calibration                               │
│  ├─ RX Quadrature Calibration                               │
│  ├─ RF DC Offset Calibration                                │
│  ├─ BB DC Offset Calibration                                │
│  └─ Gain Step Calibration                                   │
│                                                             │
│  ENSM (Enable State Machine)                                │
│  ├─ State: SLEEP → WAIT → ALERT → TX/RX → FDD/TDD          │
│  ├─ Fast Lock Profiles                                      │
│  └─ Pin Control                                             │
│                                                             │
│  Data Path Configuration                                    │
│  ├─ LVDS/CMOS Interface                                     │
│  ├─ Port Swap (RX1↔RX2, TX1↔TX2)                            │
│  ├─ Half/Full Duplex                                        │
│  └─ 1R1T / 2R2T Modes                                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 3. DMA Engine Support

**Driver:** `drivers/dma/dma-axi-dmac.c`

```
┌──────────────────────────────────────────────────────────┐
│           AXI DMA Architecture in Kernel                 │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  User Space                                              │
│      │                                                   │
│      ├─ open("/dev/iio:device3")                         │
│      │                                                   │
│      ├─ ioctl(IIO_BUFFER_SETUP)                          │
│      │    • Configure buffer size                        │
│      │    • Set sample count                             │
│      │                                                   │
│      ├─ mmap() or read()                                 │
│      │    • Map DMA buffer to user space                 │
│      │    • Zero-copy data access                        │
│      │                                                   │
│  ════════════════════════════════════════                │
│                                                          │
│  Kernel Space                                            │
│      │                                                   │
│      ├─ IIO Buffer (indio_dev->buffer)                   │
│      │    • Kfifo or DMA buffer                          │
│      │    • Watermark handling                           │
│      │    • Poll wait queues                             │
│      │                                                   │
│      ├─ DMA Engine API                                   │
│      │    • dmaengine_prep_slave_single()                │
│      │    • dmaengine_submit()                           │
│      │    • dma_async_issue_pending()                    │
│      │                                                   │
│      ├─ AXI DMAC Driver                                  │
│      │    ├─ Configure source/destination               │
│      │    ├─ Set transfer size                           │
│      │    ├─ Enable interrupts                           │
│      │    └─ Start transfer                              │
│      │                                                   │
│  ════════════════════════════════════════                │
│                                                          │
│  Hardware (FPGA)                                         │
│      │                                                   │
│      ├─ AXI Stream (from ADC)                            │
│      │       ↓                                           │
│      ├─ DMA Controller                                   │
│      │    • FIFO buffering                               │
│      │    • Burst optimization                           │
│      │    • Error detection                              │
│      │       ↓                                           │
│      └─ AXI Master → DDR3 Memory                         │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

### 4. USB Gadget Framework

**Configuration:** Multi-function composite device

```
┌──────────────────────────────────────────────────────────┐
│              USB Composite Gadget Stack                  │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  USB Gadget Configfs                                     │
│  /sys/kernel/config/usb_gadget/pluto/                    │
│                                                          │
│  Gadget Configuration:                                   │
│  ├─ idVendor: 0x0456 (Analog Devices)                   │
│  ├─ idProduct: 0xb673 (PlutoSDR)                         │
│  ├─ bcdDevice: 0x0200                                    │
│  ├─ Manufacturer: "Analog Devices Inc."                  │
│  ├─ Product: "PlutoSDR (ADALM-PLUTO)"                    │
│  └─ SerialNumber: <from EEPROM>                          │
│                                                          │
│  Functions:                                              │
│  ┌────────────────────────────────────────────┐         │
│  │ 1. RNDIS/NCM/ECM - Network Interface       │         │
│  │    └─ Creates usb0 network device          │         │
│  │       IP: 192.168.2.1/24                   │         │
│  │       DHCP server enabled                  │         │
│  │                                             │         │
│  │ 2. Mass Storage - Virtual USB Drive        │         │
│  │    └─ Exposes FAT32 image                  │         │
│  │       File: /dev/loop0                     │         │
│  │       Size: 30MB                           │         │
│  │       Contents: index.html, LICENSE.html   │         │
│  │                                             │         │
│  │ 3. ACM - Serial Console                    │         │
│  │    └─ Creates /dev/ttyGS0                  │         │
│  │       Baud: 115200                         │         │
│  │       Login shell access                   │         │
│  │                                             │         │
│  │ 4. FunctionFS - IIO Control Interface      │         │
│  │    └─ Custom endpoint for libiio           │         │
│  │       Direct IIO device access             │         │
│  │       High-speed streaming                 │         │
│  └────────────────────────────────────────────┘         │
│                                                          │
│  Underlying Drivers:                                     │
│  ├─ chipidea (USB OTG controller)                        │
│  ├─ u_ether (USB Ethernet)                               │
│  ├─ f_mass_storage (USB MSC)                             │
│  ├─ f_acm (USB CDC ACM)                                  │
│  └─ f_fs (USB FunctionFS)                                │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

## Device Tree Integration

The kernel uses device trees for hardware description:

```
Device Tree Hierarchy:

zynq.dtsi (Xilinx Zynq-7000 base)
    ↓
zynq-pluto-sdr.dtsi (PlutoSDR common)
    ├─ Memory: 512MB DDR3
    ├─ QSPI Flash partitions
    ├─ USB OTG configuration
    ├─ AD9363 transceiver (SPI0)
    │   ├─ RX/TX frequency
    │   ├─ Sample rate
    │   ├─ Gain control
    │   └─ AGC settings
    ├─ FPGA peripherals (AXI/AMBA)
    │   ├─ RX DMA @ 0x7c400000
    │   ├─ TX DMA @ 0x7c420000
    │   ├─ ADC Core @ 0x79020000
    │   ├─ DAC Core @ 0x79024000
    │   └─ I2C @ 0x41600000
    └─ Clocking
        └─ ad9364_clkin: 40MHz ±200ppm
    ↓
┌────────────────────────────────────────┐
│ zynq-pluto-sdr.dts (Rev A)             │
│ ├─ LED: GPIO0[68]                      │
│ └─ Button: None                        │
├────────────────────────────────────────┤
│ zynq-pluto-sdr-revb.dts (Rev B)        │
│ ├─ LED: GPIO0[15]                      │
│ ├─ Button: GPIO0[14]                   │
│ └─ ADM1177: I2C current monitor        │
├────────────────────────────────────────┤
│ zynq-pluto-sdr-revc.dts (Rev C)        │
│ ├─ LED: GPIO0[15]                      │
│ ├─ Button: GPIO0[14]                   │
│ └─ ADM1177: I2C current monitor        │
└────────────────────────────────────────┘
```

## Kernel Configuration

**Defconfig:** `arch/arm/configs/zynq_pluto_defconfig`

### Key Configuration Options:

```
┌─────────────────────────────────────────────────────────┐
│              Kernel Configuration Highlights            │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ARCHITECTURE                                           │
│  ├─ CONFIG_ARCH_ZYNQ=y (Xilinx Zynq platform)          │
│  ├─ CONFIG_SMP=y (Dual-core support)                   │
│  ├─ CONFIG_VFP=y (Vector Floating Point)               │
│  ├─ CONFIG_NEON=y (ARM NEON SIMD)                      │
│  └─ CONFIG_HIGHMEM=y (>512MB memory support)           │
│                                                         │
│  MEMORY & CMA                                           │
│  ├─ CONFIG_CMA=y (Contiguous Memory Allocator)         │
│  └─ CONFIG_CMA_SIZE_MBYTES=256 (256MB for DMA)         │
│                                                         │
│  INDUSTRIAL I/O                                         │
│  ├─ CONFIG_IIO=y                                        │
│  ├─ CONFIG_AD9361=y (Main RF driver)                   │
│  ├─ CONFIG_CF_AXI_DDS=y (DDS core)                     │
│  └─ CONFIG_XILINX_XADC=y (Voltage/temp monitor)        │
│                                                         │
│  USB GADGET                                             │
│  ├─ CONFIG_USB_GADGET=y                                │
│  ├─ CONFIG_USB_CONFIGFS=y                              │
│  ├─ CONFIG_USB_CONFIGFS_SERIAL=y                       │
│  ├─ CONFIG_USB_CONFIGFS_NCM=y                          │
│  ├─ CONFIG_USB_CONFIGFS_MASS_STORAGE=y                 │
│  └─ CONFIG_USB_CONFIGFS_F_FS=y                         │
│                                                         │
│  DMA                                                    │
│  ├─ CONFIG_DMADEVICES=y                                │
│  ├─ CONFIG_AXI_DMAC=y (AXI DMA controller)             │
│  └─ CONFIG_XILINX_DMA=y (Xilinx DMA engine)            │
│                                                         │
│  FPGA                                                   │
│  ├─ CONFIG_FPGA=y                                      │
│  ├─ CONFIG_FPGA_MGR_ZYNQ_FPGA=y (FPGA manager)         │
│  └─ CONFIG_XILINX_PR_DECOUPLER=y (Partial reconfig)    │
│                                                         │
│  NETWORKING                                             │
│  ├─ CONFIG_PACKET=y (Packet sockets)                   │
│  ├─ CONFIG_INET=y (TCP/IP)                             │
│  ├─ CONFIG_IP_MULTICAST=y                              │
│  └─ # CONFIG_IPV6 is not set (IPv4 only)               │
│                                                         │
│  FILESYSTEMS                                            │
│  ├─ CONFIG_MSDOS_FS=y                                  │
│  ├─ CONFIG_VFAT_FS=y (FAT32)                           │
│  ├─ CONFIG_TMPFS=y (tmpfs)                             │
│  └─ CONFIG_CONFIGFS_FS=y (ConfigFS for USB gadget)     │
│                                                         │
│  DEBUGGING (Limited for production)                     │
│  ├─ CONFIG_DEBUG_INFO=y                                │
│  ├─ CONFIG_DEBUG_FS=y (debugfs)                        │
│  └─ # CONFIG_FTRACE is not set (No ftrace overhead)    │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

## Build Process

```
┌──────────────────────────────────────────────────────────┐
│               Kernel Build Flow                          │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  1. Configure                                            │
│     make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf-    │
│          zynq_pluto_defconfig                            │
│     ├─ Load arch/arm/configs/zynq_pluto_defconfig       │
│     └─ Generate .config                                  │
│                                                          │
│  2. Compile Kernel                                       │
│     make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf-    │
│          -j $(NCORES) zImage UIMAGE_LOADADDR=0x8000     │
│     ├─ Compile core kernel                              │
│     ├─ Build drivers (IIO, USB, DMA, etc.)              │
│     ├─ Link vmlinux                                      │
│     └─ Compress to zImage                                │
│     Output: arch/arm/boot/zImage (~4.1MB)               │
│                                                          │
│  3. Compile Device Trees                                 │
│     make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf-    │
│          -j $(NCORES) DTC_FLAGS=-@ \                     │
│          zynq-pluto-sdr.dtb \                            │
│          zynq-pluto-sdr-revb.dtb \                       │
│          zynq-pluto-sdr-revc.dtb                         │
│     ├─ Compile .dts → .dtb                               │
│     ├─ Include symbols (-@) for overlays                 │
│     └─ Post-process: axi → amba node rename              │
│                                                          │
│  4. Package into FIT Image                               │
│     mkimage -f scripts/pluto.its build/pluto.itb        │
│     ├─ Bundle: zImage + DTBs + FPGA + rootfs            │
│     ├─ Add MD5 hashes                                    │
│     └─ Create multi-config image                         │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

## Runtime Kernel Parameters

**Kernel Command Line** (from U-Boot):
```
console=ttyPS0,115200
root=/dev/ram0
rw
rootfstype=ext4
earlyprintk
clk_ignore_unused
uio_pdrv_genirq.of_id=generic-uio
```

**Key Parameters:**
- `console=ttyPS0,115200` - Serial console on UART0
- `root=/dev/ram0` - Root filesystem in RAM (ramdisk)
- `clk_ignore_unused` - Don't disable unused clocks (FPGA dependent)
- `uio_pdrv_genirq.of_id=generic-uio` - Enable UIO devices from device tree

## Key Kernel Modules

While the PlutoSDR kernel is mostly monolithic, key components:

```
Built-in (=y):
├─ ad9361 (RF transceiver)
├─ cf_axi_adc (ADC core)
├─ cf_axi_dds (DAC/DDS core)
├─ dma-axi-dmac (DMA engine)
├─ usb_f_ncm (USB networking)
├─ usb_f_mass_storage (USB MSD)
├─ usb_f_acm (USB serial)
└─ zynq_fpga_mgr (FPGA management)

Not used (modules disabled for embedded footprint)
```

## Performance Characteristics

```
┌──────────────────────────────────────────────────────────┐
│            Kernel Performance Metrics                    │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  Boot Time: ~5 seconds (kernel init)                    │
│  Memory Footprint: ~40MB (kernel + drivers)             │
│  zImage Size: ~4.1MB (compressed)                       │
│  vmlinux Size: ~13MB (uncompressed in memory)           │
│                                                          │
│  IIO Buffer Performance:                                 │
│  ├─ Max Sample Rate: 61.44 MSPS                         │
│  ├─ DMA Latency: <100μs                                 │
│  ├─ Buffer Size: Configurable (typically 64KB-4MB)      │
│  └─ CPU Overhead: <5% for streaming                     │
│                                                          │
│  USB Gadget Performance:                                 │
│  ├─ Network: ~480 Mbps (USB 2.0 High-Speed)             │
│  ├─ Mass Storage: ~30 MB/s read, ~20 MB/s write         │
│  └─ Serial: 115200 baud (console)                       │
│                                                          │
│  Context Switch: ~2μs                                    │
│  Interrupt Latency: ~10μs (with PREEMPT)                │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

## Debugging and Development

### Kernel Log Access

```
# Via serial console
dmesg

# Via SSH
ssh root@192.168.2.1
dmesg | grep -i "ad9361\|iio\|dma"

# Kernel ring buffer
cat /proc/kmsg
```

### IIO Debugging

```
# List IIO devices
ls -l /sys/bus/iio/devices/

# Read AD9361 attributes
cat /sys/bus/iio/devices/iio:device2/out_altvoltage0_RX_LO_frequency
cat /sys/bus/iio/devices/iio:device2/in_voltage0_gain_control_mode

# Monitor DMA transfers
cat /sys/kernel/debug/dma-axi-dmac/7c400000.dma/stats
```

### Performance Profiling

```
# Enable perf (if compiled)
CONFIG_PERF_EVENTS=y

# CPU usage per process
top

# IRQ statistics
cat /proc/interrupts

# Memory info
cat /proc/meminfo
free -m
```

## Related Documentation

- [Device Trees](05-device-trees.md)
- [U-Boot](03-u-boot.md)
- [Buildroot](01-buildroot.md)
- [Boot Sequence](../architecture/04-boot-sequence.md)
