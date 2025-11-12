# NetBSD/alpha Boot Documentation

$NetBSD$

## Platform Overview

**Architecture**: Digital Alpha (DEC Alpha, Alpha AXP)
**Port Date**: 1995-02-13
**Boot Method**: Multi-stage via SRM console firmware (bootxx → boot → kernel)
**Firmware**: SRM Console (VMS/OSF PALcode support)
**MMU Requirements**: 3-level page tables, PALcode-managed TLB, virtual addressing at boot

## Hardware Support

- **CPU**: Digital/Compaq Alpha processors (21064, 21164, 21264, 21364)
- **Memory**: Full 64-bit address space (43-bit virtual, 44-bit physical typical)
- **Boot Devices**: SCSI disk, IDE disk, CD-ROM, network (bootp/MOP), floppy
- **Firmware**: SRM Console (System Reference Manual firmware)
- **Systems**: DEC 3000, AlphaStation, AlphaServer, Alpha PC (various models)

### Supported System Types

From `/sys/arch/alpha/include/rpb.h`, NetBSD supports over 30 Alpha system types:

- **Workstations**: AlphaStation (200, 400, 500, 600), DEC 3000 (300, 400, 500, 600, 700, 800, 900)
- **Servers**: AlphaServer (1000, 2000, 2100, 4000, 4100, 7000), DEC 21000
- **Development boards**: EB64, EB64+, EB66, EB164
- **Special systems**: Jensen, Avanti, Miata, Mikasa, Noritake, Rawhide, Wildfire, Marvel

## Boot Process

### Stage 0: SRM Console Firmware

The Alpha architecture uses SRM (System Reference Manual) console firmware, which provides:

1. **Power-On Self Test (POST)** - Hardware initialization
2. **PALcode loading** - Privileged Architecture Library code
3. **Console services** - Device I/O, memory management
4. **Boot device selection** - From specified boot device

**SRM Console Commands**:
```
>>> show device              # List bootable devices
>>> boot dka0                # Boot from SCSI device
>>> boot ewa0 -file netbsd   # Network boot
>>> boot -flags a            # Boot single-user
```

### Stage 1: Primary Bootstrap (bootxx)

**Location**: `/sys/arch/alpha/stand/bootxx_*/`
**Size**: ~15 blocks (7.5KB)
**Load Address**: Low memory (firmware-dependent)
**Purpose**: Load secondary bootloader from filesystem

#### Variants

- **bootxx_ffs**: Fast File System (FFSv1)
- **bootxx_ffsv2**: FFSv2 support
- **bootxx_lfs**: Log-structured File System
- **bootxx_cd9660**: ISO 9660 (CD-ROM)

#### Operation

From `/sys/arch/alpha/stand/common/bootxx.c`:

1. **Initialize PROM callbacks** - `init_prom_calls()`
2. **Open boot device** - Using SRM firmware services
3. **Load secondary boot** - Read `/boot` from filesystem
4. **Verify size** - Check `SECONDARY_MAX_LOAD` limit
5. **Transfer control** - Jump to `SECONDARY_LOAD_ADDRESS`

```c
(*((void(*)(int))SECONDARY_LOAD_ADDRESS))(booted_dev_fd);
```

**Key Constants**:
- `SECONDARY_LOAD_ADDRESS`: Where `/boot` is loaded
- `SECONDARY_MAX_LOAD`: Maximum secondary bootloader size

### Stage 2: Secondary Bootstrap (boot)

**Location**: `/sys/arch/alpha/stand/boot/`
**Load Address**: Determined by primary bootstrap
**Purpose**: Load and execute kernel

#### Features

From `/sys/arch/alpha/stand/common/boot.c`:

1. **PALcode switching** - Switch from VMS PALcode to OSF/1 PALcode
2. **Device recognition** - Identify boot device
3. **File system access** - Full filesystem support (FFS, LFS, CD9660, NFS)
4. **Kernel loading** - ELF and ECOFF executable support
5. **Interactive boot** - User can specify kernel and flags

#### PALcode Switching

Alpha CPUs use PALcode (Privileged Architecture Library) for privileged operations. The bootloader switches from VMS PALcode to OSF/1 (Unix) PALcode:

**From `/sys/arch/alpha/stand/common/OSFpal.c`**:

```c
void OSFpal(void) {
    struct rpb *r = (struct rpb *)HWRPB_ADDR;  // Hardware RPB at 0x10000000
    struct pcs *p = LOCATE_PCS(r, r->rpb_primary_cpu_id);

    if (p->pcs_pal_type == PAL_TYPE_OSF1) {
        // Already running OSF PALcode
        ptbr_save = ((struct alpha_pcb *)p)->apcb_ptbr;
        return;
    }

    switch_palcode();  // Switch from VMS to OSF PALcode
}
```

**PALcode Operations**:
- Memory management (TLB handling)
- Exception handling
- Interrupt dispatch
- Process context switching
- I/O operations

#### Kernel Search Order

From `boot.c`:

```c
char *kernelnames[] = {
    "netbsd",      "netbsd.gz",
    "netbsd.bak",  "netbsd.bak.gz",
    "netbsd.old",  "netbsd.old.gz",
    "onetbsd",     "onetbsd.gz",
    "netbsd.alpha", "netbsd.alpha.gz",
    NULL
};
```

### Stage 3: Kernel

**Entry Point**: Kernel `start` symbol
**Format**: ELF64 or ECOFF
**Load Address**: Kernel virtual address (typically 0xFFFFFC0000000000+)

#### Bootinfo Structure

The bootloader passes information to the kernel via `struct bootinfo_v1`:

```c
struct bootinfo_v1 {
    uint32_t ssym;              // Symbol table start
    uint32_t esym;              // Symbol table end
    char booted_kernel[64];     // Kernel filename
    char boot_flags[64];        // Boot flags
    void *hwrpb;                // Hardware RPB pointer (0x10000000)
    uint32_t hwrpbsize;         // RPB size
    void *cngetc;               // Console getc function
    void *cnputc;               // Console putc function
    void *cnpollc;              // Console poll function
};
```

#### Kernel Entry

From `/sys/arch/alpha/stand/common/start.S`:

```asm
NESTED(start, 1, ENTRY_FRAME, ra, 0, 0)
    br      pv, Lstartgp
Lstartgp:
    LDGP(pv)

    lda     a0, _edata          # Clear BSS
    xor     a1, a1, a1
    lda     a2, _end
    subq    a2, a0, a2
    CALL(memset)

    CALL(main)                  # Transfer to C code

    call_pal PAL_halt           # Halt if return
END(start)
```

The kernel receives:
- **a0**: First free page / FFP (First Free Page)
- **a1**: PTBR (Page Table Base Register)
- **a2**: BOOTINFO_MAGIC
- **a3**: Pointer to bootinfo structure
- **a4**: Bootinfo version (1)

## MMU Setup

### Alpha Virtual Memory Architecture

**Page Size**: 8KB (8192 bytes)
**Virtual Address**: 64-bit (43-bit usable on most systems)
**Physical Address**: 44-bit typical (system-dependent)
**Page Table Levels**: 3 levels
**TLB**: Software-managed via PALcode

### Page Table Structure

Alpha uses a 3-level page table hierarchy:

```
Virtual Address (43 bits):
[42:33] - Level 1 index (10 bits, 1024 entries)
[32:23] - Level 2 index (10 bits, 1024 entries)
[22:13] - Level 3 index (10 bits, 1024 entries)
[12:0]  - Page offset (13 bits, 8KB page)
```

### Page Table Entry (PTE) Format

**64-bit PTE**:

```
Bit 0:     V    (Valid)
Bit 1:     FOR  (Fault On Read)
Bit 2:     FOW  (Fault On Write)
Bit 3:     FOE  (Fault On Execute)
Bit 4:     ASM  (Address Space Match)
Bit 5:     GH   (Granularity Hint)
Bits 6-7:  Reserved
Bits 8-31: PFN  (Page Frame Number) [low 24 bits]
Bits 32-63: PFN [high 32 bits, total 56 bits physical address]
```

### PALcode TLB Management

Unlike most architectures, Alpha TLB management is handled by PALcode:

**PALcode Operations**:
- `PAL_dtbmiss`: Data TLB miss handler
- `PAL_itbmiss`: Instruction TLB miss handler
- `PAL_tbi`: TLB invalidate
- `PAL_swppal`: Switch PALcode

**PTBR (Page Table Base Register)**:
- Contains physical address of level 1 page table
- Saved during PALcode switch
- Passed to kernel at boot

### Kernel Virtual Address Space

```
0x0000000000000000 - 0x000001FFFFFFFFFF : User space (8TB)
0x0000020000000000 - 0xFFFFFBFFFFFFFFFF : Reserved/unused
0xFFFFFC0000000000 - 0xFFFFFDFFFFFFFFFF : Kernel space (8TB)
0xFFFFFE0000000000 - 0xFFFFFFFFFFFFFFFF : PALcode space (8TB)
```

**Kernel Segments**:
```
0xFFFFFC0000000000 : Kernel text (code)
                   : Kernel rodata (read-only data)
                   : Kernel data (read-write data)
                   : Kernel BSS (uninitialized data)
```

### Hardware Restart Parameter Block (HWRPB)

**Address**: 0x10000000 (256MB physical, mapped virtual)
**Purpose**: Firmware-to-OS communication

The HWRPB (`struct rpb`) contains:
- **System type**: Identifies Alpha system model
- **CPU information**: Processor type, speed, features
- **Memory map**: Available physical memory regions
- **Console callback**: Firmware services (I/O, etc.)
- **PALcode revisions**: VMS and OSF PALcode versions
- **Boot device**: Device booted from

## Network Boot

### MOP (Maintenance Operations Protocol)

Alpha systems can boot via DEC's MOP protocol:

1. **Configure boot server**: Set up MOP daemon
2. **Prepare boot image**: Copy bootloader to MOP server
3. **Boot command**: `boot ewa0 -file netbsd` at SRM prompt

### BOOTP/TFTP

Modern Alpha systems support BOOTP/TFTP:

**Location**: `/sys/arch/alpha/stand/netboot/`

**Process**:
1. BOOTP request for IP configuration
2. TFTP download of kernel image
3. Direct kernel execution

## Memory Map

### Physical Memory Layout

```
0x00000000 - 0x00007FFF : Firmware use (32KB)
0x00008000 - 0x0FFFFFFF : Available RAM (varies)
0x10000000 - 0x10007FFF : HWRPB (Hardware Restart Parameter Block)
0x10008000 - ...        : More available RAM
...                     : System-dependent I/O space
```

### Virtual Memory (after PALcode switch)

```
0x0000000000000000 : User space start
0x000001FFFFFFFFFF : User space end (8TB)

0xFFFFFC0000000000 : Kernel space start
  Kernel text, rodata, data, BSS
  Kernel heap
  Device mappings

0xFFFFFDFFFFFFFFFF : Kernel space end (8TB)

0xFFFFFE0000000000 : PALcode space (8TB)
```

## Build System

### Source Locations

- **Primary bootstrap**: `/sys/arch/alpha/stand/bootxx_*/`
- **Secondary bootstrap**: `/sys/arch/alpha/stand/boot/`, `/sys/arch/alpha/stand/common/`
- **Network boot**: `/sys/arch/alpha/stand/netboot/`
- **Utilities**: `/sys/arch/alpha/stand/mkbootimage/`, `/sys/arch/alpha/stand/setnetbootinfo/`

### Building Bootloaders

```sh
# Build all bootloaders
cd /sys/arch/alpha/stand
make

# Build specific bootloader
cd /sys/arch/alpha/stand/bootxx_ffsv2
make
```

### Key Makefiles

**`Makefile.bootprogs`**: Common definitions for boot programs
**`Makefile.bootxx`**: Rules for primary bootstraps
**`Makefile.inc`**: Include paths and flags

## Installation

### Installing Primary Bootstrap

```sh
# Install bootxx for FFSv2 filesystem
installboot /dev/rsd0a /usr/mdec/bootxx_ffsv2

# For other filesystems
installboot /dev/rsd0a /usr/mdec/bootxx_ffs    # FFSv1
installboot /dev/rsd0a /usr/mdec/bootxx_lfs    # LFS
```

### Installing Secondary Bootstrap

```sh
# Copy boot program to root filesystem
cp /usr/mdec/boot /boot
chmod 444 /boot
```

### Creating Bootable CD

```sh
# Copy CD bootloader
cp /usr/mdec/bootxx_cd9660 /tmp/cdboot

# Create bootable ISO
makefs -t cd9660 -o 'bootimage=alpha;/tmp/cdboot' netbsd.iso /cdroot
```

## Debugging

### SRM Console Debugging

**Enable verbose boot**:
```
>>> boot -flags av dka0
```

**Boot options**:
- `-flags a`: Single-user mode
- `-flags d`: Drop to kernel debugger
- `-flags v`: Verbose boot
- `-flags s`: Single-user mode

### Serial Console

**Configure at SRM prompt**:
```
>>> set console serial
>>> init
```

**Serial parameters**: 9600 baud, 8N1 (default)

### Common Boot Issues

**Problem**: "Can't open boot device"
**Solution**: Check filesystem type matches bootxx variant

**Problem**: "Secondary boot returned!"
**Solution**: `/boot` corrupted or wrong architecture

**Problem**: "PALcode switch failed"
**Solution**: System requires VMS PALcode, not supported

**Problem**: Kernel doesn't boot
**Solution**: Check kernel is for alpha architecture (`file netbsd`)

### Boot Loader Debugging

**Enable debug output** (rebuild with DEBUG):
```c
#define DEBUG 1  // In bootxx.c or boot.c
```

**Monitor PALcode**:
```
>>> show pal      # Display current PALcode
```

## Platform-Specific Considerations

### CPU Variants

**21064 (EV4)**: First-generation Alpha
- No byte/word load/store instructions
- 32-entry TLB
- Used in: DEC 3000, AlphaStation 200

**21164 (EV5)**: Second-generation
- Byte/word instructions added
- 64-entry TLB
- Used in: AlphaStation 500, AlphaServer 1000

**21264 (EV6)**: Third-generation
- Out-of-order execution
- 128-entry TLB
- Used in: AlphaServer ES40, DS20

**21364 (EV7)**: Fourth-generation
- Highest performance
- Used in: AlphaServer GS series, Marvel

### Firmware Differences

**SRM Console**: Standard Unix boot (required for NetBSD)
**AlphaBIOS**: Windows NT boot (not compatible)
**ARC Firmware**: Alternative firmware (limited support)

**Note**: NetBSD requires SRM Console firmware. Systems with AlphaBIOS must have SRM installed.

### Byte Order

**Endianness**: Little-endian
**Alignment**: Strict alignment required for 16, 32, 64-bit access
**Unaligned Access**: Software trap handler required

### Memory Requirements

**Minimum**: 32MB RAM
**Recommended**: 128MB+
**Bootloader**: Uses ~1-2MB during boot

### Disk Partitioning

Alpha uses BSD disklabels with 8 partitions (a-h):
- **a**: Root filesystem (/)
- **b**: Swap
- **c**: Whole disk
- **d-h**: Additional partitions

### HWRPB System Types

The bootloader identifies system type from HWRPB:
- Determines device drivers to use
- Configures system-specific features
- Enables/disables hardware features

## References

### Source Files

**Primary bootstrap**:
- `/sys/arch/alpha/stand/common/bootxx.c` - Main primary bootstrap
- `/sys/arch/alpha/stand/common/start.S` - Assembly entry point

**Secondary bootstrap**:
- `/sys/arch/alpha/stand/common/boot.c` - Main secondary bootstrap
- `/sys/arch/alpha/stand/common/OSFpal.c` - PALcode switching
- `/sys/arch/alpha/stand/common/prom.c` - SRM console interface

**Headers**:
- `/sys/arch/alpha/include/rpb.h` - HWRPB structure definitions
- `/sys/arch/alpha/include/prom.h` - SRM console prototypes
- `/sys/arch/alpha/include/pte.h` - Page table entry formats

### Man Pages

- `boot(8)` - General boot procedures
- `installboot(8)` - Install bootloader
- `disklabel(8)` - Disk partitioning

### External Documentation

- Alpha Architecture Reference Manual (Digital Equipment Corporation)
- Alpha System Reference Manual (SRM Console specification)
- Alpha PALcode specification

---

*Last Updated: 2025-11-12*
*Architecture Maintainer: NetBSD/alpha Port*
