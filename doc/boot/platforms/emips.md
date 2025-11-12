# NetBSD/emips Boot Documentation

$NetBSD$

## Platform Overview

**Architecture**: MIPS 32-bit (little-endian)
**Port Date**: 2010-01-23
**Boot Method**: eMIPS simulator → bootxx → boot → kernel
**Firmware**: Minimal eMIPS firmware
**MMU Requirements**: MIPS TLB

## Hardware Support

NetBSD/emips supports the **eMIPS** (Extensible MIPS) platform:

- **CPU**: Custom MIPS32-like processor
- **Memory**: Configurable (typically 64-256MB)
- **Boot Devices**: IDE disk, virtual devices
- **Platform**: Research/educational MIPS simulator
- **Developed by**: Microsoft Research

### eMIPS Simulator

**Purpose**: Educational and research MIPS platform
**Features**:
- Cycle-accurate MIPS simulation
- Configurable hardware
- Multiple bus types (PCI-like, custom)
- Virtual I/O devices

## Boot Process

### Three-Stage Boot

1. **eMIPS firmware**: Minimal initialization, loads bootxx
2. **bootxx**: Primary bootloader (filesystem-specific, ~7KB)
3. **boot**: Secondary bootloader (full-featured, ~64KB)
4. **Kernel**: NetBSD/emips kernel

### Stage 1: eMIPS Firmware

**Functions**:
- Basic hardware initialization
- Load bootxx from disk sector 0
- Transfer control to bootxx

**Minimal Firmware**: Unlike other platforms, eMIPS has very simple firmware with no interactive shell.

### Stage 2: Primary Bootloader (bootxx)

**Location**: `/sys/arch/emips/stand/bootxx_*/`
**Size**: Maximum 7680 bytes (15 sectors)

**Variants**:
- `bootxx_ffs` - Fast File System
- `bootxx_lfs` - Log-structured File System  
- `bootxx_cd9660` - ISO 9660 (CD-ROM)

**Functionality**:
- Minimal filesystem code
- Locate `/boot` file
- Load secondary bootloader
- Transfer control

### Stage 3: Secondary Bootloader (boot)

**Location**: `/sys/arch/emips/stand/boot/`
**Size**: ~64KB

**Features**:
- Full filesystem support
- ELF kernel loading
- Gzip compression support
- Direct hardware access (no firmware services)

**Source**: `/sys/arch/emips/stand/common/start.S`, `boot/boot.c`

### Stage 4: Kernel

**Entry Point**: `mach_init()`
**Source**: `/sys/arch/emips/emips/machdep.c`

## Memory Map

### Virtual Address Space

```
0x00000000 - 0x7FFFFFFF : useg   (2GB, user, TLB-mapped)
0x80000000 - 0x9FFFFFFF : kseg0  (512MB, kernel cached)
0xA0000000 - 0xBFFFFFFF : kseg1  (512MB, kernel uncached)
0xC0000000 - 0xFFFFFFFF : kseg2  (1GB, kernel TLB-mapped)
```

### Physical Memory

```
0x00000000 - 0x0FFFFFFF : Main RAM (up to 256MB)
0x10000000 - 0x1FFFFFFF : I/O devices
0x1FC00000 - 0x1FFFFFFF : Boot ROM
```

## Build Instructions

```sh
# Build bootloaders
cd /sys/arch/emips/stand
make

# Build kernel
./build.sh -m emips kernel=GENERIC

# Or manually
cd /sys/arch/emips/conf
config GENERIC
cd ../compile/GENERIC
make depend
make
```

## Installation

### Using installboot

```sh
# Install bootxx
installboot /dev/rwd0a /usr/mdec/bootxx_ffs

# Copy boot and kernel
cp /usr/mdec/boot /boot
cp netbsd /netbsd
```

### In eMIPS Simulator

The simulator typically loads disk images directly. Follow simulator documentation for disk image creation and booting.

## Debugging

**DDB Kernel Debugger**:
```
options     DDB
makeoptions COPY_SYMTAB=1
```

**eMIPS Debugging**: Use simulator's built-in debugging facilities.

## Source Code Reference

**Bootloaders**:
- `/sys/arch/emips/stand/bootxx_ffs/` - Primary bootloader
- `/sys/arch/emips/stand/boot/` - Secondary bootloader
- `/sys/arch/emips/stand/common/` - Shared code

**Kernel**:
- `/sys/arch/emips/emips/machdep.c` - Machine-dependent code
- `/sys/arch/emips/emips/autoconf.c` - Autoconfiguration

**Include**:
- `/sys/arch/emips/include/emipsreg.h` - eMIPS registers

---

*Last Updated: 2025-11-12*
*Architecture Maintainer: NetBSD/emips Port*
