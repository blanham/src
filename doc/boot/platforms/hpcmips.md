# NetBSD/hpcmips Boot Documentation

$NetBSD$

## Platform Overview

**Architecture**: MIPS 32-bit (little-endian)
**Port Date**: 1999-09-16
**Boot Method**: Windows CE → pbsdboot.exe → kernel
**Firmware**: Windows CE (Microsoft)
**MMU Requirements**: MIPS TLB managed by Windows CE

## Hardware Support

NetBSD/hpcmips supports MIPS-based handheld PCs running Windows CE:

- **CPU**: MIPS VR4100, VR4300 series processors (NEC, Toshiba)
- **Memory**: 8MB to 64MB
- **Boot Method**: Windows CE application
- **Platform**: Handheld PCs (HPCs), Palm-size PCs
- **Console**: LCD screen, touch screen

### Supported Devices

**NEC Mobile Pro**:
- Mobile Pro 700, 750C, 770, 780, 800, 880

**Casio Cassiopeia**:
- E-10, E-11, E-15, E-55, E-65, E-100, E-105, E-125

**Compaq/HP**:
- C series

**Philips Velo**:
- Velo 1, 500

**IBM WorkPad**:
- z50

**Other Manufacturers**:
- Various MIPS-based Windows CE devices

## Boot Process

### Unique Boot Method

NetBSD/hpcmips uses a **completely different boot method** from other NetBSD MIPS platforms:

**No traditional bootloader** - Instead, a **Windows CE application** (pbsdboot.exe) loads and boots NetBSD.

### Boot Stages

1. **Windows CE**: Microsoft Windows CE operating system
2. **pbsdboot.exe**: NetBSD boot application (Windows CE .exe)
3. **Kernel**: NetBSD/hpcmips kernel

### Stage 1: Windows CE

**Functions**:
- Full operating system
- GUI environment
- File system access (FAT)
- Memory management

**User Interface**:
- Start menu
- File explorer
- Applications

### Stage 2: pbsdboot.exe

**Location**: `/sys/arch/hpcmips/stand/pbsdboot/`
**Format**: Windows CE executable (.exe)
**Platform**: Win32 API for Windows CE

**Features**:
- GUI application with touch screen interface
- Kernel selection via file browser
- Boot option configuration
- Framebuffer and kernel memory setup
- Direct hardware takeover from Windows CE

**User Interface Elements**:
- **Kernel Path**: Browse for NetBSD kernel file
- **Boot Options**: Single-user, verbose, etc.
- **Root Device**: Specify root device
- **Boot Button**: Start NetBSD boot process

**Source Files** (uses Microsoft Visual C++ for Windows CE):
- `pbsdboot.cpp` - Main application (Windows CE GUI)
- `vr.c` - VR processor-specific code
- `mips.c` - MIPS architecture code
- `platid.c` - Platform identification

**Build System**:
- **Visual C++ for Windows CE**: Required to build
- **Project Files**: `.dsp`, `.dsw` files for Visual Studio
- **Win32 API**: Windows CE SDK

**Boot Process**:
```c
void boot_kernel(void)
{
    /* Disable Windows CE interrupts */
    suspend_system();

    /* Allocate memory for kernel */
    alloc_kernel_memory();

    /* Load kernel from file */
    load_kernel_file(kernel_path);

    /* Prepare boot info structure */
    setup_bootinfo();

    /* Flush caches */
    flush_cache();

    /* Transfer control to kernel entry point */
    jump_to_kernel(entry_address);
}
```

### Stage 3: Kernel

**Entry Point**: `mach_init()`
**Source**: `/sys/arch/hpcmips/hpcmips/machdep.c`

**Kernel receives**:
- Boot information structure (memory map, platform ID)
- Framebuffer information (for console)
- Platform identification (platid)

**Initialization**:
```c
void mach_init(int argc, char *argv[], struct bootinfo *bi)
{
    /* Save boot information */
    memcpy(&bootinfo, bi, sizeof(bootinfo));

    /* Identify platform */
    platid_t platid = bootinfo.platid;

    /* Clear BSS */
    memset(edata, 0, end - edata);

    /* Initialize exception vectors */
    mips_vector_init(NULL, false);

    /* Parse boot arguments */
    parse_bootargs(argc, argv);

    /* Initialize console (framebuffer or serial) */
    consinit();

    /* Bootstrap VM */
    pmap_bootstrap();
}
```

## Platform Identification (platid)

### platid System

**Purpose**: Identify specific handheld device model

**Structure**:
```c
typedef struct platid {
    uint32_t dw[2];     /* Platform ID (64-bit) */
} platid_t;
```

**Format**:
- **CPU**: Vendor and model
- **Machine**: Manufacturer and model
- **Vendor-specific bits**

**Examples**:
- NEC Mobile Pro 780: `platid = {CPU_MIPS_VR_4121, MACH_NEC_MOBILEPRO_780}`
- Casio E-105: `platid = {CPU_MIPS_VR_4121, MACH_CASIO_CASSIOPEIAE_E105}`

**Usage**:
- Device-specific initialization
- Hardware quirks
- LCD panel configuration

## Memory Map

### Virtual Address Space

```
0x00000000 - 0x7FFFFFFF : useg   (2GB, user, TLB-mapped)
0x80000000 - 0x9FFFFFFF : kseg0  (512MB, kernel cached)
0xA0000000 - 0xBFFFFFFF : kseg1  (512MB, kernel uncached)
0xC0000000 - 0xFFFFFFFF : kseg2  (1GB, kernel TLB-mapped)
```

### Physical Memory (varies by device)

**Example (NEC Mobile Pro 780)**:
```
0x00000000 - 0x03FFFFFF : Main RAM (64MB)
0x0A000000 - 0x0AFFFFFF : I/O devices (16MB)
0x0B000000 - 0x0BFFFFFF : Framebuffer (16MB)
```

**Memory Layout**: Device-specific, determined by Windows CE

## Build Instructions

### Building pbsdboot.exe

**Requirements**:
- Microsoft Visual C++ for Windows CE
- Windows CE SDK
- Windows development environment

**Build Steps**:
1. Open project in Visual C++: `hpcmips_stand.dsw`
2. Select pbsdboot project
3. Build → Build pbsdboot.exe
4. Copy to handheld device

**Output**: `pbsdboot.exe` - Windows CE executable

**Note**: Cannot be built on NetBSD; requires Windows CE development tools.

### Building Kernel

```sh
# On NetBSD or cross-compile host
./build.sh -m hpcmips kernel=GENERIC

# Or manually
cd /sys/arch/hpcmips/conf
config GENERIC
cd ../compile/GENERIC
make depend
make
```

**Output**: `netbsd` - ELF kernel
**Compression**: Can compress: `gzip netbsd` → `netbsd.gz`

## Installation

### Prerequisites

1. **Windows CE device** with MIPS processor
2. **Storage card** (CF, SD) or internal storage
3. **ActiveSync** or file transfer method

### Installation Steps

**1. Transfer Files to Device**:

Using ActiveSync or file explorer:
```
\Storage Card\netbsd           # NetBSD kernel
\Storage Card\pbsdboot.exe     # Boot application
```

**2. Run pbsdboot.exe**:

- Tap on pbsdboot.exe in File Explorer
- Application launches with GUI

**3. Configure Boot**:

- **Kernel**: Browse to `\Storage Card\netbsd`
- **Options**: Check desired boot flags
  - `-s`: Single-user mode
  - `-v`: Verbose boot
  - `-a`: Ask for root device
- **Root**: Specify root device (e.g., `wd0a`)

**4. Boot NetBSD**:

- Tap "Boot" button
- Windows CE suspends
- NetBSD boots

### Root Filesystem

**Options**:
1. **Storage Card**: Format with FFS, mount as root
2. **Ramdisk**: Use kernel with built-in ramdisk (MD_ROOT)
3. **NFS**: Network root (requires network support)

**Installing to Storage Card**:
```sh
# On NetBSD system, prepare CF/SD card
newfs /dev/sd0a
mount /dev/sd0a /mnt
cd /mnt
tar xzpf /path/to/base.tgz
# ... other sets
umount /mnt

# Transfer card to handheld
```

## Framebuffer Console

### LCD Display

**Most hpcmips devices** boot to framebuffer console:

- **Resolution**: Device-specific (320x240 to 800x600)
- **Depth**: 8-bit, 16-bit, or 24-bit color
- **Orientation**: Portrait or landscape

**Configuration**: Automatically detected from bootinfo

### Virtual Console

**wscons**: NetBSD's virtual console system

**Features**:
- Multiple virtual terminals
- Unicode support
- Color support

## Debugging

### Serial Console

**Some devices have serial port**:
- Requires special cable
- Usually not available on consumer devices

**Configure**:
```
options     CONSPEED=115200
```

### DDB Kernel Debugger

```
options     DDB
makeoptions COPY_SYMTAB=1
```

**Enter**: Difficult without keyboard; use boot flag `-d`

### Windows CE Debugging

**Before Boot**:
- Use Windows CE tools to debug pbsdboot.exe
- Visual Studio debugger

**Logging**: pbsdboot.exe can write debug log to file

## Common Issues

### pbsdboot.exe Doesn't Run

- **CPU Architecture**: Ensure pbsdboot.exe built for MIPS
- **Windows CE Version**: Requires CE 2.0 or later
- **Memory**: Need sufficient free RAM

### Boot Hangs

- **Framebuffer**: Try `-h` flag for serial console
- **Platform ID**: Incorrect detection
- **Memory**: Insufficient available memory

### Kernel Panic

- **Root Device**: Specify correct root device (`-a` flag)
- **Storage Card**: Check filesystem integrity
- **Memory Overlap**: Windows CE memory conflict

### Touch Screen Issues

- **Calibration**: May need calibration
- **Driver**: Platform-specific driver support varies

## Platform-Specific Notes

### VR4100 vs. VR4300

**VR4100 series** (NEC):
- VR4102, VR4111, VR4121, VR4122, VR4131
- Lower power consumption
- Integrated peripherals

**VR4300 series** (NEC/Toshiba):
- Higher performance
- Compatible with R4000

### Windows CE Versions

- **CE 2.0**: HPC 1.0, Palm-size PC 1.0
- **CE 2.11**: HPC 2.0, Palm-size PC 1.1
- **CE 3.0**: Pocket PC 2000, HPC 2000

**Compatibility**: pbsdboot.exe works with CE 2.0+

### Historical Context

- **Windows CE**: Microsoft's embedded OS (1996+)
- **Handheld PCs**: Popular in late 1990s
- **NetBSD Port**: Pioneered by Japanese developers
- **Modern Status**: Obsolete hardware, historical interest

## Source Code Reference

**pbsdboot (Windows CE application)**:
- `/sys/arch/hpcmips/stand/pbsdboot/` - Main application (2000+ lines)
- `/sys/arch/hpcmips/stand/pbsdboot/pbsdboot.cpp` - GUI code
- `/sys/arch/hpcmips/stand/pbsdboot/vr.c` - VR processor code
- `/sys/arch/hpcmips/stand/pbsdboot/mips.c` - MIPS code

**Kernel**:
- `/sys/arch/hpcmips/hpcmips/machdep.c` - Machine-dependent (1500+ lines)
- `/sys/arch/hpcmips/hpcmips/autoconf.c` - Autoconfiguration
- `/sys/arch/hpcmips/dev/` - Device drivers (platid-specific)

**Include**:
- `/sys/arch/hpcmips/include/bootinfo.h` - Boot information
- `/sys/arch/hpcmips/include/platid.h` - Platform identification

---

*Last Updated: 2025-11-12*
*Architecture Maintainer: NetBSD/hpcmips Port*
