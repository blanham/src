# NetBSD/ews4800mips Boot Documentation

$NetBSD$

## Platform Overview

**Architecture**: MIPS 32-bit (big-endian)
**Port Date**: 2001-02-05
**Boot Method**: EWS-UX → bootxx → boot → kernel
**Firmware**: NEC EWS-UX boot ROM
**MMU Requirements**: MIPS R3000 TLB

## Hardware Support

NetBSD/ews4800mips supports NEC EWS4800 workstations:

- **CPU**: MIPS R3000, R4000, R4400 processors
- **Memory**: 32MB to 512MB
- **Boot Devices**: SCSI disk, tape (QIC)
- **Firmware**: EWS-UX (NEC's Unix variant)
- **Console**: Serial port or graphics

### Supported Models

**EWS4800/350**: R4000 processor
**EWS4800/360**: R4400 processor
**EWS4800/360AD**: R4400, advanced model
**EWS4800/360ADII**: R4400, improved model

## Boot Process

### Boot Stages

1. **EWS-UX ROM**: Firmware initialization
2. **bootxx**: Primary bootloader (BFS or USTAR format)
3. **boot**: Secondary bootloader
4. **Kernel**: NetBSD kernel

### Stage 1: EWS-UX Firmware

**Functions**:
- Hardware initialization
- System diagnostics
- Boot device selection
- Load bootxx from disk

**Boot ROM Commands**:
```
>> b            # Boot from default device
>> b sd()       # Boot from SCSI disk
>> b st()       # Boot from tape
```

### Stage 2: Primary Bootloader (bootxx)

**Location**: `/sys/arch/ews4800mips/stand/bootxx_*/`
**Size**: ~8KB

**Variants**:
- `bootxx_bfs` - Boot File System (EWS-UX native)
- `bootxx_ustarfs` - USTAR tape archive format

**Functionality**:
- Minimal filesystem code
- Locate `/boot` file
- Load secondary bootloader

**Source**: `/sys/arch/ews4800mips/stand/bootxx_bfs/`

### Stage 3: Secondary Bootloader (boot)

**Location**: `/sys/arch/ews4800mips/stand/boot/`
**Features**:
- Full filesystem support (BFS, FFS)
- ELF kernel loading
- Boot parameter passing

**Source**: `/sys/arch/ews4800mips/stand/boot/boot.c`

### Stage 4: Kernel

**Entry Point**: `mach_init()`
**Source**: `/sys/arch/ews4800mips/ews4800mips/machdep.c`

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
0x00000000 - 0x1FFFFFFF : Main RAM (up to 512MB)
0xBE000000 - 0xBFFFFFFF : I/O devices (32MB)
0x1FC00000 - 0x1FFFFFFF : Boot ROM
```

## BFS (Boot File System)

**Format**: Simple filesystem used by EWS-UX
**Structure**:
- Fixed-size directory entries
- Simple allocation scheme
- Designed for boot partitions

**Tools**: NetBSD includes BFS support for accessing EWS4800 disks.

## Build Instructions

```sh
# Build bootloaders
cd /sys/arch/ews4800mips/stand
make

# Build kernel
./build.sh -m ews4800mips kernel=GENERIC
```

## Installation

### Disk Layout

EWS4800 disks use:
- **BFS partition**: Boot filesystem with bootloader
- **FFS partition**: NetBSD root filesystem

### Installing Bootloader

```sh
# Install bootxx to BFS partition
# (Specific procedure depends on disk layout)

# Copy boot to BFS partition
cp /usr/mdec/boot /bfs_mount/boot

# Copy kernel to root
cp netbsd /netbsd
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

## Source Code Reference

**Bootloaders**:
- `/sys/arch/ews4800mips/stand/bootxx_bfs/` - BFS primary bootloader
- `/sys/arch/ews4800mips/stand/boot/` - Secondary bootloader
- `/sys/arch/ews4800mips/stand/common/` - Shared code

**Kernel**:
- `/sys/arch/ews4800mips/ews4800mips/machdep.c` - Machine-dependent
- `/sys/arch/ews4800mips/ews4800mips/autoconf.c` - Autoconfiguration

## Platform Notes

- **Big-Endian**: EWS4800 systems are big-endian MIPS
- **NEC-Specific**: Hardware is NEC proprietary
- **Rare Hardware**: Limited availability outside Japan
- **EWS-UX**: Original OS was NEC's Unix variant based on System V

---

*Last Updated: 2025-11-12*
*Architecture Maintainer: NetBSD/ews4800mips Port*
