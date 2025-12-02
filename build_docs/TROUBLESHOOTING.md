# Nexmon CSI - Comprehensive Troubleshooting Guide

## Table of Contents

1. [Build-Time Issues](#build-time-issues)
2. [Installation Issues](#installation-issues)
3. [Runtime Issues](#runtime-issues)
4. [CSI Collection Issues](#csi-collection-issues)
5. [Platform-Specific Issues](#platform-specific-issues)
6. [Performance Issues](#performance-issues)

---

## Build-Time Issues

### Error: "make: *** No rule to make target"

**Symptoms:**
```
make: *** No rule to make target 'obj/csi_extractor.o'. Stop.
```

**Causes:**
- Wrong working directory
- Makefile not found
- Missing dependencies

**Solutions:**

```bash
# Verify correct directory
pwd
# Should end with: .../nexmon_csi

# Verify Makefile exists
ls -la Makefile*

# Clean and rebuild
make clean
make -f Makefile.rpi  # For Raspberry Pi
# or
make                  # For other platforms

# If still failing, check if NEXMON_ROOT is set
echo $NEXMON_ROOT
# Should output path to nexmon directory
```

### Error: "Kernel version mismatch"

**Symptoms:**
```
ERROR: kernel version mismatch
  /lib/modules/5.10.63-v7l+/build/Makefile:... 
```

**Causes:**
- Kernel headers don't match running kernel
- Kernel was updated but headers weren't

**Solutions:**

```bash
# Check kernel version
uname -r

# Check header version
ls /lib/modules/$(uname -r)/build/

# If missing, install headers
sudo apt-get install raspberrypi-kernel-headers

# If headers are old, update kernel
sudo apt-get update
sudo apt-get upgrade

# Reboot if kernel was updated
sudo reboot

# Verify after reboot
uname -r
ls /lib/modules/$(uname -r)/build/
```

### Error: "libisl.so.10: cannot open shared object file"

**Symptoms:**
```
/usr/bin/ld: cannot find -lisl
collect2: error: ld returned 1 exit status
```

**Causes:**
- ISL library not installed
- Library in wrong location
- Wrong architecture

**Solutions:**

```bash
# Check if library exists
ls /usr/lib/arm-linux-gnueabihf/libisl.so.10

# If not, install development package
sudo apt-get install libisl-dev

# If package doesn't provide correct version, build from source
cd ~/nexmon/buildtools/isl-0.10
./configure
make
sudo make install

# Create symlink
sudo ln -s /usr/local/lib/libisl.so /usr/lib/arm-linux-gnueabihf/libisl.so.10

# Verify
ls -la /usr/lib/arm-linux-gnueabihf/libisl.so.10
```

### Error: "libmpfr.so.4: cannot open shared object file"

**Symptoms:**
```
/usr/bin/ld: cannot find -lmpfr
collect2: error: ld returned 1 exit status
```

**Solutions:**

```bash
# Check if library exists
ls /usr/lib/arm-linux-gnueabihf/libmpfr.so.4

# If not, build from source
cd ~/nexmon/buildtools/mpfr-3.1.4
autoreconf -f -i
./configure
make
sudo make install

# Create symlink
sudo ln -s /usr/local/lib/libmpfr.so /usr/lib/arm-linux-gnueabihf/libmpfr.so.4

# Verify
ls -la /usr/lib/arm-linux-gnueabihf/libmpfr.so.4
```

### Error: "arm-linux-gnueabihf-gcc: command not found"

**Symptoms:**
```
arm-linux-gnueabihf-gcc: command not found
```

**Causes:**
- ARM toolchain not installed
- NEXMON_ROOT not set
- PATH not configured

**Solutions:**

```bash
# Install ARM toolchain
sudo apt-get install gcc-arm-linux-gnueabihf

# Verify installation
which arm-linux-gnueabihf-gcc
arm-linux-gnueabihf-gcc --version

# If using Nexmon, ensure environment is sourced
cd ~/nexmon
source setup_env.sh

# Verify CC is set
echo $CC
# Should show path to compiler
```

### Error: "permission denied" during make install

**Symptoms:**
```
make: *** [install-firmware] Error 1
cp: cannot create regular file '/lib/modules/...': Permission denied
```

**Causes:**
- Not running with sudo
- Incorrect permissions on system directories

**Solutions:**

```bash
# Use sudo for installation
sudo make -f Makefile.rpi install-firmware

# Or become root
sudo su
make -f Makefile.rpi install-firmware
exit

# Verify installation
lsmod | grep brcmfmac
```

### Error: "NEXMON_ROOT not set"

**Symptoms:**
```
Makefile:XX: *** NEXMON_ROOT not set. Stop.
```

**Causes:**
- Nexmon environment not sourced
- Wrong shell session

**Solutions:**

```bash
# Navigate to Nexmon directory
cd ~/nexmon

# Source environment
source setup_env.sh

# Verify
echo $NEXMON_ROOT
# Should show /home/pi/nexmon or similar

# Now try building again
cd ~/nexmon/patches/bcm43455c0/7_45_189/nexmon_csi
make -f Makefile.rpi
```

### Error: "NDK version mismatch" (Android)

**Symptoms:**
```
NDK version mismatch
Expected: r11c
Found: r12
```

**Causes:**
- Wrong NDK version installed
- Using newer NDK instead of r11c

**Solutions:**

```bash
# Download exact NDK r11c version
# From: https://developer.android.com/ndk/downloads/older_releases

# Extract it
unzip android-ndk-r11c-linux-x86_64.zip

# Set correct path
export NDK_ROOT=~/android-ndk-r11c

# Verify version
$NDK_ROOT/ndk-build --version
# Should show: r11c
```

---

## Installation Issues

### Error: "Failed to install firmware"

**Symptoms:**
```
make: *** [install-firmware] Error 127
```

**Causes:**
- Device not connected
- ADB not working (Android)
- SSH not working (Router)

**Solutions:**

```bash
# For Android devices
adb devices
# Should list your device

# If not listed:
adb kill-server
adb start-server
adb devices

# For Raspberry Pi (local installation)
sudo make -f Makefile.rpi install-firmware

# For Router (remote installation)
export REMOTEADDR=192.168.1.1
make install-firmware REMOTEADDR=$REMOTEADDR

# Verify SSH connection first
ssh admin@$REMOTEADDR "echo 'SSH works'"
```

### Error: "Device not found" (ADB)

**Symptoms:**
```
error: device not found
```

**Causes:**
- USB cable not connected
- USB debugging not enabled
- ADB daemon not running

**Solutions:**

```bash
# Check USB connection
lsusb | grep -i nexus

# Restart ADB
adb kill-server
adb start-server

# List devices
adb devices

# If still not found:
# 1. Enable USB debugging on device
# 2. Accept USB debugging prompt on device
# 3. Try again

# Force reconnect
adb disconnect
adb connect <device_ip>
```

### Error: "SSH connection refused" (Router)

**Symptoms:**
```
ssh: connect to host 192.168.1.1 port 22: Connection refused
```

**Causes:**
- Router IP incorrect
- SSH not enabled on router
- Firewall blocking SSH

**Solutions:**

```bash
# Verify router IP
ping 192.168.1.1

# Check if SSH is open
nmap -p 22 192.168.1.1

# Enable SSH on router (via web interface)
# ASUS RT-AC86U: System Administration > SSH Daemon > Enable

# Try with different port if configured
ssh -p 2222 admin@192.168.1.1

# Verify credentials
ssh admin@192.168.1.1 "uname -a"
```

---

## Runtime Issues

### Error: "Interface not found"

**Symptoms:**
```
nexutil: cannot find interface wlan0
```

**Causes:**
- Interface name is different
- WiFi driver not loaded
- Interface down

**Solutions:**

```bash
# List available interfaces
ifconfig -a
# or
ip link show

# Find WiFi interface (usually wlan0, wlan1, etc.)
iw dev

# Bring interface up
sudo ifconfig wlan0 up

# Or using ip command
sudo ip link set wlan0 up

# Verify driver is loaded
lsmod | grep brcmfmac

# If not loaded, load it
sudo modprobe brcmfmac
```

### Error: "Permission denied" for nexutil

**Symptoms:**
```
nexutil: permission denied
```

**Causes:**
- Not running with sudo
- nexutil not in PATH
- Incorrect file permissions

**Solutions:**

```bash
# Use sudo
sudo nexutil -Iwlan0 -k

# Or check if nexutil is installed
which nexutil

# If not found, install it
sudo make -C ~/nexmon/utilities/nexutil install

# Verify permissions
ls -la /usr/bin/nexutil
# Should be executable
```

### Error: "Channel not supported"

**Symptoms:**
```
nexutil: chanspec 0x6863 not supported
```

**Causes:**
- Channel not in regulatory domain
- Channel not in regulations.c
- Regulatory domain not set

**Solutions:**

```bash
# Check current regulatory domain
iw reg get

# Set regulatory domain (example: US)
sudo iw reg set US

# Or modify regulations.c to add channel
# Edit: src/regulations.c
# Add your channel to: additional_valid_chanspecs[]

# Rebuild firmware
make -f Makefile.rpi clean
make -f Makefile.rpi
sudo make -f Makefile.rpi install-firmware
```

### Error: "Monitor mode not working"

**Symptoms:**
```
No packets captured on monitor interface
```

**Causes:**
- Monitor interface not created
- Monitor interface not up
- Wrong interface name

**Solutions:**

```bash
# Create monitor interface
sudo iw phy `iw dev wlan0 info | gawk '/wiphy/ {printf "phy" $2}'` interface add mon0 type monitor

# Bring it up
sudo ifconfig mon0 up

# Verify
iw dev
# Should show mon0 in monitor mode

# If creation fails, try alternative method
sudo iw phy phy0 interface add mon0 type monitor
sudo ifconfig mon0 up

# For Nexus devices
sudo nexutil -Iwlan0 -m1

# For ASUS router
ssh admin@192.168.1.1 "/usr/sbin/wl -i eth6 monitor 1"
```

---

## CSI Collection Issues

### Issue: "No CSI packets received"

**Symptoms:**
```
tcpdump: listening on wlan0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
# No packets shown
```

**Causes:**
- Extractor not configured
- No traffic on channel
- CSI data not being sent to port 5500
- Firewall blocking port 5500

**Solutions:**

```bash
# 1. Verify extractor is configured
sudo nexutil -k
# Should show correct channel

# 2. Reconfigure if needed
PARAMS=$(makecsiparams -c 6/20 -C 0 -N 0 -m ff:ff:ff:ff:ff:ff -b 0x88)
sudo nexutil -Iwlan0 -s500 -b -l34 -v$PARAMS

# 3. Verify monitor interface is up
ifconfig mon0

# 4. Generate traffic
ping -c 100 8.8.8.8

# 5. Capture on correct interface
sudo tcpdump -i wlan0 dst port 5500

# 6. Check firewall
sudo iptables -L | grep 5500
# If blocked, allow it:
sudo iptables -I INPUT -p udp --dport 5500 -j ACCEPT
```

### Issue: "Packets received but CSI data is empty"

**Symptoms:**
```
tcpdump shows packets but CSI data is all zeros
```

**Causes:**
- Extractor not properly configured
- MAC address filter too restrictive
- Frame type filter not matching

**Solutions:**

```bash
# 1. Reconfigure with broadcast MAC
PARAMS=$(makecsiparams -c 6/20 -C 0 -N 0 -m ff:ff:ff:ff:ff:ff -b 0x88)
sudo nexutil -Iwlan0 -s500 -b -l34 -v$PARAMS

# 2. Try different frame types
# Data frames
makecsiparams -c 6/20 -b 0x88

# All frames
makecsiparams -c 6/20 -b 0x00

# 3. Verify traffic source MAC
# Use tcpdump to see what's actually on the channel
sudo tcpdump -i wlan0 -e

# 4. Update MAC filter to match traffic
PARAMS=$(makecsiparams -c 6/20 -C 0 -N 0 -m <actual_mac> -b 0x88)
sudo nexutil -Iwlan0 -s500 -b -l34 -v$PARAMS
```

### Issue: "CSI packets on wrong port"

**Symptoms:**
```
tcpdump shows packets on different port than 5500
```

**Causes:**
- Port number in nexutil command incorrect
- Firmware compiled with different port

**Solutions:**

```bash
# Verify port 5500 is being used
sudo tcpdump -i wlan0 -n | grep UDP

# Check all UDP ports
sudo netstat -uln | grep LISTEN

# Reconfigure with correct port
# The -s parameter in nexutil specifies the port
sudo nexutil -Iwlan0 -s500 -b -l34 -v$PARAMS

# Listen on correct port
sudo tcpdump -i wlan0 dst port 5500
```

---

## Platform-Specific Issues

### Raspberry Pi Issues

#### Issue: "brcmfmac driver fails to load"

**Symptoms:**
```
modprobe: ERROR: could not insert 'brcmfmac': Exec format error
```

**Causes:**
- Driver compiled for wrong kernel version
- Kernel version changed

**Solutions:**

```bash
# Check kernel version
uname -r

# Rebuild driver for current kernel
cd ~/nexmon/patches/bcm43455c0/7_45_189/nexmon_csi
make -f Makefile.rpi clean
make -f Makefile.rpi
sudo make -f Makefile.rpi install-firmware
sudo reboot
```

#### Issue: "wpa_supplicant interferes with CSI"

**Symptoms:**
```
Interface keeps switching channels
CSI collection stops unexpectedly
```

**Solutions:**

```bash
# Stop wpa_supplicant
sudo pkill wpa_supplicant

# Disable it permanently
sudo systemctl disable wpa_supplicant
sudo systemctl stop wpa_supplicant

# Or remove it
sudo apt-get remove wpasupplicant
```

### Nexus Device Issues

#### Issue: "ADB connection unstable"

**Symptoms:**
```
adb: error: device offline
adb: error: device not found
```

**Solutions:**

```bash
# Restart ADB
adb kill-server
adb start-server

# Disconnect and reconnect
adb disconnect
adb connect <device_ip>

# Or via USB
adb usb

# Check device status
adb devices -l
```

#### Issue: "Root access lost"

**Symptoms:**
```
adb: insufficient permissions for device
```

**Solutions:**

```bash
# Restart adb as root
adb root

# Or reconnect device
adb disconnect
adb connect <device_ip>

# Verify root access
adb shell id
# Should show uid=0
```

### ASUS RT-AC86U Issues

#### Issue: "SSH authentication fails"

**Symptoms:**
```
Permission denied (publickey,password).
```

**Solutions:**

```bash
# Try default credentials
ssh admin@192.168.1.1
# Password: admin

# Or with password prompt
ssh -o PubkeyAuthentication=no admin@192.168.1.1

# Check if SSH is enabled
# Via web interface: System Administration > SSH Daemon > Enable
```

#### Issue: "nexutil not found on router"

**Symptoms:**
```
/jffs/nexutil: command not found
```

**Solutions:**

```bash
# Verify nexutil was copied
ssh admin@192.168.1.1 "ls -la /jffs/nexutil"

# If missing, copy it again
scp ~/nexmon/utilities/nexutil/nexutil admin@192.168.1.1:/jffs/
ssh admin@192.168.1.1 "chmod +x /jffs/nexutil"

# Verify
ssh admin@192.168.1.1 "/jffs/nexutil -h"
```

---

## Performance Issues

### Issue: "Low CSI packet rate"

**Symptoms:**
```
Only receiving a few CSI packets per second
```

**Causes:**
- Low WiFi traffic
- Extractor not optimized
- Channel congestion

**Solutions:**

```bash
# Generate more traffic
iperf -c <server_ip> -t 60 -i 1

# Or use frame injection (requires Nexmon on transmitter)

# Increase traffic rate
ping -c 1000 -i 0.01 8.8.8.8

# Monitor packet rate
sudo tcpdump -i wlan0 dst port 5500 | head -100
```

### Issue: "High CPU usage"

**Symptoms:**
```
CSI collection process using 80-100% CPU
```

**Causes:**
- Too much traffic
- Inefficient packet processing

**Solutions:**

```bash
# Reduce traffic rate
ping -c 100 -i 1 8.8.8.8

# Use MAC address filter to reduce packets
PARAMS=$(makecsiparams -c 6/20 -C 0 -N 0 -m <specific_mac> -b 0x88)
sudo nexutil -Iwlan0 -s500 -b -l34 -v$PARAMS

# Use frame type filter
makecsiparams -c 6/20 -b 0x88  # Only data frames

# Process packets in batches instead of real-time
```

### Issue: "Packet loss during collection"

**Symptoms:**
```
Gaps in sequence numbers
Missing CSI data
```

**Causes:**
- Buffer overflow
- Slow disk I/O
- Network congestion

**Solutions:**

```bash
# Use faster storage
# Avoid writing to USB or network drives

# Increase buffer size
sudo sysctl -w net.core.rmem_max=134217728
sudo sysctl -w net.core.rmem_default=134217728

# Use tcpdump with larger snaplen
sudo tcpdump -i wlan0 dst port 5500 -s 65535 -w csi_capture.pcap

# Reduce traffic rate
ping -c 100 -i 0.1 8.8.8.8
```

---

## Getting Help

If you're still experiencing issues:

1. Check the original README.md for additional information
2. Review the project's GitHub issues: https://github.com/seemoo-lab/nexmon_csi/issues
3. Consult Nexmon documentation: https://nexmon.org
4. Check kernel logs: `sudo dmesg | tail -50`
5. Enable verbose logging in build: `make V=1`

