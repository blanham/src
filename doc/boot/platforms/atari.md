# NetBSD/atari Boot Documentation

## Platform Overview

NetBSD/atari supports Atari TT030, Falcon030, and Hades computer systems. These machines use Motorola 68030/68040/68060 CPUs and boot from Atari TOS (The Operating System), the native OS stored in ROM.

**Supported Models:**
- Atari TT030 - 68030 at 32 MHz with TT-RAM
- Atari Falcon030 - 68030 at 16 MHz
- Hades - Clone with 68040/68060 option
- Milan - Clone with 68040/68060

**CPU Support:**
- 68030 with integrated MMU (primary)
- 68040 with integrated MMU/FPU (Hades)
- 68060 with integrated MMU/FPU (Hades/Milan)

**Memory Types:**
- ST-RAM: Up to 14MB, limited by custom chips
- TT-RAM (FastRAM): Up to 512MB, 32-bit fast memory (TT030/Hades)

## Boot Method

The Atari platform uses TOS (The Operating System) as the boot environment, similar to how Amiga uses AmigaOS. NetBSD provides both TOS-based and native bootblock-based boot methods.

### Boot Chain

**Method 1: TOS Boot (Recommended for Installation)**

1. **TOS ROM** - System boots into TOS from ROM
2. **loadbsd.tos** - TOS program that loads kernel
3. **NetBSD kernel** - Kernel takes over system

**Method 2: Bootblock Boot (Native)**

1. **TOS ROM**
2. **Bootblock (bootxx)** - Installed in disk boot sector
3. **Secondary boot (boot)** - Loaded by bootxx from filesystem
4. **NetBSD kernel** - Final stage

### Boot Loaders

#### loadbsd.tos (Primary Method)

**Location:** `/sys/arch/atari/stand/tostools/loadbsd/`

**Purpose:** TOS program (PRG file) that loads NetBSD kernel from any TOS-accessible device

**Features:**
- Runs from TOS desktop or AUTO folder
- Supports ELF and a.out kernel formats
- Handles ST-RAM and TT-RAM configuration
- Symbol table loading for debugging
- Compressed kernel support

**Usage:**
```
loadbsd.tos [-abdDhNstVw] [-o output] [-S stsize] [-T ttsize] [kernel]
```

**Options:**
- `-a`: Boot to multiuser (auto-boot)
- `-b`: Ask for root device name
- `-d`: Enter kernel debugger (RB_KDB)
- `-D`: Debug mode (show diagnostics)
- `-h`: Display help information
- `-N`: Don't load symbol table
- `-s`: Use ST-RAM only (avoid TT-RAM)
- `-S <size>`: Override ST-RAM size detection
- `-T <size>`: Override TT-RAM size detection
- `-t`: Test mode (load but don't execute)
- `-V`: Show version information
- `-w`: Wait for keypress before proceeding
- `-o <file>`: Redirect output to file

**Default kernel:** `n:/netbsd` (drive N:, file netbsd)

**Memory Handling:**
- Detects ST-RAM (up to 14MB)
- Detects TT-RAM location and size
- Can force ST-RAM only with `-s` flag
- Override detection with `-S` and `-T`

#### bootxx/boot (Bootblock Method)

**Location:** `/sys/arch/atari/stand/bootxx/` and `/sys/arch/atari/stand/bootxxx/`

**Components:**
- `bootxx` - Primary bootblock (512 bytes)
- `boot` (aka bootxxx) - Secondary loader (up to 60KB)

**Boot Sequence:**
1. TOS ROM reads AHDI boot sector
2. bootxx (first stage) executes
3. bootxx loads boot (second stage) from filesystem
4. boot loads NetBSD kernel
5. Kernel executes

**Features:**
- Native NetBSD FFS support
- Interactive boot prompt
- Kernel selection
- Boot flags
- No TOS required after installation

**Usage at Boot Prompt:**
```
NetBSD/atari secondary bootloader ($Revision$)

Enter os-type [.netbsd] root-fs [:a] kernel [/netbsd] options [none]:
```

**Examples:**
```
/netbsd -s          # Boot single-user
/netbsd.old         # Boot alternate kernel
:b -a               # Auto-boot from partition b
-d                  # Enter debugger
```

### TOS Tools

**Location:** `/sys/arch/atari/stand/tostools/`

Additional utilities for Atari systems:

- **loadbsd** - Main boot loader
- **aptck** - Atari partition table checker
- **chg_pid** - Change partition ID
- **file2swp** - Convert file to swap partition
- **rawwrite** - Low-level disk writer
- **libtos** - TOS library for NetBSD programs

## 68030/68040/68060 MMU Setup Differences

### 68030 MMU (TT030, Falcon030)

**Characteristics:**
- Integrated PMMU (Paged Memory Management Unit)
- Three-level page tables
- Translation Control (TC) register
- Transparent Translation registers (TT0/TT1)
- CPU Root Pointer (CRP) and Supervisor Root Pointer (SRP)

**TT030 Memory Layout:**
- ST-RAM: 0x00000000 - 0x00e00000 (up to 14MB)
- TT-RAM: 0x01000000 - 0x80000000 (up to 512MB)
- I/O space: 0xff000000 - 0xffffffff

**MMU Setup in locore.s:**
```assembly
| 68030 initialization
lea     _C_LABEL(Sysseg_pa),%a0 | get physical address of page directory
movl    %a0@,%d0                | load it
pmove   %d0,%crp                | set CPU root pointer
pflusha                         | flush TLB
movl    _C_LABEL(protott),%d0   | get transparent translation
pmove   %d0,%tt0                | set TT0 register
lea     _C_LABEL(protocrp),%a0
pmove   %a0@,%tc                | enable MMU
```

**Transparent Translation:**
- TT0 used for I/O space (0xff000000-0xffffffff)
- TT1 can be used for direct-mapped regions
- Allows bypassing translation for specific ranges

### 68040 MMU (Hades 040)

**Characteristics:**
- Different architecture from 68030
- Simplified page table format
- Four-level page tables
- Address Translation Cache (ATC)
- URP/SRP instead of CRP
- TTR0/TTR1 instead of TT0/TT1

**040 Differences:**
- No PMOVE instruction (use MOVEC)
- Different register encodings
- Separate instruction and data TLBs
- Integrated FPU
- Push/invalidate cache operations (CINVA, CPUSHA)

**MMU Setup:**
```assembly
| 68040 initialization
.long   0x4e7b1807              | movc d1,srp (load root pointer)
movl    #0x80f04445,%d0         | TTR value for I/O
.long   0x4e7b0004              | movc d0,itt0
.long   0x4e7b0005              | movc d0,dtt0
.word   0xf518                  | pflusha (flush all TLB entries)
movl    #MMU40_TCR_BITS,%d0     | TC value: enable, 4K pages
.long   0x4e7b0003              | movc d0,tc
```

**040 Page Table Entry Format:**
- Simplified compared to 030
- Different status bits
- Write-back and write-through cache modes

### 68060 MMU (Hades 060, Milan)

**Characteristics:**
- Similar to 68040 but with enhancements
- Superscalar execution
- Larger branch cache
- Enhanced cache control
- Processor Control Register (PCR)

**060-Specific Features:**
- Branch prediction (requires special error handling)
- Instruction/Data cache independent control
- Misaligned access emulation
- FSLW (Fault Status Long Word) format

**060 Initialization:**
```assembly
| 68060-specific setup
movl    #1,%d0
.long   0x4e7b0808              | movc d0,pcr (enable superscalar)
movl    #0x80f04445,%d0
.long   0x4e7b0004              | movc d0,itt0
.long   0x4e7b0005              | movc d0,dtt0
.word   0xf4f8                  | cpusha bc (push and invalidate)
.word   0xf518                  | pflusha
movl    #MMU40_TCR_BITS,%d0
.long   0x4e7b0003              | movc d0,tc
```

**060 Error Handling:**
```assembly
| Branch prediction error handling (buserr60)
movel   %sp@(FR_HW+12),%d0      | FSLW
btst    #2,%d0                  | branch prediction error?
jeq     Lnobpe
movc    %cacr,%d2
orl     #IC60_CABC,%d2          | clear all branch cache entries
movc    %d2,%cacr
```

**060 Special Considerations:**
- Requires 68060 Software Package (060SP) for:
  - Integer divide emulation
  - Misaligned access support
  - FPU instruction emulation
- Branch cache coherency management
- Different cache line size (16 bytes vs 040's 16 bytes)

## Boot Process Stages

### Stage 1: TOS Execution

**Environment:** Running in TOS desktop or AUTO folder

**Process:**
1. User runs `loadbsd.tos` from TOS
2. Program calls `Super(0)` to enter supervisor mode
3. Query system information:
   - CPU type via cookies (_CPU, _FPU, _VDO)
   - Memory configuration (phystop, ramtop)
   - ST-RAM and TT-RAM sizes/locations
4. Parse command line arguments
5. Open and validate kernel file

**System Detection:**
```c
// Check CPU cookie
get_cookie("_CPU", &cpu_type);
// 0 = 68000, 30 = 68030, 40 = 68040, 60 = 68060

// Check FPU cookie
get_cookie("_FPU", &fpu_type);

// Check machine type cookie
get_cookie("_MCH", &machine);
// 0x00010000 = ST, 0x00010010 = MegaST
// 0x00020000 = TT, 0x00030000 = Falcon
```

**Memory Detection:**
```c
// ST-RAM: Always at 0x00000000
stmem_size = *((long *)0x42e);  // phystop

// TT-RAM: Typically at 0x01000000
if (ramtop = *((long *)0x5a4))
    ttmem_start = 0x01000000;
    ttmem_size = ramtop - ttmem_start;
```

### Stage 2: Kernel Loading

**Location:** loadbsd.tos code

**Process:**
1. Read kernel ELF or a.out header
2. Allocate memory for kernel:
   - Prefer TT-RAM for speed (unless `-s` flag)
   - Fall back to ST-RAM if needed
3. Load kernel sections:
   - Text segment
   - Data segment
   - BSS (zero-filled)
   - Symbol table (if requested)
4. Build boot parameter structure
5. Prepare for handoff

**ELF Loading:**
```c
// Read ELF header
read(fd, &ehdr, sizeof(ehdr));

// Validate ELF magic
if (ehdr.e_ident[EI_MAG0] != ELFMAG0 || ...)
    return -1;

// Load program headers
for (i = 0; i < ehdr.e_phnum; i++) {
    // Load each loadable segment
    if (phdr[i].p_type == PT_LOAD) {
        lseek(fd, phdr[i].p_offset, SEEK_SET);
        read(fd, dest, phdr[i].p_filesz);
    }
}
```

**Boot Parameters (osdsc_t structure):**
```c
struct osdsc {
    u_long  stmem_size;     // ST-RAM size
    u_long  ttmem_size;     // TT-RAM size
    u_long  ttmem_start;    // TT-RAM start address
    u_long  cputype;        // CPU type flags
    u_long  boothowto;      // Boot flags (RB_SINGLE, etc.)
    u_long  kstart;         // Kernel load address
    u_long  ksize;          // Kernel size
    u_long  kentry;         // Kernel entry point
    u_long  k_esym;         // End of symbol table
};
```

### Stage 3: Kernel Handoff

**Preparation:**
1. Disable interrupts (`movw #0x2700,%sr`)
2. Turn off caches
3. Set up registers with boot parameters
4. Jump to kernel entry point

**Register Convention:**
```c
d0 = boothowto flags
d1 = ttmem_size (TT-RAM size)
d2 = stmem_size (ST-RAM size)
a0 = ttmem_start (TT-RAM address)
a1 = esym (end of symbols)
```

**Assembly Handoff:**
```assembly
start_kernel:
    move.w  #0x2700,sr          | disable interrupts
    move.l  od_boothowto,d0     | boot flags
    move.l  od_ttmem_size,d1    | TT-RAM size
    move.l  od_stmem_size,d2    | ST-RAM size
    move.l  od_ttmem_start,a0   | TT-RAM address
    move.l  od_k_esym,a1        | end of symbols
    move.l  od_kentry,a2        | kernel entry
    jmp     (a2)                | jump to kernel
```

### Stage 4: Kernel Entry (locore.s)

**Location:** `/sys/arch/atari/atari/locore.s`

**Entry Point:** `start` at offset 0x400 (skip page zero)

**Initial State:**
- Supervisor mode with interrupts disabled
- Registers contain boot parameters
- MMU disabled
- Caches disabled
- Running at physical addresses

**Early Initialization:**
```assembly
    .text
    .globl  kernel_text
kernel_text:
    .fill   PAGE_SIZE/4,4,0     | Skip page zero (unmapped)

vectors:
    .include "vectors.s"         | Exception vectors

start:
    movw    #PSL_HIGHIPL,%sr    | Ensure interrupts off

    | Save boot parameters from registers
    movl    %d0,_C_LABEL(boothowto)
    movl    %d1,_C_LABEL(ttmem_size)
    movl    %d2,_C_LABEL(stmem_size)
    movl    %a0,_C_LABEL(ttmem_start)
    movl    %a1,_C_LABEL(esym)

    | Determine CPU type
    | Test for 68030/040/060 as in Amiga

    | Setup temporary stack
    lea     _ASM_LABEL(tmpstk),%sp

    | Initialize MMU for CPU type
    bsr     _C_LABEL(mmu_init)

    | Jump to C code
    jbsr    _C_LABEL(start_c)
```

**CPU Detection:**
```assembly
| Detect CPU type at runtime
| (Similar to Amiga but may use different methods)

    movl    #CACHE_OFF,%d0
    movc    %d0,%cacr           | Try to set cache control
    movc    %cacr,%d1           | Read it back

    | Test for 68030 data freeze bit
    movl    #0x200,%d0          | data freeze bit
    movc    %d0,%cacr
    movc    %cacr,%d0
    tstl    %d0
    jne     Lis030              | Bit stuck = 68030

    | Test for 68040 data cache enable
    movl    #0x80000000,%d0
    movc    %d0,%cacr
    movc    %cacr,%d0
    tstl    %d0
    jne     Lis040              | Bit stuck = 68040

    | Must be 68020
    jra     Lis020
```

### Stage 5: Machine-Dependent Init

**Location:** `/sys/arch/atari/atari/machdep.c`

**Function:** `start_c()` and `cpu_startup()`

**Actions:**
1. Setup bootinfo structure from registers
2. Initialize memory lists (ST-RAM and TT-RAM)
3. Setup page tables
4. Configure interrupt vectors
5. Initialize custom chips:
   - MFP (Multi-Function Peripheral)
   - PSG (Programmable Sound Generator)
   - ACIA (Keyboard/MIDI)
   - ST DMA (Floppy/ACSI)
   - SCSI controllers (TT/Hades)
6. Probe and configure devices
7. Mount root filesystem
8. Start init

**Memory Setup:**
```c
// Setup memory regions
avail_start = round_page(esym);
avail_end = stmem_size;

// Add TT-RAM if present
if (ttmem_size > 0) {
    vm_page_physload(
        atop(ttmem_start),
        atop(ttmem_start + ttmem_size),
        atop(ttmem_start),
        atop(ttmem_start + ttmem_size),
        VM_FREELIST_DEFAULT);
}
```

## Memory Map

### Atari TT030/Falcon Memory Layout

```
Physical Address Space:

0x00000000 - 0x00000007    System Vectors (must be RAM)
0x00000008 - 0x000003FF    Exception Vectors
0x00000400 - 0x00E00000    ST-RAM (up to 14MB)
  0x00000400 - 0x000007FF    Kernel base (page 0 skipped)
  0x00000800 - ...           Kernel text/data/bss
  ...                        ST-RAM user space

0x00E00000 - 0x00E40000    Video RAM (256KB)
0x00E80000 - 0x00EFFFFF    I/O space
  0x00FA0000 - 0x00FBFFFF    Cartridge ROM
0x00FC0000 - 0x00FEFFFF    TOS ROM (192KB)
0x00FF0000 - 0x00FFFFFF    I/O Registers
  0x00FF8000 - 0x00FF8FFF    PSG, DMA, MFP, etc.
  0x00FF8A00 - 0x00FF8A3F    Blitter
  0x00FFFA00 - 0x00FFFA3F    MFP (Multi-Function Peripheral)
  0x00FFFC00 - 0x00FFFC03    ACIA

0x01000000 - 0x80000000    TT-RAM (Fast 32-bit RAM)
  Typical: 0x01000000 - 0x08000000 (up to 128MB)

VME Bus (TT030):
0x0x00000000 - 0xFEFFFFFF    VME address space
```

### Falcon030-Specific

```
0x00FF8000 - 0x00FF9FFF    ST hardware registers
0x00FFA000 - 0x00FFBFFF    Falcon-specific hardware
  0x00FFA200 - 0x00FFA206    Videl (Video controller)
  0x00FFAx00                 DSP56001 registers
```

### Hades Memory Map

```
0x00000000 - 0x00E00000    ST-RAM (up to 14MB)
0x00FF0000 - 0x00FFFFFF    I/O space
0x01000000 - 0x40000000    TT-RAM (up to ~1GB)
0xFFF00000 - 0xFFFFFFFF    PCI I/O space (Hades)
```

### Kernel Virtual Address Layout

```
0x00000000 - 0x000007FF    Unmapped (trap NULL pointers)
0x00000800 - ...           Kernel text/data/bss
...                        Kernel heap
...                        Page tables
0x80000000 - ...           User process space (if VA != PA)
0xFF000000 - 0xFFFFFFFF    I/O device mappings
```

## Build and Installation

### Building the Bootloader

```bash
# Build TOS tools
cd /usr/src/sys/arch/atari/stand/tostools
make depend
make
make install

# Build bootblocks
cd /usr/src/sys/arch/atari/stand/bootxx
make depend
make
```

**Build Products:**
- `loadbsd.tos` - TOS executable bootloader
- `bootxx` - Primary bootblock
- `boot` (bootxxx) - Secondary bootloader
- `installboot` - Bootblock installation tool

### Building the Kernel

```bash
cd /usr/src/sys/arch/atari/conf
config ATARITT      # For TT030
# or
config FALCON       # For Falcon030
# or
config HADES        # For Hades

cd ../compile/ATARITT
make depend
make
```

### Installation Methods

#### Method 1: TOS Boot (Recommended)

**Setup:**
1. Boot Atari into TOS
2. Copy `loadbsd.tos` to TOS filesystem (e.g., C:\)
3. Copy NetBSD kernel (e.g., C:\NETBSD)
4. Create AUTO\BOOT.BAT or run manually

**AUTO Folder Boot:**
```
REM AUTO\BOOT.BAT
C:\LOADBSD.TOS C:\NETBSD
```

**Manual Boot:**
```
C:\> LOADBSD.TOS C:\NETBSD -a
```

#### Method 2: Bootblock Installation

**Prerequisites:**
- NetBSD partition on disk
- AHDI-compatible partition table

**Installation:**
```bash
# From NetBSD
cd /usr/mdec
installboot /dev/rsd0a bootxx boot

# From TOS (using NetBSD utilities)
installboot.tos A: bootxx boot
```

**AHDI Requirements:**
- Partition must be GEM/BGM type (0x01)
- Set BOOTABLE flag in partition table
- Partition must start on cylinder boundary

### Configuration Files

**Kernel Configuration:**
- `ATARITT` - TT030 with TT-RAM
- `FALCON` - Falcon030
- `HADES` - Hades 040/060
- `MILAN` - Milan 040/060

**Device Support:**
- ncr5380: ACSI/SCSI on Falcon
- ncrscsi: NCR 53C710 on TT
- esp: NCR 53C90 on Hades
- IDE: Falcon/Hades IDE interface
- nvr: Non-volatile RAM (clock)

## Debugging

### Serial Console

**Hardware:**
- Modem1 port (9-pin DIN on TT/Falcon)
- Or standard DB-25 serial port
- 9600 8N1 default

**Enable:**
```bash
# In kernel config
options CONSPEED=9600
options CONSDEV=1       # Serial port 1
```

**Or at boot:**
```
loadbsd.tos -o SER: netbsd
```

### Kernel Debugger

**Enable DDB:**
```bash
# In kernel config
options DDB
options DDB_HISTORY_SIZE=512
```

**Break into debugger:**
- Boot with `-d` flag
- Trigger panic or assertion
- Press Break on serial console

**DDB Commands:**
```
db> ps               # Process list
db> bt               # Backtrace
db> show reg         # Show registers
db> machine ddbcpu   # CPU-specific commands
```

### Common Boot Problems

#### "Not a valid TOS program"
- **Cause:** Corrupted loadbsd.tos
- **Solution:** Re-copy from distribution

#### "Unknown CPU type"
- **Cause:** Unsupported CPU or failed detection
- **Solution:** Check CPU cookies, ensure 68030+

#### "Cannot allocate memory"
- **Cause:** Insufficient ST-RAM or TT-RAM
- **Solution:**
  - Check phystop/ramtop pointers
  - Try `-s` flag (ST-RAM only)
  - Reduce kernel size

#### "MMU fault during boot"
- **Cause:** Bad memory or MMU problem
- **Solution:**
  - Test memory with TOS utilities
  - Check TT-RAM configuration
  - Try ST-RAM only boot

#### System hangs after kernel load
- **Cause:** Hardware conflict or bad kernel
- **Solution:**
  - Boot with minimal kernel
  - Disable devices in config
  - Check for VME conflicts (TT)

### TOS-Specific Issues

**AUTO Folder Problems:**
- Some TOS versions have AUTO limits
- Boot order may affect loading
- Accessories can conflict

**Memory Fragmentation:**
- TOS allocates from bottom-up
- Large programs can fragment memory
- Boot early before running programs

**TOS Version Issues:**
- TOS 1.x: May have compatibility issues
- TOS 2.x: Recommended minimum
- TOS 3.x/4.x: Best compatibility
- EmuTOS: Alternative free TOS

### Hardware-Specific Issues

**TT030:**
- VME bus conflicts
- NCR 53C710 SCSI quirks
- TT-RAM configuration
- SCU (System Control Unit) setup

**Falcon030:**
- IDE conflicts with ACSI
- Videl video compatibility
- DSP may need reset
- SCSI adapter types vary

**Hades:**
- PCI bus initialization
- 68040/060 differences
- ISA compatibility
- Different I/O addresses

**Milan:**
- BIOS compatibility layer
- PCI device mapping
- Different memory layout

## References

**Source Files:**
- `/sys/arch/atari/stand/tostools/loadbsd/` - TOS bootloader
- `/sys/arch/atari/stand/bootxx/` - Bootblock code
- `/sys/arch/atari/atari/locore.s` - Kernel entry
- `/sys/arch/atari/atari/machdep.c` - Machine init
- `/sys/arch/atari/include/cpu.h` - CPU definitions

**Documentation:**
- Atari TT030 Hardware Reference
- Atari Falcon030 Developer Documentation
- Atari Compendium (comprehensive TOS reference)
- AHDI partition table specification

**Hardware References:**
- 68882 Coprocessor manual
- MFP 68901 Multi-Function Peripheral
- NCR 53C710 SCSI controller
- Videl video controller (Falcon)

## Appendix: TOS Technical Details

**TOS Cookies:**
```c
Cookie Jar at 0x5A0:
_CPU   CPU type (0/10/20/30/40/60)
_FPU   FPU type (0/6888x)
_VDO   Video type (ST/TT/NOVA/MILAN)
_MCH   Machine type
_SND   Sound chip type
_FDC   Floppy controller type
_FLK   File locking support
```

**TOS System Variables:**
```c
0x042E (phystop)   Top of ST-RAM
0x04BA (dskbuf)    Disk buffer address
0x05A0 (_cookies)  Cookie jar pointer
0x05A4 (ramtop)    Top of TT-RAM
0x05A8 (ramvalid)  RAM configuration valid
```

**GEMDOS Call Convention:**
- Parameters pushed on stack
- Function number in word before params
- TRAP #1 to invoke
- Return value in d0

**Boot Sector Format (AHDI):**
```c
struct bootsect {
    u_int16_t branch;       // BRA.S instruction
    char      filler[6];
    char      serial[3];    // Serial number
    struct bpb bpb;         // BIOS Parameter Block
    u_int16_t checksum;     // Boot sector checksum
};
```

**Checksum Calculation:**
```c
// Boot sector must checksum to 0x1234
u_int16_t checksum = 0;
for (i = 0; i < 256; i++)
    checksum += bootsect[i];
// Adjust last word to make sum = 0x1234
```
