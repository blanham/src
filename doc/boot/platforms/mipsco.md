# NetBSD/mipsco Boot Documentation

$NetBSD$

## Platform Overview

**Architecture**: MIPS 32-bit (big-endian)
**Port Date**: 2000-10-01
**Boot Method**: MIPS firmware → bootxx → boot → kernel
**Firmware**: MIPS Computer Systems firmware
**MMU Requirements**: MIPS R3000 TLB

## Hardware Support

NetBSD/mipsco supports MIPS Computer Systems workstations:

- **CPU**: MIPS R3000 processor
- **Memory**: 16MB to 64MB
- **Boot Devices**: SCSI disk, floppy, network
- **Firmware**: MIPS Computer Systems PROM
- **Console**: Serial port

### Supported Systems

**Magnum 3000** (RC3240):
- R3000A CPU (25 or 33 MHz)
- VME bus
- SCSI-1 controller
- Ethernet (Intel i82586)

**Magnum 4000** (RC3330):
- R3000A CPU (33 MHz)
- Enhanced I/O

## Boot Process

### Boot Stages

1. **MIPS PROM**: Firmware
2. **bootxx**: Primary bootloader
3. **boot**: Secondary bootloader
4. **Kernel**: NetBSD kernel

### Stage 1: MIPS PROM Firmware

**Functions**:
- Hardware initialization
- System diagnostics
- Boot menu
- Load bootxx

**PROM Commands**:
```
> b            # Boot
> b sd         # Boot from SCSI disk
> h            # Help
```

### Stage 2: Primary Bootloader (bootxx)

**Location**: `/sys/arch/mipsco/stand/bootxx_*/`
**Size**: ~8KB

**Variants**:
- `bootxx_ffs` - Fast File System
- `bootxx_cd9660` - ISO 9660 (CD-ROM)

**Source**: `/sys/arch/mipsco/stand/bootxx_ffs/`

### Stage 3: Secondary Bootloader (boot)

**Location**: `/sys/arch/mipsco/stand/boot/`

**Features**:
- Full filesystem support
- ELF kernel loading
- PROM callbacks for I/O

**Source**: `/sys/arch/mipsco/stand/boot/boot.c`

### Stage 4: Kernel

**Entry Point**: `mach_init()`
**Source**: `/sys/arch/mipsco/mipsco/machdep.c`

## Memory Map

### Virtual Address Space

```
0x00000000 - 0x7FFFFFFF : useg   (2GB, user)
0x80000000 - 0x9FFFFFFF : kseg0  (512MB, cached)
0xA0000000 - 0xBFFFFFFF : kseg1  (512MB, uncached)
0xC0000000 - 0xFFFFFFFF : kseg2  (1GB, TLB-mapped)
```

### Physical Memory

```
0x00000000 - 0x03FFFFFF : Main RAM (up to 64MB)
0x18000000 - 0x1FFFFFFF : I/O devices
0x1FC00000 - 0x1FFFFFFF : Boot ROM
```

## Build Instructions

```sh
cd /sys/arch/mipsco/stand
make

./build.sh -m mipsco kernel=GENERIC
```

## Installation

```sh
installboot /dev/rsd0a /usr/mdec/bootxx_ffs
cp /usr/mdec/boot /boot
cp netbsd /netbsd
```

## Debugging

**Serial Console**: 9600 baud, 8N1

**DDB**:
```
options     DDB
makeoptions COPY_SYMTAB=1
```

## Source Code Reference

**Bootloaders**:
- `/sys/arch/mipsco/stand/bootxx_ffs/` - Primary
- `/sys/arch/mipsco/stand/boot/` - Secondary
- `/sys/arch/mipsco/stand/common/` - Shared

**Kernel**:
- `/sys/arch/mipsco/mipsco/machdep.c` - Machine-dependent
- `/sys/arch/mipsco/mipsco/autoconf.c` - Autoconfiguration

## Platform Notes

- **Historical**: MIPS Computer Systems (1984-1992)
- **Big-Endian**: MIPS workstations are big-endian
- **Rare Hardware**: Limited availability
- **RISC/os**: Original OS was RISC/os (BSD-derived)

---

*Last Updated: 2025-11-12*
*Architecture Maintainer: NetBSD/mipsco Port*
