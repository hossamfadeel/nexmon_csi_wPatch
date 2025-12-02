# Nexmon CSI - Complete Requirements Guide

## Overview

This document provides a comprehensive list of all system requirements, dependencies, and environment configurations needed to successfully build and deploy the Nexmon CSI extraction framework.

---

## System Requirements

### Operating System

| Platform | Supported Versions | Recommended Version |
|----------|-------------------|-------------------|
| **Raspberry Pi OS** | 4.19, 5.4, 5.10 kernel | Raspberry Pi OS Bullseye (5.10 kernel) |
| **Linux (x86_64)** | Ubuntu 16.04+ | Ubuntu 20.04 LTS or 22.04 LTS |
| **Linux (ARM64)** | Debian-based | Ubuntu 18.04+ for ARM64 |

### Hardware Requirements

#### For Raspberry Pi (bcm43455c0)

- **Device:** Raspberry Pi 3B+, 4B, or 5
- **WiFi Chip:** BCM43455C0 (built-in)
- **RAM:** Minimum 1GB (2GB+ recommended)
- **Storage:** 8GB microSD card minimum (16GB+ recommended)
- **Power Supply:** 5V/2.5A or better

#### For Nexus Devices (bcm4339, bcm4358)

- **Device:** Rooted Nexus 5 (bcm4339) or Nexus 6P (bcm4358)
- **Android Version:** 4.4+ (Nexus 5), 6.0+ (Nexus 6P)
- **USB Debugging:** Enabled
- **Root Access:** Required (via SuperSU or Magisk)

#### For Router (bcm4366c0)

- **Device:** ASUS RT-AC86U
- **SSH Access:** Enabled
- **Admin Credentials:** Required

---

## Software Dependencies

### Core Build Tools

```bash
# Essential build utilities
git                 # Version control (any recent version)
gcc                 # C compiler (4.8+)
make                # Build automation (3.81+)
gawk                # Text processing (4.0+)
flex                # Lexical analyzer (2.5+)
bison               # Parser generator (2.5+)
qpdf                # PDF utilities (5.0+)
xxd                 # Hex dump utility (any)
```

### Raspberry Pi Specific

```bash
# Kernel headers for driver compilation
raspberrypi-kernel-headers  # Matches kernel version

# Additional libraries
libgmp3-dev                 # GNU Multiple Precision library
libmpfr-dev                 # Multiple Precision Floating-Point
libisl-dev                  # Integer Set Library
automake                    # Makefile generation
autoconf                    # Configuration scripts
libtool                     # Shared library support
texinfo                     # Documentation format
```

### Android/Nexus Specific

```bash
adb                         # Android Debug Bridge
Android NDK r11c            # EXACT version required for bcm4339/bcm4358
```

### x86_64 Multilib Support (if on x86_64)

```bash
# For 32-bit compilation support
libc6:i386                  # 32-bit C library
libncurses5:i386            # 32-bit ncurses
libstdc++6:i386             # 32-bit C++ library
```

---

## Firmware Requirements

### Broadcom Firmware Files

The following firmware files are required and will be extracted from the original firmware during the build process:

| Chipset | Firmware Version | Source | Size |
|---------|-----------------|--------|------|
| bcm4339 | 6_37_34_43 | Nexus 5 | ~500KB |
| bcm43455c0 | 7_45_189 | Raspberry Pi | ~600KB |
| bcm4358 | 7_112_300_14_sta | Nexus 6P | ~550KB |
| bcm4366c0 | 10_10_122_20 | ASUS RT-AC86U | ~700KB |

**Note:** These files are extracted from the device's original firmware during the build process. The extraction is handled automatically by the Nexmon framework.

---

## Python Requirements

### Python Version

- **Minimum:** Python 3.6
- **Recommended:** Python 3.8+
- **Not Required:** Python 2.x (deprecated)

### Python Packages (Optional for CSI Analysis)

```bash
numpy               # Numerical computing
scipy               # Scientific computing
matplotlib          # Data visualization
pandas              # Data analysis
```

Install with:
```bash
pip3 install numpy scipy matplotlib pandas
```

---

## Toolchain Requirements

### Nexmon Base Framework

The Nexmon CSI project requires the base Nexmon framework:

```bash
git clone https://github.com/seemoo-lab/nexmon.git
cd nexmon
source setup_env.sh
```

**Environment Variables Set by Nexmon:**

- `NEXMON_ROOT` - Path to Nexmon framework
- `CC` - Compiler prefix
- `CCPLUGIN` - GCC plugin path
- `NEXMON_CHIP` - Chip identifier
- `NEXMON_FW_VERSION` - Firmware version

### ARM Toolchain (for bcm4366c0)

For ASUS RT-AC86U support, an ARM64 toolchain is required:

```bash
git clone https://github.com/RMerl/am-toolchains.git
export AMCC=$(pwd)/am-toolchains/brcm-arm-hnd/crosstools-aarch64-gcc-5.3-linux-4.1-glibc-2.22-binutils-2.25/usr/bin/aarch64-buildroot-linux-gnu-
```

---

## Kernel Header Requirements

### Raspberry Pi

```bash
# Automatically installed with
apt install raspberrypi-kernel-headers

# Verify installation
ls /lib/modules/$(uname -r)/build/
```

### Custom Kernel Builds

If using a custom kernel, ensure headers are available:

```bash
# Build kernel headers
cd /usr/src/linux
make modules_prepare
```

---

## Library Dependencies

### ISL (Integer Set Library)

**Requirement:** `/usr/lib/arm-linux-gnueabihf/libisl.so.10`

If missing, build from source:

```bash
cd buildtools/isl-0.10
./configure
make
sudo make install
sudo ln -s /usr/local/lib/libisl.so /usr/lib/arm-linux-gnueabihf/libisl.so.10
```

### MPFR (Multiple Precision Floating-Point)

**Requirement:** `/usr/lib/arm-linux-gnueabihf/libmpfr.so.4`

If missing, build from source:

```bash
cd buildtools/mpfr-3.1.4
autoreconf -f -i
./configure
make
sudo make install
sudo ln -s /usr/local/lib/libmpfr.so /usr/lib/arm-linux-gnueabihf/libmpfr.so.4
```

---

## Environment Variables

### Essential Variables

```bash
# Set by Nexmon setup_env.sh
export NEXMON_ROOT=/path/to/nexmon
export FW_PATH=${NEXMON_ROOT}/patches/bcm43455c0/7_45_189
export KERNEL_VERSION=$(uname -r)

# For Android builds
export NDK_ROOT=/path/to/android-ndk-r11c
export ANDROID_SDK_ROOT=/path/to/android-sdk
```

### Optional Variables

```bash
# For remote deployment
export REMOTEADDR=192.168.1.100    # Router IP for bcm4366c0
export ADBSERIAL=<device_serial>   # Android device serial

# Build optimization
export MAKEFLAGS="-j$(nproc)"      # Parallel build jobs
```

---

## Kernel Module Requirements

### For Raspberry Pi

The build process requires:

1. **Kernel Headers** - Matching the running kernel version
2. **Build Essentials** - gcc, make, etc.
3. **brcmfmac Driver** - Either stock or modified version

### For Custom Kernels

Ensure the following are available:

```bash
# Check kernel configuration
cat /boot/config-$(uname -r)

# Required options
CONFIG_WIRELESS=y
CONFIG_CFG80211=m
CONFIG_MAC80211=m
CONFIG_BRCMFMAC=m
```

---

## Network Requirements

### For Remote Deployment

- **SSH Access:** Required for bcm4366c0 (ASUS RT-AC86U)
- **Port 22:** SSH port (configurable)
- **Port 5500:** UDP port for CSI data collection

### For Local Development

- **USB Connection:** For Android devices (ADB)
- **Direct Connection:** For Raspberry Pi (SSH or direct serial)

---

## Storage Requirements

### Disk Space

| Component | Size |
|-----------|------|
| Nexmon base repository | ~500MB |
| Nexmon CSI repository | ~50MB |
| Build artifacts | ~200MB |
| Compiled firmware | ~5-10MB |
| Total (with all platforms) | ~1GB |

### Temporary Space

During compilation, approximately 500MB of temporary space is required in `/tmp`.

---

## Verification Checklist

Before starting the build, verify:

- [ ] Operating system version matches requirements
- [ ] All required packages installed: `git`, `gcc`, `make`, `gawk`, `flex`, `bison`, `qpdf`, `xxd`
- [ ] Kernel headers installed and match running kernel
- [ ] Nexmon base framework cloned and environment sourced
- [ ] Sufficient disk space available (>1GB)
- [ ] Network connectivity for downloading dependencies
- [ ] Device connectivity (USB for Android, SSH for Pi/Router)
- [ ] Required firmware files accessible
- [ ] Appropriate toolchain installed for target platform

---

## Troubleshooting Common Issues

### Missing Library Errors

**Error:** `libisl.so.10: cannot open shared object file`

**Solution:**
```bash
sudo apt install libisl-dev
sudo ln -s /usr/lib/*/libisl.so.10 /usr/lib/arm-linux-gnueabihf/
```

### Kernel Header Mismatch

**Error:** `kernel version mismatch`

**Solution:**
```bash
# Update kernel headers
sudo apt update
sudo apt install raspberrypi-kernel-headers
sudo reboot
```

### Compiler Not Found

**Error:** `arm-linux-gnueabihf-gcc: command not found`

**Solution:**
```bash
# Install ARM toolchain
sudo apt install gcc-arm-linux-gnueabihf
```

### NDK Version Issues (Android)

**Error:** `NDK version mismatch`

**Solution:** Download exactly NDK r11c from:
```
https://developer.android.com/ndk/downloads/older_releases
```

---

## Version Compatibility Matrix

| Kernel | Raspberry Pi OS | Status |
|--------|-----------------|--------|
| 4.19 | Buster | Supported |
| 5.4 | Bullseye | Supported |
| 5.10 | Bullseye/Bookworm | Supported (Recommended) |
| 5.15+ | Bookworm | Requires Makefile.rpi |

---

## Next Steps

After verifying all requirements:

1. Clone the Nexmon base repository
2. Set up the build environment
3. Clone the Nexmon CSI repository
4. Follow platform-specific build instructions
5. Refer to `FULL_BUILD_INSTRUCTIONS.md` for detailed steps

