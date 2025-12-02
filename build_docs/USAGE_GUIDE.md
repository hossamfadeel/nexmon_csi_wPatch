# Nexmon CSI - Complete Usage Guide

## Table of Contents

1. [Quick Start](#quick-start)
2. [Understanding makecsiparams](#understanding-makecsiparams)
3. [Configuring the Extractor](#configuring-the-extractor)
4. [Enabling Monitor Mode](#enabling-monitor-mode)
5. [Collecting CSI Data](#collecting-csi-data)
6. [Analyzing CSI Data](#analyzing-csi-data)
7. [Advanced Usage](#advanced-usage)

---

## Quick Start

### Minimal Example (Raspberry Pi)

```bash
# 1. Stop wpa_supplicant
sudo pkill wpa_supplicant

# 2. Bring up interface
sudo ifconfig wlan0 up

# 3. Generate parameters (channel 6, 20MHz, core 0, spatial stream 0)
makecsiparams -c 6/20 -C 0 -N 0 -m ff:ff:ff:ff:ff:ff -b 0x88

# 4. Configure extractor (replace with your base64 string)
sudo nexutil -Iwlan0 -s500 -b -l34 -v<BASE64_STRING>

# 5. Enable monitor mode
sudo iw phy `iw dev wlan0 info | gawk '/wiphy/ {printf "phy" $2}'` interface add mon0 type monitor
sudo ifconfig mon0 up

# 6. Capture CSI packets
sudo tcpdump -i wlan0 dst port 5500 -w csi_capture.pcap

# 7. In another terminal, generate traffic (ping, iperf, etc.)
ping -c 100 8.8.8.8
```

---

## Understanding makecsiparams

### Purpose

`makecsiparams` generates a base64-encoded configuration string that tells the firmware which CSI data to extract and how to filter it.

### Command Syntax

```bash
makecsiparams [OPTIONS]
```

### Available Options

| Option | Argument | Description | Default |
|--------|----------|-------------|---------|
| `-c` | `CHANNEL/BANDWIDTH` | WiFi channel and bandwidth | Required |
| `-C` | `CORE_NUM` | Antenna core (0-3) | 0 |
| `-N` | `SPATIAL_STREAM` | Spatial stream (0-3) | 0 |
| `-m` | `MAC_ADDR_LIST` | MAC addresses to capture (comma-separated) | All |
| `-b` | `BYTE_VALUE` | Frame type filter (hex byte) | All |
| `-h` | - | Show help message | - |

### Channel and Bandwidth Specifications

```bash
# 20 MHz channels
6/20    # 2.4 GHz band, channel 6
11/20   # 2.4 GHz band, channel 11

# 40 MHz channels
36/40   # 5 GHz band, channel 36
149/40  # 5 GHz band, channel 149

# 80 MHz channels (5 GHz only)
36/80   # 5 GHz band, channels 36-48
149/80  # 5 GHz band, channels 149-165
```

### Core and Spatial Stream Numbers

- **Core:** 0-3 (antenna element)
- **Spatial Stream:** 0-3 (MIMO stream)

For single-antenna systems, use Core 0 and Spatial Stream 0.

### MAC Address Filtering

```bash
# Capture from specific device
makecsiparams -c 6/20 -m aa:bb:cc:dd:ee:ff

# Capture from multiple devices
makecsiparams -c 6/20 -m aa:bb:cc:dd:ee:ff,11:22:33:44:55:66

# Capture from all devices
makecsiparams -c 6/20 -m ff:ff:ff:ff:ff:ff
```

### Frame Type Filtering

Frame type is specified as a hex byte (the first byte of the frame):

```bash
# Data frames
makecsiparams -c 6/20 -b 0x88

# Management frames
makecsiparams -c 6/20 -b 0x80

# All frames
makecsiparams -c 6/20 -b 0x00
```

### Complete Examples

```bash
# Example 1: Channel 6, 20MHz, all devices, data frames
makecsiparams -c 6/20 -C 0 -N 0 -m ff:ff:ff:ff:ff:ff -b 0x88

# Example 2: Channel 157, 80MHz, specific device, all frames
makecsiparams -c 157/80 -C 1 -N 1 -m 00:11:22:33:44:55 -b 0x00

# Example 3: Channel 36, 40MHz, multiple cores/streams
makecsiparams -c 36/40 -C 2 -N 2 -m aa:bb:cc:dd:ee:ff -b 0x88
```

---

## Configuring the Extractor

### nexutil Configuration

After generating parameters with `makecsiparams`, configure the firmware using `nexutil`:

```bash
nexutil -Iwlan0 -s500 -b -l34 -v<BASE64_STRING>
```

### nexutil Options Explained

| Option | Argument | Description |
|--------|----------|-------------|
| `-I` | `INTERFACE` | WiFi interface name (usually `wlan0`) |
| `-s` | `IOCTL_ID` | IOCTL command ID (500 for CSI) |
| `-b` | - | Binary mode (required for CSI) |
| `-l` | `LENGTH` | Parameter length (34 bytes for CSI) |
| `-v` | `BASE64_STRING` | Base64-encoded parameters |

### Configuration Example

```bash
# Generate parameters
PARAMS=$(makecsiparams -c 6/20 -C 0 -N 0 -m ff:ff:ff:ff:ff:ff -b 0x88)

# Configure extractor
sudo nexutil -Iwlan0 -s500 -b -l34 -v$PARAMS

# Verify configuration
sudo nexutil -k  # Check current channel
```

### Verify Configuration

```bash
# Check if channel is correctly set
nexutil -k

# Expected output: chanspec 0x1006 (for channel 6)
# If output shows 0x6863, the interface is not up
```

---

## Enabling Monitor Mode

### For Raspberry Pi (bcm43455c0)

```bash
# Create monitor interface
sudo iw phy `iw dev wlan0 info | gawk '/wiphy/ {printf "phy" $2}'` interface add mon0 type monitor

# Bring up monitor interface
sudo ifconfig mon0 up

# Verify
iw dev
# Should show mon0 in monitor mode
```

### For Nexus Devices (bcm4339, bcm4358)

```bash
# Enable monitor mode directly
nexutil -Iwlan0 -m1
```

### For ASUS RT-AC86U (bcm4366c0)

```bash
# SSH into router
ssh admin@192.168.1.1

# Enable monitor mode
/usr/sbin/wl -i eth6 monitor 1
```

### Verify Monitor Mode

```bash
# Check interface status
iw dev

# Should show:
# phy#0
#   Interface mon0
#       ifindex 3
#       wdev 0x1
#       addr xx:xx:xx:xx:xx:xx
#       type monitor
#       txpower 20.00 dBm
```

---

## Collecting CSI Data

### Method 1: tcpdump (Recommended)

```bash
# Capture CSI packets to file
sudo tcpdump -i wlan0 dst port 5500 -w csi_capture.pcap

# Capture with verbose output
sudo tcpdump -i wlan0 dst port 5500 -v

# Capture for specific duration (60 seconds)
sudo timeout 60 tcpdump -i wlan0 dst port 5500 -w csi_capture.pcap
```

### Method 2: netcat

```bash
# Listen on UDP port 5500
nc -u -l 5500 > csi_data.bin
```

### Method 3: Custom Python Script

```python
#!/usr/bin/env python3
import socket
import struct

# Create UDP socket
sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
sock.bind(('0.0.0.0', 5500))

# Receive CSI packets
with open('csi_capture.bin', 'wb') as f:
    while True:
        data, addr = sock.recvfrom(4096)
        print(f"Received {len(data)} bytes from {addr}")
        f.write(data)
```

### Generating Traffic

To trigger CSI collection, you need WiFi traffic:

```bash
# Method 1: Ping
ping -c 100 8.8.8.8

# Method 2: iperf (requires iperf on both devices)
iperf -c <server_ip> -t 30

# Method 3: Frame injection (requires Nexmon on transmitter)
# See Nexmon documentation for frame injection

# Method 4: SSH traffic
ssh user@<remote_ip> "cat /dev/urandom" > /dev/null
```

---

## Analyzing CSI Data

### CSI Packet Structure

Each UDP packet contains:

```
Offset  Size    Field
------  ----    -----
0       2       Magic bytes (0x1111)
2       6       Source MAC address
8       2       Sequence number
10      2       Core and spatial stream (bits 0-2: core, bits 3-5: stream)
12      2       Channel specification
14      2       Chip version
16      N*4     CSI data (N depends on bandwidth)
```

### CSI Data Format

| Chipset | Format | Size (20MHz) | Size (40MHz) | Size (80MHz) |
|---------|--------|--------------|--------------|--------------|
| bcm4339 | int16 real/imag | 256 bytes | 512 bytes | 1024 bytes |
| bcm43455c0 | int16 real/imag | 256 bytes | 512 bytes | 1024 bytes |
| bcm4358 | float (custom) | 256 bytes | 512 bytes | 1024 bytes |
| bcm4366c0 | float (custom) | 256 bytes | 512 bytes | 1024 bytes |

### MATLAB Analysis

The project includes MATLAB scripts for CSI analysis:

```matlab
% Navigate to utils/matlab/
cd utils/matlab/

% For bcm4358 or bcm4366c0, compile the MEX file first
mex unpack_float.c

% Configure csireader.m with your capture file
% Edit the configuration section:
% - pcap_file = 'example.pcap'
% - output_dir = './output'

% Run the reader
csireader
```

### Python Analysis Example

```python
#!/usr/bin/env python3
import struct
import numpy as np

def parse_csi_packet(data):
    """Parse a CSI packet"""
    if len(data) < 16:
        return None
    
    # Parse header
    magic = struct.unpack('>H', data[0:2])[0]
    if magic != 0x1111:
        return None
    
    src_mac = data[2:8]
    seq_num = struct.unpack('>H', data[8:10])[0]
    core_stream = struct.unpack('>H', data[10:12])[0]
    chanspec = struct.unpack('>H', data[12:14])[0]
    chip_ver = struct.unpack('>H', data[14:16])[0]
    
    # Parse CSI data (int16 format for bcm43455c0)
    csi_data = data[16:]
    csi_values = np.frombuffer(csi_data, dtype=np.int16)
    
    # Separate real and imaginary parts
    real_parts = csi_values[::2]
    imag_parts = csi_values[1::2]
    csi_complex = real_parts + 1j * imag_parts
    
    return {
        'magic': magic,
        'src_mac': src_mac,
        'seq_num': seq_num,
        'core': core_stream & 0x7,
        'spatial_stream': (core_stream >> 3) & 0x7,
        'chanspec': chanspec,
        'chip_ver': chip_ver,
        'csi': csi_complex
    }

# Read and parse PCAP file
import dpkt

with open('csi_capture.pcap', 'rb') as f:
    pcap = dpkt.pcap.Reader(f)
    for ts, buf in pcap:
        eth = dpkt.ethernet.Ethernet(buf)
        ip = eth.data
        udp = ip.data
        csi_packet = parse_csi_packet(udp.data)
        if csi_packet:
            print(f"CSI from {csi_packet['src_mac'].hex()}: {len(csi_packet['csi'])} subcarriers")
```

---

## Advanced Usage

### Multi-Core CSI Collection

```bash
# Collect from multiple cores simultaneously
# Run multiple nexutil commands with different core numbers

# Core 0, Stream 0
PARAMS0=$(makecsiparams -c 6/20 -C 0 -N 0 -m ff:ff:ff:ff:ff:ff -b 0x88)
sudo nexutil -Iwlan0 -s500 -b -l34 -v$PARAMS0

# Core 1, Stream 0
PARAMS1=$(makecsiparams -c 6/20 -C 1 -N 0 -m ff:ff:ff:ff:ff:ff -b 0x88)
sudo nexutil -Iwlan0 -s500 -b -l34 -v$PARAMS1

# Both will send CSI data to port 5500
```

### Capturing Specific Frame Types

```bash
# Data frames (0x88)
makecsiparams -c 6/20 -b 0x88

# Management frames (0x80)
makecsiparams -c 6/20 -b 0x80

# QoS data frames (0x88)
makecsiparams -c 6/20 -b 0x88

# Beacon frames (0x80)
makecsiparams -c 6/20 -b 0x80
```

### Channel Hopping

```bash
#!/bin/bash
# Capture CSI on multiple channels

for channel in 1 6 11; do
    echo "Capturing on channel $channel"
    
    # Generate parameters
    PARAMS=$(makecsiparams -c $channel/20 -C 0 -N 0 -m ff:ff:ff:ff:ff:ff -b 0x88)
    
    # Configure extractor
    sudo nexutil -Iwlan0 -s500 -b -l34 -v$PARAMS
    
    # Capture for 10 seconds
    sudo timeout 10 tcpdump -i wlan0 dst port 5500 -w csi_ch${channel}.pcap
done
```

### Real-time CSI Monitoring

```bash
#!/usr/bin/env python3
import socket
import struct
import numpy as np

def monitor_csi():
    sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
    sock.bind(('0.0.0.0', 5500))
    
    packet_count = 0
    while True:
        data, addr = sock.recvfrom(4096)
        
        if len(data) < 16:
            continue
        
        # Parse header
        magic = struct.unpack('>H', data[0:2])[0]
        if magic != 0x1111:
            continue
        
        seq_num = struct.unpack('>H', data[8:10])[0]
        core_stream = struct.unpack('>H', data[10:12])[0]
        
        # Parse CSI
        csi_data = data[16:]
        csi_values = np.frombuffer(csi_data, dtype=np.int16)
        csi_complex = csi_values[::2] + 1j * csi_values[1::2]
        
        # Calculate amplitude
        amplitude = np.abs(csi_complex)
        
        packet_count += 1
        print(f"Packet {packet_count}: Seq={seq_num}, Core={core_stream & 0x7}, "
              f"Stream={(core_stream >> 3) & 0x7}, Mean Amplitude={np.mean(amplitude):.2f}")

if __name__ == '__main__':
    monitor_csi()
```

---

## Troubleshooting

### No CSI Packets Received

**Problem:** tcpdump shows no packets on port 5500

**Solutions:**
1. Verify interface is up: `ifconfig wlan0`
2. Check channel configuration: `nexutil -k`
3. Ensure traffic is being generated
4. Verify firewall isn't blocking port 5500
5. Check if driver is loaded: `lsmod | grep brcmfmac`

### Incorrect Channel

**Problem:** Channel doesn't match what was configured

**Solution:**
```bash
# Check current channel
nexutil -k

# If incorrect, reconfigure
PARAMS=$(makecsiparams -c 6/20 -C 0 -N 0 -m ff:ff:ff:ff:ff:ff -b 0x88)
sudo nexutil -Iwlan0 -s500 -b -l34 -v$PARAMS
```

### Monitor Mode Issues

**Problem:** Monitor interface not created

**Solution:**
```bash
# Verify WiFi phy exists
iw phy list

# Try creating monitor interface manually
sudo iw phy phy0 interface add mon0 type monitor
sudo ifconfig mon0 up
```

---

## Next Steps

- Review `PATCH_NOTES.md` for code modifications
- Check `TROUBLESHOOTING.md` for runtime issues
- Consult the original README.md for additional information

