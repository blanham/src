# NetBSD/algor Boot Documentation

$NetBSD$

## Platform Overview

**Architecture**: MIPS 32-bit (little-endian)
**Port Date**: 2001-06-06
**Boot Method**: Direct boot from PMON firmware
**Firmware**: PMON (Algorithmics MIPS Monitor)
**MMU Requirements**: MIPS TLB, no special bootloader setup required

## Hardware Support

NetBSD/algor supports three Algorithmics evaluation board families:

- **CPU**: MIPS R4000, R5000 series processors
- **Memory**: Up to 512MB (board-dependent)
- **Boot Devices**: IDE disk, network (via PMON)
- **Firmware**: PMON (Portable Monitor)
- **Supported Boards**:
  - **P-4032**: MIPS R4000PC processor, V962PBC PCI bridge
  - **P-5064**: MIPS R5000 processor, AMD PCnet-PCI Ethernet
  - **P-6032**: MIPS R4000/R5000, dual PCI bus

## Boot Process

### Overview

Unlike most other MIPS platforms, NetBSD/algor has **no standalone bootloader**. The kernel is loaded and executed directly by the PMON firmware.

### Boot Sequence

**Stage 1: PMON Firmware**

1. **Power-On**: System starts executing PMON from boot ROM
2. **Hardware initialization**: PMON initializes memory, PCI bus, devices
3. **Console setup**: Serial or VGA console activated
4. **PMON prompt**: User can enter commands or autoboot

**Stage 2: Kernel Loading**

PMON loads the NetBSD kernel directly using one of several methods:

**From IDE Disk**:
```
PMON> load ide0:netbsd
PMON> g
```

**From Network (TFTP)**:
```
PMON> ifaddr synopsys0 192.168.1.10
PMON> load tftp://192.168.1.1/netbsd
PMON> g
```

**From Flash**:
```
PMON> load flash:netbsd
PMON> g
```

**Stage 3: Kernel Initialization**

1. **Entry Point**: Kernel `mach_init()` function
2. **Arguments Passed**:
   - `argc`, `argv`, `envp` from PMON
   - Environment variables provide memory size, Ethernet address, etc.
3. **Bootstrap Process**:
   ```c
   mach_init(int argc, char *argv[], char *envp[])
   ```

### PMON Environment Variables

PMON passes critical configuration through environment variables:

- `memsize` - Physical memory size in MB
- `ethaddr` - Ethernet MAC address
- `bootdev` - Boot device identifier
- `bootopts` - Boot options (e.g., `-s`, `-a`)

**Accessing in Kernel**:
```c
const char *memsize_str = pmon_getenv("memsize");
const char *ethaddr_str = pmon_getenv("ethaddr");
```

**Source**: `/sys/arch/algor/algor/pmon.c`

## MIPS Memory Segments

NetBSD/algor uses standard MIPS 32-bit memory segmentation:

### Segment Layout

```
0x00000000 - 0x7FFFFFFF : useg   (User segment, 2GB)
                          - Mapped via TLB
                          - User mode accessible

0x80000000 - 0x9FFFFFFF : kseg0  (Kernel unmapped cached, 512MB)
                          - Physical = Virtual & 0x1FFFFFFF
                          - Cached
                          - Kernel mode only
                          - Used for kernel text/data

0xA0000000 - 0xBFFFFFFF : kseg1  (Kernel unmapped uncached, 512MB)
                          - Physical = Virtual & 0x1FFFFFFF
                          - Uncached
                          - Used for device registers
                          - Early boot console access

0xC0000000 - 0xFFFFFFFF : kseg2  (Kernel mapped, 1GB)
                          - Mapped via TLB
                          - Used for user process pages
```

### Physical Memory Map

**P-4032**:
```
0x00000000 - 0x0FFFFFFF : DRAM (up to 256MB)
0x10000000 - 0x13FFFFFF : PCI I/O space
0x14000000 - 0x17FFFFFF : PCI memory space (low)
0x40000000 - 0x7FFFFFFF : PCI memory space (high)
0x1FC00000 - 0x1FC7FFFF : Boot ROM (512KB)
```

**P-5064**:
```
0x00000000 - 0x1FFFFFFF : DRAM (up to 512MB)
0x40000000 - 0x5FFFFFFF : PCI memory space
0x80000000 - 0x9FFFFFFF : PCI I/O space
0x1FC00000 - 0x1FCFFFFF : Boot ROM (1MB)
```

**P-6032**:
```
0x00000000 - 0x0FFFFFFF : DRAM (up to 256MB)
0x20000000 - 0x3FFFFFFF : PCI0 memory space
0x40000000 - 0x5FFFFFFF : PCI1 memory space
0x1FC00000 - 0x1FCFFFFF : Boot ROM
```

## MMU and TLB

### MIPS TLB Overview

The MIPS TLB (Translation Lookaside Buffer) provides virtual-to-physical address translation for mapped segments (useg, kseg2).

**TLB Characteristics** (MIPS III):
- **Entries**: 48 (R4000), 48 (R5000)
- **Page Sizes**: 4KB, 16KB, 64KB, 256KB, 1MB, 4MB, 16MB
- **Wired Entries**: First N entries pinned for kernel use

### TLB Entry Format

```
EntryHi:  [VPN2 (27 bits)] [ASID (8 bits)]
EntryLo0: [PFN (24 bits)] [C (3)] [D (1)] [V (1)] [G (1)]
EntryLo1: [PFN (24 bits)] [C (3)] [D (1)] [V (1)] [G (1)]
PageMask: [Mask (12 bits)] - Determines page size
```

**Bit Fields**:
- **VPN2**: Virtual Page Number (divided by 2, covers even/odd pair)
- **ASID**: Address Space Identifier (process ID)
- **PFN**: Physical Frame Number
- **C**: Cache algorithm (0-7)
- **D**: Dirty (writable) bit
- **V**: Valid bit
- **G**: Global bit (ignore ASID)

### Cache Coherency Algorithms

**C field values**:
```
0: Reserved
1: Reserved
2: Uncached
3: Cacheable non-coherent
4: Cacheable coherent exclusive
5: Cacheable coherent exclusive on write
6: Cacheable coherent update on write
7: Uncached accelerated (write-combining)
```

### CP0 Registers

**Key Coprocessor 0 (CP0) registers**:

```
CP0_INDEX    ($0)  : TLB entry index
CP0_RANDOM   ($1)  : TLB random index
CP0_ENTRYLO0 ($2)  : TLB EntryLo0
CP0_ENTRYLO1 ($3)  : TLB EntryLo1
CP0_CONTEXT  ($4)  : Kernel PTE pointer
CP0_PAGEMASK ($5)  : TLB page mask
CP0_WIRED    ($6)  : TLB wired boundary
CP0_BADVADDR ($8)  : Bad virtual address
CP0_COUNT    ($9)  : Timer count
CP0_ENTRYHI  ($10) : TLB EntryHi
CP0_COMPARE  ($11) : Timer compare
CP0_STATUS   ($12) : Status register
CP0_CAUSE    ($13) : Exception cause
CP0_EPC      ($14) : Exception PC
CP0_PRID     ($15) : Processor ID
CP0_CONFIG   ($16) : Configuration
CP0_LLADDR   ($17) : Load-linked address
CP0_WATCHLO  ($18) : Watchpoint low
CP0_WATCHHI  ($19) : Watchpoint high
CP0_ECC      ($26) : ECC register
CP0_CACHEERR ($27) : Cache error
CP0_TAGLO    ($28) : Cache tag low
CP0_TAGHI    ($29) : Cache tag high
CP0_ERROREPC ($30) : Error exception PC
```

### Status Register (CP0_STATUS)

```
Bit  Name   Description
0    IE     Interrupt Enable
1    EXL    Exception Level
2    ERL    Error Level
3    KSU0   Kernel/Supervisor/User mode bit 0
4    KSU1   Kernel/Supervisor/User mode bit 1
5    UX     64-bit User mode enable
6    SX     64-bit Supervisor mode enable
7    KX     64-bit Kernel mode enable
8-15 IM     Interrupt Mask (8 levels)
16   DE     Disable Cache Errors
17   CE     Cache Error occurred
22   BEV    Bootstrap Exception Vectors
25   RE     Reverse Endian
27   CU0    Coprocessor 0 usable
28   CU1    Coprocessor 1 usable (FPU)
```

**Boot State**: BEV=1 (use bootstrap vectors at 0xBFC00000)
**Normal State**: BEV=0 (use vectors at 0x80000000)

### Boot-Time TLB Setup

**Performed by kernel** in `mips_vector_init()`:

```assembly
; No TLB setup required for kseg0/kseg1 access
; Early kernel runs entirely in unmapped segments

; Clear all TLB entries during initialization
li      t0, MIPS_KSEG0_START
mtc0    t0, CP0_ENTRYHI
mtc0    zero, CP0_ENTRYLO0
mtc0    zero, CP0_ENTRYLO1
li      t0, 47                   # Last TLB entry
1:
    mtc0    t0, CP0_INDEX
    nop
    tlbwi                        # Write Indexed TLB entry
    addi    t0, t0, -1
    bgez    t0, 1b
    nop
```

### Cache Operations

**MIPS cache instructions**:

**Cache OP Codes**:
```
0: Index Invalidate
1: Index Load Tag
2: Index Store Tag
4: Hit Invalidate
5: Hit Writeback Invalidate (D-cache)
5: Fill (I-cache)
6: Hit Writeback (D-cache)
```

**Cache Initialization**:
```c
/* Performed in mips_vector_init() */
mips_icache_sync_all();         /* Invalidate I-cache */
mips_dcache_wbinv_all();        /* Writeback-invalidate D-cache */
```

**Source**: `/sys/arch/mips/mips/cache.c`, `/sys/arch/mips/mips/mips_machdep.c`

## Kernel Initialization Sequence

**Source**: `/sys/arch/algor/algor/machdep.c`

```c
void mach_init(int argc, char *argv[], char *envp[])
{
    /* 1. Clear BSS segment */
    led_display('b', 's', 's', ' ');
    kernstart = (vaddr_t) mips_trunc_page(kernel_text) - 2 * NBPG;
    kernend   = (vaddr_t) mips_round_page(end);
    memset(edata, 0, kernend - (vaddr_t)edata);

    /* 2. Initialize exception vectors and caches */
    led_display('v', 'e', 'c', 'i');
    mips_vector_init(NULL, false);
    /* PMON calls are no longer valid after this */

    /* 3. Initialize PAGE_SIZE-dependent variables */
    led_display('p', 'g', 's', 'z');
    uvm_md_init();

    /* 4. Initialize platform-specific hardware */
    /* P4032: V962PBC PCI controller */
    /* P5064: Bonito64 PCI controller */
    /* P6032: Dual PCI buses */

    /* 5. Initialize bus space tags */
    /* - Local I/O bus */
    /* - PCI I/O space */
    /* - PCI memory space */

    /* 6. Attach console device */
    /* Usually serial (16550 UART) on all platforms */

    /* 7. Parse PMON environment */
    pmon_init(envp);

    /* 8. Determine memory size */
    /* From PMON memsize variable or hardcoded MEMSIZE option */

    /* 9. Parse boot arguments */
    /* -s (single user), -a (ask root), -d (debugger) */

    /* 10. Build memory cluster list */
    /* mem_clusters[] array describes physical RAM layout */

    /* 11. Bootstrap VM system */
    pmap_bootstrap();

    /* 12. Enable debugger if requested */
    if (boothowto & RB_KDB) {
        #ifdef DDB
            ddb_init();
        #endif
    }
}
```

**LED Display**: Algor boards have 4-character LED displays showing boot progress.

## Platform-Specific Details

### P-4032

**CPU**: MIPS R4000PC (IDT R4640 or equivalent)
**Clock**: 133-150 MHz
**Memory**: Up to 256MB SDRAM
**PCI**: V962PBC PCI bridge (32-bit, 33MHz)
**Ethernet**: DEC 21143 (dc) or equivalent
**Serial**: 16550 UART at COM1/COM2
**IDE**: VIA VT82C586 IDE controller

**Configuration**: `/sys/arch/algor/conf/P4032`

### P-5064

**CPU**: IDT R5000 or QED RM5261
**Clock**: 200-300 MHz
**Memory**: Up to 512MB SDRAM
**PCI**: Bonito64 system controller with integrated PCI
**Ethernet**: AMD PCnet-PCI (le)
**Serial**: 16550 UART
**RTC**: M48T37 NVRAM/RTC

**Configuration**: `/sys/arch/algor/conf/P5064`

**64-bit Support**: P5064-64 configuration for 64-bit kernel (experimental)

### P-6032

**CPU**: R4000/R5000 series
**Clock**: 133-200 MHz
**Memory**: Up to 256MB
**PCI**: Two independent 32-bit PCI buses
**Ethernet**: DEC 21143 or Intel i82559
**Features**: Hot-swap PCI, CompactPCI backplane support

**Configuration**: `/sys/arch/algor/conf/P6032`

## Build Instructions

### Building the Kernel

```sh
# From NetBSD source tree root
cd /usr/src

# Build toolchain (if cross-compiling)
./build.sh -m algor tools

# Build kernel for P-5064
./build.sh -m algor kernel=P5064

# Or manually
cd sys/arch/algor/conf
config P5064
cd ../compile/P5064
make depend
make

# Resulting kernel: netbsd
```

### Kernel Size Considerations

**No size constraints**: Kernel is loaded entirely into RAM by PMON.

**Typical sizes**:
- Minimal kernel: ~2-3 MB
- GENERIC-equivalent: ~5-8 MB
- With modules: ~10+ MB

## Installation

### Installing NetBSD

**Prerequisites**:
- Working PMON firmware
- Network or IDE/SCSI disk access from PMON

**Installation Steps**:

1. **Obtain Installation Media**

```sh
# Download from FTP
ftp ftp.netbsd.org
cd /pub/NetBSD/NetBSD-<version>/algor/
get base.tgz
get etc.tgz
# ... other sets
```

2. **Partition Disk** (from another NetBSD system or use PMON fdisk)

```sh
# NetBSD disklabel
disklabel -e wd0
# Create partitions: a (root), b (swap), e (usr), etc.
```

3. **Create Filesystems**

```sh
newfs /dev/rwd0a        # root
newfs /dev/rwd0e        # usr
```

4. **Extract Sets**

```sh
mount /dev/wd0a /mnt
cd /mnt
tar xzpf /path/to/base.tgz
tar xzpf /path/to/etc.tgz
# ... other sets
```

5. **Configure PMON Autoboot**

```
PMON> set autoboot "load ide0:netbsd; g"
PMON> set bootdelay 5
```

6. **Boot NetBSD**

```
PMON> boot
```

### Network Boot Installation

**Server Setup**:

```sh
# On TFTP server
cp netbsd /tftpboot/netbsd-algor

# On NFS server
export /export/algor/root
```

**PMON Commands**:

```
PMON> ifaddr synopsys0 192.168.1.10
PMON> load tftp://192.168.1.1/netbsd-algor
PMON> g root=/dev/nfs nfsroot=192.168.1.1:/export/algor/root
```

## Boot Configuration

### PMON Configuration Variables

**Set variables**:
```
PMON> set <variable> <value>
```

**Important Variables**:
- `autoboot` - Command to execute on autoboot
- `bootdelay` - Seconds to wait before autoboot
- `netaddr` - IP address for network boot
- `gateway` - Default gateway
- `bootfile` - Default boot file path

**Save Configuration**:
```
PMON> save
```

### Kernel Boot Arguments

**Passed via PMON**:

```
PMON> g -s              # Single-user mode
PMON> g -a              # Ask for root device
PMON> g -d              # Enter debugger
PMON> g -v              # Verbose boot
```

**Combined**:
```
PMON> g -sv             # Single-user, verbose
```

## Debugging

### Serial Console

**All Algor platforms default to serial console**.

**Serial Parameters**:
- **Baud Rate**: 115200 (default) or 38400
- **Data Bits**: 8
- **Parity**: None
- **Stop Bits**: 1
- **Flow Control**: None

**Port**: COM1 (0x3F8)

**Changing Baud Rate** (in kernel config):
```
options CONSPEED=38400
```

### PMON Debugging Commands

**Memory Dump**:
```
PMON> d 0x80000000 100          # Dump 256 bytes
PMON> d -w 0x80000000 100       # Dump words
```

**Disassemble**:
```
PMON> dis 0x80000000 20         # Disassemble 20 instructions
```

**Register Display**:
```
PMON> r                         # Show all registers
```

**Breakpoints**:
```
PMON> b 0x80000000              # Set breakpoint
PMON> c                         # Continue execution
```

### Kernel Debugging (DDB)

**Enable in kernel config**:
```
options     DDB
makeoptions COPY_SYMTAB=1
```

**Enter debugger**:
```
# From kernel: press Control-Alt-Esc
# At boot: g -d
# On panic: automatic
```

**DDB Commands**:
```
db> trace                       # Stack trace
db> show registers              # Show register contents
db> x/x 0x80000000,10          # Examine memory
db> break mach_init             # Set breakpoint
```

### Common Issues

**"No autoboot command"**:
- PMON autoboot variable not set
- Solution: `set autoboot "load ide0:netbsd; g"`

**"Cannot find kernel"**:
- Wrong device name or path
- Solution: Try `load ide0:netbsd` or `load scsi0:netbsd`

**"Illegal instruction"**:
- Kernel compiled for wrong CPU architecture
- P4032 needs MIPS3, not MIPS4
- Check kernel config CPU options

**Serial console garbled**:
- Baud rate mismatch
- Solution: Check CONSPEED in kernel, match PMON baud rate

**Hangs after "NetBSD"**:
- Console device misconfiguration
- Try different PMON console settings
- Check cable connections

## Source Code Reference

### Critical Source Files

**Platform-specific**:
- `/sys/arch/algor/algor/machdep.c` - Main bootstrap and initialization (550 lines)
- `/sys/arch/algor/algor/pmon.c` - PMON interface (103 lines)
- `/sys/arch/algor/algor/autoconf.c` - Device autoconfiguration
- `/sys/arch/algor/algor/algor_p4032_*.c` - P-4032 support
- `/sys/arch/algor/algor/algor_p5064_*.c` - P-5064 support
- `/sys/arch/algor/algor/algor_p6032_*.c` - P-6032 support

**Include files**:
- `/sys/arch/algor/include/pmon.h` - PMON interface definitions
- `/sys/arch/algor/include/autoconf.h` - Autoconfiguration structures
- `/sys/arch/algor/algor/algor_p4032reg.h` - P-4032 register definitions
- `/sys/arch/algor/algor/algor_p5064reg.h` - P-5064 register definitions
- `/sys/arch/algor/algor/algor_p6032reg.h` - P-6032 register definitions

**MIPS-generic**:
- `/sys/arch/mips/mips/mips_machdep.c` - MIPS machine-dependent code
- `/sys/arch/mips/mips/cache.c` - Cache operations
- `/sys/arch/mips/mips/tlb.c` - TLB management
- `/sys/arch/mips/include/cpuregs.h` - CP0 register definitions (1000+ lines)
- `/sys/arch/mips/include/pte.h` - Page table entry structures

### Man Pages

- `boot(8)` - General boot procedures
- `intro(4)` - Device driver introduction
- `pmon(8)` - PMON monitor commands (if available)

### External Documentation

- **MIPS Run-Time Architecture (RTA)** - MIPS calling conventions, exception handling
- **MIPS R4000 Microprocessor User's Manual** - CPU architecture, CP0 registers, TLB
- **MIPS R5000 Microprocessor Technical Backgrounder** - R5000-specific features
- **Algorithmics SDE-MIPS** - Software Development Environment manual
- **PMON User Manual** - Algorithmics PMON firmware documentation

## Notes

- **No bootloader required**: PMON firmware loads kernel directly
- **Little-endian only**: NetBSD/algor does not support big-endian mode
- **Serial console**: All platforms use serial console by default
- **PCI support**: All platforms have PCI bus with various controllers
- **Historical platform**: Algorithmics boards were primarily used for MIPS software development and evaluation
- **Limited production**: These boards had limited deployment outside development environments

---

*Last Updated: 2025-11-12*
*Architecture Maintainer: NetBSD/algor Port*
