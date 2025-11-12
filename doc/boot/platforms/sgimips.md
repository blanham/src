# NetBSD/sgimips Boot Documentation

$NetBSD$

## Platform Overview

**Architecture**: MIPS 32-bit and 64-bit (big-endian)
**Port Date**: 2000-06-14
**Boot Method**: SGI ARCS/PROM → boot64 or bootiris → kernel
**Firmware**: SGI ARCS (ARC-compatible) or IP12 PROM
**MMU Requirements**: MIPS TLB, initialized by firmware

## Hardware Support

NetBSD/sgimips supports Silicon Graphics MIPS workstations and servers:

- **CPU**: MIPS R3000, R4000, R4400, R5000, R10000, R12000, R14000, R16000
- **Memory**: 32MB to 16GB (model-dependent)
- **Boot Devices**: SCSI disk, CD-ROM, network (BOOTP/TFTP)
- **Firmware**: ARCS firmware (IP20+) or PROM (IP12)
- **Console**: Graphics + keyboard or serial

### Supported Systems

**Indy/Indigo2** (IP22, IP24):
- R4000, R4400, R4600, R5000 processors
- ARCS firmware
- PCI or GIO bus expansion

**O2** (IP32):
- R5000, RM5200, RM7000, R10000, R12000
- ARCS firmware, built-in graphics

**Indigo** (IP20):
- R4000 processor
- ARCS firmware

**Indigo** (IP12):
- R3000 processor
- Old PROM firmware

**Octane** (IP30):
- R10000, R12000, R14000 processors
- ARCS firmware (64-bit)

**Origin/Onyx** (IP27):
- R10000, R12000 processors
- ARCS firmware, ccNUMA architecture

## Boot Process

### Overview

SGI systems use ARCS (Advanced RISC Computing Specification) firmware, which is similar to but incompatible with ARC BIOS used on arc platform.

### Boot Stages

**IP22/IP24/IP32 (32-bit)**:
1. ARCS firmware
2. `boot` bootloader (ECOFF format)
3. NetBSD kernel (ELF)

**IP30/IP27 (64-bit)**:
1. ARCS firmware
2. `boot64` bootloader (ECOFF64 format)
3. NetBSD kernel (ELF64)

**IP12 (Old Indigo)**:
1. IP12 PROM
2. `bootiris` bootloader
3. NetBSD kernel

### Stage 1: SGI ARCS Firmware

**Functions**:
- Hardware initialization
- System diagnostics
- Device tree enumeration
- Boot menu and shell

**ARCS Shell Commands**:
```
>> boot                    # Boot default device
>> boot -f dksc(0,1,8)netbsd  # Boot specific kernel
>> setenv OSLoadPartition dksc(0,1,8)  # Set boot device
>> printenv                # Show variables
>> hinv                    # Hardware inventory
```

**Device Naming**:
- `dksc(0,1,0)` - SCSI controller 0, ID 1, LUN 0 (disk)
- `dksc(0,1,8)` - SCSI controller 0, ID 1, partition 8 (volume header)
- `bootp()` - Network boot via BOOTP/TFTP

**Volume Header**:

SGI disks use a **volume header** (first 512-byte sector) containing:
- Partition table (up to 16 partitions)
- Boot files embedded in volume header area
- Directory of bootable files

**Partition 8**: Volume header itself (512KB at start of disk)
**Partition 10**: Usually entire disk

### Stage 2: Bootloader

**Location**: `/sys/arch/sgimips/stand/`

**Variants**:
- `boot/boot` - 32-bit bootloader (ECOFF format) for IP22/IP24/IP32
- `boot64/boot64` - 64-bit bootloader (ECOFF64) for IP30/IP27
- `bootiris/bootiris` - Bootloader for IP12 (R3000)
- `sgivol/sgivol` - Volume header manipulation tool

**Format**: ECOFF (Executable and Linkable Format, MIPS variant)
- Required by ARCS firmware
- NetBSD kernel is ELF, but bootloader is ECOFF

**Installation**:

Bootloader is installed into SGI volume header:

```sh
# View volume header
sgivol /dev/rsd0d

# Install bootloader
sgivol -w boot /usr/mdec/boot /dev/rsd0d

# Or for 64-bit
sgivol -w boot /usr/mdec/boot64 /dev/rsd0d
```

**Bootloader Features**:
- ARCS firmware callbacks for I/O
- Filesystem support (FFS, EFS)
- ELF kernel loading
- Gzip-compressed kernel support
- Boot parameter passing

**Source Files**:
- `common/start.S` - Entry point (150 lines)
- `boot/boot.c` - Main logic (300 lines)
- `sgivol/sgivol.c` - Volume header tool (600 lines)

**Initialization** (`start.S`):
```assembly
start:
    .set  noreorder

    # Save arguments from ARCS
    move  s0, a0              # argc
    move  s1, a1              # argv
    move  s2, a2              # envp

    # Get ARCS SPB
    PTR_LA v0, 0x80001000     # ARCBIOS_SPB address
    PTR_L  v0, 8*SZREG(v0)    # FirmwareVector
    PTR_S  v0, ARCBIOS

    # Flush caches via ARCS
    PTR_L  v0, 136(v0)        # FlushAllCache
    jalr   v0
    nop

    # Clear BSS
    PTR_LA a0, edata
    move   a1, zero
    PTR_LA a2, end
    jal    memset
    PTR_SUBU a2, a2, a0

    # Call main(argc, argv, envp)
    move  a0, s0
    move  a1, s1
    jal   main
    move  a2, s2
```

### Stage 3: Kernel

**Entry Point**: `mach_init()`

**Arguments from bootloader**:
- `a0`: argc
- `a1`: argv
- `a2`: envp
- `a3`: Bootinfo structure (NetBSD-specific)

**Kernel Initialization**:
```c
void mach_init(int argc, char *argv[], char *envp[], void *bootinfo)
{
    /* Identify platform via ARCS component tree */
    arcbios_init();
    ident_platform();

    /* Clear BSS */
    memset(edata, 0, end - edata);

    /* Copy exception vectors */
    mips_vector_init(NULL, false);
    /* ARCS calls no longer safe after this */

    /* Parse boot arguments */
    parse_bootargs(argc, argv);

    /* Detect memory via ARCS memory descriptors */
    mem_init();

    /* Console initialization */
    consinit();

    /* Bootstrap VM */
    pmap_bootstrap();
}
```

**Source**: `/sys/arch/sgimips/sgimips/machdep.c`

## SGI Volume Header

### Structure

**Sector 0**: Volume header (512 bytes)

```c
struct sgi_boot_block {
    uint32_t  magic;            /* 0x0BE5A941 */
    int16_t   root;             /* Root partition number */
    int16_t   swap;             /* Swap partition number */
    char      bootfile[16];     /* Default boot file */
    /* ... */
    struct {
        char     name[8];
        int32_t  block;         /* Block number in volume header */
        int32_t  bytes;         /* File size */
    } voldir[15];               /* Directory of boot files */
    struct {
        int32_t  blocks;        /* Partition size in blocks */
        int32_t  first;         /* First block */
        int32_t  type;          /* Partition type */
    } partitions[16];           /* Partition table */
};
```

**Magic Number**: `0x0BE5A941`

### Partition Types

- **0**: Volume header (partition 8)
- **1**: Track replacements
- **4**: SGI EFS filesystem
- **5**: Swap
- **6**: Raw data
- **7**: BSD 4.2 filesystem (FFS)
- **8**: IRIX system V filesystem
- **10**: Volume directory
- **83**: Linux native (often used for NetBSD)

### Volume Directory

The volume header contains a directory of up to 15 bootable files:

**Common files**:
- `sash` - IRIX standalone shell
- `sashARCS` - ARCS-based sash
- `boot` - NetBSD bootloader
- `netbsd` - Kernel (if small enough)

**Files stored in blocks 2+ of volume header area**.

## Memory Map

### Virtual Address Space (32-bit)

```
0x00000000 - 0x7FFFFFFF : useg   (2GB, user, TLB-mapped)
0x80000000 - 0x9FFFFFFF : kseg0  (512MB, kernel cached, unmapped)
0xA0000000 - 0xBFFFFFFF : kseg1  (512MB, kernel uncached, unmapped)
0xC0000000 - 0xFFFFFFFF : kseg2  (1GB, kernel, TLB-mapped)
```

### Virtual Address Space (64-bit)

```
0x0000000000000000 - 0x0000FFFFFFFFFFFF : xuseg  (User, TLB-mapped)
0x4000000000000000 - 0x400FFFFFFFFFFFFF : xsseg  (Supervisor, TLB-mapped)
0x8000000000000000 - 0x87FFFFFFFFFFFFFF : xkphys (Uncached physical)
0x9000000000000000 - 0x97FFFFFFFFFFFFFF : xkphys (Cached physical)
0xC000000000000000 - 0xFFFFFFFFFFFFFFFF : xkseg  (Kernel, TLB-mapped)
```

### Physical Memory

**IP22 (Indy)**:
```
0x00000000 - 0x1FFFFFFF : Main RAM (up to 512MB)
0x1F000000 - 0x1FFFFFFF : I/O devices (16MB)
0x1FC00000 - 0x1FFFFFFF : Boot PROM (4MB)
```

**IP32 (O2)**:
```
0x00000000 - 0x7FFFFFFF : Main RAM (up to 2GB)
0x1F000000 - 0x1FFFFFFF : I/O devices
0x1FC00000 - 0x1FFFFFFF : Boot PROM
```

**IP30 (Octane)**:
```
0x0000000000000000 - 0x000000FFFFFFFFFF : Main RAM (up to 4GB per node)
0x0000900000000000 - 0x00009FFFFFFFFFFF : PCI64 space
0x00000FFFFFFFFC00000 - 0x00000FFFFFFFFFFFFF : Boot PROM
```

## MMU and TLB

### TLB Sizes

- **R3000**: 64 entries, 4KB pages
- **R4000/R4400**: 48 entries, 4KB to 16MB pages
- **R5000**: 48 entries, 4KB to 16MB pages
- **R10000**: 64 entries, 4KB to 16MB pages
- **R12000+**: 64 entries, 4KB to 256MB pages

### Cache Hierarchies

**R10000/R12000**:
- **L1 I-cache**: 32KB, 2-way
- **L1 D-cache**: 32KB, 2-way
- **L2 cache**: 1-16MB, external

**R5000**:
- **L1 I-cache**: 32KB, 2-way
- **L1 D-cache**: 32KB, 2-way
- **L2 cache**: None or external

## Build Instructions

### Building Bootloaders

```sh
cd /sys/arch/sgimips/stand
make

# For 32-bit systems
cd boot
make

# For 64-bit systems
cd boot64
make
```

**Output**:
- `boot` - ECOFF bootloader for 32-bit systems
- `boot64` - ECOFF64 bootloader for 64-bit systems

### Building Kernel

```sh
# 32-bit kernel (IP22, IP24, IP32)
./build.sh -m sgimips kernel=GENERIC32_IP2x

# 64-bit kernel (IP30)
./build.sh -m sgimips kernel=GENERIC32_IP3x

# Or manually
cd /sys/arch/sgimips/conf
config GENERIC32_IP2x
cd ../compile/GENERIC32_IP2x
make depend
make
```

## Installation

### Disk Preparation

**Create volume header and partitions**:

```sh
# Write volume header
dd if=/dev/zero of=/dev/rsd0d bs=512 count=1
sgivol -i /dev/rsd0d

# Create partitions
# Partition 8: Volume header (0-4095 blocks, 2MB)
# Partition 0: Root filesystem (4096+ blocks)

echo -e "8\t4096\t0" | sgivol -w /dev/rsd0d
echo -e "0\t...\t7" | sgivol -w /dev/rsd0d
```

**Or use existing SGI disk** with volume header.

### Installing Bootloader

```sh
# Install boot program into volume header
sgivol -w boot /usr/mdec/boot /dev/rsd0d

# Verify
sgivol /dev/rsd0d
```

### Installing System

```sh
# Create root filesystem on partition 0
newfs /dev/rsd0a

# Mount and extract
mount /dev/sd0a /mnt
cd /mnt
tar xzpf /path/to/base.tgz
# ... other sets

# Copy kernel
cp /path/to/netbsd /mnt/netbsd

# Unmount
umount /mnt
```

### Setting Boot Variables

```
>> setenv OSLoadPartition dksc(0,1,0)
>> setenv OSLoadFilename netbsd
>> setenv OSLoader boot
>> setenv AutoLoad yes
>> setenv SystemPartition dksc(0,1,8)
```

## Boot Configuration

### ARCS Environment Variables

**View**:
```
>> printenv
```

**Key Variables**:
- `OSLoadPartition`: Root filesystem partition (e.g., `dksc(0,1,0)`)
- `OSLoadFilename`: Kernel name (e.g., `netbsd`)
- `OSLoader`: Bootloader name (e.g., `boot`)
- `SystemPartition`: Volume header partition (e.g., `dksc(0,1,8)`)
- `AutoLoad`: Autoboot enable (`Yes` or `No`)
- `OSLoadOptions`: Boot flags

**Set**:
```
>> setenv <variable> <value>
```

### Boot Commands

**From bootloader**:
```
boot> netbsd                # Boot default kernel
boot> netbsd -s             # Single-user
boot> netbsd -a             # Ask for root
boot> netbsd -d             # Enter debugger
```

## Debugging

### Serial Console

**Configure in ARCS**:
```
>> setenv console d         # Serial console
>> setenv console g         # Graphics console
```

**Serial Parameters**: 9600 or 19200 baud, 8N1

### DDB Kernel Debugger

**Enable**:
```
options     DDB
makeoptions COPY_SYMTAB=1
```

**Enter**: `Ctrl-Alt-Esc` or `L1-A` (keyboard-dependent)

### ARCS Debugging

```
>> hinv                     # Hardware inventory
>> single                   # Boot single-user
>> boot -f dksc(0,1,8)sash  # Boot standalone shell
```

## Platform-Specific Notes

### Big-Endian

SGI systems are **big-endian** (unlike most other NetBSD MIPS platforms which are little-endian).

**Implications**:
- Different byte order for multi-byte values
- Bootloader and kernel must match endianness
- Cross-compilation requires `-EB` flag

### Graphics

Many SGI systems have built-in graphics:
- **IP22**: Newport, XL, or Extreme graphics
- **IP32**: O2 "CRM" graphics
- **IP30**: VPro or Impact graphics

**Console**: Graphics console with SGI keyboard, or serial

### Historical Context

- **Silicon Graphics, Inc. (SGI)**: Leading graphics workstation vendor (1980s-2000s)
- **IRIX**: SGI's Unix variant
- **End of MIPS Production**: ~2006
- **Hardware Availability**: Still found on surplus market

## Source Code Reference

### Bootloader

- `/sys/arch/sgimips/stand/common/start.S` - Entry point (150 lines)
- `/sys/arch/sgimips/stand/boot/boot.c` - 32-bit bootloader (300 lines)
- `/sys/arch/sgimips/stand/boot64/boot.c` - 64-bit bootloader (300 lines)
- `/sys/arch/sgimips/stand/sgivol/sgivol.c` - Volume header tool (600 lines)

### Kernel

- `/sys/arch/sgimips/sgimips/machdep.c` - Machine-dependent code (1000+ lines)
- `/sys/arch/sgimips/sgimips/arcbios.c` - ARCS interface (400 lines)
- `/sys/arch/sgimips/sgimips/ip*.c` - Platform-specific support

### Include

- `/sys/dev/arcbios/arcbios.h` - ARCS definitions
- `/sys/arch/sgimips/include/sgivol.h` - Volume header structures

### Man Pages

- `boot(8)` - Boot procedures
- `sgivol(8)` - Manipulate SGI volume headers
- `installboot(8)` - Install bootloader

---

*Last Updated: 2025-11-12*
*Architecture Maintainer: NetBSD/sgimips Port*
