# Firmware Deployment Methods

## Overview

PlutoSDR firmware can be deployed using multiple methods, each suited for different scenarios:

```
Deployment Methods:
├─ USB Mass Storage Device (MSD)
│  └─ Drag-and-drop firmware update
│  └─ Most user-friendly
│
├─ Device Firmware Update (DFU)
│  └─ Command-line dfu-util
│  └─ Programmatic/scripted updates
│
├─ JTAG Bootstrap
│  └─ Direct FPGA programming
│  └─ Recovery/initial setup
│
└─ SSH + DFU (automated testing)
   └─ Combined with device_reboot command
   └─ CI/CD integration
```

## Method 1: USB Mass Storage Device (MSD)

### Overview

The simplest firmware update method. The device appears as a USB drive when connected in normal (SDR) mode.

### Process

```
Step 1: Build Firmware
        └─ build/pluto.frm
        └─ build/boot.frm (optional, for bootloader update)

Step 2: Connect Device
        ├─ Plug PlutoSDR into USB port
        ├─ Wait for device enumeration
        └─ Should appear as removable storage

Step 3: Device Auto-Detection
        ├─ Device checks for .frm files
        ├─ Auto-mounts as USB drive
        ├─ Shows firmware update prompt (LED blink)
        └─ Approximately 2-3 seconds

Step 4: Copy Firmware File
        ├─ Mount point: /media/[username]/ADALM-PLUTO/
        ├─ Copy: cp build/pluto.frm /media/[username]/ADALM-PLUTO/
        ├─ Or drag-and-drop in file explorer
        └─ File should copy immediately

Step 5: Wait for Update
        ├─ Device detects new firmware
        ├─ LED blinks during programming
        ├─ QSPI flash write: ~30-60 seconds
        ├─ Device reboots automatically
        └─ LED stabilizes when complete

Step 6: Verify
        ├─ Device re-enumerates
        ├─ Device no longer appears as USB drive
        ├─ USB IIO interface available
        └─ SSH: ssh root@192.168.2.1
           Password: analog
```

### File Format Details

```
FRM File Structure:
┌────────────────────────────────────────┐
│ Header (32 bytes)                      │
├────────────────────────────────────────┤
│ Firmware Data (variable length)        │
│ • Flattened Image Tree (FIT)          │
│ • Contains: DTB + Kernel + Rootfs     │
│ • Contains: FPGA bitstream            │
├────────────────────────────────────────┤
│ CRC32 Checksum (4 bytes)              │
├────────────────────────────────────────┤
│ MD5 Hash (16 bytes)                   │
├────────────────────────────────────────┤
│ DFU Suffix (16 bytes)                 │
│ • Device VID/PID: 0456:b673           │
│ • Firmware revision                   │
│ • Signature: "UFD" (0x55, 0x46, 0x44) │
└────────────────────────────────────────┘
```

### Advantages
- User-friendly (no special tools needed)
- Works on Windows, Mac, Linux
- No command-line knowledge required
- Automatic error detection and retry

### Limitations
- Slower than DFU (USB 2.0 storage speed)
- Requires device to be in normal mode
- No progress feedback during update
- Can't update bootloader easily

## Method 2: Device Firmware Update (DFU)

### Overview

DFU mode is specifically designed for firmware updates. The device exposes DFU endpoints for rapid, programmatic updates.

### Enable DFU Mode

```
On PlutoSDR:
1. Power cycle device (unplug + replug USB)
2. Or SSH: ssh root@192.168.2.1 → reboot
3. Device boots with DFU enabled if:
   └─ boot.bin in QSPI is valid
   └─ U-Boot detects DFU request (button or env)

Detect DFU Device:
$ dfu-util -l

Output shows:
Found DFU: [0456:b674] ver=0100, devnum=X, cfg=1, intf=0, alt=0 ...
(Note: Product ID 0xb674 indicates DFU mode)
```

### Update via DFU

```bash
#!/bin/bash
# Script: update-via-dfu.sh

# 1. Verify firmware file
if [ ! -f build/pluto.dfu ]; then
    echo "Error: build/pluto.dfu not found"
    exit 1
fi

# 2. Verify device is in DFU mode
dfu-util -l | grep -q "0456:b674" || {
    echo "Error: No DFU device found"
    echo "Please put device in DFU mode"
    exit 1
}

# 3. Download firmware via DFU
echo "Updating firmware..."
dfu-util -d 0456:b674 -D build/pluto.dfu -R

# -d 0456:b674 = Device VID:PID
# -D = Download to device
# -R = Reset device after transfer

# Output:
# dfu-util: Invalid DFU suffix signature
# Continuing anyway...
# Opening DFU capable USB device...
# ID 0456:b674
# Run-time device DFU version ...
# Claiming USB DFU Interface...
# Setting Alternate Setting ...
# Determining device status...
# DFU mode device DFU version ...
# DFU State/Status (Transport): dfuIDLE
# Download [=========================] 100%  11582464 bytes
# Download done.
# DFU State/Status (Transport): dfuDnload-Idle
# state(7) = dfuDnload-Idle
# Resetting USB to switch back to runtime mode

echo "Firmware update complete!"
echo "Device rebooting..."
sleep 5

# 4. Verify success
if dfu-util -l 2>&1 | grep -q "0456:b673"; then
    echo "Device re-enumerated in normal mode!"
    echo "Firmware update successful"
else
    echo "Warning: Device not yet ready"
fi
```

### DFU Targets (Advanced)

```bash
# Update FPGA bitstream only (mtd3)
dfu-util -d 0456:b674 -a 3,0 -D build/pluto.dfu

# Update bootloader (mtd0)
dfu-util -d 0456:b674 -a 0,0 -D build/boot.dfu

# Update U-Boot environment (mtd1)
dfu-util -d 0456:b674 -a 1,0 -D build/uboot-env.dfu

# Note: Partition numbers may vary by firmware version
```

### Advantages
- Faster than MSD (DFU optimized)
- Programmatic/scriptable
- Supports partial updates
- Better error handling
- Used by Linux IIO drivers

### Limitations
- Requires dfu-util tool
- Device must be in DFU mode
- No visual feedback during transfer
- Complex command-line syntax

## Method 3: JTAG Bootstrap

### Overview

JTAG provides direct access to the FPGA for recovery, debugging, or initial programming.

### Requirements

```
Hardware:
├─ JTAG cable (Xilinx Platform Cable USB, compatible)
├─ USB connection to development PC
└─ JTAG header on PlutoSDR

Software:
├─ Vivado or XSDT (Xilinx tools)
├─ Drivers for JTAG cable
└─ JTAG bootstrap scripts (run.tcl or run-xsdb.tcl)
```

### Process

```bash
#!/bin/bash
# JTAG Bootstrap Process

# Step 1: Start XSDB tool
# (XSDB = Xilinx System Debug Tool)
cd plutosdr-fw/scripts

# Step 2: Run bootstrap script
xsdb run-xsdb.tcl
# Script executes:
# ├─ connect             (connect to JTAG)
# ├─ target 2            (select ARM target)
# ├─ rst                 (reset)
# ├─ source ps7_init.tcl (load HW init)
# ├─ ps7_init            (initialize PS7)
# ├─ ps7_post_config     (post config)
# ├─ dow u-boot.elf      (download u-boot)
# └─ con                 (continue/run)

# Step 3: U-Boot runs in RAM
# Access via:
# ├─ Serial console (115200 baud)
# └─ Or telnet localhost 3121 (XSDB default)

# Step 4: Program QSPI Flash from U-Boot
# Use sf (SPI Flash) commands:
# sf probe
# sf erase 0 0x1000000
# loady 0x1000000       (load via YMODEM)
# sf write 0x1000000 0 0x1000000

# Step 5: Reboot
# => boot
```

### Use Cases

```
Recovery Scenarios:
├─ Corrupted bootloader in QSPI
│  └─ Device won't boot at all
│  └─ Use JTAG to load U-Boot into RAM
│  └─ Program QSPI from U-Boot
│
├─ Testing new bootloader
│  └─ Test in RAM before flashing
│  └─ Verify functionality
│  └─ Debug boot process
│
├─ Initial provisioning
│  └─ Brand new devices
│  └─ Program QSPI for first time
│  └─ Install production firmware
│
└─ Hardware debugging
   └─ Inspect memory contents
   └─ Set breakpoints
   └─ Single-step execution
```

### Advantages
- Recovery from corrupted bootloader
- Direct FPGA/PS access
- Minimal firmware requirement
- Useful for development/debugging

### Limitations
- Requires JTAG hardware
- Slow compared to other methods
- Complex setup and tools
- Not practical for field updates

## Method 4: SSH + DFU (Automated Testing)

### Overview

Combined method for CI/CD and automated testing environments.

### Script

```bash
#!/bin/bash
# Script: download_and_test.sh
# Usage: ./download_and_test.sh

# Purpose:
# 1. SSH to device
# 2. Reboot to RAM mode
# 3. Download firmware via DFU
# 4. Verify update success

# Configuration
DEVICE_IP="192.168.2.1"
DEVICE_USER="root"
DEVICE_PASSWORD="analog"
FIRMWARE_FILE="build/pluto.dfu"
DFU_TIMEOUT=10

# Verify firmware exists
if [ ! -f "$FIRMWARE_FILE" ]; then
    echo "Error: $FIRMWARE_FILE not found"
    exit 1
fi

# Step 1: SSH to device and reboot to RAM
echo "Connecting to device via SSH..."
sshpass -p "$DEVICE_PASSWORD" \
    ssh -o StrictHostKeyChecking=no \
    $DEVICE_USER@$DEVICE_IP \
    "device_reboot ram"

# Step 2: Wait for DFU enumeration
echo "Waiting for DFU device to appear..."
START_TIME=$(date +%s)
while [ $(( $(date +%s) - START_TIME )) -lt $DFU_TIMEOUT ]; do
    if dfu-util -l 2>/dev/null | grep -q "0456:b674"; then
        echo "DFU device found!"
        break
    fi
    sleep 1
done

# Step 3: Download firmware
echo "Downloading firmware..."
dfu-util -d 0456:b674 -D "$FIRMWARE_FILE" -R

if [ $? -eq 0 ]; then
    echo "Firmware update successful!"
    echo "Device rebooting..."
    sleep 5

    # Verify device re-enumerated
    if ping -c 1 $DEVICE_IP 2>/dev/null; then
        echo "Device is online and accessible!"
        exit 0
    fi
fi

echo "Error: Firmware update may have failed"
exit 1
```

### Advantages
- Fully automated
- Suitable for CI/CD pipelines
- Tests both SSH and DFU paths
- Verifies post-update device state

### Limitations
- Requires network connectivity
- SSH password required (security consideration)
- Depends on working bootloader
- May be slower (network overhead)

## Comparison Table

```
┌──────────────┬──────────┬──────────┬───────────┬─────────────────┐
│ Method       │ Speed    │ Ease     │ Tools     │ Use Case        │
├──────────────┼──────────┼──────────┼───────────┼─────────────────┤
│ MSD          │ Slow     │ Very Easy│ None      │ End users       │
│ DFU          │ Fast     │ Moderate │ dfu-util  │ Power users,    │
│              │          │          │           │ scripting       │
│ JTAG         │ Very Slow│ Complex  │ Xilinx   │ Recovery,       │
│              │          │          │ tools     │ debugging       │
│ SSH + DFU    │ Fast     │ Easy     │ dfu-util, │ CI/CD, testing  │
│              │          │          │ SSH       │                 │
└──────────────┴──────────┴──────────┴───────────┴─────────────────┘
```

## Troubleshooting Deployment

### Issue: Device Not Recognized

```
Symptom: Device doesn't appear in /dev or dfu-util -l

Solutions:
1. Check USB cable (try different port)
2. Verify device is powered
3. Check permissions:
   └─ sudo usermod -a -G plugdev $USER
   └─ Logout and login
4. Load udev rules:
   └─ sudo cp scripts/53-adi-plutosdr-usb.rules /etc/udev/rules.d/
   └─ sudo udevadm control --reload

Verify installation:
$ dfu-util -l | grep 0456
```

### Issue: DFU Device Not Entering DFU Mode

```
Symptom: Device stays in normal mode (0456:b673)

Solutions:
1. Try manual reboot in DFU mode:
   └─ ssh root@192.168.2.1
   └─ dmesg (check boot logs)
2. Check bootloader is valid
3. Try JTAG bootstrap to recover
4. Verify U-Boot environment has DFU enabled
```

### Issue: Firmware Update Fails Midway

```
Symptom: dfu-util stops or reports error

Solutions:
1. Retry the update:
   └─ Power cycle device first
   └─ Rerun dfu-util command
2. Try via MSD method as fallback
3. If both fail, use JTAG recovery
4. Check USB port power (use powered hub)
```

### Issue: Device Reboots in Bootloop

```
Symptom: LED blinks continuously, device doesn't boot

Solutions:
1. Don't panic - device is likely checking firmware
2. Wait 30-60 seconds (first boot verification)
3. If continues:
   └─ Use JTAG to verify QSPI integrity
   └─ Or re-flash via JTAG bootstrap
4. Check build log for errors
```

## Related Documentation

- [Build Process](./build-process.md) - How to build firmware
- [Scripts & Automation](../components/scripts-automation.md) - Deployment scripts
- [Device Access](../api-reference/device-interfaces.md) - USB interface details
