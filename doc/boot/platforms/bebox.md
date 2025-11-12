# NetBSD/bebox Boot Documentation

## Platform Overview

NetBSD/bebox is the port of NetBSD to Be Inc.'s BeBox dual-processor PowerPC systems. The BeBox was a unique computer designed for the BeOS operating system, featuring:

- Dual PowerPC 603/603e processors (66-133 MHz)
- Up to 256 MB RAM
- GeekPort I/O panel with LEDs
- BeOS boot ROM (not OpenFirmware)
- ISA and PCI expansion buses
- Integrated graphics and audio

The BeBox has a proprietary boot ROM compatible with the BeOS DR8 (Developer Release 8) boot format.

## Boot Method

**Primary Boot Method:** BeOS Boot ROM

Unlike most PowerPC platforms, the BeBox does not use OpenFirmware. Instead, it has:
- Custom boot ROM firmware
- BeOS DR8-compatible boot loader format
- Support for OBFS (Old BeOS File System) boot images
- Direct kernel loading from floppy or IDE/SCSI

### Boot Sequence

1. **Hardware Reset** → BeOS ROM initialization
2. **Boot ROM** → Searches for bootable media
3. **Boot Loader** → Loaded from floppy/disk
4. **NetBSD Kernel** → Direct kernel load via boot loader

## Boot Loader Implementation

### Primary Bootloader: boot

**Location:** `/sys/arch/bebox/stand/boot/`

The BeBox bootloader is a standalone program that:
- Runs under control of the BeOS boot ROM
- Supports FFS, IDE, SCSI, and floppy devices
- Provides console selection (VGA, framebuffer, serial)
- Loads ELF kernel images

**Key Source Files:**
- `boot/main.c` - Main boot logic
- `common/` - Shared boot code
- `boot_com0/` - Serial console variant
- `boot_vga/` - VGA console variant

### Console Selection

The bootloader supports three console types (selected at compile time):

```c
/* In Makefile: */
CPPFLAGS+= -DCONS_VGA        /* S3 Trio64, etc. */
CPPFLAGS+= -DCONS_BE         /* Trio64v+, Millennium, Mystique */
CPPFLAGS+= -DCONS_SERIAL     /* Serial port */
```

**Note:** After changing CPPFLAGS, run `make cleandir` before rebuilding.

### Installation Format

The bootloader must be packaged in BeOS DR8 filesystem format using `mkbootimage`:

```bash
# Standalone bootloader
nbpowerpc-mkbootimage -I -m bebox -b boot/boot /tmp/fd.img

# With embedded kernel
nbpowerpc-mkbootimage -m bebox -b boot/boot \
    -k ../compile/INSTALL/netbsd /tmp/fd.img
```

## BAT Register Setup

**Source:** `/sys/arch/bebox/bebox/machdep.c:initppc()`

The BeBox uses minimal BAT setup through the `prep_initppc()` common code:

```c
void initppc(u_int startkernel, u_int endkernel, u_int args, void *btinfo)
{
    /* Copy bootinfo from loader */
    memcpy(bootinfo, btinfo, sizeof(bootinfo));

    /* Set up memory regions */
    physmemr[0].start = 0;
    physmemr[0].size = meminfo->memsize & ~PGOFSET;
    availmemr[0].start = (endkernel + PGOFSET) & ~PGOFSET;
    availmemr[0].size = meminfo->memsize - availmemr[0].start;

    /* Initialize with BeBox mainboard register mapping */
    prep_initppc(startkernel, endkernel, args,
        0x7ffff000, BAT_BL_8M,    /* BeBox mainboard registers */
        0);
}
```

### BeBox BAT Configuration

```
DBAT0: 0x7ffff000 - 8MB  (BeBox mainboard registers and I/O)
```

Additional BATs are configured by the prep common code for PCI I/O space.

## Boot Process Stages

### Stage 1: BeOS ROM Execution

The BeBox boot ROM performs:
1. CPU detection and initialization
2. Memory detection
3. PCI bus enumeration
4. Boot device search (floppy → IDE → SCSI)
5. Load boot sector/boot image

### Stage 2: CPU Detection and Initialization

**Source:** `/sys/arch/bebox/bebox/locore.S:__start`

```assembly
__start:
    /* Disable FPU/MMU/exceptions */
    li      0, 0
    mtmsr   0
    isync

    /* CPU detection - BeBox specific */
    lis     8, 0x7FFF
    ori     8, 8, 0xF3F0
    lwz     9, 0(8)              # Read processor ID
    andis.  9, 9, 0x0200         # Check bit 6
    cmpwi   0, 9, 0              # 0=CPU0, non-zero=CPU1
    bne     __start_cpu1
    b       __start_cpu0
```

### Stage 3: Multi-Processor Handling

**CPU 1 (Secondary Processor):**
```assembly
__start_cpu1:
    /* Disable caches for spinup */
    li      8, 0
    mtspr   SPR_HID0, 8
    sync
    isync

#ifdef MULTIPROCESSOR
    li      3, 0x1              # CPU ID 1
    ba      cpu_spinstart       # Jump to spin start
#else
1:  b       1b                  # Infinite loop if !MP
#endif
```

**CPU 0 (Boot Processor):**
```assembly
__start_cpu0:
    /* Enable data and instruction caches */
    mfspr   8, SPR_HID0
    andi.   8, 8, (HID0_ICE|HID0_DCE)@l
    andi.   0, 8, HID0_DCE
    ori     7, 8, HID0_ICFI
    bne     1f
    ori     7, 7, HID0_DCFI
1:
    sync
    mtspr   SPR_HID0, 7         # Flash invalidate
    sync
    mtspr   SPR_HID0, 8         # Enable caches
    sync
    isync
```

### Stage 4: Symbol Table and Memory Setup

```assembly
    /* Compute end of kernel memory */
#if defined(DDB) || NKSYMS || defined(MODULAR)
    lis     7, _C_LABEL(startsym)@ha
    addi    7, 7, _C_LABEL(startsym)@l
    stw     3, 0(7)             # Save symbol start
    lis     7, _C_LABEL(endsym)@ha
    addi    7, 7, _C_LABEL(endsym)@l
    stw     4, 0(7)             # Save symbol end
#else
    lis     4, _C_LABEL(end)@ha
    addi    4, 4, _C_LABEL(end)@l
#endif

    /* Initialize CPU info structure */
    INIT_CPUINFO(4, 1, 9, 0)

    /* Call C initialization */
    lis     3, __start@ha
    addi    3, 3, __start@l
    bl      _C_LABEL(initppc)
    bl      _C_LABEL(main)
```

### Stage 5: C Initialization

The `initppc()` function:
1. Copies bootinfo from loader
2. Sets up memory regions from bootinfo
3. Gets CPU clock from bootinfo
4. Calls `prep_initppc()` for common PReP initialization

## MMU Requirements

### Initial State

- **MMU:** Disabled during early locore.S execution
- **Caches:** Enabled after CPU detection
- **Real Mode:** Boot runs in real mode

### Cache Configuration

The BeBox uses PowerPC 603/603e processors with:
- 16KB instruction cache (2-way set associative)
- 16KB data cache (2-way set associative)
- Cache line size: 32 bytes
- Hardware cache coherency in multiprocessor mode

### Memory Translation

After BAT setup:
- BATs map I/O regions
- Page tables handle main memory
- Segment registers configured for kernel/user split

## Memory Map

### Physical Memory Layout

```
0x00000000 - 0x00003fff    Exception vectors
0x00004000 - RAM_END       Main memory (up to 256MB)
0x7ffff000 - 0x7fffffff    BeBox mainboard registers
0x80000000 - 0x807fffff    ISA I/O space (8MB)
0x80800000 - 0xffffffff    PCI memory/I/O space
```

### BeBox Mainboard Registers

```
0x7ffff000                 CPU ID register (bit 6: CPU 0/1)
0x7ffff0c0                 GeekPort LED control
0x7ffff0e0 - 0x7ffff0ff   Other mainboard controls
```

### Virtual Memory Layout

Standard PowerPC OEA layout:
```
0x00000000 - 0x0fffffff    User space (segment 0)
0x10000000 - 0xefffffff    User space (segments 1-14)
0xf0000000 - 0xffffffff    Kernel space (segment 15)
```

## Build and Installation

### Building the Bootloader

```bash
cd /sys/arch/bebox/stand
make cleandir
make depend
make
```

This builds:
- `boot/boot` - Main bootloader (default console)
- `boot_com0/boot` - Serial console variant
- `boot_vga/boot` - VGA console variant

### Creating Boot Floppy

```bash
# Method 1: Standalone bootloader
cd /sys/arch/bebox/stand
make
nbpowerpc-mkbootimage -I -m bebox -b boot/boot /tmp/fd.img

# Method 2: With embedded kernel
nbpowerpc-mkbootimage -m bebox -b boot/boot \
    -k ../compile/YOUR_KERNEL/netbsd /tmp/boot.img

# Write to floppy
dd if=/tmp/fd.img of=/dev/rfd0a
```

### Installing to Hard Disk

```bash
# Partition disk for BeBox boot
# Create small FAT partition for boot files
newfs_msdos /dev/rsd0a
mount -t msdos /dev/sd0a /mnt

# Install bootloader in BeOS format
mkbootimage -m bebox -b boot/boot -k netbsd /mnt/bootimg

# Create NetBSD root partition
newfs /dev/rsd0b
```

### Multi-Console Support

Build separate boot images for different consoles:

```bash
# VGA console
cd boot_vga
make
mkbootimage -m bebox -b boot bootimg.vga

# Serial console
cd boot_com0
make
mkbootimage -m bebox -b boot bootimg.serial

# Framebuffer console
cd boot
make CPPFLAGS=-DCONS_BE
mkbootimage -m bebox -b boot bootimg.fb
```

## Debugging

### GeekPort LEDs

The BeBox has a distinctive "GeekPort" with diagnostic LEDs:

```c
/* Access GeekPort LEDs */
#define BEBOX_SET_LED(val) \
    *(volatile uint8_t *)0x800c00 = (val)

/* In kernel code: */
void debug_led(int value)
{
    lis     4, 0x8000
    ori     4, 4, 0x0c00
    stb     3, 0(4)             # Write to LED register
    blr
}
```

Use for boot debugging:
```c
BEBOX_SET_LED(0x01);  /* Entry to function */
BEBOX_SET_LED(0x02);  /* After initialization */
BEBOX_SET_LED(0xFF);  /* Error condition */
```

### Serial Console Debugging

Configure serial boot variant:
```bash
cd boot_com0
make
```

Serial parameters:
- Port: COM1 (0x3F8)
- Speed: 9600 baud (default)
- Format: 8N1

### Multiprocessor Debugging

Enable MP debug output:
```c
options MULTIPROCESSOR
options MP_DEBUG
```

Check CPU startup:
- CPU 0 proceeds to boot
- CPU 1 spins in `cpu_spinstart()`
- IPI mechanism activates secondary CPU

### Common Boot Issues

**Problem:** Bootloader not found
- **Cause:** Image not in BeOS DR8 format
- **Solution:** Use `mkbootimage` with `-m bebox`

**Problem:** Second CPU not starting
- **Cause:** Multiprocessor support not enabled
- **Solution:** Build with `options MULTIPROCESSOR`

**Problem:** Hanging at initppc
- **Cause:** Bad bootinfo structure
- **Solution:** Verify bootloader version matches kernel

### Boot ROM Diagnostics

The BeBox boot ROM provides minimal diagnostics:
- Screen flashes indicate boot progress
- Beep codes for hardware failures
- LED patterns on front panel

### Low-Level Debugging

Insert debug markers in locore.S:
```assembly
/* Debug: Flash LED pattern */
lis     %r3, 0x8000
ori     %r3, %r3, 0x0c00
li      %r4, 0xAA           # Test pattern
stb     %r4, 0(%r3)
```

## Platform-Specific Notes

### Dual CPU Support

The BeBox is one of the few dual-CPU PowerPC systems:
- Both CPUs are PowerPC 603/603e
- Symmetric multiprocessing (SMP) capable
- Requires MULTIPROCESSOR kernel option
- CPU detection via memory-mapped register

### Interrupt Controller

BeBox uses a custom interrupt controller:
- BeBox Interrupt Controller (BIC)
- 32 interrupt sources
- Cascaded with i8259 PIC for ISA
- IRQ 26 cascades to i8259

### PCI Bus

- Single PCI bus
- Motorola MPC105 host bridge
- Shared with ISA devices
- Limited PCI slots

### Graphics

Supported graphics cards:
- S3 Trio64 (VGA mode)
- S3 Trio64v+ (framebuffer)
- Matrox Millennium I/II
- Matrox Mystique 220

## References

### Source Files

- `/sys/arch/bebox/bebox/locore.S` - Assembly boot code
- `/sys/arch/bebox/bebox/machdep.c` - Platform initialization
- `/sys/arch/bebox/stand/boot/` - Bootloader
- `/sys/arch/powerpc/stand/mkbootimage/` - Boot image tool

### Hardware Documentation

- Be Inc. BeBox Developer's Guide
- BeOS DR8 Boot Format Specification
- PowerPC 603e RISC Microprocessor User's Manual
- Motorola MPC105 PCI Bridge/Memory Controller

### Historical Notes

The BeBox was produced from 1995-1997 and is now a collector's item. NetBSD/bebox preserves the ability to run modern software on this unique dual-CPU PowerPC platform.

### Boot Examples

```bash
# Create installation floppy
mkbootimage -m bebox -b boot/boot -k INSTALL /tmp/install.img
dd if=/tmp/install.img of=/dev/rfd0a

# Create hard disk boot image
mkbootimage -m bebox -b boot/boot -k GENERIC /tmp/hd_boot.img
```
