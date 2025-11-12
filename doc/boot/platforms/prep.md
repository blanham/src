# NetBSD/prep Boot Documentation

## Platform Overview

NetBSD/prep is the port of NetBSD to PowerPC Reference Platform (PReP) compliant systems. PReP was an open standard developed by IBM, Motorola, and others to create a common PC-like architecture for PowerPC systems.

Supported systems include:
- IBM RS/6000 43P series (non-CHRP models)
- Motorola PowerStack and FirePower series
- Various PReP-compliant workstations and servers
- Some embedded PReP systems

PReP systems use a PC BIOS-like firmware with PReP boot specification compliance.

## Boot Method

**Primary Boot Method:** PReP Boot Specification

PReP systems use a hybrid boot architecture:
- PC-style BIOS for initial boot
- PReP-compliant boot sector
- Residual data structure for hardware configuration
- Direct kernel loading or multi-stage boot

### Boot Sequence

1. **Power-On** → PReP BIOS initialization
2. **BIOS** → Load boot sector from bootable partition
3. **Boot Sector** → Loads `boot` program
4. **Bootloader** → Loads NetBSD kernel with residual data
5. **Kernel** → Initializes using residual data

## Boot Loader Implementation

### Primary Bootloader: boot

**Location:** `/sys/arch/prep/stand/boot/`

The PReP bootloader:
- Runs in real mode with BIOS services
- Reads residual data from firmware
- Supports FFS, IDE, and SCSI devices
- Provides serial and VGA console options
- Loads ELF kernels

**Key Components:**
- `boot/` - Main bootloader
- `boot_com0/` - Serial console variant
- `boot_com0_vreset/` - Serial with video reset
- `common/` - Shared boot code
- `installboot/` - Installation utility

### Console Variants

Three bootloader variants for different console configurations:

```bash
boot/           # VGA console (default)
boot_com0/      # Serial console on COM1
boot_com0_vreset/  # Serial with VGA reset support
```

### Boot Image Format

The bootloader must be packaged in PReP boot format:

```bash
# Create bootable floppy image
mkbootimage boot/boot /tmp/boot.fs

# Create bootable floppy with kernel
mkbootimage -m prep -b boot/boot -k ../compile/YOUR_KERNEL/netbsd /tmp/boot.fs
```

## BAT Register Setup

**Source:** `/sys/arch/prep/prep/machdep.c:initppc()`

### BAT Configuration

```c
void initppc(u_int startkernel, u_int endkernel, u_int args, void *btinfo)
{
    /* Copy bootinfo and residual data */
    memcpy(bootinfo, btinfo, sizeof(bootinfo));
    memcpy(&resdata, resinfo->addr, sizeof(resdata));
    res = &resdata;

    /* Set memory and clock from residual data */
    physmemr[0].start = 0;
    physmemr[0].size = res->TotalMemory & ~PGOFSET;

    /* Initialize with rs6k-style PCI bridge mapping */
    prep_initppc(startkernel, endkernel, args,
        0xbf800000, BAT_BL_8M,    /* rs6k-style PCI bridge */
        0);
}
```

### Common BAT Setup (prep_initppc)

**Source:** `/sys/arch/powerpc/prep/prep_machdep.c`

The prep_initppc() function sets up standard PReP BATs:

```c
/* Typical PReP BAT layout */
DBAT0: 0xbf800000 - 8MB   (PCI configuration/I/O)
DBAT1: 0x80000000 - 256MB (PCI memory space)
DBAT2: I/O regions as needed
```

The exact BAT configuration varies by platform and is determined by:
- Residual data from firmware
- Platform quirks table
- PCI bridge type

## Boot Process Stages

### Stage 1: Firmware and Boot Sector

The PReP BIOS:
1. Performs POST (Power-On Self Test)
2. Enumerates PCI devices
3. Creates residual data structure
4. Searches for PReP boot partition (type 0x41)
5. Loads and executes boot sector

### Stage 2: Assembly Entry Point

**Source:** `/sys/arch/prep/prep/locore.S:__start`

```assembly
__start:
    /* Disable FPU/MMU/exceptions */
    li      0, 0
    mtmsr   0
    isync

    /* Compute end of kernel memory */
#if NKSYMS || defined(DDB) || defined(MODULAR)
    lis     7, _C_LABEL(startsym)@ha
    addi    7, 7, _C_LABEL(startsym)@l
    stw     3, 0(7)
    lis     7, _C_LABEL(endsym)@ha
    addi    7, 7, _C_LABEL(endsym)@l
    stw     4, 0(7)
#else
    lis     4, _C_LABEL(end)@ha
    addi    4, 4, _C_LABEL(end)@l
#endif

    /* Initialize CPU info */
    INIT_CPUINFO(4, 1, 9, 0)

    /* Call C initialization */
    lis     3, __start@ha
    addi    3, 3, __start@l
    bl      _C_LABEL(initppc)
```

### Stage 3: Cache Initialization

```assembly
    /* Enable internal i/d-cache */
    mfpvr   9
    rlwinm  9, 9, 16, 16, 31
    cmpwi   %r9, 1
    beq     3f                   # Skip for 601

    mfspr   11, SPR_HID0
    andi.   0, 11, HID0_DCE
    ori     11, 11, HID0_ICE|HID0_DCE
    ori     8, 11, HID0_ICFI
    bne     1f
    ori     8, 8, HID0_DCI       # Invalidate D-cache if disabled
1:
    sync
    mtspr   SPR_HID0, 8          # Enable and invalidate
    sync
    mtspr   SPR_HID0, 11         # Enable caches
    sync
    isync
```

### Stage 4: CPU-Specific Optimizations

```assembly
    /* Check for 604/604e/mach5 */
    cmpwi   %r9, 4               # 604
    cmpwi   %cr1, %r9, 9         # 604e
    cmpwi   %cr2, %r9, 10        # mach5
    cror    2, 2, 6
    cror    2, 2, 10
    bne     3f

    /* Enable 604-specific features */
    ori     11, 11, HID0_SIED|HID0_BHTE
    bne     2, 2f
    ori     11, 11, HID0_BTCD
2:
    mtspr   SPR_HID0, 11
3:
    sync
    isync

    bl      _C_LABEL(main)
```

### Stage 5: C Initialization (initppc)

The initppc() function:

1. **Copies bootinfo structure** from bootloader
2. **Copies residual data** - Contains hardware configuration
3. **Sets memory regions** from residual data TotalMemory field
4. **Configures CPU clock** from residual VPD (Vital Product Data)
5. **Calls prep_initppc()** for common PReP initialization

### Stage 6: Platform Setup (cpu_startup)

```c
void cpu_startup(void)
{
    /* Common OEA startup */
    oea_startup(res->VitalProductData.PrintableModel);

    /* Initialize interrupts using residual data */
    prep_init();

    /* Enable hardware interrupts */
    splraise(-1);
    __asm volatile ("mfmsr %0; ori %0,%0,%1; mtmsr %0"
                  : "=r"(msr) : "K"(PSL_EE));

    /* Allow malloc in bus_space */
    bus_space_mallocok();
}
```

## MMU Requirements

### Initial State

- **MMU:** Disabled in early boot
- **Caches:** Enabled after CPU detection
- **Real Mode:** Early boot code runs in real mode

### Supported Processors

- MPC601 (special case, different BAT handling)
- MPC603/603e
- MPC604/604e
- MPC750 (G3)
- Later variants

### Cache Configuration

Varies by processor:
- **603/603e:** 16KB I-cache, 16KB D-cache
- **604/604e:** 32KB I-cache, 32KB D-cache
- **750:** 32KB I-cache, 32KB D-cache

### Page Table Setup

After boot:
- Hash page table initialized
- Segment registers configured
- BATs map I/O regions
- Page tables handle RAM

## Memory Map

### Physical Memory Layout

```
0x00000000 - 0x00003fff    Exception vectors
0x00004000 - RAM_END       Main memory (varies)
0x80000000 - 0x8fffffff    PCI memory space (256MB)
0xbf800000 - 0xbfffffff    PCI configuration/I/O (8MB)
0xc0000000 - 0xfeffffff    I/O and device memory
0xfff00000 - 0xffffffff    Firmware ROM (1MB)
```

### Residual Data Structure

Located in low memory, contains:
- Total memory size
- CPU type and speed
- PCI device information
- Boot device information
- Vital Product Data (VPD)

**Key Fields:**
```c
RESIDUAL resdata;
resdata.TotalMemory        /* Physical RAM size */
resdata.VitalProductData.ProcessorBusHz  /* Bus frequency */
resdata.VitalProductData.PrintableModel  /* Model string */
```

### Virtual Memory Layout

```
0x00000000 - 0x0fffffff    User space (segment 0)
0x10000000 - 0xefffffff    User space (segments 1-14)
0xf0000000 - 0xffffffff    Kernel space (segment 15)
```

## Build and Installation

### Building the Bootloader

```bash
cd /sys/arch/prep/stand
make cleandir
make depend
make

# For cross-compilation:
for i in common boot_com0 boot; do (cd $i; ppc-make); done
cd ../../powerpc/stand/mkbootimage
make
```

### Creating Boot Media

#### Boot Floppy

```bash
# Bootloader only
mkbootimage boot/boot /tmp/boot.fs

# With kernel
mkbootimage -m prep -b boot/boot -k ../compile/YOUR_KERNEL/netbsd /tmp/boot.fs

# Write to floppy
dd if=/tmp/boot.fs of=/dev/rfd0a
```

#### Hard Disk Installation

```bash
# Create PReP boot partition (type 0x41)
fdisk /dev/rwd0c
# Create ~1MB partition, type 0x41 (PReP Boot)

# Install bootloader
installboot -v /dev/rwd0a /usr/mdec/boot /usr/mdec/bootxx

# Create root filesystem
newfs /dev/rwd0b
```

### PReP Partition Layout

```
Partition 1: PReP Boot (type 0x41) - 1-2 MB
Partition 2: NetBSD root (type 0xA9) - Remaining space
```

### Console Selection

Choose console at build time:

```bash
# VGA console
cd boot
make

# Serial console (COM1)
cd boot_com0
make

# Serial with video reset
cd boot_com0_vreset
make
```

## Debugging

### Residual Data Examination

Add debug output to initppc():

```c
void initppc(...)
{
    printf("Residual Data:\n");
    printf("  Total Memory: %lu MB\n", res->TotalMemory >> 20);
    printf("  Model: %s\n", res->VitalProductData.PrintableModel);
    printf("  CPU Bus: %lu MHz\n",
           be32toh(res->VitalProductData.ProcessorBusHz) / 1000000);
}
```

### Serial Console Debugging

Configure for serial output:
```bash
# Build serial console bootloader
cd boot_com0
make

# Kernel boot with serial console
boot -h  # Halt for serial console selection
```

Serial port settings:
- Port: COM1 (0x3F8)
- Default speed: 9600
- Format: 8N1

### Interrupt Vector Register

PReP systems have a special interrupt vector register:

```c
/* Mapped during cpu_startup() */
vaddr_t prep_intr_reg;
uint32_t prep_intr_reg_off;

/* Access interrupt vector */
uint8_t vector = *(uint8_t *)(prep_intr_reg + prep_intr_reg_off);
```

### Boot Loader Debugging

Enable debug output:
```c
/* In boot source, add: */
#define DEBUG 1

/* Will enable verbose output during boot */
```

### Common Boot Issues

**Problem:** "not found residual information in bootinfo"
- **Cause:** Bootloader/kernel version mismatch
- **Solution:** Rebuild both bootloader and kernel together

**Problem:** Incorrect memory size
- **Cause:** Residual data corruption
- **Solution:** Update firmware, check bootloader

**Problem:** PCI devices not found
- **Cause:** Residual data missing PCI info
- **Solution:** Firmware issue, update if possible

**Problem:** Wrong CPU speed reported
- **Cause:** VPD data incorrect
- **Solution:** Can be ignored, or patched in kernel

### Platform Quirks

Some PReP systems require special handling:

```c
/* Platform quirks table */
struct platform_quirkdata platform_quirks[] = {
    { "IBM,7043-140", QUIRK_NOCLOCK },
    { "MOT,PowerStack", QUIRK_NOLEGACY },
    /* ... */
};
```

### Low-Level Debugging

Insert debug points in locore.S:

```assembly
/* Debug output to serial port */
lis     %r10, 0x3f8 >> 16
ori     %r10, %r10, 0x3f8 & 0xffff
li      %r11, 'X'
stb     %r11, 0(%r10)
```

## Platform-Specific Notes

### IBM RS/6000 43P

- Non-CHRP firmware
- Uses residual data extensively
- May have SCSI or IDE boot device
- Typically has serial console

### Motorola PowerStack

- PReP-compliant
- PCI-based
- Often has VGA graphics
- Good PReP reference implementation

### FirePower Systems

- Originally from Powerhouse (FirePower Systems Inc.)
- Later sold by Motorola
- Clean PReP implementation
- Good hardware documentation

### Interrupt Controllers

PReP systems typically use:
- i8259 PIC for legacy ISA interrupts
- OpenPIC for PCI (some models)
- Custom interrupt controller (some models)

The interrupt controller type is determined from residual data.

### PCI Configuration

PReP systems use different PCI bridge types:
- IBM Eagle (RS6000 style)
- Motorola Raven
- Motorola Falcon
- Standard PCI-ISA bridges

Bridge type affects BAT setup and I/O mapping.

## References

### Source Files

- `/sys/arch/prep/prep/locore.S` - Assembly boot code
- `/sys/arch/prep/prep/machdep.c` - Platform initialization
- `/sys/arch/prep/stand/boot/` - Bootloader
- `/sys/arch/powerpc/prep/prep_machdep.c` - Common PReP code

### Specifications

- PowerPC Reference Platform Specification (PReP 1.1)
- PowerPC Microprocessor Common Hardware Reference Platform (CHRP)
- Residual Data Format Specification
- PowerPC Processor Binding to IEEE 1275

### PReP Resources

- IBM RS/6000 43P documentation
- Motorola PowerStack technical manuals
- PReP specification documents
- Residual data structure definitions

### Boot Command Examples

```bash
# Create installation media
mkbootimage -m prep -b boot/boot -k INSTALL /tmp/install.img

# Create hard disk boot
mkbootimage -m prep -b boot/boot -k GENERIC /boot/prep.img

# Serial console boot
mkbootimage -m prep -b boot_com0/boot -k GENERIC /boot/prep_serial.img
```

### Firmware Notes

Different PReP systems have different firmware capabilities:
- Some support OpenFirmware subset
- Most use BIOS-like interface
- Residual data is the common interface
- Boot from SCSI, IDE, or floppy supported
