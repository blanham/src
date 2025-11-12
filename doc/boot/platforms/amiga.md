# NetBSD/amiga Boot Documentation

## Platform Overview

NetBSD/amiga supports Commodore Amiga computers including the A500, A600, A1200, A2000, A3000, A4000 series, and DraCo systems. The platform uses the Motorola 68k CPU family (68020, 68030, 68040, 68060) with custom chipsets (OCS, ECS, AGA) and various expansion capabilities.

**Supported Models:**
- Amiga 500/600/1000/1200/2000/3000/4000
- DraCo workstations (MacroSystem)
- Various accelerator boards with 68030/040/060 CPUs

**CPU Support:**
- 68020 minimum requirement
- 68030 with PMMU
- 68040 with integrated MMU/FPU
- 68060 with integrated MMU/FPU

## Boot Method

The Amiga platform uses a two-stage boot process that starts from AmigaOS (Kickstart ROM and Workbench):

### Boot Chain

1. **AmigaOS Environment** - System boots into Kickstart ROM → Workbench
2. **First Stage: loadbsd** - AmigaOS executable that loads the kernel
3. **Second Stage: bootxx** - Optional bootblock-based boot from disk
4. **Kernel** - NetBSD kernel loaded into memory and executed

### Boot Loaders

#### loadbsd (Primary Method)

**Location:** `/sys/arch/amiga/stand/loadbsd/`

**Purpose:** AmigaOS program that loads NetBSD kernel from AmigaDOS filesystem

**Features:**
- Runs from AmigaOS/Workbench CLI
- Loads kernel from any AmigaDOS-accessible device
- Supports ELF and a.out kernel formats
- Memory management and configuration detection
- Symbol table loading for debugging

**Usage:**
```
loadbsd [-abhklpstACDSVZ] [-c machine] [-m size] [-M size] [-n mode] [-I sync-inhibit] kernel
```

**Key Options:**
- `-a`: Boot to multiuser mode
- `-b`: Ask for root device
- `-c <machine>`: Force machine type (e.g., 3000, 4000, 32000+N for DraCo rev N)
- `-s`: Boot to single-user mode (default)
- `-S`: Include kernel symbol table
- `-D`: Enter kernel debugger
- `-k`: Reserve first 4MB of fast memory
- `-p`: Use highest priority fastmem segment (default)
- `-l`: Use largest memory segment
- `-n <mode>`: Enable non-contiguous memory (0-3)
- `-C`: Use serial console
- `-A`: Enable AGA modes
- `-I <mask>`: Inhibit SCSI sync negotiation (bit mask)
- `-Z`: Force kernel load to chip memory
- `-M <size>`: Minimum memory segment size in MB (default: 2MB)

**Memory Management:**
- Detects chip memory (always starts at 0x0)
- Finds fastest/largest fast memory segment
- Handles non-contiguous memory configurations
- Supports memory priority-based selection
- DraCo MMU table handling

#### bootxx (Bootblock Method)

**Location:** `/sys/arch/amiga/stand/bootblock/`

**Purpose:** Firmware-level bootblock installed in disk boot sector

**Components:**
- `bootxx_ffs` - FFS (Fast File System) bootblock
- `bootxx_ffsv2` - FFSv2 bootblock
- `boot` - Second-stage loader loaded by bootxx

**Boot Sequence:**
1. Amiga Kickstart ROM
2. RDB (Rigid Disk Block) bootblock (bootxx)
3. Secondary boot loader (boot)
4. NetBSD kernel

**Features:**
- Boots directly from NetBSD FFS partition
- No AmigaOS required after installation
- RDB-based partition support
- Limited to 7KB first stage + 50KB second stage

## 68030/68040/68060 MMU Setup Differences

### 68020 + 68851 PMMU
- External 68851 PMMU required
- Two-level page tables
- 256-byte page table entries
- TC register for translation control
- CRP/SRP registers for root pointers

### 68030 MMU
- Integrated PMMU compatible with 68851
- Three-level page tables (root, pointer, page)
- Translation Control (TC) register
- Root Pointer registers (CRP/SRP for CPU/Supervisor)
- Transparent Translation registers (TT0/TT1)
- Function codes (FC0-FC7) for address spaces

**Setup in locore.s:**
```assembly
lea     _C_LABEL(Sysseg_pa),%a0
movl    %a0@,%d0                | get RP
pmove   %d0,%crp                | load CRP
pflusha                         | flush entire TLB
pmove   %a1@,%tc                | load TC
```

### 68040 MMU
- Redesigned MMU architecture
- Four-level page tables
- Address Translation Cache (ATC)
- Separate instruction and data TLBs
- Different register names (SRP replaces CRP)
- TTR0/TTR1 (Transparent Translation Registers)
- URP/SRP for user/supervisor root pointers

**Setup in locore.s:**
```assembly
.long   0x4e7b1807              | movc d1,srp
.word   0xf518                  | pflusha
movl    #MMU40_TCR_BITS,%d0
.long   0x4e7b0003              | movc d0,tc
```

**040 Differences:**
- Simplified page descriptors
- No need for FC (Function Code)
- Integrated FPU
- Branch cache must be cleared on TLB misses
- Different cache control

### 68060 MMU
- Similar to 68040 but enhanced
- Larger branch cache
- Superscalar execution affects MMU
- PCR (Processor Control Register) for features
- Enhanced cache control
- Branch prediction errors require special handling

**060-Specific Setup:**
```assembly
movl    #1,%d0
.long   0x4e7b0808              | movc d0,pcr
movl    #0xa0808000,%d0
.long   0x4e7b0004              | movc d0,itt0
.long   0x4e7b0005              | movc d0,dtt0
```

**060 Gotchas:**
- Branch prediction errors (handled in buserr60)
- Requires 68060-specific software support package (060SP)
- Some instructions emulated in software
- Misaligned access handling
- FSLW (Fault Status Long Word) format differs

## Boot Process Stages

### Stage 1: AmigaOS Execution

**Location:** AmigaOS filesystem (e.g., Work:netbsd)

**Process:**
1. User runs `loadbsd` from CLI/Shell
2. Exec.library version check (requires V36+)
3. Parse command line options
4. Detect CPU type via AttnFlags and FindResident()
5. Query Expansion.library for hardware configuration
6. Build memory list from Exec MemList

**CPU Detection:**
```c
cpuid |= SysBase->AttnFlags;    // AFB_68020, AFB_68030, etc.
if (FindResident("A4000 Bonus"))
    cpuid |= 4000 << 16;
else if (FindResident("A3000 Bonus"))
    cpuid |= 3000 << 16;
else if (OpenResource("card.resource")) {
    // A600 or A1200 based on Alice revision
}
```

**Memory Detection:**
```c
// Walk Exec memory list
for (mh = SysBase->MemLst; mh->next; mh = mh->next) {
    // CachePreDMA() to get physical address
    // Separate chip memory (MEMF_CHIP) from fast (MEMF_FAST)
    // Handle DraCo MMU kludges
    // Find best memory segment by size or priority
}
```

### Stage 2: Kernel Loading

**Location:** loadbsd code at `/sys/arch/amiga/stand/loadbsd/loadbsd.c`

**Process:**
1. Parse kernel path and options
2. Call `loadfile()` twice:
   - First with COUNT flags to determine size
   - Allocate memory (chip or fast depending on kernel version)
   - Second with LOAD flags to actually load kernel
3. Verify kernel startup interface version (in kernel at entry-2)
4. Append boot information:
   - ConfigDev list (Zorro expansion boards)
   - Memory segment list
   - Boot flags and parameters
5. Copy startup code to end of kernel image
6. Disable interrupts and caches
7. Jump to startup code

**Version Checking:**
```c
kvers = *(u_short *)(marks[MARK_ENTRY] - 2);
if (kvers > KERNEL_STARTUP_VERSION_MAX)
    err(20, "newer loadbsd required: %d\n", kvers);
if (kvers < KERNEL_STARTUP_VERSION)
    err(20, "kernel too old for bootblock\n");
```

**Kernel Startup Interface Versions:**
- Version 1: First loadbsd release
- Version 2: Pass esym in a4
- Version 3: Load kernel into fast memory
- Version 9: Max supported version

### Stage 3: Startup Code Execution

**Location:** Copy of startit() function

**Executed with:**
- Interrupts disabled
- Caches off
- Running in physical memory
- AmigaOS still in ROM but unused

**Arguments passed:**
```c
startit(
    kernel_ptr,          // d0: kernel base address
    kernel_size,         // d1: kernel size
    entry_offset,        // d2: entry point offset
    fastmem_ptr,         // a0: fast memory base
    fastmem_size,        // d3: fast memory size
    chipmem_size,        // d4: chip memory size
    boothowto,           // d5: boot flags (RB_SINGLE, etc.)
    esym,                // a4: end of symbol table
    cpuid,               // d6: CPU/machine ID
    eclock_freq,         // d7: E-clock frequency
    amiga_flags,         // stack: AGA, console, memory flags
    sync_inhibit,        // stack: SCSI sync inhibit mask
    aio_base,            // stack: AIO base (tape boot)
    load_to_fastmem      // stack: 1 if loaded to fast mem
);
```

**Actions:**
1. Copy kernel to fast memory if needed
2. Clean caches with CacheClearU()
3. Disable all interrupts
4. Jump to kernel entry point

### Stage 4: Kernel Entry (locore.s)

**Location:** `/sys/arch/amiga/amiga/locore.s`

**Entry point:** `start` or offset 0x400 in kernel image

**Initial State:**
- %sr = 0x2700 (interrupts disabled, supervisor mode)
- Registers contain boot parameters
- MMU disabled
- Caches disabled
- Still in physical memory

**Process:**
```assembly
L_base:
    .long   0x4ef80400+PAGE_SIZE    // jmp 0x400
    .fill   PAGE_SIZE/4-1,4,0        // padding

start:
    movw    #PSL_HIGHIPL,%sr        // ensure interrupts off

    // Detect CPU type
    // - Test for 68030 data freeze bit
    // - Test for 68040 data cache enable
    // - Default to 68020+68851

    // Setup MMU based on CPU type
    // 68020/030: Use PMMU instructions
    // 68040/060: Use different opcodes

    // Enable MMU with VA==PA mapping
    // Setup kernel page tables
    // Turn on caches

    // Setup temporary stack
    lea     _ASM_LABEL(tmpstk),%sp

    // Call C function to complete bootstrap
    jsr     _C_LABEL(start_c)
```

**CPU-Specific MMU Initialization:**

**68030:**
```assembly
Lstart3:
    lea     _C_LABEL(protorp),%a0
    pmove   %a0@,%crp              // load root pointer
    lea     _C_LABEL(protocrp),%a0
    pmove   %a0@,%tc               // load translation control
```

**68040:**
```assembly
Lstart4:
    .long   0x4e7b1807             // movc d1,srp
    .word   0xf518                 // pflusha
    movl    #MMU40_TCR_BITS,%d0
    .long   0x4e7b0003             // movc d0,tc
```

**68060:**
```assembly
Lstart6:
    movl    #1,%d0
    .long   0x4e7b0808             // movc d0,pcr (enable superscalar)
    // Setup transparent translation
    .long   0x4e7b0004             // movc d0,itt0
    .long   0x4e7b0005             // movc d0,dtt0
```

### Stage 5: Machine-Dependent Initialization

**Location:** `/sys/arch/amiga/amiga/machdep.c`

**Function:** `start_c()` and `cpu_startup()`

**Actions:**
1. Parse boot arguments from registers
2. Setup configuration device list
3. Initialize memory management structures
4. Detect and configure custom chips (Paula, Denise, Alice, Lisa)
5. Setup CIA timers
6. Configure interrupt system
7. Mount root filesystem
8. Start init

**Configuration Device Handling:**
```c
// ConfigDev structures passed from loadbsd
ncd = *(int *)endkernel;
cd = (struct cfdev *)(endkernel + 4);
for (i = 0; i < ncd; i++) {
    // Process each Zorro device
    // Adjust addresses for DraCo Z2 mapping
}
```

## Memory Map

### Standard Amiga Memory Layout

```
Physical Address Space:
0x00000000 - 0x001FFFFF    Chip Memory (up to 2MB)
  0x00000000 - 0x000003FF    Exception Vectors
  0x00000400 - 0x0007FFFF    Chip RAM
  0x00080000 - 0x001FFFFF    Chip RAM (ECS/AGA)

0x00200000 - 0x009FFFFF    Ranger/Spare/SlowMem (optional)

0x00A00000 - 0x00BFFFFF    CIA/RTC/Custom chips
  0x00BFD000               CIA-A
  0x00BFE001               CIA-B
  0x00DFF000 - 0x00DFF1FF  Custom chip registers

0x00C00000 - 0x00DFFFFF    Internal expansion (generally reserved)

0x00E00000 - 0x00E7FFFF    Autoconfig region (Zorro-II)
0x00E80000 - 0x00EFFFFF    More autoconfig

0x00F00000 - 0x00F7FFFF    Reserved (some accelerators)
0x00F80000 - 0x00FFFFFF    ROM (Kickstart, generally 512KB)

0x01000000 - 0x01FFFFFF    Zorro-II RAM/ROM (8MB)

0x02000000 - 0x09FFFFFF    Reserved

0x10000000 - 0xFFFFFFFF    Zorro-III space (A3000/A4000)
  0x10000000 - 0x7FFFFFFF  Zorro-III RAM
  0xFF000000 - 0xFFFFFFFF  Zorro-III ROM/IO
```

### Kernel Virtual Address Layout

```
0x00000000 - 0x000007FF    Unmapped (trap NULL pointers)
0x00000800 - 0x001FFFFF    Kernel text/data/bss
0x00200000 - ...           Kernel heap/malloc
...                        Process page tables
...                        User process space
0xFFxxxxxx                 I/O device mapping
```

### DraCo Memory Layout

DraCo systems have a different memory map due to the built-in MMU:

```
0x04000000 - 0x05FFFFFF    Main RAM (8MB or 32MB)
  0x04000000 - 0x041FFFFF  MMU translation table reserved
  0x04200000 - ...         Usable RAM

0x10000000 - 0x1FFFFFFF    Optional RAM expansion

// Zorro-II space mapped differently
0x30000000 - 0x30FFFFFF    Zorro-II virtual mapping
```

**DraCo Boot Parameters:**
- cpuid = 0x7Dxxxxxx (126000 + revision)
- Memory margin: 2MB reserved at 0x04000000
- Z2 offset: 0x03000000 added to Zorro-II addresses

## Build and Installation

### Building the Bootloader

```bash
cd /usr/src/sys/arch/amiga/stand
make depend
make
make install DESTDIR=/targetpath
```

**Build products:**
- `loadbsd` - AmigaOS executable
- `bootxx_ffs` - FFS bootblock
- `bootxx_ffsv2` - FFSv2 bootblock
- `installboot` - Tool to install bootblock

### Building the Kernel

```bash
cd /usr/src/sys/arch/amiga/conf
config GENERIC
cd ../compile/GENERIC
make depend
make
```

**Kernel types:**
- `netbsd` - Standard ELF kernel
- `netbsd.gz` - Compressed kernel (loadbsd can uncompress)

### Installation Methods

#### Method 1: Using loadbsd (Recommended for New Installations)

1. Boot into AmigaOS
2. Copy `loadbsd` to AmigaDOS filesystem
3. Copy NetBSD kernel (e.g., to `Work:netbsd`)
4. Create a startup script:

```
.key kernel/a
.bra {
.ket }
loadbsd -b -D <kernel>
```

5. Boot: `loadbsd netbsd -b`

#### Method 2: Bootblock Installation (Advanced)

1. Install NetBSD to NetBSD FFS partition
2. Use `installboot` to write bootblock to RDB:

```bash
cd /usr/mdec
installboot /dev/rsd0a bootxx_ffs
```

3. Configure RDB to boot from NetBSD partition
4. Reboot and select NetBSD partition from Kickstart boot menu

**RDB Configuration:**
- Partition must be marked as bootable
- Priority determines boot order
- DosType must match filesystem type

### Common Boot Options

**Development/Debugging:**
```
loadbsd -D -S netbsd      # Enter debugger with symbols
```

**Single-user with serial console:**
```
loadbsd -s -C netbsd      # Single-user, serial console
```

**Specific machine type:**
```
loadbsd -c 4000 netbsd    # Force A4000 detection
```

**Non-contiguous memory:**
```
loadbsd -n 2 netbsd       # Use all memory segments
```

## Debugging

### Serial Console Setup

**Hardware:**
- Use Amiga serial port (DB-25)
- Connect to null-modem cable
- 9600 8N1 default

**Enable in loadbsd:**
```
loadbsd -C netbsd
```

**Or set amiga_flags in bootloader:**
```c
amiga_flags |= (1 << 3);  // Serial console bit
```

### Kernel Debugger (DDB)

**Enable at boot:**
```
loadbsd -D netbsd
```

**Break into debugger:**
- Press Ctrl-Alt-Alt simultaneously
- Or trigger panic/assertion

**Useful DDB commands:**
```
db> ps          # Process list
db> bt          # Backtrace
db> show map    # Virtual memory map
db> show page   # Page table
db> x/i addr    # Disassemble
db> show reg    # Registers
```

### Remote Debugging (KGDB)

**Setup:**
1. Build kernel with `options KGDB`
2. Configure serial line in kernel config
3. Connect serial cable to debug host

**Debug session:**
```bash
# On NetBSD/amiga
boot netbsd -d

# On debug host
kgdb /path/to/netbsd
target remote /dev/tty00
```

### Common Boot Problems

#### "Exec V36 required"
- **Cause:** Kickstart ROM too old
- **Solution:** Upgrade to Kickstart 2.0 (V36) or later

#### "CPU not supported"
- **Cause:** 68000 or 68010 CPU
- **Solution:** Upgrade to 68020 or better

#### "kernel too old for bootblock"
- **Cause:** Kernel interface version mismatch
- **Solution:** Update kernel or loadbsd to matching versions

#### "failed alloc memory"
- **Cause:** Insufficient memory or fragmented memory
- **Solution:**
  - Free up memory in AmigaOS
  - Use `-M` to lower minimum memory requirement
  - Defragment memory
  - Try `-p` or `-l` options

#### "newer loadbsd required"
- **Cause:** Kernel too new for loadbsd
- **Solution:** Update loadbsd from newer NetBSD release

#### System hangs after "Loading kernel"
- **Cause:** Bad memory, incompatible expansion card, cache issues
- **Solution:**
  - Try `-k` to reserve first 4MB
  - Try `-Z` to force chip memory load
  - Disable CPU cache/accelerator
  - Remove expansion cards one by one

### Memory Debugging

**Check memory configuration:**
```
loadbsd -t netbsd     # Test mode, show memory
```

**Force specific memory size:**
```
loadbsd -m 4096 netbsd    # Limit to 4MB fast memory
```

**Memory segment priority:**
```
loadbsd -p netbsd     # Use highest priority segment
loadbsd -l netbsd     # Use largest segment
```

### Bootblock Debugging

**Check RDB:**
```bash
amiga# disklabel sd0
```

**Verify bootblock installation:**
```bash
amiga# dd if=/dev/rsd0a bs=512 count=1 | od -x
```

**Look for:**
- Boot signature: "DOS\0" or "DOS\1"
- Checksum validity
- Load segment pointers

### Hardware-Specific Issues

**DraCo Systems:**
- Must set cpuid: `loadbsd -c 32000`
- Memory mapping differs
- Z2 space offset required

**A1200/A600 (AGA machines):**
- May need `-A` flag for AGA modes
- PCMCIA issues common
- Be careful with gayle/alchemy revisions

**A3000/A4000:**
- SCSI sync negotiation issues: use `-I 0xFF`
- NCR 53C710/720 controller quirks
- Bridgeboard conflicts

**Accelerator Boards:**
- Some boards need `-k` flag
- Cache disable may be needed
- Check for Kickstart conflicts
- MMU setup may vary

## References

**Source Files:**
- `/sys/arch/amiga/stand/loadbsd/loadbsd.c` - Main bootloader
- `/sys/arch/amiga/stand/bootblock/` - Bootblock code
- `/sys/arch/amiga/amiga/locore.s` - Kernel entry
- `/sys/arch/amiga/amiga/machdep.c` - Machine-dependent init
- `/sys/arch/amiga/include/cpu.h` - CPU definitions
- `/sys/arch/amiga/amiga/pmap.c` - MMU management

**Documentation:**
- Amiga Hardware Reference Manual
- Commodore A3000/A4000 Technical Reference
- Motorola 68030/68040/68060 User's Manuals
- RKM: Libraries (Exec, Expansion)

**Hardware Specifications:**
- Zorro-II: 8-bit AutoConfig protocol, 16MB address space
- Zorro-III: 32-bit, up to 4GB address space
- RDB (Rigid Disk Block): Amiga partition table format
- Kickstart: ROM-based OS (512KB-1MB)

## Appendix: Boot Flags

**RB_* flags (in sys/reboot.h):**
```c
#define RB_AUTOBOOT    0x00000000  // Auto boot (multi-user)
#define RB_ASKNAME     0x00000001  // Ask for root device
#define RB_SINGLE      0x00000002  // Boot to single-user
#define RB_NOSYNC      0x00000004  // Don't sync disks
#define RB_HALT        0x00000008  // Halt, don't reboot
#define RB_INITNAME    0x00000010  // Init path name
#define RB_DFLTROOT    0x00000020  // Use compiled-in root
#define RB_KDB         0x00000040  // Enter kernel debugger
#define RB_RDONLY      0x00000080  // Mount root read-only
#define RB_DUMP        0x00000100  // Dump kernel memory
#define RB_MINIROOT    0x00000200  // Use miniroot
#define RB_STRING      0x00000400  // Use boot string
```

**Amiga-specific flags:**
```c
#define AB_QUIET       0x00010000  // Quiet boot
#define AB_VERBOSE     0x00020000  // Verbose boot
#define AB_COMPAT      0x00040000  // Compatibility mode
#define AB_AGA         0x00000001  // AGA chipset enabled
#define AB_SERIALCONS  0x00000008  // Serial console
```

**Memory flags (-n option):**
```c
0: Disabled - use only one memory segment
1: Two segments - split into 2 regions
2: All available - use all segments
3: All available - same as 2
```
