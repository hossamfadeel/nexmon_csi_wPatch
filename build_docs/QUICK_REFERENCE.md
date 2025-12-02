# Nexmon CSI - Quick Reference Guide

## Installation Quick Start (Raspberry Pi)

```bash
# 1. Update system
sudo apt-get update && sudo apt-get upgrade -y

# 2. Install dependencies
sudo apt-get install -y git gawk qpdf flex bison xxd make gcc libgmp3-dev automake autoconf libtool texinfo raspberrypi-kernel-headers

# 3. Clone and setup Nexmon
cd ~ && git clone https://github.com/seemoo-lab/nexmon.git && cd nexmon
source setup_env.sh && make && cd utilities && make

# 4. Clone Nexmon CSI
cd ~/nexmon/patches/bcm43455c0/7_45_189/
git clone https://github.com/seemoo-lab/nexmon_csi.git && cd nexmon_csi

# 5. Build and install
make -f Makefile.rpi && sudo make -f Makefile.rpi install-firmware

# 6. Install utilities
cd ~/nexmon/utilities/nexutil && make && sudo make install
cd ~/nexmon/patches/bcm43455c0/7_45_189/nexmon_csi/utils/makecsiparams && make && sudo make install

# 7. Reboot
sudo reboot
```

---

## CSI Extraction Quick Start

```bash
# 1. Stop wpa_supplicant
sudo pkill wpa_supplicant

# 2. Bring up interface
sudo ifconfig wlan0 up

# 3. Generate parameters
PARAMS=$(makecsiparams -c 6/20 -C 0 -N 0 -m ff:ff:ff:ff:ff:ff -b 0x88)

# 4. Configure extractor
sudo nexutil -Iwlan0 -s500 -b -l34 -v$PARAMS

# 5. Create monitor interface
sudo iw phy `iw dev wlan0 info | gawk '/wiphy/ {printf "phy" $2}'` interface add mon0 type monitor
sudo ifconfig mon0 up

# 6. Capture CSI
sudo tcpdump -i wlan0 dst port 5500 -w csi.pcap

# 7. In another terminal, generate traffic
ping -c 100 8.8.8.8
```

---

## Common makecsiparams Examples

```bash
# Channel 6, 20MHz, all devices
makecsiparams -c 6/20 -C 0 -N 0 -m ff:ff:ff:ff:ff:ff -b 0x88

# Channel 157, 80MHz, specific device
makecsiparams -c 157/80 -C 0 -N 0 -m aa:bb:cc:dd:ee:ff -b 0x88

# Channel 36, 40MHz, all devices
makecsiparams -c 36/40 -C 0 -N 0 -m ff:ff:ff:ff:ff:ff -b 0x88

# Channel 11, 20MHz, multiple devices
makecsiparams -c 11/20 -C 0 -N 0 -m aa:bb:cc:dd:ee:ff,11:22:33:44:55:66 -b 0x88

# All channels, all devices, all frames
makecsiparams -c 6/20 -C 0 -N 0 -m ff:ff:ff:ff:ff:ff -b 0x00
```

---

## Common nexutil Commands

```bash
# Check current channel
nexutil -k

# Get driver version
nexutil -v

# Configure CSI extraction
sudo nexutil -Iwlan0 -s500 -b -l34 -v<BASE64_STRING>

# Enable monitor mode (Nexus devices)
sudo nexutil -Iwlan0 -m1

# Get interface info
nexutil -I wlan0
```

---

## Troubleshooting Quick Fixes

```bash
# Interface not found
ifconfig -a
sudo ifconfig wlan0 up

# Kernel headers missing
sudo apt-get install raspberrypi-kernel-headers
sudo reboot

# libisl.so.10 not found
sudo apt-get install libisl-dev
sudo ln -s /usr/lib/arm-linux-gnueabihf/libisl.so.15 /usr/lib/arm-linux-gnueabihf/libisl.so.10

# libmpfr.so.4 not found
sudo apt-get install libmpfr-dev
sudo ln -s /usr/lib/arm-linux-gnueabihf/libmpfr.so.6 /usr/lib/arm-linux-gnueabihf/libmpfr.so.4

# No CSI packets
sudo nexutil -k  # Check channel
sudo pkill wpa_supplicant  # Stop interference
ping -c 100 8.8.8.8  # Generate traffic

# Monitor mode not working
sudo iw phy phy0 interface add mon0 type monitor
sudo ifconfig mon0 up
```

---

## File Locations

```
Nexmon Framework:        ~/nexmon/
Nexmon CSI:             ~/nexmon/patches/bcm43455c0/7_45_189/nexmon_csi/
Compiled Firmware:      ~/nexmon/patches/bcm43455c0/7_45_189/nexmon_csi/brcmfmac.bin
makecsiparams:          /usr/bin/makecsiparams
nexutil:                /usr/bin/nexutil
Kernel Headers:         /lib/modules/$(uname -r)/build/
Driver Source:          ~/nexmon/patches/bcm43455c0/7_45_189/nexmon_csi/brcmfmac_5.10.y-nexmon/
```

---

## Environment Variables

```bash
# Set after cloning Nexmon
export NEXMON_ROOT=~/nexmon
source $NEXMON_ROOT/setup_env.sh

# For remote deployment (ASUS router)
export REMOTEADDR=192.168.1.1

# For Android devices
export ADBSERIAL=<device_serial>
export NDK_ROOT=~/android-ndk-r11c
```

---

## Build Commands

```bash
# Clean build
make -f Makefile.rpi clean
make -f Makefile.rpi

# Verbose build (for debugging)
make -f Makefile.rpi V=1

# Install firmware
sudo make -f Makefile.rpi install-firmware

# Install utilities
cd ~/nexmon/utilities/nexutil && sudo make install
cd ~/nexmon/patches/bcm43455c0/7_45_189/nexmon_csi/utils/makecsiparams && sudo make install

# Uninstall (restore original driver)
sudo apt-get install --reinstall raspberrypi-kernel
sudo reboot
```

---

## Channel Reference

### 2.4 GHz Band (20 MHz only)
```
Channel 1:   2412 MHz
Channel 6:   2437 MHz
Channel 11:  2462 MHz
Channel 14:  2484 MHz (Japan only)
```

### 5 GHz Band (20/40/80 MHz)
```
UNII-1:  36, 40, 44, 48 (5150-5250 MHz)
UNII-2:  52, 56, 60, 64 (5250-5350 MHz)
UNII-3:  100-144 (5470-5725 MHz)
UNII-4:  149-165 (5725-5850 MHz)
```

---

## Frame Type Filters

```
0x88  - QoS Data frames (recommended)
0x80  - Management frames
0x00  - All frames
0x40  - Null frames
0xc0  - QoS Null frames
```

---

## Useful Commands

```bash
# Check WiFi interface
iw dev

# List available networks
sudo iw dev wlan0 scan

# Connect to network
sudo nmcli dev wifi connect <SSID> password <PASSWORD>

# Monitor traffic
sudo tcpdump -i wlan0 -n

# Check network statistics
ifconfig wlan0
ip link show wlan0

# Monitor system resources
top
free -h
df -h

# Check kernel logs
sudo dmesg | tail -20

# Check driver status
lsmod | grep brcmfmac
modinfo brcmfmac
```

---

## Performance Tuning

```bash
# Increase network buffer sizes
sudo sysctl -w net.core.rmem_max=134217728
sudo sysctl -w net.core.rmem_default=134217728

# Disable power saving (may improve stability)
sudo iw wlan0 set power_save off

# Set fixed channel (prevent hopping)
sudo iw dev wlan0 set channel 6

# Monitor CPU usage
watch -n 1 'top -bn1 | head -20'
```

---

## Data Analysis

### Parse CSI in Python

```python
import struct
import numpy as np

def parse_csi_packet(data):
    magic = struct.unpack('>H', data[0:2])[0]
    src_mac = data[2:8]
    seq_num = struct.unpack('>H', data[8:10])[0]
    core_stream = struct.unpack('>H', data[10:12])[0]
    chanspec = struct.unpack('>H', data[12:14])[0]
    
    csi_data = data[16:]
    csi_values = np.frombuffer(csi_data, dtype=np.int16)
    csi_complex = csi_values[::2] + 1j * csi_values[1::2]
    
    return {
        'seq': seq_num,
        'core': core_stream & 0x7,
        'stream': (core_stream >> 3) & 0x7,
        'csi': csi_complex
    }
```

### Analyze CSI in MATLAB

```matlab
% Load and plot CSI
addpath('utils/matlab/');
mex unpack_float.c
csireader  % Configure and run
```

---

## Getting Help

```bash
# Show help for makecsiparams
makecsiparams -h

# Show help for nexutil
nexutil -h

# Check Nexmon documentation
# https://nexmon.org

# Check project repository
# https://github.com/seemoo-lab/nexmon_csi

# View kernel logs for errors
sudo journalctl -xe

# Check system messages
sudo dmesg | grep -i wifi
sudo dmesg | grep -i brcm
```

---

## Important Notes

1. **Always use sudo** for system-level operations
2. **Stop wpa_supplicant** before CSI collection
3. **Generate traffic** to trigger CSI packets
4. **Use broadcast MAC** (ff:ff:ff:ff:ff:ff) if unsure
5. **Check channel** with `nexutil -k` after configuration
6. **Reboot after driver installation** to ensure proper loading
7. **Keep kernel headers updated** to match running kernel

---

## Quick Diagnostics

```bash
# Complete system check
echo "=== System Info ===" && uname -a
echo "=== Kernel Version ===" && uname -r
echo "=== WiFi Interface ===" && iw dev
echo "=== Driver Status ===" && lsmod | grep brcmfmac
echo "=== Kernel Headers ===" && ls /lib/modules/$(uname -r)/build/
echo "=== makecsiparams ===" && which makecsiparams && makecsiparams -h | head -5
echo "=== nexutil ===" && which nexutil && nexutil -h | head -5
echo "=== Current Channel ===" && sudo nexutil -k
```

---

## Emergency Recovery

```bash
# If driver breaks WiFi
sudo apt-get install --reinstall raspberrypi-kernel
sudo reboot

# If Nexmon environment is broken
cd ~/nexmon
source setup_env.sh
make clean
make

# If compilation fails
make -f Makefile.rpi clean
rm -rf obj/ gen/ log/
make -f Makefile.rpi
```

