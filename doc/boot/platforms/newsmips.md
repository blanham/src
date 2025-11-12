# NetBSD/newsmips Boot Documentation

$NetBSD$

## Platform Overview

**Architecture**: MIPS 32-bit (big-endian)
**Port Date**: 1998-02-17
**Boot Method**: Sony firmware → bootxx → boot → kernel
**Firmware**: Sony NEWS MIPS PROM
**MMU Requirements**: MIPS R3000/R4000 TLB

## Hardware Support

NetBSD/newsmips supports Sony NEWS MIPS workstations:

- **CPU**: MIPS R3000, R4000, R4400 processors
- **Memory**: 16MB to 256MB
- **Boot Devices**: SCSI disk, floppy, network
- **Firmware**: Sony NEWS-OS PROM
- **Console**: Serial or graphics

### Supported Systems

**NEWS-3000 series** (R3000):
- NWS-3200, 3260, 3410, 3430, 3440, 3460, 3470, 3710, 3720

**NEWS-5000 series** (R4000):
- NWS-5000

**APbus-based systems** (advanced):
- Various NEWS workstations with APbus

## Boot Process

### Boot Stages

1. **Sony NEWS PROM**: Firmware
2. **bootxx**: Primary bootloader (8KB)
3. **boot**: Secondary bootloader
4. **Kernel**: NetBSD kernel

### Stage 1: Sony NEWS PROM

**Functions**:
- Hardware initialization
- System diagnostics
- Boot device selection
- Load bootxx from disk

**PROM Interfaces**:
- **APCALL**: APbus-based machines
- **ROMCALL**: Older NEWS-3000 machines

**Boot Command**:
```
>> b           # Boot from default device
>> b sd        # Boot from SCSI disk
```

### Stage 2: Primary Bootloader (bootxx)

**Location**: `/sys/arch/newsmips/stand/bootxx/`
**Size**: ~8KB
**Format**: Raw binary for NEWS disk layout

**Functionality**:
- Minimal filesystem code (FFS)
- Locate `/boot` file
- Load secondary bootloader
- Transfer control

**Source**: `/sys/arch/newsmips/stand/bootxx/bootxx.c`

### Stage 3: Secondary Bootloader (boot)

**Location**: `/sys/arch/newsmips/stand/boot/`

**Features**:
- Full FFS support
- ELF kernel loading
- Gzip compression support
- PROM callbacks (APCALL/ROMCALL)

**Source**: `/sys/arch/newsmips/stand/boot/boot.c` (includes detection of APbus vs. non-APbus)

**Boot Logic**:
```c
/* Detect machine type */
if (a3 >= 0x80000000)
    apbus = 1;      /* APbus-based machine */
else
    apbus = 0;      /* NEWS-3000 series */

/* Use appropriate PROM interface */
if (apbus)
    use_apcall();
else
    use_romcall();
```

### Stage 4: Kernel

**Entry Point**: `mach_init()`
**Source**: `/sys/arch/newsmips/newsmips/machdep.c`

## Memory Map

### Virtual Address Space

```
0x00000000 - 0x7FFFFFFF : useg   (2GB, user, TLB-mapped)
0x80000000 - 0x9FFFFFFF : kseg0  (512MB, kernel cached)
0xA0000000 - 0xBFFFFFFF : kseg1  (512MB, kernel uncached)
0xC0000000 - 0xFFFFFFFF : kseg2  (1GB, kernel TLB-mapped)
```

### Physical Memory

**NEWS-3000**:
```
0x00000000 - 0x0FFFFFFF : Main RAM (up to 256MB)
0xB0000000 - 0xBFFFFFFF : I/O devices
0x1FC00000 - 0x1FFFFFFF : Boot ROM
```

**APbus-based**:
```
0x00000000 - 0x0FFFFFFF : Main RAM
0x10000000 - 0x1FFFFFFF : APbus space
0xB0000000 - 0xBFFFFFFF : I/O devices
```

## APbus vs. Non-APbus

### Non-APbus (NEWS-3000)

**Characteristics**:
- Older NEWS-3000 series
- ROMCALL interface
- Simpler I/O architecture

**PROM Interface**: ROMCALL functions

### APbus (Advanced)

**Characteristics**:
- Newer systems
- APCALL interface
- Advanced bus architecture
- System information structure

**PROM Interface**: APCALL functions + system info pointer

## Build Instructions

```sh
# Build bootloaders
cd /sys/arch/newsmips/stand
make

# Build kernel
./build.sh -m newsmips kernel=GENERIC
```

## Installation

### Disk Layout

NEWS systems use a specific disk layout:
- **Sector 0**: Boot block
- **Sectors 1-15**: bootxx (primary bootloader)
- **Filesystem**: FFS starting at higher sectors

### Installing Bootloader

```sh
# Install bootxx
dd if=/usr/mdec/bootxx of=/dev/rsd0c bs=512 seek=1 count=15

# Copy boot to filesystem
mount /dev/sd0a /mnt
cp /usr/mdec/boot /mnt/boot
cp netbsd /mnt/netbsd
umount /mnt
```

## Debugging

**Serial Console**:
- Baud: 9600
- Data: 8N1

**DDB Kernel Debugger**:
```
options     DDB
makeoptions COPY_SYMTAB=1
```

**PROM Debugging**:
```
>> t           # Test devices
>> h           # Help
```

## Source Code Reference

**Bootloaders**:
- `/sys/arch/newsmips/stand/bootxx/` - Primary bootloader (150 lines)
- `/sys/arch/newsmips/stand/boot/` - Secondary bootloader (300 lines)
- `/sys/arch/newsmips/stand/common/` - Shared code

**Kernel**:
- `/sys/arch/newsmips/newsmips/machdep.c` - Machine-dependent (1000+ lines)
- `/sys/arch/newsmips/newsmips/autoconf.c` - Autoconfiguration
- `/sys/arch/newsmips/apbus/` - APbus support

**Include**:
- `/sys/arch/newsmips/include/apcall.h` - APCALL interface
- `/sys/arch/newsmips/include/romcall.h` - ROMCALL interface

## Platform Notes

- **Sony NEWS**: Sony's NEWS (Network Extensible Workstation System)
- **Big-Endian**: All NEWS MIPS systems are big-endian
- **NEWS-OS**: Original OS was NEWS-OS (BSD-derived)
- **Historical**: Sony ceased NEWS production in mid-1990s
- **Rare Hardware**: Limited availability outside Japan

---

*Last Updated: 2025-11-12*
*Architecture Maintainer: NetBSD/newsmips Port*
