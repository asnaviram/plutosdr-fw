# Device Interfaces Documentation

## Overview

PlutoSDR provides multiple interfaces for host communication and control:

```
Host Computer
    │
    ├─ USB (IIO Interface)
    │  └─ libiio client library
    │  └─ Real-time RF data streaming
    │
    ├─ SSH/Network
    │  └─ Remote command execution
    │  └─ Device management
    │  └─ SCP file transfer
    │
    └─ Serial Console
       └─ Debug output
       └─ U-Boot interactive shell
       └─ Kernel logs
```

## USB Interface (IIO - Industrial I/O)

### Overview

The primary interface for RF data and control, using the Linux IIO subsystem exposed via USB.

### Device Identification

```
USB Enumeration:
├─ Vendor ID: 0x0456 (Analog Devices)
├─ Product ID: 0xb673 (PlutoSDR in normal mode)
└─ Product ID: 0xb674 (PlutoSDR in DFU mode)

Device appears as:
/dev/iio:device0      (IIO device character device)
/sys/bus/iio/devices/ (Sysfs interface)
```

### IIO Device Structure

```
ad9361-phy (IIO Device)
│
├─ Attributes (RW):
│  ├─ Frequency (RX/TX tuning)
│  ├─ Sample Rate
│  ├─ Bandwidth
│  ├─ Gain
│  ├─ Mode (TDD, FDD, etc.)
│  └─ ...
│
├─ Channels:
│  ├─ RX (Input channels)
│  │  ├─ RX1_I, RX1_Q (In-phase, Quadrature)
│  │  ├─ RX2_I, RX2_Q (if dual-channel)
│  │  └─ Attributes: gain, offset, scale
│  │
│  └─ TX (Output channels)
│     ├─ TX1_I, TX1_Q
│     ├─ TX2_I, TX2_Q (if dual-channel)
│     └─ Attributes: gain, offset, scale
│
└─ Buffers:
   ├─ RX buffer (circular, configurable size)
   ├─ TX buffer (circular, configurable size)
   └─ Channels: streaming raw I/Q samples
```

### Using libiio

```c
#include <iio.h>

// Example: Receive I/Q samples
int main(void) {
    struct iio_context *ctx;
    struct iio_device *dev;
    struct iio_channel *ch;
    struct iio_buffer *rxbuf;

    // Connect to IIO device (auto-detect USB)
    ctx = iio_create_default_context();

    // Get AD9361 device
    dev = iio_context_find_device(ctx, "ad9361-phy");

    // Set RX frequency to 1 GHz
    ch = iio_device_find_channel(dev, "RX1", false);
    iio_channel_attr_write_longlong(ch, "frequency", 1000000000);

    // Set sample rate to 2 MSps
    ch = iio_device_find_channel(dev, "RX1", false);
    iio_channel_attr_write_longlong(ch, "sampling_frequency", 2000000);

    // Enable RX channels for buffering
    iio_device_identify_filename(dev, "RX1_I", &filename, &ch_id);
    iio_channel_enable(ch);

    // Create RX buffer (4 MB)
    rxbuf = iio_device_create_buffer(dev, 1024, false);

    // Read samples
    uint8_t *buf = iio_buffer_first(rxbuf, ch);
    int bytes_read = iio_buffer_refill(rxbuf);

    // Process samples...

    // Cleanup
    iio_buffer_destroy(rxbuf);
    iio_context_destroy(ctx);

    return 0;
}
```

### Python Example with PyADI-IIO

```python
import adi

# Create IIO device context (auto-detects USB)
sdr = adi.Pluto()

# Configure RX
sdr.rx_lo = 1000000000  # 1 GHz
sdr.rx_rf_bandwidth = 1000000  # 1 MHz bandwidth
sdr.rx_buffer_size = 1024
sdr.gain_control_mode_chan0 = "slow_attack"

# Configure TX
sdr.tx_lo = 1000000000  # 1 GHz
sdr.tx_rf_bandwidth = 1000000  # 1 MHz

# Receive samples
data = sdr.rx()  # Returns numpy array [I samples, Q samples]

# Transmit samples
sdr.tx(tx_data)

# Access native IIO device
print(sdr.iio_device.attrs)
```

### Buffer Structure

```
RX Buffer (Circular):
┌─────────────────────────────────┐
│ Write Pointer (by DMA)          │
│                                 │
│ [Sample N-1] [Sample N]         │ ← Read Pointer (by host)
│              [Sample N+1]       │
│              [Sample N+2]       │
│                                 │
│ (Oldest samples)                │
│                                 │
│ Total Size: Configurable        │
│ Default: 16 MB                  │
│ Min: 4 KB                       │
│ Max: Available DDR3             │
└─────────────────────────────────┘

Circular operation:
• DMA writes continuously
• Host reads at own pace
• Buffer wraps around
• Overrun if host too slow
• Underrun risk minimized by size
```

## SSH Interface

### Network Access

```
SSH Connection:
$ ssh root@192.168.2.1
Password: analog

Alternative (if configured for different subnet):
$ ssh root@<device-ip-address>
```

### Available Commands

```bash
# Device information
uname -a                    # OS version
cat /sys/firmware/devicetree/base/model  # Hardware version
iio_info                    # IIO device details

# System status
df -h                       # Disk usage (QSPI flash)
free -h                     # Memory usage
ps aux                      # Running processes
top -b -n 1                 # Process activity

# Network configuration
ip addr                     # IP configuration
ifconfig                    # Network interfaces
ping 8.8.8.8               # Test connectivity

# RF transceiver control (via sysfs)
cat /sys/bus/iio/devices/iio:device0/name
cat /sys/bus/iio/devices/iio:device0/frequency

# View kernel logs
dmesg                       # Kernel messages
dmesg | tail -50           # Last 50 lines

# Control services
systemctl status ssh        # Check SSH daemon
systemctl restart network   # Restart networking

# Transfer files
scp local_file root@192.168.2.1:/tmp/
scp root@192.168.2.1:/tmp/remote_file ./
```

### SCP (Secure Copy)

```bash
# Upload file to device
scp myapp root@192.168.2.1:/home/root/

# Download file from device
scp root@192.168.2.1:/tmp/logfile.txt ./

# Remote execution
ssh root@192.168.2.1 "iio_attr -r ad9361-phy frequency"
```

### X11 Forwarding (if GUI apps needed)

```bash
# Enable X11 forwarding
ssh -X root@192.168.2.1

# Run GUI application
glxgears  # Test graphics
```

## Serial Console Interface

### Connection Setup

```bash
# Identify serial port
ls /dev/ttyUSB*     # Linux
ls /dev/ttyS*       # Windows (COM port)
ls /dev/cu.usbserial* # macOS

# Connect via picocom
picocom -b 115200 /dev/ttyUSB0

# Or minicom
minicom -D /dev/ttyUSB0

# Or screen
screen /dev/ttyUSB0 115200

# Exit picocom: Ctrl+A Ctrl+X
# Exit minicom: Ctrl+A Ctrl+X
# Exit screen: Ctrl+A :quit
```

### U-Boot Prompt

```
U-Boot bootloader command line:

=> printenv
# Print environment variables

=> setenv ethaddr 00:11:22:33:44:55
# Set MAC address

=> saveenv
# Save environment to flash

=> dfu 0 mmc 0
# Enter DFU firmware update mode

=> help [command]
# Show available commands

=> boot
# Boot kernel manually

=> run bootcmd
# Execute boot command
```

### Kernel Output

```
Typical boot messages:
[    0.000000] Booting Linux on physical CPU 0x0
[    0.000000] Linux version 4.18.0-xlnx-v2018.1 (build@...) (gcc version 7.3.0)
[    0.000000] Command line: console=ttyPS0,115200 root=/dev/mtdblock3
[    0.000000] KERNEL supported cpus:
[    0.000000]   ARMv7 Processor [412fc09a] revision 10 (ARMv7)
...
[    2.345678] ad9361 ad9361-phy: AD9361 Transceiver initialized
[    2.456789] iio ad9361-phy: Registered device iio:device0
[    3.567890] usb 1-1: new high-speed USB device number 2 using ci_hdrc
[    4.678901] gadget_serial: Gadget Serial v2.4
[    5.789012] Freeing initrd memory: 8024K
[    6.890123] usb_gadget: USB Gadget registered as 1d0
[    7.901234] Read-only file system
```

### Interactive Shell

```
Once booted, interactive shell available:

# Change directory
cd /tmp
cd /home/root

# List files
ls -la
ls -h

# Edit files
vi /tmp/test.txt
nano /tmp/test.txt

# Execute programs
./my_app --option value

# Check AD9361
iio_info
iio_attr -r ad9361-phy frequency

# Exit shell
exit
reboot
```

## IIO Attributes and Channels

### Commonly Used Attributes

```
RX Frequency:
$ iio_attr -r ad9361-phy RX1 frequency
RX1_frequency: 1000000000  (1 GHz)

TX Frequency:
$ iio_attr -r ad9361-phy TX1 frequency
TX1_frequency: 1000000000  (1 GHz)

RX Gain:
$ iio_attr -r ad9361-phy RX1 hardwaregain
RX1_hardwaregain: 21.000000 dB

RX Sample Rate:
$ iio_attr -r ad9361-phy in_voltage_sampling_frequency
in_voltage_sampling_frequency: 2000000 (2 MSps)

RX Bandwidth:
$ iio_attr -r ad9361-phy in_voltage_filter_fir_en
in_voltage_filter_fir_en: 1  (enabled)

AGC Mode:
$ iio_attr -r ad9361-phy RX1 gain_control_mode
RX1_gain_control_mode: slow_attack

TX Attenuation:
$ iio_attr -r ad9361-phy TX1 hardwaregain
TX1_hardwaregain: -10.000000 dB
```

### Setting Attributes

```bash
# Set RX frequency to 2.4 GHz
iio_attr -w ad9361-phy RX1 frequency 2400000000

# Set TX frequency to 2.4 GHz
iio_attr -w ad9361-phy TX1 frequency 2400000000

# Set RX sample rate to 4 MSps
iio_attr -w ad9361-phy in_voltage_sampling_frequency 4000000

# Set RX gain to 30 dB
iio_attr -w ad9361-phy RX1 hardwaregain 30

# Enable AGC (Automatic Gain Control)
iio_attr -w ad9361-phy RX1 gain_control_mode slow_attack

# Disable AGC (use manual gain)
iio_attr -w ad9361-phy RX1 gain_control_mode manual
```

## DMA Ring Buffers

### Buffer Configuration

```
RX Buffer:
├─ Allocated from DDR3
├─ Default size: 16 MB
├─ Write by ADC DMA
├─ Read by application
├─ Circular (wrap-around)
└─ Interrupt on threshold

TX Buffer:
├─ Allocated from DDR3
├─ Default size: 16 MB
├─ Read by DAC DMA
├─ Write by application
├─ Circular (wrap-around)
└─ Interrupt on empty
```

### Buffer Control (libiio)

```c
// Create 4 MB RX buffer
struct iio_buffer *rxbuf =
    iio_device_create_buffer(dev, 1024*1024, false);

// Enable channels
struct iio_channel *ch = iio_device_get_channel(dev, 0);
iio_channel_enable(ch);

// Refill from device (blocking read)
ssize_t nbytes = iio_buffer_refill(rxbuf);

// Access data
void *data = iio_buffer_first(rxbuf, ch);

// Get number of samples
size_t sample_count = iio_buffer_step(rxbuf) / 4; // 4 bytes per sample

// Destroy buffer
iio_buffer_destroy(rxbuf);
```

## Error Handling

### Common Issues

```
Issue: "Connection refused"
- Device not powered on
- Wrong IP address
- Wrong USB port

Issue: "dmesg: permission denied"
- Run with sudo: sudo dmesg
- Or add user to adm group: sudo usermod -a -G adm $USER

Issue: "iio_info: No devices found"
- USB cable disconnected
- Device in DFU mode (not normal mode)
- Driver not loaded

Issue: "iio_attr: Error: No such attribute"
- Wrong attribute name
- Device/channel doesn't support attribute
- Outdated libiio version
```

### Debugging Steps

```bash
# 1. Check USB connection
lsusb | grep "0456"
# Should show: Analog Devices, Inc. [0456:b673]

# 2. Check IIO device
iio_info
# Should show: ad9361-phy with attributes

# 3. Check kernel driver
dmesg | grep -i ad9361

# 4. Check permissions
ls -la /dev/iio:device0
# Should be readable/writable by user

# 5. Test raw IIO
iio_attr -r ad9361-phy frequency
# Should return current frequency
```

## Performance Considerations

### Bandwidth

```
USB 2.0 High-Speed:
├─ Theoretical max: 480 Mbps
├─ Practical with IIO overhead: 350-400 Mbps
├─ 16-bit I/Q pairs at 2 Mbps: ~80 Mbps actual
└─ Sustained multi-second transfers possible

Achievable Sample Rates:
├─ 4 MSps I/Q: ~256 Mbps needed
├─ 2 MSps I/Q: ~128 Mbps needed
├─ 1 MSps I/Q: ~64 Mbps needed
└─ All well within USB 2.0 capability
```

### Latency

```
Typical latencies:
├─ IIO buffer transfer: <10 ms
├─ USB round-trip: <1 ms
├─ SSH command execution: <100 ms
└─ Total system latency: <50 ms typical
```

## Related Documentation

- [System Overview](../architecture/system-overview.md) - System architecture
- [Build Process](../build-system/build-process.md) - Firmware build
- [Boot Sequence](./boot-sequence.md) - Boot process details
