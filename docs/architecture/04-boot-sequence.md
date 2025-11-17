# PlutoSDR Firmware - Boot Sequence

## Overview

This document provides a detailed analysis of the PlutoSDR boot sequence from power-on to a fully operational Linux system.

## Complete Boot Flow Diagram

```
┌──────────────────────────────────────────────────────────────────────┐
│                    PLUTOSDR BOOT SEQUENCE                            │
└──────────────────────────────────────────────────────────────────────┘

    POWER ON
       │
       ▼
┌─────────────────────────────────────────────────────────────────────┐
│  STAGE 0: Zynq BootROM (Hardwired in Silicon)                      │
├─────────────────────────────────────────────────────────────────────┤
│  • Executed from on-chip ROM                                        │
│  • Reads boot mode pins                                             │
│  • Configures QSPI flash controller                                 │
│  • Loads boot.bin header from QSPI @ 0x0                            │
│  • Validates boot image (checksums)                                  │
│  • Loads FSBL to OCM (On-Chip Memory)                               │
│  • Jumps to FSBL entry point                                        │
│                                                                      │
│  Location: On-chip ROM                                              │
│  Duration: ~100ms                                                    │
└─────────────────────────────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────────────────────┐
│  STAGE 1: FSBL (First Stage Boot Loader)                           │
├─────────────────────────────────────────────────────────────────────┤
│  • Runs from OCM (256KB)                                            │
│  • Executes ps7_init:                                               │
│    ┌────────────────────────────────────────────────┐              │
│    │ PS7 Initialization Sequence:                   │              │
│    │ 1. MIO Pin Configuration                       │              │
│    │ 2. PLL Configuration (ARM, DDR, IO)            │              │
│    │ 3. Clock Configuration                         │              │
│    │ 4. DDR Controller Setup                        │              │
│    │ 5. DDR3 Memory Training                        │              │
│    │ 6. Peripheral Clocks Enable                    │              │
│    │ 7. Level Shifter Configuration                 │              │
│    └────────────────────────────────────────────────┘              │
│  • Reads U-Boot from boot.bin                                       │
│  • Copies U-Boot to DDR @ 0x4000000                                 │
│  • Sets up execution environment                                    │
│  • Jumps to U-Boot                                                  │
│                                                                      │
│  Location: OCM (0xFFFF0000)                                         │
│  Duration: ~500ms                                                    │
│  Flash Source: mtd0 (qspi-fsbl-uboot, 1MB)                         │
└─────────────────────────────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────────────────────┐
│  STAGE 2: U-Boot (Universal Boot Loader)                           │
├─────────────────────────────────────────────────────────────────────┤
│  • Runs from DDR @ 0x4000000                                        │
│  • Initializes:                                                      │
│    - Serial console (UART @ 115200 baud)                            │
│    - USB controller                                                  │
│    - QSPI flash driver                                              │
│    - FIT image support                                              │
│    - DFU (Device Firmware Update) mode                              │
│                                                                      │
│  • Reads environment from mtd1 (128KB @ 0x100000)                   │
│    ┌────────────────────────────────────────────────┐              │
│    │ Key Environment Variables:                     │              │
│    │ • bootcmd: Main boot command                   │              │
│    │ • bootargs: Kernel command line                │              │
│    │ • fit_load_address: FIT image load address     │              │
│    │ • qspi_boot_cmd: QSPI boot sequence            │              │
│    │ • dfu_alt_info: DFU partition mapping          │              │
│    └────────────────────────────────────────────────┘              │
│                                                                      │
│  • Boot Decision Tree:                                              │
│    ┌──────────────┐                                                 │
│    │ Check GPIO   │                                                 │
│    │ Button Press?│                                                 │
│    └──────┬───────┘                                                 │
│           │                                                          │
│    ┌──────┴───────┐                                                 │
│    │Yes        No │                                                 │
│    ▼              ▼                                                 │
│  ┌────┐      ┌─────────┐                                           │
│  │DFU │      │Normal   │                                           │
│  │Mode│      │Boot     │                                           │
│  └────┘      └─────────┘                                           │
│                   │                                                 │
│                   ▼                                                 │
│  • Normal Boot Sequence:                                            │
│    1. sf probe 0 0 0 (Initialize QSPI)                             │
│    2. sf read <addr> <offset> <size> (Read FIT image)              │
│       Source: mtd3 @ 0x200000 (30MB partition)                     │
│       Destination: DDR @ 0x2080000                                  │
│    3. bootm <fit_addr>#<config>                                    │
│       Load and verify FIT image components                          │
│                                                                      │
│  Location: DDR @ 0x4000000                                          │
│  Duration: ~2 seconds                                                │
└─────────────────────────────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────────────────────┐
│  STAGE 3: FIT Image Processing                                     │
├─────────────────────────────────────────────────────────────────────┤
│  FIT (Flattened Image Tree) Structure:                             │
│  ┌──────────────────────────────────────────────────────┐          │
│  │  pluto.itb (Multi-component boot image)              │          │
│  ├──────────────────────────────────────────────────────┤          │
│  │  • Images:                                            │          │
│  │    ┌────────────────────────────────────────┐        │          │
│  │    │ fpga@1:   system_top.bit               │        │          │
│  │    │           Load @ 0xF000000              │        │          │
│  │    │           Size: ~943KB                  │        │          │
│  │    │           MD5: Verified                 │        │          │
│  │    └────────────────────────────────────────┘        │          │
│  │    ┌────────────────────────────────────────┐        │          │
│  │    │ linux_kernel@1:  zImage                │        │          │
│  │    │           Load @ 0x8000                 │        │          │
│  │    │           Entry @ 0x8000                │        │          │
│  │    │           Size: ~4.1MB                  │        │          │
│  │    │           MD5: Verified                 │        │          │
│  │    └────────────────────────────────────────┘        │          │
│  │    ┌────────────────────────────────────────┐        │          │
│  │    │ ramdisk@1:  rootfs.cpio.gz             │        │          │
│  │    │           Type: ramdisk                 │        │          │
│  │    │           Compression: gzip             │        │          │
│  │    │           Size: ~5.3MB                  │        │          │
│  │    │           MD5: Verified                 │        │          │
│  │    └────────────────────────────────────────┘        │          │
│  │    ┌────────────────────────────────────────┐        │          │
│  │    │ fdt@1/2/3:  Device Tree Blobs          │        │          │
│  │    │           Rev A / Rev B / Rev C         │        │          │
│  │    │           Size: ~22-23KB each           │        │          │
│  │    └────────────────────────────────────────┘        │          │
│  │                                                       │          │
│  │  • Configuration Selection:                          │          │
│  │    Hardware detection → Select config@N              │          │
│  │    config@0-7, 9-10: Different HW variants           │          │
│  └──────────────────────────────────────────────────────┘          │
│                                                                      │
│  U-Boot FIT Processing:                                             │
│  1. Parse FIT header                                                │
│  2. Verify magic string: "ITB PlutoSDR (ADALM-PLUTO)"              │
│  3. Select configuration based on hardware                          │
│  4. Verify MD5 checksums for all images                            │
│  5. Load FPGA bitstream to memory                                   │
│  6. Program FPGA via PCAP (Processor Configuration Access Port)     │
│  7. Load kernel to 0x8000                                           │
│  8. Load ramdisk to memory                                          │
│  9. Load device tree to memory                                      │
│  10. Setup kernel boot arguments                                    │
│  11. Jump to kernel entry point                                     │
│                                                                      │
│  Duration: ~3 seconds                                                │
└─────────────────────────────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────────────────────┐
│  STAGE 4: FPGA Configuration                                        │
├─────────────────────────────────────────────────────────────────────┤
│  • U-Boot programs FPGA fabric via PCAP interface                   │
│  • Bitstream contains:                                              │
│    ┌────────────────────────────────────────────┐                  │
│    │ • AXI DMA Controllers (RX/TX)              │                  │
│    │ • ADC Core (cf_ad9364_adc)                 │                  │
│    │ • DAC Core (cf_ad9364_dac)                 │                  │
│    │ • AXI Interconnect                         │                  │
│    │ • GPIO Controllers                         │                  │
│    │ • I2C Controllers                          │                  │
│    └────────────────────────────────────────────┘                  │
│  • FPGA configuration: ~100ms                                       │
│  • After configuration, PL peripherals accessible via AXI bus       │
│                                                                      │
│  FPGA Memory Map (AXI Address Space):                              │
│    0x43C00000: MathWorks IP Core                                    │
│    0x79020000: ADC Core (24KB)                                      │
│    0x79024000: DAC Core (4KB)                                       │
│    0x7C400000: RX DMA (64KB)                                        │
│    0x7C420000: TX DMA (64KB)                                        │
│    0x41600000: I2C Controller (64KB)                                │
└─────────────────────────────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────────────────────┐
│  STAGE 5: Linux Kernel Initialization                              │
├─────────────────────────────────────────────────────────────────────┤
│  Kernel Boot Parameters (from U-Boot):                              │
│  ┌──────────────────────────────────────────────────────┐          │
│  │ console=ttyPS0,115200                                 │          │
│  │ root=/dev/ram0 rw                                     │          │
│  │ rootfstype=ext4                                       │          │
│  │ earlyprintk                                           │          │
│  │ clk_ignore_unused                                     │          │
│  └──────────────────────────────────────────────────────┘          │
│                                                                      │
│  Kernel Initialization Sequence:                                    │
│  1. Decompress kernel (zImage → vmlinux)                           │
│  2. Setup ARM architecture                                          │
│  3. Parse device tree                                               │
│  4. Initialize memory management                                    │
│  5. Setup SMP (Symmetric Multi-Processing)                          │
│     - CPU0: Primary core                                            │
│     - CPU1: Secondary core                                          │
│  6. Initialize core subsystems:                                     │
│     ┌────────────────────────────────────────────┐                 │
│     │ • IRQ (Interrupt Request) Controller       │                 │
│     │ • Timers and Clocksource                   │                 │
│     │ • GPIO Subsystem                           │                 │
│     │ • DMA Engine                                │                 │
│     │ • I2C Buses                                 │                 │
│     │ • SPI Buses                                 │                 │
│     │ • USB OTG Controller                        │                 │
│     │ • MTD (Flash) Subsystem                     │                 │
│     └────────────────────────────────────────────┘                 │
│  7. Mount root filesystem (ramdisk)                                 │
│  8. Initialize Industrial I/O (IIO) framework                       │
│  9. Load AD9361/AD9364 driver                                       │
│     ┌────────────────────────────────────────────┐                 │
│     │ AD9361 Driver Initialization:              │                 │
│     │ • Detect transceiver via SPI               │                 │
│     │ • Read chip ID and revision                │                 │
│     │ • Load default configuration               │                 │
│     │ • Initialize RF synthesizers (RX/TX)       │                 │
│     │ • Configure AGC (Automatic Gain Control)   │                 │
│     │ • Setup sample rate clocks                 │                 │
│     │ • Calibrate RF paths                       │                 │
│     │ • Register IIO device (/dev/iio:device*)   │                 │
│     └────────────────────────────────────────────┘                 │
│  10. Initialize DMA buffers                                         │
│  11. Start init process (/sbin/init)                                │
│                                                                      │
│  Duration: ~5 seconds                                                │
└─────────────────────────────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────────────────────┐
│  STAGE 6: Init System (SysV Init)                                  │
├─────────────────────────────────────────────────────────────────────┤
│  Init scripts execution order (from /etc/init.d/):                  │
│                                                                      │
│  S10mdev           Device manager initialization                    │
│    │ • Populate /dev with device nodes                              │
│    │ • Setup mdev hotplug                                           │
│    ▼                                                                │
│  S15watchdog       Watchdog timer setup                             │
│    │ • Start hardware watchdog                                      │
│    │ • Configure timeout                                            │
│    ▼                                                                │
│  S20urandom        Random number generator                          │
│    │ • Seed /dev/urandom                                            │
│    ▼                                                                │
│  S21misc           Miscellaneous initialization                     │
│    │ • Mount debugfs                                                │
│    │ • Mount JFFS2 persistent storage                               │
│    │ • System preparation                                           │
│    ▼                                                                │
│  S23udc            USB Device Controller gadget configuration       │
│    │ • Setup USB composite device:                                  │
│    │   ┌─────────────────────────────────────┐                     │
│    │   │ Function 1: RNDIS/NCM/ECM (Network) │                     │
│    │   │ Function 2: Mass Storage Device      │                     │
│    │   │ Function 3: ACM (Serial Console)     │                     │
│    │   │ Function 4: FunctionFS (IIO Control) │                     │
│    │   └─────────────────────────────────────┘                     │
│    │ • Set USB descriptors (VID/PID)                                │
│    │ • Enable USB gadget                                            │
│    ▼                                                                │
│  S40network        Network interface configuration                  │
│    │ • Configure loopback (lo)                                      │
│    │ • Configure USB network (usb0/usb1)                            │
│    │ • Setup IP: 192.168.2.1/24                                     │
│    │ • Generate config files:                                       │
│    │   - /etc/hostname                                              │
│    │   - /etc/network/interfaces                                    │
│    │   - /etc/udhcpd.conf (DHCP server)                             │
│    │   - /opt/config.txt (user config)                              │
│    │ • Update web interface                                         │
│    ▼                                                                │
│  S41network        Additional network services                      │
│    │ • Start DHCP server (udhcpd)                                   │
│    │ • Configure wireless if present                                │
│    ▼                                                                │
│  S45msd            Mass Storage Device management                   │
│    │ • Create FAT filesystem image                                  │
│    │ • Mount to /mnt/msd                                            │
│    │ • Populate with:                                               │
│    │   - index.html (web interface)                                 │
│    │   - LICENSE.html                                               │
│    │   - System information                                         │
│    │ • Start update.sh daemon                                       │
│    │   (monitors for .frm files)                                    │
│    ▼                                                                │
│  S50dropbear       SSH server                                       │
│    │ • Start Dropbear SSH daemon                                    │
│    │ • Listen on port 22                                            │
│    ▼                                                                │
│  S60avahi          mDNS/DNS-SD service                              │
│    │ • Start Avahi daemon                                           │
│    │ • Advertise: pluto.local                                       │
│    ▼                                                                │
│  S70iiod           IIO Daemon                                       │
│    │ • Start iiod (IIO network daemon)                              │
│    │ • Listen on port 30431                                         │
│    │ • Provide remote IIO access                                    │
│    ▼                                                                │
│  S98autostart      Execute user autorun script                      │
│    │ • Run /mnt/jffs2/autorun.sh if exists                          │
│    │ • User customization hook                                      │
│    ▼                                                                │
│                                                                      │
│  System Ready: Login prompt on serial console                       │
│  Duration: ~3 seconds                                                │
└─────────────────────────────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────────────────────┐
│                   SYSTEM FULLY OPERATIONAL                          │
├─────────────────────────────────────────────────────────────────────┤
│  Available Services:                                                │
│  • SSH:      Port 22 (root/analog)                                  │
│  • IIO:      Port 30431 (libiio network backend)                    │
│  • HTTP:     Via USB MSD (index.html)                               │
│  • mDNS:     pluto.local / sidekiqz2.local                          │
│  • Serial:   ttyPS0 @ 115200 baud                                   │
│  • USB:      Network (192.168.2.1), MSD, Serial, IIO               │
│                                                                      │
│  Total Boot Time: ~11 seconds (power-on to ready)                   │
└─────────────────────────────────────────────────────────────────────┘
```

## Boot Time Breakdown

```
┌───────────────────────────────────────────────────────────────┐
│                    Boot Time Analysis                         │
├───────────────────────────────────────────────────────────────┤
│                                                               │
│  BootROM        [####] 100ms                                  │
│  FSBL           [##########] 500ms                            │
│  U-Boot         [████████████████████] 2000ms                 │
│  FPGA Config    [##] 100ms                                    │
│  Linux Kernel   [██████████████████████████] 5000ms          │
│  Init Scripts   [██████████] 3000ms                           │
│                 └─────────────────────────────────┘           │
│                 0s    2s    4s    6s    8s   10s  11s         │
│                                                               │
│  Total: ~11 seconds                                           │
└───────────────────────────────────────────────────────────────┘
```

## Memory Layout During Boot

```
┌────────────────────────────────────────────────────────────────┐
│            DDR3 Memory Layout (512MB Total)                    │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  0x00000000 ┌──────────────────────────────────┐              │
│             │  Reserved / NULL Pointer Trap    │              │
│  0x00008000 ├──────────────────────────────────┤              │
│             │  Linux Kernel (zImage)           │              │
│             │  Load Address: 0x8000            │              │
│             │  Entry Point: 0x8000             │              │
│             │  Size: ~4.1MB                    │              │
│  0x00500000 ├──────────────────────────────────┤              │
│             │  Device Tree Blob                │              │
│             │  Size: ~23KB                     │              │
│  0x00520000 ├──────────────────────────────────┤              │
│             │  Initial Ramdisk                 │              │
│             │  Size: ~5.3MB                    │              │
│  0x01000000 ├──────────────────────────────────┤              │
│             │  Kernel Runtime Memory           │              │
│             │  - Page Tables                   │              │
│             │  - Kernel Stack                  │              │
│             │  - DMA Buffers                   │              │
│  0x02080000 ├──────────────────────────────────┤              │
│             │  FIT Image Load Area             │              │
│             │  (Temporary during boot)         │              │
│             │  Size: ~11MB                     │              │
│  0x03000000 ├──────────────────────────────────┤              │
│             │  CMA (Contiguous Memory Alloc)   │              │
│             │  256MB Reserved for:             │              │
│             │  - DMA Transfers                 │              │
│             │  - IIO Buffers                   │              │
│             │  - FPGA Data Streaming           │              │
│  0x13000000 ├──────────────────────────────────┤              │
│             │  User Space Memory               │              │
│             │  - Application Code/Data         │              │
│             │  - Shared Libraries              │              │
│             │  - Heap                          │              │
│             │  ~240MB                          │              │
│  0x1FFFFFFF └──────────────────────────────────┘              │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

## DFU Boot Mode

When the button is pressed during power-on or reset:

```
┌─────────────────────────────────────────────────────────────┐
│              DFU (Device Firmware Update) Mode              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  FSBL → U-Boot → Check GPIO Button                         │
│                         │                                   │
│                         ▼                                   │
│                  [Button Pressed]                           │
│                         │                                   │
│                         ▼                                   │
│              ┌─────────────────────┐                        │
│              │  Enter DFU Mode     │                        │
│              └─────────────────────┘                        │
│                         │                                   │
│                         ▼                                   │
│              ┌─────────────────────┐                        │
│              │  Initialize USB OTG │                        │
│              │  as DFU Device      │                        │
│              └─────────────────────┘                        │
│                         │                                   │
│                         ▼                                   │
│              ┌─────────────────────┐                        │
│              │  USB Enumeration:   │                        │
│              │  VID: 0x0456        │                        │
│              │  PID: 0xb674        │                        │
│              │  (DFU mode PID)     │                        │
│              └─────────────────────┘                        │
│                         │                                   │
│                         ▼                                   │
│              ┌─────────────────────┐                        │
│              │  Wait for DFU       │                        │
│              │  Commands from Host │                        │
│              │                     │                        │
│              │  Supported:         │                        │
│              │  - firmware.dfu     │                        │
│              │  - boot.dfu         │                        │
│              │  - uboot-env.dfu    │                        │
│              └─────────────────────┘                        │
│                         │                                   │
│                         ▼                                   │
│              ┌─────────────────────┐                        │
│              │  Receive Firmware   │                        │
│              │  Write to Flash     │                        │
│              └─────────────────────┘                        │
│                         │                                   │
│                         ▼                                   │
│              ┌─────────────────────┐                        │
│              │  DFU Detach         │                        │
│              │  Reboot Device      │                        │
│              └─────────────────────┘                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## Key Boot Parameters

### U-Boot Environment Variables

| Variable | Value | Purpose |
|----------|-------|---------|
| `bootcmd` | sf probe; sf read... | Main boot command |
| `bootargs` | console=ttyPS0,115200... | Kernel parameters |
| `baudrate` | 115200 | Serial console baud rate |
| `fit_load_address` | 0x2080000 | FIT image load address |
| `dfu_alt_info` | firmware.dfu... | DFU partition map |
| `ethaddr` | 00:xx:xx:xx:xx:xx | MAC address |
| `ipaddr` | 192.168.2.1 | Device IP address |

### Kernel Command Line

```
console=ttyPS0,115200
root=/dev/ram0
rw
rootfstype=ext4
earlyprintk
clk_ignore_unused
uio_pdrv_genirq.of_id=generic-uio
```

## Error Handling and Recovery

### Boot Failure Scenarios

1. **Corrupted FSBL/U-Boot**
   - Symptom: No serial output, device doesn't enumerate
   - Recovery: JTAG programming required
   - Use: `scripts/run-xsdb.tcl` to bootstrap U-Boot

2. **Corrupted Firmware (mtd3)**
   - Symptom: U-Boot works, kernel fails to load
   - Recovery: DFU mode update
   - Button + Power → DFU mode → Flash new firmware

3. **Bad U-Boot Environment**
   - Symptom: U-Boot shell appears, doesn't auto-boot
   - Recovery: Reflash uboot-env.dfu or manually set environment

4. **Watchdog Reset**
   - Symptom: Device reboots during init
   - Cause: Init script hangs
   - Recovery: Disable watchdog or fix init scripts

## Performance Optimization

Boot time can be reduced by:

1. **Kernel Configuration**: Disable unused drivers (~1-2s savings)
2. **Init Scripts**: Optimize S* scripts execution (~1s savings)
3. **Ramdisk Size**: Reduce root filesystem size (~0.5s savings)
4. **U-Boot Delay**: Set bootdelay=0 (already optimized)

## Related Documentation

- [Hardware Architecture](02-hardware-architecture.md)
- [U-Boot Configuration](../components/03-u-boot.md)
- [Linux Kernel](../components/02-linux-kernel.md)
- [Buildroot](../components/01-buildroot.md)
