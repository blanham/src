# NetBSD/arc Boot Documentation

$NetBSD$

## Platform Overview

**Architecture**: MIPS 32-bit (little-endian)
**Port Date**: 2000-01-23
**Boot Method**: ARC BIOS firmware → boot program → kernel
**Firmware**: ARC BIOS (Advanced RISC Computing BIOS)
**MMU Requirements**: MIPS TLB initialization by firmware

## Hardware Support

NetBSD/arc supports MIPS-based machines conforming to the Advanced RISC Computing (ARC) specification:

- **CPU**: MIPS R4000, R4400, R4600 series processors
- **Memory**: Up to 2GB (varies by platform)
- **Boot Devices**: SCSI disk, IDE disk, CD-ROM, floppy, network
- **Firmware**: ARC BIOS (Microsoft/MIPS ARC specification)
- **Console**: VGA + keyboard, or serial port

### Supported Platforms

**NEC Express5800**:
- **J96A**: Express 5800/240 (EISA, R4400)
- **JC94**: Express 5800/230 (PCI, R4400)
- **R94**: RISCstation 2200 (EISA)
- **R96**: Express RISCserver (PCI)
- **RAX94**: RISCstation 2200 (PCI)
- **RD94**: RISCstation 2250

**Acer/PICA**:
- **PICA-61**: Acer PICA (Personal RISC Computer Architecture)
- Also compatible with NEC ImageRISCstation

**DeskStation**:
- **Tyne**: DeskStation Tyne
- **rPC44**: DeskStation Arcstation I

**Microsoft/MIPS**:
- **Jazz**: MIPS Magnum R4000

**Siemens**:
- **RM200PCI**: SNI (Siemens Nixdorf) RM200

## Boot Process

### Overview

NetBSD/arc uses a **two-stage boot process**: ARC firmware loads a standalone bootloader, which then loads the kernel.

### Stage 1: ARC BIOS Firmware

**Functions**:
1. **Hardware Initialization**: CPU, memory, devices
2. **Self-Test**: POST (Power-On Self-Test)
3. **Device Enumeration**: Build system configuration tree
4. **Boot Menu**: Interactive or automatic boot

**ARC BIOS provides**:
- Device I/O services (disk, network, console)
- Memory management
- Environment variables
- Configuration database
- Timer services

**System Parameter Block (SPB)**:
- Located at physical address 0x00001000
- Contains firmware entry points
- Provides FirmwareVector table

```c
struct arcbios_spb {
    uint32_t  SPBSignature;        /* 'ARCS' or 'SCRA' */
    uint32_t  SPBLength;
    uint16_t  Version;
    uint16_t  Revision;
    int32_t   RestartBlock;
    int32_t   DebugBlock;
    int32_t   GEVector;            /* General Exception */
    int32_t   UTLBMissVector;
    uint32_t  FirmwareVectorLength;
    int32_t   FirmwareVector;      /* ARCBIOS function table */
    uint32_t  PrivateVectorLength;
    int32_t   PrivateVector;
    uint32_t  AdapterCount;
    uint32_t  AdapterType;
    uint32_t  AdapterVectorLength;
    int32_t   AdapterVector;
};
```

**Source**: `/sys/dev/arcbios/arcbios.h`

### Stage 2: Bootloader (boot)

**Location**: `/sys/arch/arc/stand/boot/`
**Size**: ~16KB (no strict size limit)

**Load Process**:
1. **Firmware** reads boot program from boot device
2. **Load Address**: Varies, typically around 0x80100000
3. **Entry Point**: `start` symbol

**Source Files**:
- `start.S` - Assembly entry point (123 lines)
- `boot.c` - Main bootloader logic (200+ lines)
- `devopen.c` - Device open routines (178 lines)
- `disk.c` - Disk I/O via ARCBIOS (179 lines)
- `bootinfo.c` - Boot information structure (83 lines)

**Functionality**:
- **ARC BIOS interface**: Uses firmware for all I/O
- **Filesystem support**: FFS (BSD Fast File System)
- **Kernel loading**: ELF executable, optional gzip compression
- **Boot parameters**: Command line arguments to kernel
- **Symbol table**: Optional kernel symbol loading for DDB

**Initialization** (`start.S`):
```assembly
start:
    # Save arguments from ARCBIOS
    sw      a0, 0(sp)              # argc
    sw      a1, 4(sp)              # argv
    sw      a2, 8(sp)              # envp

    # Get ARCBIOS function vector
    la      v0, 0x80001000         # ARCBIOS_SPB address
    lw      v0, 8 * 4(v0)          # Load FirmwareVector
    sw      v0, ARCBIOS            # Save for later use

    # Flush all caches
    lw      v0, 136(v0)            # (*ARCBIOS->FlushAllCache)()
    jalr    v0

    # Clear BSS
    la      a0, edata
    move    a1, zero
    la      a2, end
    jal     memset
    subu    a2, a2, a0

    # Call main(argc, argv, envp)
    lw      a0, 0(sp)
    lw      a1, 4(sp)
    lw      a2, 8(sp)
    jal     main
```

**Main Function** (`boot.c`):
```c
void main(int argc, char *argv[], char *envp[])
{
    /* Initialize ARCBIOS interface */
    arcbios_init(argc, argv, envp);

    /* Initialize console */
    cninit();

    /* Parse boot arguments */
    parsebootargs(argv);

    /* Try to load kernel */
    for (i = 0; kernelnames[i] != NULL; i++) {
        if (loadfile(kernelnames[i], marks, LOAD_KERNEL) == 0)
            break;
    }

    /* Prepare boot information */
    bi_init(bootinfo_buffer);
    bi_add_btinfo_magic();
    bi_add_btinfo_bootpath(bootdev);
    if (marks[MARK_SYM] != marks[MARK_END])
        bi_add_btinfo_symtab(marks);

    /* Transfer control to kernel */
    (*entry)(argc, argv, envp, bootinfo_buffer);
}
```

**Kernel Names Tried** (in order):
1. `netbsd.arc`
2. `netbsd`
3. `netbsd.gz`
4. `netbsd.bak`
5. `netbsd.old`
6. `onetbsd`
7. `onetbsd.gz`

### Stage 3: Kernel

**Entry Point**: Kernel `start` symbol → `mach_init()`

**Arguments Passed**:
- `a0`: argc
- `a1`: argv
- `a2`: envp
- `a3`: bootinfo structure pointer

**Kernel Initialization**:
```c
void mach_init(int argc, char *argv[], char *envp[], void *bootinfo)
{
    /* Save bootinfo for later use */
    memcpy(bi_buf, bootinfo, BOOTINFO_SIZE);

    /* Clear BSS */
    memset(edata, 0, end - edata);

    /* Initialize exception vectors */
    mips_vector_init(NULL, false);
    /* ARCBIOS calls no longer valid after this */

    /* Parse bootinfo */
    bootpath = lookup_bootinfo(BTINFO_BOOTPATH);
    symtab = lookup_bootinfo(BTINFO_SYMTAB);

    /* Identify platform */
    arc_identify_platform();

    /* Parse boot arguments */
    parse_boot_args(argc, argv);

    /* Scan memory via ARCBIOS descriptors */
    arc_memory_init();

    /* Initialize console */
    consinit();

    /* Initialize platform-specific hardware */
    (*platform->init)();

    /* Bootstrap VM system */
    pmap_bootstrap();
}
```

**Source**: `/sys/arch/arc/arc/machdep.c`

## ARC BIOS Interface

### Firmware Vector Table

**Location**: Pointed to by SPB at 0x80001000

**Key Functions** (subset):
```c
struct arcbios_fv {
    /* Memory */
    int32_t (*GetMemoryDescriptor)(void *);

    /* I/O */
    int32_t (*Open)(char *, uint32_t, uint32_t *);
    int32_t (*Close)(uint32_t);
    int32_t (*Read)(uint32_t, void *, uint32_t, uint32_t *);
    int32_t (*Write)(uint32_t, void *, uint32_t, uint32_t *);
    int32_t (*Seek)(uint32_t, int64_t *, uint32_t);

    /* Environment */
    char *(*GetEnvironmentVariable)(char *);
    int32_t (*SetEnvironmentVariable)(char *, char *);

    /* Time */
    void *(*GetTime)(void);

    /* Cache */
    void (*FlushAllCache)(void);

    /* Configuration */
    void *(*GetChild)(void *);
    void *(*GetPeer)(void *);
    void *(*GetConfigurationData)(void *, void *);
};
```

**Usage in Bootloader**:
```c
/* Open device */
fd = (*ARCBIOS->Open)(devname, OpenReadOnly, &fd);

/* Read data */
count = (*ARCBIOS->Read)(fd, buffer, size, &actual);

/* Close device */
(*ARCBIOS->Close)(fd);
```

### Memory Descriptors

**ARC BIOS provides memory map**:

```c
struct arcbios_mem {
    uint32_t  Type;         /* Memory type */
    uint32_t  BasePage;     /* Starting page (4KB pages) */
    uint32_t  PageCount;    /* Number of pages */
};
```

**Memory Types**:
- `ARCBIOS_MEM_ExceptionBlock` (0): Exception handler space
- `ARCBIOS_MEM_SystemParameterBlock` (1): ARCBIOS SPB
- `ARCBIOS_MEM_FreeMemory` (2): Available RAM
- `ARCBIOS_MEM_BadMemory` (3): Bad RAM (ECC errors)
- `ARCBIOS_MEM_LoadedProgram` (4): Bootloader/kernel area
- `ARCBIOS_MEM_FirmwareTemporary` (5): Firmware scratch space
- `ARCBIOS_MEM_FirmwarePermanent` (6): Firmware reserved

**Kernel uses these descriptors to build `mem_clusters[]` array**.

### Configuration Tree

ARC BIOS builds a **hierarchical device tree**:

**Component Structure**:
```c
struct arcbios_component {
    uint32_t  Class;              /* System, Processor, Cache, etc. */
    uint32_t  Type;               /* Specific device type */
    uint32_t  Flags;              /* ConsoleIn, ConsoleOut, etc. */
    uint16_t  Version;
    uint16_t  Revision;
    uint32_t  Key;                /* Class-specific data */
    uint32_t  AffinityMask;
    uint32_t  ConfigurationDataSize;
    uint32_t  IdentifierLength;
    int32_t   Identifier;         /* Device name string */
};
```

**Classes**:
- `SystemClass`: System board
- `ProcessorClass`: CPU, FPU
- `CacheClass`: L1/L2 caches
- `AdapterClass`: EISA, SCSI, etc.
- `ControllerClass`: Disk, network, serial controllers
- `PeripheralClass`: Actual devices
- `MemoryClass`: Memory modules

**Tree Navigation**:
- `GetChild()`: First child component
- `GetPeer()`: Next sibling component
- Depth-first traversal to enumerate all devices

## MIPS Memory Map

### Virtual Address Segments

Standard MIPS32 segmentation:

```
0x00000000 - 0x7FFFFFFF : useg   (2GB, user space, TLB-mapped)
0x80000000 - 0x9FFFFFFF : kseg0  (512MB, kernel, unmapped, cached)
0xA0000000 - 0xBFFFFFFF : kseg1  (512MB, kernel, unmapped, uncached)
0xC0000000 - 0xFFFFFFFF : kseg2  (1GB, kernel, TLB-mapped)
```

**Bootloader runs in kseg0** (cached, unmapped).
**Device access uses kseg1** (uncached, unmapped).
**Kernel uses all segments** after MMU initialization.

### Physical Memory Layouts

**PICA-61 / NEC ImageRISCstation**:
```
0x00000000 - 0x00000FFF : Exception vectors
0x00001000 - 0x00001FFF : ARCBIOS SPB
0x00002000 - 0x000FFFFF : ARCBIOS firmware
0x00100000 - 0x0FFFFFFF : Main memory (varies, up to 256MB)
0x80000000 - 0x803FFFFF : I/O devices (4MB)
0x90000000 - 0x9FFFFFFF : ISA I/O space
0xF0000000 - 0xFFFFFFFF : EISA I/O space
```

**NEC Express RISCserver**:
```
0x00000000 - 0x00000FFF : Exception vectors
0x00001000 - 0x00001FFF : ARCBIOS SPB
0x00100000 - 0x1FFFFFFF : Main memory (up to 512MB)
0x10000000 - 0x1FFFFFFF : PCI memory space
0x80000000 - 0x9FFFFFFF : PCI I/O space
```

**DeskStation Tyne**:
```
0x00000000 - 0x07FFFFFF : Main memory (up to 128MB)
0x02000000 - 0x03FFFFFF : ISA memory
0x80000000 - 0x9FFFFFFF : I/O space
0xE0000000 - 0xFFFFFFFF : PCI configuration
```

## MMU and TLB

### TLB Overview

ARC platforms use standard MIPS R4000-class TLB:

- **Entries**: 48 (R4000/R4400), 32 (R4600)
- **Page Sizes**: 4KB to 16MB (configurable)
- **Wired Entries**: Firmware uses some for I/O mappings

**Entry Format** (same as all MIPS):
- EntryHi: Virtual Page Number + ASID
- EntryLo0/Lo1: Physical Frame Number + flags (even/odd pair)
- PageMask: Page size selector

### Boot-Time MMU State

**At bootloader entry**:
- MMU enabled by firmware
- kseg0/kseg1 don't require TLB
- Some wired TLB entries for firmware use

**At kernel entry**:
- Bootloader has flushed caches
- TLB state preserved from firmware
- Kernel must initialize exception vectors
- Kernel clears/rebuilds TLB after `mips_vector_init()`

**Exception Vector Copying**:
```c
/* mips_vector_init() copies exception handlers */
/* From: template code in kernel */
/* To:   0x80000000 (TLB miss), 0x80000080 (general) */

/* After this, ARCBIOS calls are no longer safe */
```

## Build Instructions

### Building the Bootloader

```sh
cd /sys/arch/arc/stand
make

# Or from top-level build.sh
./build.sh -m arc tools
./build.sh -m arc distribution
```

**Output**: `boot` - standalone bootloader (ELF executable)

**Installation**: Copy to root of boot partition as `/boot`

### Building the Kernel

```sh
# Using build.sh
./build.sh -m arc kernel=GENERIC

# Or manually
cd /sys/arch/arc/conf
config GENERIC
cd ../compile/GENERIC
make depend
make
```

**Output**: `netbsd` - ELF kernel

### Kernel Configuration

**Key options for arc**:

```
# Platform selection (choose one or more)
options     PLATFORM_ACER_PICA_61
options     PLATFORM_NEC_R94
options     PLATFORM_NEC_R96
# ... (see GENERIC for full list)

# CPU architecture
makeoptions CPUFLAGS="-march=mips3 -mabi=32"

# Console options
options     CONSPEED=9600           # Serial console baud rate
#options    CONADDR=0x3f8           # Serial port address
```

## Installation

### From ARC BIOS

**Prerequisites**:
- Bootable NetBSD media (CD-ROM or pre-installed disk)
- ARC BIOS firmware with boot manager

**Boot Menu**:
1. Power on system
2. Press appropriate key for BIOS setup (varies by platform)
3. Select boot device
4. ARC BIOS loads `/boot` from boot partition
5. Bootloader loads kernel

### Installing Boot Program

**From running NetBSD**:

```sh
# Copy bootloader to root partition
cp /usr/mdec/boot /boot

# Ensure partition is bootable
# (ARC BIOS uses partition type and active flag)
fdisk -u /dev/rsd0
```

**From another OS**:

```sh
# Mount NetBSD partition
mount /dev/sd0a /mnt

# Copy bootloader
cp boot /mnt/boot

# Unmount
umount /mnt
```

### Disk Partitioning

**MBR Partition Table**:
- ARC BIOS requires MBR-style partition table
- NetBSD partition type: 0xA5 or 0x165
- Mark partition as active (bootable)

**NetBSD Disklabel**:
```sh
disklabel -e sd0

# Create partitions:
#   a: root (/)
#   b: swap
#   e: /usr
#   f: /var
#   g: /home
```

### Installing System

```sh
# Newfs partitions
newfs /dev/rsd0a
newfs /dev/rsd0e
newfs /dev/rsd0f

# Mount and extract sets
mount /dev/sd0a /mnt
mkdir /mnt/usr /mnt/var
mount /dev/sd0e /mnt/usr
mount /dev/sd0f /mnt/var

cd /mnt
tar xzpf /path/to/base.tgz
tar xzpf /path/to/etc.tgz
# ... other sets

# Copy kernel
cp /path/to/netbsd /mnt/netbsd

# Unmount
umount /mnt/usr /mnt/var /mnt
```

## Boot Configuration

### ARC BIOS Environment Variables

**View variables**:
```
>> printenv
```

**Common variables**:
- `OSLOADPARTITION`: Boot partition (e.g., `scsi(0)disk(0)rdisk(0)partition(1)`)
- `OSLOADFILENAME`: Boot file (e.g., `\boot`)
- `OSLOADOPTIONS`: Boot options
- `SYSTEMPARTITION`: System/firmware partition
- `AUTOLOAD`: `yes` or `no` (autoboot enable)
- `COUNTDOWN`: Autoboot countdown in seconds

**Set variable**:
```
>> setenv OSLOADPARTITION scsi(0)disk(0)rdisk(0)partition(1)
>> setenv OSLOADFILENAME \boot
>> setenv AUTOLOAD yes
>> setenv COUNTDOWN 5
```

### Bootloader Commands

**Boot specific kernel**:
```
boot> boot netbsd
```

**Boot with flags**:
```
boot> boot netbsd -s      # Single-user mode
boot> boot netbsd -a      # Ask for root device
boot> boot netbsd -d      # Enter debugger (DDB)
boot> boot netbsd -v      # Verbose boot
```

**List files**:
```
boot> ls
```

**Help**:
```
boot> ?
```

## Debugging

### Serial Console

**Configure in kernel**:
```
options     CONSPEED=9600
options     CONADDR=0x3f8       # COM1
#options    CONADDR=0x2f8       # COM2
```

**Or at boot time**: ARC BIOS console setup

**Serial Parameters**:
- Baud: 9600 or 19200 (default varies by platform)
- Data: 8 bits
- Parity: None
- Stop: 1 bit
- Flow control: None

### DDB Kernel Debugger

**Enable in kernel config**:
```
options     DDB
#options    DDB_HISTORY_SIZE=100
makeoptions COPY_SYMTAB=1
```

**Enter DDB**:
- At boot: `boot netbsd -d`
- Running system: `Ctrl-Alt-Esc` (console-dependent)
- On panic: Automatic

**Common DDB commands**:
```
db> trace               # Stack backtrace
db> ps                  # Process list
db> show registers      # CPU registers
db> x/x 0x80000000,10  # Examine memory
db> break function      # Set breakpoint
db> continue            # Continue execution
```

### ARCBIOS Debugging

**Bootloader Debug Mode**:

Build with `BOOT_DEBUG`:
```
cd /sys/arch/arc/stand/boot
make CPPFLAGS=-DBOOT_DEBUG
```

**Firmware Debug Commands**:
```
>> dump 0x80000000      # Memory dump
>> go 0x80100000        # Jump to address
```

### Common Boot Issues

**"Cannot find boot file"**:
- Check OSLOADPARTITION environment variable
- Verify `/boot` exists in root of partition
- Check partition is active/bootable

**"Cannot load kernel"**:
- Verify `netbsd` or `netbsd.arc` exists in root
- Check file is valid ELF executable
- Try explicit: `boot netbsd`

**Hangs after "NetBSD"**:
- Console device mismatch
- Try serial console
- Check CONSPEED matches serial terminal

**"Exception" or crash in bootloader**:
- Incompatible ARCBIOS version
- Corrupted bootloader
- Try rebuilding boot program

**No output after firmware**:
- Bootloader not found
- Check OSLOADFILENAME path
- Verify partition type and active flag

## Platform-Specific Notes

### PICA-61 / NEC ImageRISCstation

- **CPU**: R4000 or R4400
- **Console**: VGA + PS/2 keyboard default
- **Serial**: 16550 UART, configurable
- **SCSI**: NCR 53c94
- **Ethernet**: AMD LANCE (le)
- **Boot Device**: Usually SCSI disk

### NEC Express RISCserver

- **Models**: Various R96, J96A, JC94
- **Bus**: PCI or EISA
- **Console**: Usually serial
- **RAID**: Some models have hardware RAID

### DeskStation Tyne

- **CPU**: R4600
- **Bus**: ISA and local bus
- **Quirks**: Non-standard memory layout
- **Limited Support**: Less common platform

### Microsoft Jazz / MIPS Magnum

- **Historical**: One of the original ARC platforms
- **EISA bus**
- **S3 graphics**
- **NCR SCSI**

## Source Code Reference

### Bootloader Source

**Stand-alone boot**:
- `/sys/arch/arc/stand/boot/start.S` - Entry point (123 lines)
- `/sys/arch/arc/stand/boot/boot.c` - Main logic (250 lines)
- `/sys/arch/arc/stand/boot/devopen.c` - Device open (178 lines)
- `/sys/arch/arc/stand/boot/disk.c` - Disk I/O (179 lines)
- `/sys/arch/arc/stand/boot/conf.c` - Configuration (108 lines)
- `/sys/arch/arc/stand/boot/bootinfo.c` - Boot info (83 lines)

### Kernel Source

**Platform-specific**:
- `/sys/arch/arc/arc/machdep.c` - Machine-dependent code (800+ lines)
- `/sys/arch/arc/arc/autoconf.c` - Autoconfiguration (250+ lines)
- `/sys/arch/arc/arc/arcbios.c` - ARCBIOS interface (300+ lines)

**Include files**:
- `/sys/arch/arc/include/bootinfo.h` - Boot information structures
- `/sys/dev/arcbios/arcbios.h` - ARCBIOS definitions (600+ lines)
- `/sys/arch/arc/include/platform.h` - Platform abstraction

**Platform support**:
- `/sys/arch/arc/arc/p_*.c` - Platform-specific initialization files

### Man Pages

- `boot(8)` - System bootstrap procedures
- `installboot(8)` - Install bootloader
- `intro(4)` - Device driver introduction

### External Documentation

- **ARC Specification**: Microsoft RISC Specification (riscspec.zip)
  - Available from Microsoft Hardware Developer archive
  - Defines ARC BIOS interface, system parameter block, configuration tree
- **MIPS R4000 User's Manual**: CPU architecture, TLB, caches
- **Platform-specific manuals**: NEC, DeskStation, Acer documentation

## Notes

- **Historical Platform**: ARC was a 1990s initiative to standardize MIPS workstations
- **Limited Hardware**: Most ARC machines are no longer in active use
- **BIOS Dependency**: Heavy reliance on ARC BIOS for all I/O
- **Little-Endian Only**: ARC platforms were exclusively little-endian MIPS
- **Multi-Vendor**: Platform abstraction supports diverse hardware with common firmware interface
- **Obsolete Specification**: ARC was superseded by UEFI; mainly of historical interest

---

*Last Updated: 2025-11-12*
*Architecture Maintainer: NetBSD/arc Port*
