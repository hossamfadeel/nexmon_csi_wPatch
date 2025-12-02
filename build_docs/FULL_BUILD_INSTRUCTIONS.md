# Nexmon CSI - Complete Build Instructions

## Table of Contents

1. [Raspberry Pi (bcm43455c0)](#raspberry-pi-bcm43455c0)
2. [Nexus Devices (bcm4339/bcm4358)](#nexus-devices)
3. [ASUS RT-AC86U (bcm4366c0)](#asus-rt-ac86u-bcm4366c0)
4. [Verification & Testing](#verification--testing)

---

## Raspberry Pi (bcm43455c0)

### Prerequisites

Ensure you have:
- Raspberry Pi 3B+, 4B, or 5 with Raspberry Pi OS installed
- SSH access to the Pi (or direct terminal access)
- Internet connectivity on the Pi
- Minimum 8GB microSD card

### Step 1: Update System and Install Dependencies

```bash
# SSH into your Raspberry Pi
ssh pi@raspberrypi.local
# or: ssh pi@<your-pi-ip>

# Update package lists
sudo apt-get update
sudo apt-get upgrade -y

# Install essential build tools
sudo apt-get install -y \
    git \
    gawk \
    qpdf \
    flex \
    bison \
    xxd \
    make \
    gcc \
    libgmp3-dev \
    automake \
    autoconf \
    libtool \
    texinfo
```

### Step 2: Install Kernel Headers

```bash
# Install Raspberry Pi kernel headers
sudo apt-get install -y raspberrypi-kernel-headers

# Verify installation
ls /lib/modules/$(uname -r)/build/

# You should see kernel header files. If not, reboot and try again:
sudo reboot
```

### Step 3: Build ISL and MPFR Libraries (if needed)

Check if the required libraries exist:

```bash
# Check for ISL
ls /usr/lib/arm-linux-gnueabihf/libisl.so.10

# Check for MPFR
ls /usr/lib/arm-linux-gnueabihf/libmpfr.so.4
```

If either is missing, build from source:

```bash
# Clone Nexmon to access build tools
cd ~
git clone https://github.com/seemoo-lab/nexmon.git
cd nexmon

# Build ISL if needed
cd buildtools/isl-0.10
./configure
make
sudo make install
sudo ln -s /usr/local/lib/libisl.so /usr/lib/arm-linux-gnueabihf/libisl.so.10
cd ../..

# Build MPFR if needed
cd buildtools/mpfr-3.1.4
autoreconf -f -i
./configure
make
sudo make install
sudo ln -s /usr/local/lib/libmpfr.so /usr/lib/arm-linux-gnueabihf/libmpfr.so.4
cd ../..
```

### Step 4: Set Up Nexmon Build Environment

```bash
# From the nexmon directory
cd ~/nexmon

# Source the environment setup script
source setup_env.sh

# Verify environment variables are set
echo $NEXMON_ROOT
# Should output: /home/pi/nexmon (or your path)

# Extract firmware components (this may take a few minutes)
make

# Navigate to utilities and build them
cd utilities
make
```

### Step 5: Clone Nexmon CSI Repository

```bash
# Navigate to the appropriate patch directory
cd ~/nexmon/patches/bcm43455c0/7_45_189/

# Clone the Nexmon CSI repository
git clone https://github.com/seemoo-lab/nexmon_csi.git

# Enter the directory
cd nexmon_csi
```

### Step 6: Build Nexmon CSI Firmware

```bash
# From the nexmon_csi directory
# Using the Raspberry Pi specific Makefile

make -f Makefile.rpi

# This will compile the CSI extraction firmware
# You should see output like:
#   COMPILING src/csi_extractor.c => obj/csi_extractor.o
#   LINKING OBJECTS => gen/patch.elf
#   APPLYING PATCHES => brcmfmac_5.10.y-nexmon/brcmfmac.ko
#   etc.

# If successful, you'll have a compiled firmware file
```

### Step 7: Install Compiled Firmware

```bash
# Install the compiled firmware
sudo make -f Makefile.rpi install-firmware

# This will:
# 1. Backup the original brcmfmac driver
# 2. Install the patched driver
# 3. Load the new driver into the kernel
```

### Step 8: Build and Install Nexutil

```bash
# Navigate back to Nexmon utilities
cd ~/nexmon/utilities/nexutil

# Compile nexutil
make

# Install nexutil system-wide
sudo make install

# Verify installation
which nexutil
nexutil -h
```

### Step 9: Optional - Remove wpa_supplicant

For better control over the WiFi interface, you may want to remove wpa_supplicant:

```bash
sudo apt-get remove -y wpasupplicant
```

### Step 10: Build makecsiparams Utility

```bash
# Navigate to the makecsiparams directory
cd ~/nexmon/patches/bcm43455c0/7_45_189/nexmon_csi/utils/makecsiparams

# Compile makecsiparams
make

# Install it
sudo make install

# Verify installation
which makecsiparams
makecsiparams -h
```

### Step 11: Reboot and Verify Installation

```bash
# Reboot the Raspberry Pi
sudo reboot

# After reboot, verify the driver is loaded
lsmod | grep brcmfmac

# Check that the interface is available
ifconfig wlan0
# or
ip link show wlan0
```

---

## Nexus Devices

### Prerequisites for bcm4339 (Nexus 5) and bcm4358 (Nexus 6P)

- Rooted Android device (Nexus 5 or 6P)
- USB debugging enabled
- ADB installed on build machine
- Android NDK r11c (EXACT version)
- Ubuntu 16.04 LTS or compatible

### Step 1: Install Build Dependencies on Linux Machine

```bash
# On your Ubuntu build machine (NOT on the phone)
sudo apt-get update
sudo apt-get install -y \
    git \
    gawk \
    qpdf \
    adb \
    flex \
    bison \
    xxd \
    make \
    gcc

# For x86_64 systems, install 32-bit support
sudo dpkg --add-architecture i386
sudo apt-get update
sudo apt-get install -y \
    libc6:i386 \
    libncurses5:i386 \
    libstdc++6:i386
```

### Step 2: Download Android NDK r11c

```bash
# Download from (exact version required):
# https://developer.android.com/ndk/downloads/older_releases

# Extract it
cd ~
unzip android-ndk-r11c-linux-x86_64.zip

# Set environment variable
export NDK_ROOT=~/android-ndk-r11c
```

### Step 3: Clone and Setup Nexmon

```bash
cd ~
git clone https://github.com/seemoo-lab/nexmon.git
cd nexmon

# Source environment
source setup_env.sh

# Extract firmware
make

# Build utilities
cd utilities
make
```

### Step 4: Clone Nexmon CSI

For Nexus 5 (bcm4339):
```bash
cd ~/nexmon/patches/bcm4339/6_37_34_43/
git clone https://github.com/seemoo-lab/nexmon_csi.git
cd nexmon_csi
```

For Nexus 6P (bcm4358):
```bash
cd ~/nexmon/patches/bcm4358/7_112_300_14_sta/
git clone https://github.com/seemoo-lab/nexmon_csi.git
cd nexmon_csi
```

### Step 5: Connect Device and Build

```bash
# Connect your rooted Nexus device via USB
# Verify ADB connection
adb devices

# Build the firmware
make

# Install on device
make install-firmware
```

---

## ASUS RT-AC86U (bcm4366c0)

### Prerequisites

- ASUS RT-AC86U router
- SSH access enabled on router
- Admin credentials
- Ubuntu 18.04.3 LTS (ARM64 compatible)
- ARM64 toolchain

### Step 1: Install Dependencies

```bash
sudo apt-get update
sudo apt-get install -y \
    git \
    gawk \
    qpdf \
    flex \
    bison \
    xxd \
    make \
    gcc

# For x86_64 systems
sudo dpkg --add-architecture i386
sudo apt-get install -y \
    libc6:i386 \
    libncurses5:i386 \
    libstdc++6:i386
```

### Step 2: Clone Nexmon and Setup

```bash
cd ~
git clone https://github.com/seemoo-lab/nexmon.git
cd nexmon

source setup_env.sh
make
cd utilities
make
```

### Step 3: Clone ARM64 Toolchain

```bash
cd ~
git clone https://github.com/RMerl/am-toolchains.git

# Set compiler environment
export AMCC=$(pwd)/am-toolchains/brcm-arm-hnd/crosstools-aarch64-gcc-5.3-linux-4.1-glibc-2.22-binutils-2.25/usr/bin/aarch64-buildroot-linux-gnu-
export LD_LIBRARY_PATH=$(pwd)/am-toolchains/brcm-arm-hnd/crosstools-aarch64-gcc-5.3-linux-4.1-glibc-2.22-binutils-2.25/usr/lib
```

### Step 4: Clone Nexmon CSI

```bash
cd ~/nexmon/patches/bcm4366c0/10_10_122_20/
git clone https://github.com/seemoo-lab/nexmon_csi.git
cd nexmon_csi
```

### Step 5: Build Firmware

```bash
# Set your router's IP address
export REMOTEADDR=192.168.1.1  # Change to your router's IP

make

# Install on router
make install-firmware REMOTEADDR=$REMOTEADDR
```

### Step 6: Build and Install Nexutil for Router

```bash
cd ~/nexmon/utilities/libnexio

# Compile libnexio
${AMCC}gcc -c libnexio.c -o libnexio.o -DBUILD_ON_RPI
${AMCC}ar rcs libnexio.a libnexio.o

# Build nexutil
cd ../nexutil
echo "typedef uint32_t uint;" > types.h
sed -i 's/argp-extern/argp/' nexutil.c
${AMCC}gcc -static -o nexutil nexutil.c bcmutils.c bcmwifi_channels.c b64-encode.c b64-decode.c \
    -DBUILD_ON_RPI -DVERSION=0 \
    -I. -I./include -I../libnexio -I../../patches/include \
    -L../libnexio/ -lnexio

# Copy to router
scp nexutil admin@$REMOTEADDR:/jffs/nexutil
ssh admin@$REMOTEADDR "/bin/chmod +x /jffs/nexutil"
```

---

## Verification & Testing

### Verify Installation on Raspberry Pi

```bash
# Check driver is loaded
lsmod | grep brcmfmac

# Check interface is up
ifconfig wlan0

# Test makecsiparams
makecsiparams -h

# Test nexutil
nexutil -h
```

### Test CSI Extraction (Raspberry Pi)

```bash
# 1. Ensure wpa_supplicant is not running
pkill wpa_supplicant

# 2. Bring up the interface
ifconfig wlan0 up

# 3. Generate CSI parameters for channel 6 with 20MHz bandwidth
makecsiparams -c 6/20 -C 1 -N 1 -m 00:11:22:33:44:55 -b 0x88

# Output should be a base64 string like:
# m+IBEQGIAgAAESIzRFWqu6q7qrsAAAAAAAAAAAAAAAAAAA==

# 4. Configure the extractor with the parameters
nexutil -Iwlan0 -s500 -b -l34 -v<your-base64-string>

# 5. Create monitor interface
iw phy `iw dev wlan0 info | gawk '/wiphy/ {printf "phy" $2}'` interface add mon0 type monitor
ifconfig mon0 up

# 6. Listen for CSI packets on UDP port 5500
tcpdump -i wlan0 dst port 5500

# In another terminal, generate WiFi traffic to trigger CSI collection
# You should see UDP packets with CSI data
```

### Common Test Commands

```bash
# Check current channel
nexutil -k

# Get driver version
nexutil -v

# Monitor mode verification
iw dev

# Interface statistics
ifconfig wlan0
```

---

## Build Troubleshooting

### Issue: "make: *** No rule to make target"

**Solution:**
```bash
# Ensure you're in the correct directory
pwd  # Should show nexmon_csi directory

# Verify Makefile exists
ls -la Makefile*

# Try cleaning and rebuilding
make clean
make -f Makefile.rpi
```

### Issue: "Kernel version mismatch"

**Solution:**
```bash
# Update kernel headers
sudo apt-get install --reinstall raspberrypi-kernel-headers

# Reboot
sudo reboot

# Verify
uname -r
ls /lib/modules/$(uname -r)/build/
```

### Issue: "libisl.so.10 not found"

**Solution:**
```bash
# Install ISL development package
sudo apt-get install libisl-dev

# Create symlink if needed
sudo ln -s /usr/lib/arm-linux-gnueabihf/libisl.so.15 /usr/lib/arm-linux-gnueabihf/libisl.so.10
```

### Issue: "Permission denied" during installation

**Solution:**
```bash
# Use sudo for installation
sudo make -f Makefile.rpi install-firmware

# Or use sudo su to become root
sudo su
make -f Makefile.rpi install-firmware
exit
```

### Issue: "ADB device not found"

**Solution:**
```bash
# Check USB connection
adb devices

# If not listed, try:
adb kill-server
adb start-server
adb devices

# Enable USB debugging on device and try again
```

---

## Next Steps

After successful build and installation:

1. Refer to `USAGE_GUIDE.md` for CSI extraction procedures
2. Check `TROUBLESHOOTING.md` for runtime issues
3. Review `PATCH_NOTES.md` for code modifications
4. Consult the original README.md for advanced usage

