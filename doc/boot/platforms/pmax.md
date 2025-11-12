# NetBSD/pmax Boot Documentation

$NetBSD$

## Platform Overview

**Architecture**: MIPS 32-bit (little-endian)
**Port Date**: 1992-11-13
**Boot Method**: DECstation firmware → bootxx → boot → kernel
**Firmware**: DECstation PROM / REX (MIPS Ultrix)
**MMU Requirements**: MIPS TLB, initialized by firmware

## Hardware Support

NetBSD/pmax supports Digital Equipment Corporation DECstation and DECsystem MIPS workstations:

- **CPU**: MIPS R2000, R3000, R3000A processors (MIPS I)
- **Memory**: 8MB to 480MB (model-dependent)
- **Boot Devices**: SCSI disk, network (MOP, BOOTP/TFTP)
- **Firmware**: DECstation PROM or REX firmware
- **Console**: Serial (DZ11) or graphics (PMAG)

### Supported Systems

**Personal DECstation**:
- **5000/xx (3MIN, 3MAX, 3MAXPLUS)**: R3000A, TURBOchannel
- **5000/1xx (Personal DECstation)**: R3000, built-in SCSI/Ethernet
- **5000/200 (3MAX)**: R3000, TURBOchannel expansion
- **5000/240, 260 (3MAX+)**: R3000A, faster TURBOchannel

**DECsystem**:
- **5100**: Rackmount R3000 server
- **5400, 5500, 5800, 5900**: High-end R3000 servers

**DECstation 2100, 3100**:
- Entry-level R2000/R3000 workstations

## Boot Process

### Three-Stage Boot

**Stage 1**: DECstation PROM firmware
**Stage 2**: Primary bootloader `bootxx` (filesystem-specific, max 7680 bytes)
**Stage 3**: Secondary bootloader `boot` (full-featured, ~128KB)
**Stage 4**: Kernel

### Stage 1: DECstation PROM

**Functions**:
- Hardware initialization and diagnostics
- Console setup (serial or graphics)
- Boot device selection
- Load primary bootloader from disk

**PROM Commands**:
```
>> boot                    # Boot from default device
>> boot 3/rz0/netbsd      # Boot specific device/file
>> boot -a                 # Ask for root device
>> boot -s                 # Single-user mode
>> setenv bootpath ...     # Set default boot device
>> printenv                # Show environment variables
```

**Boot Device Syntax**:
- `3/rz0a/netbsd` - SCSI controller 3, disk 0, partition a
- `mop()` - MOP network boot
- `bootp()` - BOOTP/TFTP network boot

### Stage 2: Primary Bootloader (bootxx)

**Location**: `/sys/arch/pmax/stand/bootxx_*/`
**Size**: Maximum 7680 bytes (15 sectors)
**Load Address**: 0xA0700000 (kseg1, uncached)

**Filesystem Variants**:
- `bootxx_ffs` - BSD Fast File System (FFS v1)
- `bootxx_ffsv2` - FFS v2 (64-bit)
- `bootxx_lfs` - Log-structured File System
- `bootxx_cd9660` - ISO 9660 (CD-ROM)

**Functionality**:
- Minimal filesystem code
- Locate `/boot` in root directory
- Load secondary bootloader to memory
- Transfer control to `/boot`

**Source**: `/sys/arch/pmax/stand/bootxx_ffs/bootxx.c` (~200 lines)

### Stage 3: Secondary Bootloader (boot)

**Location**: `/sys/arch/pmax/stand/boot/`
**Size**: ~128KB (no strict limit)
**Load Address**: 0x80700000 (kseg0, cached)

**Features**:
- Full filesystem support (FFS, LFS, CD9660)
- ELF and a.out kernel loading
- Gzip-compressed kernel support
- Interactive command line
- PROM callbacks for I/O
- Network boot support (MOP, TFTP, NFS)

**Commands**:
```
boot> boot netbsd         # Boot kernel
boot> boot -s             # Single-user
boot> boot netbsd.old     # Boot alternate kernel
boot> ls /                # List files
boot> help                # Show help
```

**Source Files**:
- `start.S` - Entry point (100 lines)
- `boot.c` - Main logic (200+ lines)
- `devopen.c` - Device open (150 lines)
- `conf.c` - Device configuration (100 lines)

**Initialization** (`start.S`):
```assembly
start:
    .set  noreorder
    la    gp, _gp
    la    sp, start - CALLFRAME_SIZ

    # Save arguments from PROM
    move  s0, a0              # argc
    move  s1, a1              # argv
    move  s2, a2              # REX magic or envp
    sw    a2, _C_LABEL(callv)

    # Clear BSS
    la    a0, edata
    move  a1, zero
    la    a2, end
    jal   memset
    subu  a2, a2, a0

    # Call main
    move  a0, s0
    move  a1, s1
    jal   main
    move  a2, s2
```

### Stage 4: Kernel

**Entry Point**: `mach_init()`

**Source**: `/sys/arch/pmax/pmax/machdep.c`

## Memory Map

### Virtual Address Space

```
0x00000000 - 0x7FFFFFFF : useg   (2GB, user, TLB-mapped)
0x80000000 - 0x9FFFFFFF : kseg0  (512MB, kernel cached, unmapped)
0xA0000000 - 0xBFFFFFFF : kseg1  (512MB, kernel uncached, unmapped)
0xC0000000 - 0xFFFFFFFF : kseg2  (1GB, kernel, TLB-mapped)
```

### Physical Memory

**DECstation 5000 series**:
```
0x00000000 - 0x1FFFFFFF : Main RAM (up to 480MB)
0x10000000 - 0x17FFFFFF : TURBOchannel slot 0
0x18000000 - 0x1BFFFFFF : TURBOchannel slot 1
0x1C000000 - 0x1FFFFFFF : TURBOchannel slot 2
0x1FC00000 - 0x1FFFFFFF : Boot ROM (4MB)
```

**DECstation 3100**:
```
0x00000000 - 0x01FFFFFF : Main RAM (up to 32MB)
0x10000000 - 0x17FFFFFF : I/O devices
0x1C000000 - 0x1C07FFFF : SCSI (NCR 53C94)
0x1C800000 - 0x1C8003FF : Lance Ethernet
0x1D000000 - 0x1D0FFFFF : Serial (DZ11)
0x1FC00000 - 0x1FFFFFFF : Boot ROM
```

## MMU and TLB

### MIPS R2000/R3000 TLB

- **Entries**: 64 (R2000/R3000)
- **Page Sizes**: 4KB (fixed)
- **Wired Entries**: 8 (reserved by PROM)

**TLB Entry Format**:
```
EntryHi:  [VPN (20 bits)] [PID (6 bits)]
EntryLo:  [PFN (20 bits)] [N (1)] [D (1)] [V (1)] [G (1)]
```

**Bits**:
- **VPN**: Virtual Page Number
- **PID**: Process ID (ASID)
- **PFN**: Physical Frame Number
- **N**: Non-cacheable
- **D**: Dirty (writable)
- **V**: Valid
- **G**: Global (ignore PID)

### Cache

**R3000 Cache**:
- **I-cache**: 4KB or 64KB (direct-mapped)
- **D-cache**: 4KB or 64KB (direct-mapped)
- **Line Size**: 4 bytes (R3000), 16 bytes (R3000A)

**Cache Operations**:
- Flush via indexed invalidate
- No cache coherence protocol
- Software must manage consistency

## Build Instructions

### Building Bootloaders

```sh
cd /sys/arch/pmax/stand
make

# Or using build.sh
./build.sh -m pmax tools
./build.sh -m pmax distribution
```

**Output**:
- `bootxx_ffs` - Primary bootloader for FFS
- `boot` - Secondary bootloader

### Building Kernel

```sh
./build.sh -m pmax kernel=GENERIC

# Or manually
cd /sys/arch/pmax/conf
config GENERIC
cd ../compile/GENERIC
make depend
make
```

## Installation

### Using installboot(8)

```sh
# Install bootxx to root partition
installboot /dev/rsd0a /usr/mdec/bootxx_ffs

# Copy boot to root filesystem
cp /usr/mdec/boot /boot

# Copy kernel
cp /usr/mdec/netbsd /netbsd
```

### Disk Partitioning

```sh
disklabel -e sd0

# Example:
#  a: /        1GB
#  b: swap     256MB
#  c: whole disk
#  d: /usr     remaining
```

### Network Boot

**MOP (Maintenance Operations Protocol)**:

```sh
# On MOP server
mopd -d -i le0 /tftpboot/boot.pmax

# On DECstation
>> boot 3/mop
```

**TFTP/BOOTP**:

```sh
# On DECstation
>> boot 3/tftp/netbsd
```

## Debugging

### Serial Console

**DZ11 Serial Port**:
- Baud: 9600 or 19200
- Data: 8 bits
- Parity: None
- Stop: 1 bit

**Connection**: DB25 serial port on back panel

### DDB Kernel Debugger

**Enable**:
```
options     DDB
makeoptions COPY_SYMTAB=1
```

**Enter DDB**: `Ctrl-Alt-Esc` or panic

### PROM Debugging

```
>> cnfg         # Show configuration
>> t rz(0,0,0)  # Test SCSI disk
>> boot -d      # Boot with debugger
```

## Source Code Reference

**Bootloaders**:
- `/sys/arch/pmax/stand/bootxx_ffs/` - Primary bootloader
- `/sys/arch/pmax/stand/boot/` - Secondary bootloader
- `/sys/arch/pmax/stand/common/` - Shared code

**Kernel**:
- `/sys/arch/pmax/pmax/machdep.c` - Machine-dependent code
- `/sys/arch/pmax/pmax/autoconf.c` - Autoconfiguration
- `/sys/arch/pmax/pmax/dec_3100.c` - DECstation 3100 support
- `/sys/arch/pmax/pmax/dec_5100.c` - DECsystem 5100 support

**Include**:
- `/sys/arch/pmax/include/dec_prom.h` - PROM definitions

---

*Last Updated: 2025-11-12*
*Architecture Maintainer: NetBSD/pmax Port*
