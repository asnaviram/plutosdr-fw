# PlutoSDR Firmware - Development Workflow

## Overview

This document provides comprehensive workflows for developing, testing, and debugging PlutoSDR firmware modifications.

## Development Environment Setup

### Prerequisites Installation

```
┌──────────────────────────────────────────────────────────┐
│           Development Environment Setup                  │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  STEP 1: Install System Dependencies                    │
│  ┌────────────────────────────────────────┐             │
│  │ sudo apt-get update                    │             │
│  │ sudo apt-get install -y \              │             │
│  │   git build-essential fakeroot \       │             │
│  │   libncurses5-dev libssl-dev ccache \  │             │
│  │   dfu-util u-boot-tools \              │             │
│  │   device-tree-compiler mtools \        │             │
│  │   bc python cpio zip unzip rsync \     │             │
│  │   file wget curl                       │             │
│  └────────────────────────────────────────┘             │
│                                                          │
│  STEP 2: Install Xilinx Vivado (Optional)               │
│  ┌────────────────────────────────────────┐             │
│  │ Download from xilinx.com                │             │
│  │ Version: 2023.2 (recommended)           │             │
│  │ Install to: /opt/Xilinx/Vivado/2023.2  │             │
│  │                                         │             │
│  │ Components needed:                      │             │
│  │ ├─ Vivado Design Suite                 │             │
│  │ ├─ Zynq-7000 device support            │             │
│  │ └─ SDK/Vitis (for FSBL/XSCT)           │             │
│  └────────────────────────────────────────┘             │
│                                                          │
│  STEP 3: Clone Repository                               │
│  ┌────────────────────────────────────────┐             │
│  │ git clone --recursive \                 │             │
│  │   https://github.com/analogdevicesinc\  │             │
│  │   /plutosdr-fw.git                      │             │
│  │ cd plutosdr-fw                          │             │
│  └────────────────────────────────────────┘             │
│                                                          │
│  STEP 4: Setup Environment Variables                    │
│  ┌────────────────────────────────────────┐             │
│  │ export VIVADO_SETTINGS=/opt/Xilinx/    │             │
│  │        Vivado/2023.2/settings64.sh     │             │
│  │ export CROSS_COMPILE=\                  │             │
│  │        arm-linux-gnueabihf-             │             │
│  │ export TARGET=pluto                     │             │
│  │                                         │             │
│  │ # Add to ~/.bashrc for persistence      │             │
│  └────────────────────────────────────────┘             │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

## Complete Build Workflow

```
┌──────────────────────────────────────────────────────────────────────┐
│                      COMPLETE BUILD WORKFLOW                         │
├──────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  Initial Build                                                        │
│  ============                                                         │
│  $ cd plutosdr-fw                                                     │
│  $ make                                                               │
│      │                                                                │
│      ├─ Builds toolchain (~5 min)                                    │
│      ├─ Compiles U-Boot (~3 min)                                     │
│      ├─ Builds Linux kernel (~15 min)                                │
│      ├─ Builds FPGA or downloads XSA (~30 sec or 1-2 hours)          │
│      ├─ Builds Buildroot rootfs (~30 min)                            │
│      └─ Packages firmware (~1 min)                                   │
│                                                                       │
│  Output: build/pluto.frm, build/pluto.dfu                            │
│  Total Time: ~55 min (without FPGA) or ~2.5 hours (with FPGA)        │
│                                                                       │
├───────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  Incremental Development Workflows                                    │
│  =================================                                    │
│                                                                       │
│  1. KERNEL DEVELOPMENT                                                │
│  ┌─────────────────────────────────────────────────┐                 │
│  │ Modify kernel source:                            │                 │
│  │   linux/drivers/iio/adc/ad9361.c                │                 │
│  │                                                  │                 │
│  │ Rebuild kernel only:                             │                 │
│  │   make -C linux ARCH=arm \                       │                 │
│  │        CROSS_COMPILE=arm-linux-gnueabihf- \      │                 │
│  │        -j$(nproc) zImage                         │                 │
│  │                                                  │                 │
│  │ Rebuild firmware image:                          │                 │
│  │   rm build/pluto.itb build/pluto.frm             │                 │
│  │   make build/pluto.frm                           │                 │
│  │                                                  │                 │
│  │ Test (without flashing):                         │                 │
│  │   ./download_and_test.sh                         │                 │
│  │                                                  │                 │
│  │ Time: ~15 min (kernel compile) + ~1 min (package)│                 │
│  └─────────────────────────────────────────────────┘                 │
│                                                                       │
│  2. DEVICE TREE DEVELOPMENT                                           │
│  ┌─────────────────────────────────────────────────┐                 │
│  │ Modify device tree:                              │                 │
│  │   linux/arch/arm/boot/dts/zynq-pluto-sdr.dtsi   │                 │
│  │                                                  │                 │
│  │ Rebuild DTB:                                     │                 │
│  │   make -C linux ARCH=arm \                       │                 │
│  │        CROSS_COMPILE=arm-linux-gnueabihf- \      │                 │
│  │        zynq-pluto-sdr-revb.dtb                   │                 │
│  │                                                  │                 │
│  │ Post-process:                                    │                 │
│  │   dtc -I dtb -O dts linux/arch/arm/boot/dts/\   │                 │
│  │       zynq-pluto-sdr-revb.dtb | \                │                 │
│  │   sed 's/axi {/amba {/g' | \                     │                 │
│  │   dtc -I dts -O dtb -o \                         │                 │
│  │       build/zynq-pluto-sdr-revb.dtb              │                 │
│  │                                                  │                 │
│  │ Rebuild firmware:                                │                 │
│  │   rm build/pluto.itb build/pluto.frm             │                 │
│  │   make build/pluto.frm                           │                 │
│  │                                                  │                 │
│  │ Time: ~30 seconds                                │                 │
│  └─────────────────────────────────────────────────┘                 │
│                                                                       │
│  3. U-BOOT DEVELOPMENT                                                │
│  ┌─────────────────────────────────────────────────┐                 │
│  │ Modify U-Boot:                                   │                 │
│  │   u-boot-xlnx/common/main.c                      │                 │
│  │                                                  │                 │
│  │ Rebuild U-Boot:                                  │                 │
│  │   make -C u-boot-xlnx ARCH=arm \                 │                 │
│  │        CROSS_COMPILE=arm-linux-gnueabihf-        │                 │
│  │   cp u-boot-xlnx/u-boot build/u-boot.elf         │                 │
│  │                                                  │                 │
│  │ Rebuild boot image:                              │                 │
│  │   rm build/boot.bin build/boot.frm               │                 │
│  │   make build/boot.frm                            │                 │
│  │                                                  │                 │
│  │ Flash bootloader (CAREFUL!):                     │                 │
│  │   make dfu-sf-uboot                              │                 │
│  │                                                  │                 │
│  │ Time: ~3 minutes                                 │                 │
│  └─────────────────────────────────────────────────┘                 │
│                                                                       │
│  4. ROOTFS/USERSPACE DEVELOPMENT                                      │
│  ┌─────────────────────────────────────────────────┐                 │
│  │ Modify init scripts:                             │                 │
│  │   buildroot/board/pluto/S40network               │                 │
│  │                                                  │                 │
│  │ Modify web interface:                            │                 │
│  │   buildroot/board/pluto/msd/index.html           │                 │
│  │                                                  │                 │
│  │ Rebuild rootfs:                                  │                 │
│  │   rm buildroot/output/images/rootfs.cpio.gz      │                 │
│  │   make -C buildroot all                          │                 │
│  │   cp buildroot/output/images/rootfs.cpio.gz \    │                 │
│  │      build/rootfs.cpio.gz                        │                 │
│  │                                                  │                 │
│  │ Rebuild firmware:                                │                 │
│  │   rm build/pluto.itb build/pluto.frm             │                 │
│  │   make build/pluto.frm                           │                 │
│  │                                                  │                 │
│  │ Time: ~5 minutes                                 │                 │
│  └─────────────────────────────────────────────────┘                 │
│                                                                       │
│  5. FPGA/HDL DEVELOPMENT                                              │
│  ┌─────────────────────────────────────────────────┐                 │
│  │ Modify HDL design:                               │                 │
│  │   hdl/projects/pluto/system_top.v                │                 │
│  │                                                  │                 │
│  │ Rebuild FPGA:                                    │                 │
│  │   source $VIVADO_SETTINGS                        │                 │
│  │   make -C hdl/projects/pluto                     │                 │
│  │   cp hdl/projects/pluto/pluto.sdk/\              │                 │
│  │      system_top.xsa build/                       │                 │
│  │                                                  │                 │
│  │ Extract bitstream:                               │                 │
│  │   unzip -p build/system_top.xsa \                │                 │
│  │      system_top.bit > build/system_top.bit       │                 │
│  │                                                  │                 │
│  │ Rebuild firmware:                                │                 │
│  │   rm build/pluto.itb build/pluto.frm             │                 │
│  │   make build/pluto.frm                           │                 │
│  │                                                  │                 │
│  │ Time: ~1-2 hours (synthesis + implementation)     │                 │
│  └─────────────────────────────────────────────────┘                 │
│                                                                       │
└───────────────────────────────────────────────────────────────────────┘
```

## Testing Workflows

### 1. RAM Boot Testing (Non-Destructive)

```
┌──────────────────────────────────────────────────────────┐
│              RAM Boot Testing Workflow                   │
│         (Tests firmware without flashing)                │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  Prerequisites:                                          │
│  ├─ PlutoSDR connected via USB                          │
│  ├─ SSH access configured (192.168.2.1)                 │
│  ├─ sshpass installed                                   │
│  └─ dfu-util installed                                  │
│                                                          │
│  Workflow:                                               │
│  ┌────────────────────────────────────────┐             │
│  │ 1. Device Running Normally              │             │
│  │    IP: 192.168.2.1                      │             │
│  │    User: root / Pass: analog            │             │
│  │         │                               │             │
│  │         ▼                               │             │
│  │ 2. SSH Command: device_reboot ram       │             │
│  │    ├─ Device reboots to RAM mode        │             │
│  │    └─ U-Boot waits in DFU mode          │             │
│  │         │                               │             │
│  │         ▼                               │             │
│  │ 3. Wait for USB Re-enumeration          │             │
│  │    ├─ Old device: 0x0456:0xb673         │             │
│  │    └─ New device: 0x0456:0xb674 (DFU)   │             │
│  │         │                               │             │
│  │         ▼                               │             │
│  │ 4. Upload Firmware via DFU              │             │
│  │    dfu-util -D build/pluto.dfu \        │             │
│  │             -a firmware.dfu              │             │
│  │    ├─ Firmware loaded to RAM             │             │
│  │    └─ Flash contents unchanged           │             │
│  │         │                               │             │
│  │         ▼                               │             │
│  │ 5. Device Boots New Firmware            │             │
│  │    ├─ Test new features                 │             │
│  │    ├─ Verify functionality               │             │
│  │    └─ Check logs: dmesg                  │             │
│  │         │                               │             │
│  │         ▼                               │             │
│  │ 6. Power Cycle to Restore               │             │
│  │    └─ Device boots original flash image │             │
│  └────────────────────────────────────────┘             │
│                                                          │
│  Automated Script:                                       │
│  $ ./download_and_test.sh                                │
│                                                          │
│  Advantages:                                             │
│  ✓ Non-destructive (flash unchanged)                    │
│  ✓ Fast iteration (~30 seconds)                         │
│  ✓ Easy rollback (power cycle)                          │
│  ✓ Safe for experimentation                             │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

### 2. Permanent Flash Update

```
┌──────────────────────────────────────────────────────────┐
│           Permanent Flash Update Methods                 │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  METHOD A: USB Mass Storage Device                      │
│  ┌────────────────────────────────────────┐             │
│  │ 1. Connect PlutoSDR via USB             │             │
│  │    ├─ Device appears as USB drive       │             │
│  │    └─ Mount point: /media/PlutoSDR      │             │
│  │                                         │             │
│  │ 2. Copy firmware to device              │             │
│  │    cp build/pluto.frm /media/PlutoSDR/  │             │
│  │    ├─ update.sh daemon detects file     │             │
│  │    ├─ Validates MD5 checksum            │             │
│  │    ├─ Checks magic string               │             │
│  │    └─ LED blinks during update          │             │
│  │                                         │             │
│  │ 3. Wait for completion (~30 seconds)    │             │
│  │    ├─ File deleted when done            │             │
│  │    └─ Device auto-reboots               │             │
│  │                                         │             │
│  │ 4. Verify new firmware                  │             │
│  │    ├─ Check version: cat /opt/VERSION   │             │
│  │    └─ Test functionality                │             │
│  └────────────────────────────────────────┘             │
│                                                          │
│  METHOD B: DFU Mode                                      │
│  ┌────────────────────────────────────────┐             │
│  │ 1. Enter DFU Mode                       │             │
│  │    ├─ Hold button during power-on       │             │
│  │    OR                                   │             │
│  │    └─ SSH: device_reboot sf             │             │
│  │                                         │             │
│  │ 2. Verify DFU enumeration               │             │
│  │    dfu-util -l                          │             │
│  │    └─ Should show 0x0456:0xb674         │             │
│  │                                         │             │
│  │ 3. Flash firmware                       │             │
│  │    dfu-util -D build/pluto.dfu \        │             │
│  │             -a firmware.dfu              │             │
│  │    dfu-util -e (eject/reboot)           │             │
│  │                                         │             │
│  │ 4. Device reboots with new firmware     │             │
│  └────────────────────────────────────────┘             │
│                                                          │
│  METHOD C: Makefile Targets                             │
│  ┌────────────────────────────────────────┐             │
│  │ make dfu-pluto                          │             │
│  │   └─ Uploads firmware only              │             │
│  │                                         │             │
│  │ make dfu-all  (CAUTION!)                │             │
│  │   ├─ Uploads firmware                   │             │
│  │   ├─ Uploads bootloader                 │             │
│  │   └─ Uploads U-Boot environment         │             │
│  │       (Requires user confirmation)       │             │
│  └────────────────────────────────────────┘             │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

### 3. JTAG Recovery and Initial Programming

```
┌──────────────────────────────────────────────────────────┐
│          JTAG Programming Workflow                       │
│      (For recovery or factory programming)               │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  Prerequisites:                                          │
│  ├─ Xilinx Platform Cable USB II (or compatible)        │
│  ├─ JTAG connection to PlutoSDR header                  │
│  ├─ Vivado/XSDB tools installed                         │
│  └─ jtag-bootstrap package built                        │
│                                                          │
│  Workflow:                                               │
│  ┌────────────────────────────────────────┐             │
│  │ 1. Build JTAG bootstrap package         │             │
│  │    make jtag-bootstrap                  │             │
│  │    Output: plutosdr-jtag-bootstrap-\    │             │
│  │            v0.39.zip                    │             │
│  │         │                               │             │
│  │         ▼                               │             │
│  │ 2. Extract package                      │             │
│  │    unzip plutosdr-jtag-bootstrap-*.zip  │             │
│  │    Contents:                            │             │
│  │    ├─ u-boot.elf.strip                  │             │
│  │    ├─ ps7_init.tcl                      │             │
│  │    ├─ system_top.bit                    │             │
│  │    ├─ run.tcl (XMD)                     │             │
│  │    └─ run-xsdb.tcl (XSDB)               │             │
│  │         │                               │             │
│  │         ▼                               │             │
│  │ 3. Connect JTAG hardware                │             │
│  │    ├─ Power on PlutoSDR                 │             │
│  │    └─ Connect Platform Cable            │             │
│  │         │                               │             │
│  │         ▼                               │             │
│  │ 4. Run bootstrap script                 │             │
│  │    source $VIVADO_SETTINGS              │             │
│  │    xsdb run-xsdb.tcl                    │             │
│  │    ├─ Connect to JTAG chain             │             │
│  │    ├─ Reset ARM cores                   │             │
│  │    ├─ Run ps7_init (DDR, clocks)        │             │
│  │    ├─ Load U-Boot to RAM                │             │
│  │    └─ Start execution                   │             │
│  │         │                               │             │
│  │         ▼                               │             │
│  │ 5. U-Boot running from RAM              │             │
│  │    ├─ Enter DFU mode                    │             │
│  │    └─ Flash firmware via dfu-util       │             │
│  │                                         │             │
│  │ 6. Permanent firmware in flash          │             │
│  │    └─ Remove JTAG, power cycle          │             │
│  └────────────────────────────────────────┘             │
│                                                          │
│  Use Cases:                                              │
│  • Corrupted bootloader recovery                        │
│  • Factory programming (new boards)                     │
│  • Development debugging                                │
│  • FPGA bitstream testing                               │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

## Debugging Workflows

### Serial Console Debugging

```
┌──────────────────────────────────────────────────────────┐
│             Serial Console Access                        │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  Hardware Serial (UART):                                 │
│  ┌────────────────────────────────────────┐             │
│  │ Connection:                             │             │
│  │  PlutoSDR   │  USB-Serial Adapter       │             │
│  │  ─────────────────────────────────      │             │
│  │  UART TX    →  RX                       │             │
│  │  UART RX    ←  TX                       │             │
│  │  GND        ─  GND                      │             │
│  │                                         │             │
│  │ Settings: 115200 baud, 8N1              │             │
│  │                                         │             │
│  │ Access:                                 │             │
│  │   sudo screen /dev/ttyUSB0 115200       │             │
│  │   OR                                    │             │
│  │   sudo minicom -D /dev/ttyUSB0          │             │
│  └────────────────────────────────────────┘             │
│                                                          │
│  USB Serial (Gadget):                                    │
│  ┌────────────────────────────────────────┐             │
│  │ Automatic when USB connected            │             │
│  │ Device: /dev/ttyACM0 (Linux/Mac)        │             │
│  │                                         │             │
│  │ Access:                                 │             │
│  │   sudo screen /dev/ttyACM0 115200       │             │
│  └────────────────────────────────────────┘             │
│                                                          │
│  Boot Messages Captured:                                 │
│  ├─ BootROM output (minimal)                            │
│  ├─ FSBL initialization                                 │
│  ├─ U-Boot startup and environment                      │
│  ├─ Kernel boot messages                                │
│  ├─ Init script execution                               │
│  └─ Login prompt                                        │
│                                                          │
│  Useful Commands:                                        │
│  # View kernel log                                       │
│  dmesg | less                                            │
│                                                          │
│  # Monitor IIO devices                                   │
│  cat /sys/kernel/debug/iio/iio:device2/direct_reg_access│
│                                                          │
│  # Check DMA status                                      │
│  cat /sys/kernel/debug/dma-axi-dmac/*/stats             │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

### Network Debugging (SSH)

```
┌──────────────────────────────────────────────────────────┐
│                  SSH Debugging Access                    │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  Connect:                                                │
│    ssh root@192.168.2.1                                  │
│    Password: analog                                      │
│                                                          │
│  System Information:                                     │
│    cat /opt/VERSION         # Firmware version           │
│    uname -a                 # Kernel version             │
│    free -m                  # Memory usage               │
│    top                      # Process list               │
│    df -h                    # Disk space                 │
│                                                          │
│  IIO Debugging:                                          │
│    iio_info                 # List IIO devices           │
│    iio_attr -C              # List IIO channels          │
│    iio_attr -c ad9361-phy \ # Read attribute             │
│            voltage0 \                                    │
│            hardwaregain                                  │
│                                                          │
│  Network Debugging:                                      │
│    ifconfig                 # Network interfaces         │
│    netstat -tulpn           # Listening ports            │
│    ping -c 4 192.168.2.10   # Test connectivity          │
│                                                          │
│  RF Configuration:                                       │
│    cat /sys/bus/iio/devices/iio:device2/\               │
│        out_altvoltage0_RX_LO_frequency                   │
│    echo 2400000000 > /sys/bus/iio/devices/\             │
│         iio:device2/out_altvoltage0_RX_LO_frequency      │
│                                                          │
│  Log Files:                                              │
│    dmesg                    # Kernel messages            │
│    journalctl               # System logs (if systemd)   │
│    /var/log/                # Log directory              │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

## Git Workflow

```
┌──────────────────────────────────────────────────────────┐
│              Git Development Workflow                    │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  1. Create Feature Branch                                │
│     git checkout -b feature/my-new-feature               │
│                                                          │
│  2. Make Changes                                         │
│     # Edit files                                         │
│     vim linux/drivers/iio/adc/ad9361.c                   │
│                                                          │
│  3. Build and Test                                       │
│     make                                                 │
│     ./download_and_test.sh                               │
│                                                          │
│  4. Commit Changes                                       │
│     git add linux/drivers/iio/adc/ad9361.c               │
│     git commit -m "iio: ad9361: Add XYZ feature"         │
│                                                          │
│  5. Handle Submodules                                    │
│     # If you modified a submodule:                       │
│     cd linux                                             │
│     git add drivers/iio/adc/ad9361.c                     │
│     git commit -m "iio: ad9361: Add XYZ feature"         │
│     cd ..                                                │
│     git add linux  # Update submodule reference          │
│     git commit -m "linux: Update to include XYZ"         │
│                                                          │
│  6. Push and Create Pull Request                         │
│     git push origin feature/my-new-feature               │
│     # Create PR on GitHub                                │
│                                                          │
│  Submodule Management:                                   │
│    git submodule update --init --recursive  # Init all   │
│    git submodule update --remote            # Update all │
│    make git-update-all                      # Convenience│
│                                                          │
└──────────────────────────────────────────────────────────┘
```

## Common Development Tasks

### Adding a New Init Script

```bash
# 1. Create script
vim buildroot/board/pluto/S99mycustom

# 2. Mark executable
chmod +x buildroot/board/pluto/S99mycustom

# 3. Reference in post-build.sh
vim buildroot/board/pluto/post-build.sh
# Add: cp board/pluto/S99mycustom ${TARGET_DIR}/etc/init.d/

# 4. Rebuild rootfs
make -C buildroot all

# 5. Test
./download_and_test.sh
```

### Modifying Web Interface

```bash
# 1. Edit HTML
vim buildroot/board/pluto/msd/index.html

# 2. Rebuild (fast, no recompile)
make -C buildroot all

# 3. Test
make dfu-ram  # or ./download_and_test.sh
```

### Adding Kernel Module

```bash
# 1. Add driver source
cp my_driver.c linux/drivers/iio/adc/

# 2. Modify Kconfig
vim linux/drivers/iio/adc/Kconfig
# Add config option

# 3. Modify Makefile
vim linux/drivers/iio/adc/Makefile
# Add: obj-y += my_driver.o

# 4. Enable in defconfig
vim linux/arch/arm/configs/zynq_pluto_defconfig
# Add: CONFIG_MY_DRIVER=y

# 5. Rebuild kernel
make -C linux ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- \
     zynq_pluto_defconfig
make -C linux ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- \
     -j$(nproc) zImage
```

## Related Documentation

- [Testing Workflow](02-testing-workflow.md)
- [Release Workflow](03-release-workflow.md)
- [Build Flow](../build-system/03-build-flow.md)
- [Components](../components/)
