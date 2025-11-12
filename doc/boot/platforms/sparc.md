# NetBSD/sparc Boot Documentation

## Platform Overview

NetBSD/sparc is the port of NetBSD to Sun Microsystems' 32-bit SPARC (Scalable Processor ARChitecture) workstations and servers. This includes:

- **sun4** - First generation SPARC systems (Sun-4/110, Sun-4/200, Sun-4/300, Sun-4/400)
- **sun4c** - Second generation with MMU improvements (SPARCstation 1, 1+, 2, IPC, IPX, SLC, ELC)
- **sun4m** - Multiprocessor-capable systems with SRMMU (SPARCstation 4, 5, 10, 20, LX, Classic, etc.)
- **sun4d** - High-end multiprocessor servers (SPARCcenter 1000, SPARCcenter 2000)

The platform supports machines with Sun's OpenBoot PROM (versions 1.x through 2.x) and older sun4 Monitor ROM firmware.

## Boot Method

**Primary Boot Method:** OpenBoot PROM / Sun Monitor

The sparc boot process depends on the system generation:

### OpenBoot PROM (sun4c/sun4m/sun4d)
OpenBoot PROM is based on IEEE 1275-1994 (Open Firmware) and provides:
- Device tree enumeration
- Forth interpreter for scripting
- Boot device selection and management
- Runtime services during boot
- Network booting capabilities (RARP/TFTP)

### Sun Monitor (sun4)
Older sun4 systems use the Sun Monitor ROM:
- Simple command interface
- Limited device support
- Basic loading capabilities

### Boot Sequence

1. **Hardware Reset** → PROM/Monitor initialization
2. **PROM** → Loads first-stage bootloader (bootblk or bootxx)
3. **First-stage** → Loads second-stage bootloader (boot or ofwboot)
4. **Second-stage** → Loads NetBSD kernel
5. **Kernel** → System initialization

## Boot Loader Implementation

### Primary Bootloaders

NetBSD/sparc supports two boot paths depending on the PROM version:

#### 1. OpenBoot PROM Boot Path (sun4c/sun4m)

**Location:** `/sys/arch/sparc/stand/`

**Components:**
- **bootblk** - First-stage OpenBoot bootloader (written in Forth)
- **ofwboot** - Second-stage OpenFirmware bootloader
- **boot** - Second-stage bootloader (for older PROM versions)

#### 2. Traditional Boot Path (sun4/older sun4c)

**Components:**
- **bootxx** - First-stage bootloader
- **boot** - Second-stage bootloader

### First-Stage Bootloader: bootblk (OpenBoot)

**Location:** `/sys/arch/sparc/stand/bootblk/`

**Key Source Files:**
- `bootblk.fth` - Forth-based bootblock that parses disklabel and filesystem
- `genfth.cf` - Configuration for FFS bootblock generation
- `genlfs.cf` - Configuration for LFS bootblock generation

**Implementation Details:**

The bootblock is written in Forth and runs directly in the OpenBoot PROM environment. It:

1. Parses the NetBSD disklabel to locate the filesystem
2. Understands FFS (UFS1/UFS2) and LFS filesystem structures
3. Locates and loads `/ofwboot` from the root filesystem
4. Transfers control to ofwboot

**Key Features:**
```forth
\ From bootblk.fth
\ IEEE 1275 Open Firmware Boot Block
\ Parses disklabel and UFS and loads the file called 'ofwboot'

: do-boot ( bootfile -- )
   ." NetBSD IEEE 1275 Multi-FS Bootblock" cr
   boot-path load-file ( -- load-base )
   dup 0<>  if  " init-program " evaluate  then
;
```

**Filesystem Support:**
- FFS v1 (UFS1)
- FFS v2 (UFS2)
- LFS v1 and v2
- RAIDframe partitions (detected via superblock offset)

### First-Stage Bootloader: bootxx (Traditional)

**Location:** `/sys/arch/sparc/stand/bootxx/`

**Key Source Files:**
- `bootxx.c` - Main bootxx implementation

**Implementation Details:**

The bootxx loader is a minimal C program that:

1. Reads block table installed by `installboot(8)`
2. Loads the second-stage bootloader (`boot`) using PROM I/O services
3. Handles a.out executable format with ZMAGIC, NMAGIC, or OMAGIC

**Code Structure:**
```c
// From bootxx.c
struct shared_bbinfo bbinfo = {
    { SPARC_BBINFO_MAGIC },
    0,
    SHARED_BBINFO_MAXBLOCKS,
    { 0 }
};

void loadboot(struct open_file *f, char *addr)
{
    // Load blocks specified in bbinfo.bbi_block_table
    for (i = 0; i < bbinfo.bbi_block_count; i++) {
        blk = bbinfo.bbi_block_table[i];
        (f->f_dev->dv_strategy)(f->f_devdata, F_READ, blk,
            bbinfo.bbi_block_size, buf, &n);
        memcpy(addr, buf, bbinfo.bbi_block_size);
        addr += n;
    }
}
```

### Second-Stage Bootloader: boot

**Location:** `/sys/arch/sparc/stand/boot/`

**Key Source Files:**
- `boot.c` - Main boot program
- `bootinfo.c` - Boot information structure
- `prompatch.c` - PROM patching for compatibility

**Implementation Details:**

The `boot` program provides:
- Interactive kernel selection
- Boot option parsing
- Memory management and MMU setup
- Kernel loading (a.out and ELF formats)
- Bootinfo structure creation

**Kernel Search Order:**
```c
// From boot.c
char *kernels[] = {
    "netbsd",
    "netbsd.gz",
    "netbsd.old",
    "netbsd.old.gz",
    "onetbsd",
    "onetbsd.gz",
    "vmunix",
    NULL
};
```

**Memory Management:**

The bootloader must find suitable physical memory for the kernel:

```c
// From boot.c
static paddr_t getphysmem(u_long size)
{
    // Find physical memory from PROM
    npmemarr = prom_makememarr(pmemarr, npmemarr, MEMARR_AVAILPHYS);

    // Search for suitable location (avoiding bootloader itself)
    for (mp = pmemarr, i = npmemarr; --i >= 0; mp++) {
        paddr_t pa = pmemarr[i].addr;
        u_long len = pmemarr[i].len;

        // Try to fit before bootloader
        if (pa < bstart && len >= size && (bstart - pa) >= size)
            return pa;

        // Try to fit after bootloader
        if (pa < bend) {
            len -= bend - pa;
            pa = bend;
        }

        if (len >= size)
            return pa;
    }
    return (paddr_t)-1;
}
```

**Compatibility Mode:**

NetBSD/sparc supports loading older kernels that expect to run at virtual address 0:

```c
// Compatibility mode for old kernels
if (compatmode) {
    // Double-map at VA 0 for compatibility
    if (pa != 0 && pmap_map(0, pa, size) != 0) {
        error = EFAULT;
        goto out;
    }
    loadaddrmask = 0x07ffffffUL;
}
```

### Second-Stage Bootloader: ofwboot

**Location:** `/sys/arch/sparc/stand/ofwboot/`

**Key Source Files:**
- `boot.c` - Main ofwboot implementation
- `Locore.c` - Low-level OpenFirmware interface
- `ofdev.c` - OpenFirmware device abstraction
- `net.c` - Network boot support
- `loadfile_machdep.c` - Architecture-specific loading

**Implementation Details:**

The ofwboot loader is shared between sparc and sparc64 platforms. It provides:

- Full OpenFirmware integration
- Support for multiple filesystems
- Network booting (TFTP)
- Boot configuration files (`boot.cfg`)
- Interactive boot menu
- Both compatibility and modern kernel formats

**Entry Point:**
```c
// From boot.c
void main(void *ofw)
{
    // Initialize OpenFirmware
    romp = ofw;
    prom_init();

    printf("\r>> %s, Revision %s\n", bootprog_name, bootprog_rev);

    // Parse boot arguments
    strncpy(bootdev, prom_getbootpath(), sizeof(bootdev) - 1);
    kboothowto = boothowto =
        bootoptions(prom_getbootargs(), bootdev, kernel, bootline);

    // Load and start kernel
    start_kernel(kernel, bootline, ofw, isfloppy, kboothowto);
}
```

## MMU Requirements

The SPARC architecture uses different MMU implementations depending on the system generation:

### sun4 and sun4c MMU

**Type:** Sun-4 MMU (2-level or 3-level page tables)

**Characteristics:**
- Context-based virtual memory (8 contexts on sun4c, varies on sun4)
- Segment-based addressing
- Page size: 4 KB or 8 KB
- Software-managed TLB (via trap handlers)

**Address Space Layout:**
```
sun4c:
0x00000000 - 0x0FFFFFFF   User space (256 MB)
0x10000000 - 0xEFFFFFFF   Unused
0xF0000000 - 0xFFFFFFFF   Kernel space (256 MB)

sun4:
0x00000000 - 0x0FFFFFFF   User space (256 MB)
0x10000000 - 0xEFFFFFFF   Unused
0xF0000000 - 0xFFC00000   Kernel space
0xFFD00000 - 0xFFF00000   OpenBoot PROM
```

**MMU Implementation in Bootloader:**

**Location:** `/sys/arch/sparc/stand/common/mmu.c`

```c
// From mmu.c
int pmap_map4(vaddr_t va, paddr_t pa, psize_t size)
{
    u_int n = (size + NBPG - 1) >> PGSHIFT;
    u_int pte;

    if (sun4_mmu3l)
        setregmap((va & -NBPRG), rcookie++);

    setsegmap((va & -NBPSG), ++scookie);
    while (n--) {
        pte = PG_S | PG_V | PG_W | PG_NC |
              ((pa >> PGSHIFT) & PG_PFNUM);
        setpte4(va, pte);
        va += NBPG;
        pa += NBPG;
        if ((va & (NBPSG - 1)) == 0) {
            setsegmap(va, ++scookie);
        }
    }
    return 0;
}
```

**PTE Format (sun4/sun4c):**
```
Bits 31-26: Physical Page Number (upper bits)
Bits 25-2:  Physical Page Number (lower bits)
Bit 1:      Modified
Bit 0:      Referenced
Bits [type]:
    PG_V    - Valid
    PG_W    - Writable
    PG_S    - Supervisor
    PG_NC   - Non-cacheable
```

### sun4m MMU (SRMMU)

**Type:** SPARC Reference MMU (3-level page tables)

**Characteristics:**
- Context-based virtual memory (up to 256 contexts)
- Hardware page table walk
- Page sizes: 4 KB, 256 KB, 16 MB
- Support for multiprocessor cache coherency

**Address Space Layout:**
```
0x00000000 - 0xEFFFFFFF   User/Kernel dynamic space
0xF0000000 - 0xFFFFFFFF   Kernel space (256 MB)
```

**MMU Implementation in Bootloader:**

```c
// From mmu.c
int pmap_map_srmmu(vaddr_t va, paddr_t pa, psize_t size)
{
    char buf[64];

    // Use PROM's Forth interpreter to map pages
    snprintf(buf, sizeof(buf), "%lx %x %lx %lx map-pages",
        pa, obmem, va, size);

    prom_interpret(buf);
    return 0;
}
```

**Context Table Entry (SRMMU):**
```
Bits 31-2:  Page Table Pointer (PTP)
Bits 1-0:   Entry Type (01 = PTD, 10 = PTE)
```

**Page Table Entry (SRMMU):**
```
Bits 31-8:  Physical Page Number
Bits 7:     Cacheable
Bits 6:     Modified
Bits 5:     Referenced
Bits 4-2:   Access permissions
Bits 1-0:   Entry Type (10 = PTE)
```

## Memory Map

### Physical Memory Layout

```
Sun4c/Sun4m typical layout:
0x00000000 - 0x00003FFF   Trap vectors and low memory
0x00004000 - 0x000FFFFF   Boot loader and early kernel
0x00100000 - MEMSIZE      Kernel and user memory
0xF0000000 - 0xFFFFFFFF   I/O and device space
```

### Virtual Memory Layout

```
Kernel Virtual Address Space:
0xF0000000 - 0xF07FFFFF   Kernel text and data
0xF0800000 - 0xFDFFFFFF   Kernel dynamic allocations
0xFE000000 - 0xFEFFFFFF   Device mappings
0xFF000000 - 0xFFBFFFFF   More device space
0xFFC00000 - 0xFFFFFFFF   PROM/Monitor space
```

### Boot Loader Memory Usage

**Load Address:** `0x4000` (PROM_LOADADDR)

The bootloader reserves:
- Stack: ~16 KB below load address
- Heap: Dynamic allocation from PROM
- Kernel: Loaded at physical address determined by getphysmem()

## OpenBoot Commands and Device Aliases

### Common OpenBoot PROM Commands

#### Basic Commands

```forth
ok boot                     # Boot default device/kernel
ok boot disk                # Boot from disk
ok boot net                 # Boot from network
ok boot disk:a netbsd       # Boot specific kernel from partition
ok boot -s                  # Boot single-user mode
ok boot -a                  # Ask for root device
ok boot -d                  # Boot with kernel debugger
ok boot -v                  # Verbose boot
```

#### Device Management

```forth
ok show-devs                # Show all devices
ok devalias                 # Show device aliases
ok devalias disk /sbus/esp@0,800000/sd@3,0
ok printenv                 # Show environment variables
ok setenv auto-boot? false  # Disable auto-boot
ok setenv boot-device disk  # Set default boot device
ok setenv boot-file netbsd  # Set default kernel
```

#### Diagnostic Commands

```forth
ok .registers               # Display CPU registers
ok ctrace                   # C stack trace
ok words                    # List Forth words
ok see [word]              # Decompile a Forth word
ok dump [addr] [len]       # Memory dump
```

#### Device Testing

```forth
ok test /memory             # Test memory
ok test /sbus/le           # Test ethernet
ok test /fd                # Test floppy
ok test /sd                # Test SCSI disk
ok probe-scsi              # Probe SCSI bus
ok probe-ide               # Probe IDE bus (sun4m)
```

### Device Tree Exploration

```forth
ok dev /                    # Go to root device node
ok ls                       # List children
ok .properties             # Show current node properties
ok dev /sbus               # Go to SBus node
ok pwd                     # Show current path
ok cd disk                 # Change to disk node
```

### Common Device Aliases

#### Disk Devices

```
disk        Primary disk (typically SCSI ID 3)
disk0       SCSI disk ID 0
disk1       SCSI disk ID 1
disk2       SCSI disk ID 2
disk3       SCSI disk ID 3
cdrom       CD-ROM drive
floppy      Floppy drive
```

#### Network Devices

```
net         Primary network interface
le          Lance Ethernet
ie          Intel Ethernet
hme         Happy Meal Ethernet
qe          Quad Ethernet
```

#### Serial Devices

```
ttya        Serial port A
ttyb        Serial port B
keyboard    Keyboard
screen      Display
```

### Boot Device Specification

Format: `device:[partition][file] [options]`

Examples:
```forth
ok boot disk:a                          # Partition 'a'
ok boot disk:b netbsd.old              # Partition 'b', specific kernel
ok boot /sbus/esp@0,800000/sd@3,0:a    # Full device path
ok boot net:dhcp                        # DHCP network boot
ok boot net:,192.168.1.100,,netbsd     # RARP boot with server IP
```

### Setting Boot Variables

```forth
# Automatic boot
ok setenv auto-boot? true
ok setenv boot-device disk
ok setenv boot-file netbsd
ok setenv boot-command boot

# Network boot setup
ok setenv boot-device net
ok setenv boot-file netbsd-GENERIC

# Diagnostic mode
ok setenv diag-switch? true
ok setenv diag-device net
```

### OpenBoot PROM Versions

Different machines use different PROM versions:

- **OBP 1.x** - Early sun4c (SPARCstation 1, 1+, IPC)
- **OBP 2.x** - Later sun4c and sun4m (SPARCstation 2, 5, 10, 20)
- **OBP 3.x** - Rare on 32-bit systems

Version differences affect available commands and device naming.

## Build and Installation Instructions

### Building the Bootloaders

#### Building from Source Tree

```bash
# Build all boot components
cd /usr/src
./build.sh -m sparc tools
./build.sh -m sparc distribution

# Or build just the bootloaders
cd /usr/src/sys/arch/sparc/stand
make cleandir
make depend
make
make install
```

#### Building Individual Components

```bash
# Build bootblocks
cd /usr/src/sys/arch/sparc/stand/bootblk
make

# Build bootxx
cd /usr/src/sys/arch/sparc/stand/bootxx
make

# Build boot
cd /usr/src/sys/arch/sparc/stand/boot
make

# Build ofwboot
cd /usr/src/sys/arch/sparc/stand/ofwboot
make
```

### Installation Methods

#### Method 1: Using installboot (Recommended)

For OpenBoot PROM systems:

```bash
# Install bootloader on root filesystem
installboot /dev/rsd0a /usr/mdec/bootxx /ofwboot

# For older systems
installboot /dev/rsd0a /usr/mdec/bootxx /boot
```

**How installboot works:**
1. Writes bootxx/bootblk to the boot blocks area (first 8KB of partition)
2. Stores block list of second-stage loader in bootxx
3. Second-stage loader (boot/ofwboot) is placed in root filesystem

#### Method 2: Manual Installation

```bash
# Write boot blocks manually (advanced)
dd if=/usr/mdec/bootxx of=/dev/rsd0a bs=512 count=16

# Install second-stage loader
cp /usr/mdec/ofwboot /ofwboot
```

#### Method 3: Network Boot Setup

Configure RARP and TFTP services:

```bash
# On boot server:
# 1. Configure /etc/ethers
08:00:20:12:34:56  sparc-client

# 2. Configure /etc/hosts
192.168.1.100  sparc-client

# 3. Configure /etc/bootparams
sparc-client  root=server:/export/sparc-root

# 4. Setup TFTP
cp /usr/mdec/ofwboot /tftpboot/
chmod 644 /tftpboot/ofwboot

# 5. Start services
rarpd -a
bootparamd
tftpd -l -s /tftpboot
```

### Installing on Different Partition Types

#### FFS (Fast File System)

```bash
newfs /dev/rsd0a
mount /dev/sd0a /mnt
cp /usr/mdec/ofwboot /mnt/
installboot /dev/rsd0a /usr/mdec/bootxx /ofwboot
```

#### LFS (Log-structured File System)

```bash
newfs_lfs /dev/rsd0a
mount -t lfs /dev/sd0a /mnt
cp /usr/mdec/ofwboot /mnt/
installboot /dev/rsd0a /usr/mdec/bootxx /ofwboot
```

### Disk Label Requirements

The bootloader requires a valid NetBSD disklabel:

```bash
# Create disklabel
disklabel -e sd0

# Typical layout:
# a: root partition (starting at cylinder 0)
# b: swap partition
# c: whole disk (NetBSD portion)
# d: whole disk (physical)
```

**Important:** The 'a' partition should start at cylinder 0 (or very early) for bootability.

### Troubleshooting Installation

#### Boot Block Checksum Errors

```bash
# Regenerate boot blocks
cd /usr/src/sys/arch/sparc/stand
make cleandir
make
make install
installboot /dev/rsd0a /usr/mdec/bootxx /ofwboot
```

#### Cannot Find Second-Stage Loader

```bash
# Ensure ofwboot is in root directory
ls -l /ofwboot
# If missing:
cp /usr/mdec/ofwboot /

# Reinstall boot blocks
installboot /dev/rsd0a /usr/mdec/bootxx /ofwboot
```

#### Old PROM Cannot Boot

Some early sun4c machines need the traditional boot loader:

```bash
# Use boot instead of ofwboot
installboot /dev/rsd0a /usr/mdec/bootxx /boot
cp /usr/mdec/boot /
```

## Debugging with OpenBoot PROM

### Entering the PROM

#### From Running System

```bash
# Send break signal (depends on console type)
# On serial: Send BREAK
# On keyboard: Stop-A (L1-A)
```

This drops to the PROM prompt if `diag-switch?` is not set, or to the kernel debugger (DDB) if kernel debugging is enabled.

#### At Boot

```
# During power-on, press:
Stop (L1 key)  # Interrupt boot sequence
```

### Debug Commands

#### Memory Inspection

```forth
ok dump 0xf0000000 100         # Dump 256 bytes
ok 0xf0000000 20 dump          # Alternative syntax
ok .memory                      # Show memory configuration
ok .registers                   # Show CPU registers
ok .locals                      # Show local variables (if any)
```

#### CPU State

```forth
ok .registers                   # Display all registers
ok %g0                         # Show specific register
ok %pc                         # Show program counter
ok %psr                        # Show processor state register
```

#### Stack Traces

```forth
ok ctrace                      # C stack trace
ok .trap-registers             # Show trap state
```

#### Memory Modification

```forth
ok 0xf0000000 l@               # Read long word
ok 0xf0000000 10 l!            # Write long word
ok 0xf0000000 w@               # Read word (16-bit)
ok 0xf0000000 c@               # Read byte
```

#### Disassembly

```forth
ok dis 0xf0000000 100          # Disassemble from address
ok ctrace                      # Shows return addresses for disassembly
```

### Boot Debugging Options

#### Verbose Boot

```forth
ok boot -v                     # Verbose kernel boot
ok boot -V                     # Very verbose (PROM level)
```

#### Debug Flags

```forth
ok boot -d                     # Enable kernel debugger (DDB)
ok boot -D                     # Extra debug output in bootloader
ok boot -c                     # User kernel configuration
```

#### Bootloader Debugging

The bootloader supports debug options:

```c
// Set via boot arguments
ok boot -D                     # Debug bootloader
ok boot -C                     # Compatibility mode
```

### PROM Diagnostics

#### Hardware Tests

```forth
ok test-all                    # Test all devices
ok test /memory                # Memory test
ok test /sbus/esp             # SCSI controller test
ok watch-net                   # Monitor network packets
ok watch-clock                 # Show clock ticks
```

#### Probing Hardware

```forth
ok probe-scsi                  # Scan SCSI bus
ok probe-scsi-all             # Scan all SCSI buses
ok show-devs                   # List all devices
ok .properties                 # Show device properties
```

### Common Debug Scenarios

#### Kernel Won't Boot

```forth
# Check bootloader loading
ok boot -V disk:a netbsd

# Try compatibility mode
ok boot -C disk:a netbsd

# Check kernel location
ok dir disk:a

# Load kernel manually (advanced)
ok load disk:a netbsd
ok init-program
```

#### Network Boot Issues

```forth
# Watch network activity
ok watch-net

# Boot with network debugging
ok boot net -V

# Check network device
ok test net
ok probe-net

# Show network properties
ok dev /sbus/le
ok .properties
```

#### Memory Issues

```forth
# Test memory
ok test /memory

# Show memory configuration
ok .memory

# Check physical memory regions
ok dev /memory
ok .properties
ok reg
```

#### MMU Debugging

For sun4/sun4c systems:

```forth
ok .context                    # Show current context
ok .maps                       # Show virtual mappings
```

For sun4m systems (SRMMU):

```forth
ok .mmu                        # Show MMU registers
ok .mmu-table                  # Show page tables
```

### Custom PROM Scripts

Create boot scripts for automated debugging:

```forth
# Define custom word
ok : my-boot boot disk:a netbsd -d ;
ok my-boot

# Save to NVRAM
ok nvalias myboot " boot disk:a netbsd -d"
ok setenv boot-command myboot
```

### Reset and Recovery

```forth
ok reset                       # Soft reset
ok reset-all                   # Hard reset (reinit PROM)
ok set-defaults               # Reset NVRAM to defaults
ok sync                        # Sync before reset
```

### Serial Console Debugging

For headless systems:

```forth
# Configure serial console
ok setenv input-device ttya
ok setenv output-device ttya
ok reset-all

# Serial port settings
ok setenv ttya-mode 9600,8,n,1,-
```

### PROM Password Recovery

If PROM security password is set:

1. Open system case
2. Locate NVRAM chip or jumper
3. Temporarily remove/short to clear settings
4. Power on and use `set-defaults`
5. Replace NVRAM/jumper

**Warning:** This varies by system model. Consult hardware documentation.

## Advanced Topics

### Multi-Boot Configuration

Create a `boot.cfg` file in the root filesystem:

```
banner=NetBSD/sparc Boot Menu
timeout=10
default=1

menu=Boot NetBSD:boot netbsd
menu=Boot NetBSD (single user):boot netbsd -s
menu=Boot NetBSD (verbose):boot netbsd -v
menu=Boot NetBSD.old:boot netbsd.old
menu=Drop to boot prompt:prompt
```

### Custom Bootloader Builds

Modify bootloader configuration:

```bash
# Edit build options
cd /usr/src/sys/arch/sparc/stand/ofwboot
vi Makefile

# Add debug flags
CPPFLAGS+= -DDEBUG
CPPFLAGS+= -DBOOT_DEBUG

# Rebuild
make clean
make
```

### Cross-Building Bootloaders

```bash
# Build on another platform
cd /usr/src
./build.sh -m sparc -U tools
./build.sh -m sparc -U distribution

# Bootloaders will be in:
# obj/usr/src/sys/arch/sparc/stand/
```

## References

### Source Code Locations

- `/sys/arch/sparc/stand/` - All bootloader code
- `/sys/arch/sparc/stand/bootblk/` - OpenBoot bootblock (Forth)
- `/sys/arch/sparc/stand/bootxx/` - Traditional first-stage
- `/sys/arch/sparc/stand/boot/` - Traditional second-stage
- `/sys/arch/sparc/stand/ofwboot/` - OpenFirmware second-stage
- `/sys/arch/sparc/stand/common/` - Shared code (MMU, devices)
- `/sys/arch/sparc/include/` - Architecture headers
- `/sys/arch/sparc/sparc/locore.s` - Kernel entry point

### Documentation

- `installboot(8)` - Bootloader installation utility
- `boot(8)` - Boot procedures and options
- `openprom(4)` - OpenBoot PROM interface
- IEEE 1275-1994 - Open Firmware standard

### Hardware Manuals

- Sun4 Architecture Manual
- SPARC Architecture Manual, Version 8
- Sun-4c Architecture Manual
- SPARC Reference MMU Architecture Manual (sun4m)
- OpenBoot PROM Toolkit User's Guide

### NetBSD Documentation

- NetBSD/sparc FAQ
- NetBSD Installation Guide
- NetBSD Kernel Developer Documentation
