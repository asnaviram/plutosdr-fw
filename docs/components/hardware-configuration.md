# Hardware Configuration Component Documentation

## Device Tree Overview

The PlutoSDR hardware is described to the Linux kernel via Device Tree Binary (DTB) files. These files define the hardware structure, peripherals, and their addresses for both the ARM Processing System (PS) and FPGA Fabric (PL).

## Device Tree Files

### PlutoSDR Variants

```
Device Tree Sources (linux/arch/arm/boot/dts/):
├── zynq-pluto-sdr.dts              # Revision A (base)
├── zynq-pluto-sdr-revb.dts         # Revision B (enhanced)
├── zynq-pluto-sdr-revc.dts         # Revision C (latest)
└── zynq-pluto-sdr.dtsi             # Shared includes

Generated DTBs (build/):
├── zynq-pluto-sdr.dtb              # Compiled Revision A
├── zynq-pluto-sdr-revb.dtb         # Compiled Revision B
└── zynq-pluto-sdr-revc.dtb         # Compiled Revision C
```

### SidekiqZ2 Variant

```
Device Tree Sources:
└── zynq-sidekiqz2-revb.dts         # Revision B only

Generated DTB:
└── zynq-sidekiqz2-revb.dtb         # Compiled variant
```

## Device Tree Structure

### Device Tree Hierarchy

```
/ (Root Node)
├── #address-cells = <1>
├── #size-cells = <1>
│
├── cpu (ARM Cortex-A9 #0)
│   ├── compatible = "arm,cortex-a9"
│   ├── reg = <0>
│   ├── clocks = <&clkc 3>
│   └── operating-points-v2 = <&cpu_opp_table>
│
├── clocks (Clock distribution)
│   ├── #address-cells = <1>
│   ├── #size-cells = <0>
│   ├── armpll (ARM PLL)
│   │   ├── compatible = "xlnx,zynq-pll"
│   │   └── clock-frequency = <1333333333>
│   │
│   └── ddrpll (DDR PLL)
│       ├── compatible = "xlnx,zynq-pll"
│       └── clock-frequency = <1066666667>
│
├── amba (FPGA Fabric Peripherals)
│   ├── compatible = "simple-bus"
│   ├── ranges = <0 0 0 0xffffffff>
│   │
│   ├── axi_dmac (DMA Controller)
│   │   ├── compatible = "adi,axi-dmac-1.00.a"
│   │   ├── reg = <0x44a30000 0x10000>
│   │   ├── interrupts = <0 57 IRQ_TYPE_LEVEL_HIGH>
│   │   ├── #dma-cells = <1>
│   │   ├── dma-channels = <2>
│   │   └── clock-names = "core"
│   │
│   ├── axi_ad9361 (RF Transceiver Interface)
│   │   ├── compatible = "adi,axi-ad9361-6.00.a"
│   │   ├── reg = <0x44a00000 0x10000>
│   │   ├── interrupts = <0 55 IRQ_TYPE_LEVEL_HIGH>
│   │   ├── dmas = <&axi_dmac 0>, <&axi_dmac 1>
│   │   ├── dma-names = \"rx\", \"tx\"
│   │   ├── clocks = <&clkc 15>
│   │   └── clock-names = \"sampler_clk\"
│   │
│   └── [Other FPGA-based IP cores...]
│
├── memory@0 (DDR3 RAM)
│   ├── compatible = "memory"
│   ├── device_type = "memory"
│   ├── reg = <0 0x20000000>        # 512 MB DDR3
│   └── accessible = "true"
│
├── ps7_gpio_0 (GPIO Controller)
│   ├── compatible = "xlnx,zynq-gpio-1.0"
│   ├── reg = <0xe000a000 0x1000>
│   ├── gpio-controller
│   ├── #gpio-cells = <2>
│   ├── interrupts = <0 20 IRQ_TYPE_LEVEL_HIGH>
│   ├── interrupt-controller
│   └── #interrupt-cells = <2>
│
├── ps7_i2c_0 (I2C Master)
│   ├── compatible = "xlnx,zynq-i2c-1.0"
│   ├── reg = <0xe0004000 0x1000>
│   ├── interrupts = <0 25 IRQ_TYPE_LEVEL_HIGH>
│   ├── clocks = <&clkc 38>
│   ├── clock-frequency = <100000>
│   ├── i2c-max-frequency = <100000>
│   └── #address-cells = <1>
│
├── ps7_qspi_0 (QSPI Flash)
│   ├── compatible = "xlnx,zynq-qspi-1.0"
│   ├── reg = <0xe0009000 0x1000>
│   ├── interrupts = <0 26 IRQ_TYPE_LEVEL_HIGH>
│   ├── clocks = <&clkc 10>
│   ├── clock-names = "ref_clk"
│   ├── #address-cells = <1>
│   ├── #size-cells = <0>
│   │
│   └── spi-flash@0 (QSPI Flash Device)
│       ├── compatible = "s25fl256s1"
│       ├── reg = <0>
│       ├── spi-max-frequency = <50000000>
│       ├── #address-cells = <1>
│       ├── #size-cells = <1>
│       │
│       └── MTD Partitions:
│           ├── mtd0: QSPI-FSBL-UBOOT (1 MB)
│           ├── mtd1: QSPI-UBOOT-ENV (128 KB)
│           ├── mtd2: QSPI-NVMFS (896 KB)
│           └── mtd3: QSPI-LINUX (30 MB)
│
├── ps7_spi_0 (SPI Master)
│   ├── compatible = "xlnx,zynq-spi-1.0"
│   ├── reg = <0xe0006000 0x1000>
│   ├── interrupts = <0 26 IRQ_TYPE_LEVEL_HIGH>
│   ├── clocks = <&clkc 25, &clkc 26>
│   ├── clock-names = "ref_clk", "pclk"
│   ├── num-cs = <3>
│   └── [SPI Slave Devices...]
│
├── ps7_uart_1 (UART for Console)
│   ├── compatible = "xlnx,xuartps-1.00.a"
│   ├── reg = <0xe0001000 0x1000>
│   ├── interrupts = <0 50 IRQ_TYPE_LEVEL_HIGH>
│   ├── clocks = <&clkc 24, &clkc 45>
│   ├── clock-names = "uart_clk", "pclk"
│   ├── current-speed = <115200>
│   └── xlnx,has-modem = <0>
│
└── [Additional Peripherals: USB, Ethernet, SD/MMC, etc...]
```

## MTD Flash Partitions

### QSPI Flash Layout

```
QSPI Flash Memory (32 MB total)
┌──────────────────────────────────────┐
│ 0x00000000 - 0x000FFFFF             │
│ mtd0: QSPI-FSBL-UBOOT (1 MB)         │
│ • FSBL (First-Stage Bootloader)      │
│ • U-Boot (Second-Stage Bootloader)   │
├──────────────────────────────────────┤
│ 0x00100000 - 0x0011FFFF             │
│ mtd1: QSPI-UBOOT-ENV (128 KB)        │
│ • U-Boot environment variables       │
│ • Default command & boot settings    │
├──────────────────────────────────────┤
│ 0x00120000 - 0x001FFFFF             │
│ mtd2: QSPI-NVMFS (896 KB)            │
│ • Non-Volatile Filesystem            │
│ • EEPROM data, calibration           │
├──────────────────────────────────────┤
│ 0x00200000 - 0x01FFFFFF             │
│ mtd3: QSPI-LINUX (30 MB)             │
│ • Flattened Image Tree (FIT)         │
│   • Device Tree Blob (.dtb)          │
│   • Linux Kernel (zImage)            │
│   • Root Filesystem (rootfs.cpio.gz) │
│   • FPGA Bitstream (system_top.bit)  │
│   • MD5 checksums for verification   │
└──────────────────────────────────────┘
```

## GPIO and Peripheral Configuration

### PlutoSDR Pin Assignments

#### RF Transceiver Signals (AD9361)

| Signal | Pin | I/O | Voltage | Purpose |
|--------|-----|-----|---------|---------|
| rx_clk | L12 | In | 1.8V | RX sampling clock (61.5 MHz) |
| rx_frame | N13 | In | 1.8V | RX frame synchronization |
| rx_d[0:11] | H14-K15 | In | 1.8V | RX data (12-bit parallel) |
| tx_clk | P10 | Out | 1.8V | TX clock output |
| tx_frame | L14 | Out | 1.8V | TX frame synchronization |
| tx_d[0:11] | P15-R7 | Out | 1.8V | TX data (12-bit parallel) |

#### GPIO Control Lines

```
GPIO Bank (MIO)
├── gpio_resetb (P9)         - Active-low transceiver reset
├── gpio_en_agc (L13)        - Enable automatic gain control
├── gpio_ctl[0] (F13)        - Control line 0
├── gpio_ctl[1] (F14)        - Control line 1
├── gpio_ctl[2] (F15)        - Control line 2
├── gpio_ctl[3] (E15)        - Control line 3
│
└── gpio_status[0:7]
    ├── gpio_status[0] (L15) - Status bit 0
    ├── gpio_status[1] (N8)  - Status bit 1
    ├── gpio_status[2] (K13) - Status bit 2
    ├── gpio_status[3] (L13) - Status bit 3
    ├── gpio_status[4] (M13) - Status bit 4
    ├── gpio_status[5] (M14) - Status bit 5
    ├── gpio_status[6] (N14) - Status bit 6
    └── gpio_status[7] (N13) - Status bit 7
```

#### Communication Interfaces

```
I2C Bus (MIO):
├── iic_scl (M14)  - Serial clock (with 10K pull-up)
├── iic_sda (N14)  - Serial data (with 10K pull-up)
└── Devices:
    ├── AD9361 Configuration (~0x39)
    ├── EEPROM (~0x50)
    └── Other sensors

SPI Bus (MIO):
├── spi_clk (E11)  - Master clock
├── spi_mosi (E13) - Master out, slave in
├── spi_miso (F12) - Master in, slave out
└── spi_csn (E12)  - Chip select (active low)

UART (MIO):
├── uart_rxd       - UART receive
└── uart_txd       - UART transmit
    └── Default: 115200 baud, 8N1
```

## Boot Configuration

### U-Boot Environment Variables

Default environment extracted from compiled bootloader:

```bash
# Boot command sequence
bootcmd=run boot_config
boot_config=if test -n $m_ip; then run bootargs_ip; else run bootargs; fi; bootm 0x3000000

# Device configuration
ethaddr=00:AD:DE:AD:BE:EF
netmask=255.255.255.0
ipaddr=192.168.0.100

# MTD partition table
mtdparts=n25q256a:...

# Kernel arguments
bootargs=console=ttyPS0,115200 root=/dev/mtdblock3 rw rootfstype=squashfs

# Default IP (when no network)
m_ip=192.168.2.1
m_gw=192.168.2.254
m_netmask=255.255.255.0

# DFU mode settings
dfu_altinfo=boot ram 0x3000000 0x500000;uImage ram 0x3000000 0x500000
```

## Device Memory Map

### Physical Memory Map

```
Address Space: 32-bit (4 GB total)

0x00000000 - 0x3FFFFFFF (1 GB)
┌─────────────────────────────────────┐
│ On-Chip RAM and QSPI (CS0)          │
│                                     │
│ 0x00000000 - 0x01FFFFFF: QSPI Flash │
│ 0x02000000 - 0x03FFFFFF: Reserved  │
└─────────────────────────────────────┘

0x40000000 - 0x7FFFFFFF (1 GB)
┌─────────────────────────────────────┐
│ DDR3 Memory                         │
│                                     │
│ 0x40000000 - 0x5FFFFFFF: 512 MB    │
│ 0x60000000 - 0x7FFFFFFF: Reserved  │
└─────────────────────────────────────┘

0x80000000 - 0xBFFFFFFF (1 GB)
┌─────────────────────────────────────┐
│ PS Peripherals and I/O              │
│                                     │
│ 0xE0000000 - 0xE0100000: PS Core   │
│ • GPIO, I2C, SPI, UART, USB        │
│ • Clock/Reset Management           │
└─────────────────────────────────────┘

0xC0000000 - 0xDFFFFFFF (512 MB)
┌─────────────────────────────────────┐
│ FPGA Fabric AXI Slave Space         │
│                                     │
│ 0xC0000000 - 0xDFFFFFFF: Fabric IPs │
│ • AD9361 AXI Interface              │
│ • DMA Controllers                   │
│ • Clock Generators                  │
│ • Other custom IP cores             │
└─────────────────────────────────────┘

0xE0000000 - 0xFFFFFFFF (512 MB)
┌─────────────────────────────────────┐
│ PS Peripherals (detailed)           │
│                                     │
│ 0xE0000000: UART0                  │
│ 0xE0001000: UART1                  │
│ 0xE0004000: I2C0                   │
│ 0xE0005000: I2C1                   │
│ 0xE0006000: SPI0                   │
│ 0xE0007000: SPI1                   │
│ 0xE0009000: QSPI                   │
│ 0xE000A000: GPIO                   │
│ 0xE000D000: SD/MMC                 │
│ 0xE0100000: USB OTG                │
│ ...                                 │
│ 0xF8000000: System Level (PL to PS)│
│ ...                                 │
│ 0xFFFFFFFF: Top of address space   │
└─────────────────────────────────────┘
```

## Timing Constraints

### Device Tree Timing Specifications

```devicetree
/* Clock domains and frequency constraints */

ps7_clock {
    clock-frequency = <667000000>;      /* ARM PS clock: 667 MHz */
};

fabric_clock_0 {
    clock-frequency = <100000000>;      /* FPGA0: 100 MHz */
};

fabric_clock_1 {
    clock-frequency = <200000000>;      /* FPGA1: 200 MHz */
};

/* RF Clock (derived) */
rf_clock {
    clock-frequency = <61440000>;       /* ~61.44 MHz from AD9361 */
};

/* I2C Timing */
i2c_bus {
    i2c-max-frequency = <100000>;       /* 100 kHz standard mode */
};

/* SPI Timing */
spi_master {
    spi-max-frequency = <50000000>;     /* 50 MHz max */
};

/* UART Timing */
uart {
    current-speed = <115200>;           /* 115200 baud */
};
```

## U-Boot Flattened Image Tree (FIT)

The FIT image format combines all boot components:

```
Image Tree Source (pluto.its):
/ {
    description = "ITB PlutoSDR (ADALM-PLUTO)";

    images {
        fdt@1 { data = <DTB RevA>; };
        fdt@2 { data = <DTB RevB>; };
        fdt@3 { data = <DTB RevC>; };

        fpga@1 {
            data = <system_top.bit>;
            load = <0xF000000>;         /* FPGA bitstream load address */
        };

        linux_kernel@1 {
            data = <zImage>;
            load = <0x8000>;            /* Kernel load address */
        };

        ramdisk@1 {
            data = <rootfs.cpio.gz>;
            compression = "gzip";
        };
    };

    configurations {
        default = "config@0";

        config@0 {
            kernel = "linux_kernel@1";
            fdt = "fdt@1", "fdt@2", "fdt@3";
            ramdisk = "ramdisk@1";
        };
    };
};
```

## Calibration and EEPROM Data

### Non-Volatile Configuration Storage

```
EEPROM Layout (mtd2 - NVMFS):
├── Device Serial Number (8 bytes)
├── MAC Address (6 bytes)
├── ADC/DAC Calibration Data
│   ├── RX Gain Calibration (gain, phase per band)
│   ├── TX Attenuation Values
│   └── Temperature Coefficients
├── RF Transceiver Configuration
│   ├── AD9361 PLL Settings
│   ├── Filter Bandwidth Defaults
│   └── Gain Control Parameters
└── CRC/Validation Data
```

## Related Documentation

- [System Overview](../architecture/system-overview.md) - High-level architecture
- [Boot Sequence](../api-reference/boot-sequence.md) - Detailed boot process
- [FPGA Design](../architecture/fpga-design.md) - FPGA hardware details
