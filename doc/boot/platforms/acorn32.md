# NetBSD/acorn32 Boot Process Documentation

## Platform Overview

NetBSD/acorn32 supports Acorn ARM-based computers including:
- Acorn RiscPC systems (ARM 6, ARM 7, StrongARM)
- Acorn A7000 series
- Network Computer (NC) systems
- Systems with ARM6, ARM7, or StrongARM SA-110 processors

These systems originally ran RISC OS, a proprietary operating system developed by Acorn Computers. The NetBSD bootloader must operate within the RISC OS environment before transitioning to NetBSD.

## Boot Method

### RISC OS Module Bootloader

acorn32 uses a unique two-stage boot process that operates from within RISC OS:

1. **boot32**: A RISC OS relocatable module that loads and starts NetBSD
2. **nbfs**: An optional NetBSD filesystem module for accessing NetBSD filesystems from RISC OS

The bootloader is implemented as a RISC OS relocatable module (`boot32`) that can be executed directly from RISC OS.

### Bootloader Components

Location: `/sys/arch/acorn32/stand/`

Key components:
- `boot32/` - Primary bootloader
- `nbfs/` - NetBSD filesystem support module
- `lib/` - Shared bootloader library with RISC OS interface routines

## Boot Process Stages

### Stage 1: RISC OS Module Initialization

**File: `stand/boot32/rmheader.S`**

The bootloader starts as a RISC OS relocatable module:

```
Module Header Structure:
- Start code offset
- Initialization code offset
- Finalization code offset
- Service call handler offset
- Title string offset
- Help string offset
- Command keyword table offset
- Module flags (32-bit compatible)
```

When started:
1. Module is loaded at an arbitrary address by RISC OS
2. Code relocates itself to application space at 0x8000
3. Copies code/data from module location to 0x8000
4. Synchronizes data cache by reading 128KB region
5. Jumps to `_start` entry point

### Stage 2: Boot Preparation

**File: `stand/boot32/boot32.c`**

The bootloader initializes its data structures:

1. **Memory Allocation**
   - Queries RISC OS for page size and memory table size
   - Allocates 99% of available heap for memory image buffer
   - Allocates relocation instruction table (max 4096 relocations)
   - Allocates page info structures for physical/virtual mapping
   - Allocates 16KB for initial page tables

2. **Memory Configuration Discovery**
   - Reads RISC OS memory arrangement table
   - Identifies memory types: DRAM, VRAM, ROM, I/O, Processor-only SDRAM
   - Maps video memory from DRAM if no VRAM present
   - Splits off top 1MB of DRAM for kernel requirements
   - Records physical memory blocks

3. **Physical Memory Mapping**
   - Uses RISC OS page operations to get physical addresses
   - Sorts pages by physical address
   - Identifies contiguous physical memory regions
   - Finds first mapped DRAM and PODRAM pages

### Stage 3: Kernel Loading

The bootloader loads the NetBSD kernel:

1. **Kernel File Loading**
   - Prompts for kernel path (default: "netbsd")
   - Parses boot flags (-a, -s, -d, -v, etc.)
   - Uses `loadfile()` twice:
     - First pass: COUNT_KERNEL to get sizes and marks
     - Second pass: LOAD_KERNEL to actually load

2. **Custom Memory Operations**
   - Implements custom read/memcpy/memset operations
   - Allocates physical pages through relocation table
   - Maps kernel to physical address in DRAM0 or PODRAM0
   - Kernel physical address determined by first available memory block

3. **Physical/Virtual Offset Calculation**
   ```c
   pv_offset = marks[MARK_START] - kernel_physical_start;
   ```

### Stage 4: Page Table Creation

**Function: `create_initial_page_tables()`**

Creates ARM MMU page tables for kernel boot:

1. **Section Mapping Setup**
   - Uses 1MB sections (no page tables initially)
   - Section descriptor: AP=01, CB=0, Domain 0
   - Creates 4096 entries for 4GB address space

2. **Initial 1:1 Mapping**
   - Maps all 4GB of address space 1:1
   - Allows bootloader code to continue running

3. **Special Mappings**
   - Maps top 1MB of DRAM to virtual address 0 (for vectors)
   - Maps 16MB of kernel at KERNEL_BASE (typically 0xF0000000)
   - Video memory remains 1:1 mapped

4. **L1 Page Table Placement**
   - L1 tables placed at top of physical DRAM
   - Must be 16KB aligned (4 pages)
   - Relocated to physical memory via relocation mechanism

### Stage 5: Configuration Structure

**Function: `create_configuration()`**

Creates bootconfig structure for kernel:

```c
struct bootconfig {
    u_int magic;                  // 0x43112233
    u_int version;                // 0x2
    u_char machine_id[4];         // Unique machine ID
    char kernelname[80];          // Kernel name
    char args[512];               // Boot arguments

    // Kernel memory info
    u_int kernvirtualbase;
    u_int kernphysicalbase;
    u_int kernsize;
    u_int ksym_start, ksym_end;

    // Display info
    u_int display_phys;
    u_int display_start;
    u_int display_size;
    u_int width, height;
    u_int log2_bpp;
    u_int framerate;

    // Memory blocks
    u_int pagesize;
    u_int drampages, vrampages;
    u_int dramblocks, vramblocks;
    phys_mem dram[32];
    phys_mem vram[16];
};
```

Information gathered:
- Monitor type and sync rate
- IOEB and SuperIO flags
- LCD flags
- Unique machine ID
- Display configuration
- Physical memory layout

### Stage 6: Relocation Table

The bootloader builds a relocation table:

1. **Relocation Entries**
   - Each entry: [source, destination, length]
   - First word contains count of relocations
   - Tracks all physical page moves needed

2. **Compaction**
   - Merges adjacent relocations
   - Reduces relocation count
   - Optimizes transfer time

### Stage 7: Kernel Launch

**File: `stand/boot32/start.S`**

The bootloader transitions to kernel:

1. **Pre-launch Preparation**
   ```assembly
   - Copy relocate_code to dedicated page
   - Copy relocation table after code
   - Dismount all RISC OS filesystems
   - Send service_pre_reset to RISC OS modules
   ```

2. **Enter Supervisor Mode**
   ```assembly
   - Issue OS_EnterOS SWI (enter OS mode)
   - Switch to SVC32 mode
   - Disable interrupts (I32_bit | F32_bit)
   ```

3. **Processor Detection**
   - Read CPU ID from CP15 c0
   - Detect ARM6, ARM7, or ARMv4+
   - Select appropriate coprocessor instructions

4. **Physical Transition**
   ```assembly
   - Flush instruction and data caches
   - Flush TLB
   - Disable MMU, I-cache, D-cache, write buffer
   - Branch to physical address
   ```

5. **Physical Memory Relocation**
   - Screen border turns red
   - Execute all relocations from table
   - Copy pages in 4-byte chunks
   - Screen border turns green when done

6. **New MMU Setup**
   ```assembly
   - Flush caches and TLB again
   - Load new L1 page table address (CP15 c2)
   - Enable MMU, caches, write buffer
   - Continue executing (now with new mappings)
   ```

7. **Kernel Entry**
   - Screen border turns blue
   - Call kernel entry point with:
     - r0 = bootconfig structure (physical address)
   - Kernel takes over

## ARM MMU Setup Requirements

### ARMv3 (ARM6/ARM7)

Control Register (CP15 c1):
- MMU enable (bit 0)
- Alignment fault (bit 1)
- Data cache enable (bit 2)
- Write buffer enable (bit 3)
- 32-bit program space (bit 4)
- 32-bit data space (bit 5)
- Late abort timing (bit 6)

Cache/TLB Operations:
- Flush ID cache: `MCR p15, 0, r0, c7, c0, 0`
- Flush TLB: `MCR p15, 0, r0, c5, c0, 0`

### ARMv4+ (StrongARM SA-110)

Additional operations:
- Separate I-cache flush: `MCR p15, 0, r0, c7, c7, 0`
- Drain write buffer: `MCR p15, 0, r0, c7, c10, 4`
- Invalidate I-cache on MMU disable: `MCR p15, 0, r1, c7, c5, 0`

### L1 Page Table Requirements

- 16KB aligned (4 * 4KB pages)
- 4096 entries for 4GB address space
- Section descriptors (1MB granularity)
- Domain 0, AP=01 (supervisor read/write)

## Memory Map

### Physical Memory Layout

**RiscPC/A7000 Configuration:**
```
0x00000000 - 0x1FFFFFFF : DRAM Bank 0 (up to 512MB)
0x20000000 - 0x2FFFFFFF : DRAM Bank 1 (if present)
0x30000000 - 0x3FFFFFFF : Processor-only SDRAM (if present)
0x03200000 - 0x033FFFFF : I/O Space (2MB)
0x03400000 - 0x035FFFFF : ROM (2MB)
0x02000000 - 0x02FFFFFF : VRAM (if present, up to 2MB)
```

**Kinetic PODRAM Cards:**
```
0x20000000+ : Processor-only DRAM (not video-accessible)
```

### Virtual Memory Layout

**Kernel Address Space:**
```
0xF0000000 - 0xF0FFFFFF : Kernel text/data (16MB)
0xF1000000 - 0xFCFFFFFF : Kernel VM space (192MB)
0xFD000000 - 0xFFFFFFFF : Device mappings
```

**Special Mappings:**
```
0x00000000 - 0x000FFFFF : Top 1MB of DRAM (for vectors)
VIDEOMEM   : 1:1 mapping   : Display memory
```

### RISC OS Constraints

- Bootloader runs in RISC OS application space (0x8000+)
- Must use RISC OS SWIs for file access
- Memory allocated via RISC OS heap
- Page tables built in RISC OS memory

## Build and Installation

### Building boot32

```bash
cd /sys/arch/acorn32/stand
make
```

Produces:
- `boot32/boot32,ffa` - RISC OS module
- `nbfs/nbfs,ffa` - NetBSD filesystem module

### Installation

1. **On RISC OS System:**
   ```
   - Copy boot32,ffa to RISC OS system
   - Copy kernel as "netbsd" to accessible filesystem
   - Double-click boot32 icon to load module
   ```

2. **Running Bootloader:**
   ```
   *boot32 [kernel] [flags]
   ```

   Common flags:
   - `-a` : Ask for root device
   - `-s` : Single user mode
   - `-d` : Drop to debugger
   - `-v` : Verbose boot

3. **From Command Line:**
   ```
   *boot32 netbsd -s
   *boot32 wd0a:netbsd.old
   *boot32 adfs::4.$.netbsd -d
   ```

### Filesystem Access

RISC OS filesystem paths:
- `adfs::4.$.netbsd` - ADFS drive 4, root, netbsd file
- `wd0a:netbsd` - NetBSD-style (with nbfs module)
- `scsi::0.$.kernels.test` - SCSI drive

## Debugging

### Early Boot Debugging

1. **Screen Border Colors**
   - Red: Entered physical relocation code
   - Green: Memory relocation complete
   - Blue: New MMU enabled, about to enter kernel

2. **Debug Output**
   - Set `int debug = 1` in boot32.c
   - Prints memory configuration
   - Shows relocation statistics
   - Displays kernel load addresses

### RISC OS Debugging

Monitor bootloader with RISC OS debugger:
```
*BreakSet boot32_main
*Debug
```

### Common Issues

**Issue: "Insufficient memory"**
- Cause: < 256KB heap space available
- Solution: Quit RISC OS applications, increase application slot

**Issue: "Out of pages"**
- Cause: Kernel + bootloader exceeds available memory
- Solution: Reduce kernel size or increase RAM

**Issue: "Kernel is bigger than first DRAM module"**
- Cause: Kernel exceeds first physical memory bank
- Solution: Install more RAM in first bank or reduce kernel size

**Issue: "Found no DRAM mapped in bootloader"**
- Cause: Memory configuration detection failed
- Solution: Check RISC OS memory configuration

### Memory Debugging

View memory layout:
```c
// Memory blocks discovered
for (count = 0; count < dram_blocks; count++) {
    printf("DRAM (%d) at 0x%x for %d k\n",
           count, DRAM_addr[count],
           (DRAM_pages[count]*nbpp)>>10);
}
```

View relocation table:
```c
printf("%ld entries, first: 0x%lx->0x%lx for %lx bytes\n",
       reloc_instruction_table[0],
       reloc_instruction_table[1],
       reloc_instruction_table[2],
       reloc_instruction_table[3]);
```

### Kernel Entry Debugging

Kernel receives:
- r0: Physical address of bootconfig structure
- PC: Kernel entry point
- MMU: Enabled with new page tables
- Mode: SVC32 (Supervisor mode, ARM32)
- Interrupts: Disabled

Verify bootconfig magic:
```c
if (bootconfig.magic != BOOTCONFIG_MAGIC)
    panic("Bad bootconfig magic: 0x%x", bootconfig.magic);
```

## Technical Notes

### RISC OS Integration

The bootloader uses RISC OS SWIs extensively:
- `OS_Memory` - Memory operations
- `OS_File` - File operations
- `OS_ReadVduVariables` - Display info
- `OS_ReadSysInfo` - System information
- `OS_FSControl` - Filesystem control
- `OS_EnterOS` - Enter supervisor mode

### Relocation Mechanism

All kernel pages must be relocated because:
1. Kernel loaded at virtual addresses in RISC OS space
2. Must move to physical DRAM0 addresses
3. Bootloader memory must also be moved
4. Page tables constructed in RISC OS heap

The relocation runs in physical mode to avoid TLB conflicts.

### Cache Coherency

Critical cache operations:
1. After loading kernel: No explicit flush needed (loaded via read)
2. After copying relocation code: Read 128KB to flush D-cache
3. Before MMU disable: Flush I+D caches, drain write buffer
4. After memory relocation: Flush I+D caches
5. Before MMU enable: Flush TLB

### Processor Compatibility

Bootloader supports:
- ARM6 (ARMv3)
- ARM7 (ARMv3)
- ARM7500, ARM7500FE (ARMv3)
- StrongARM SA-110 (ARMv4)

Detection based on CPU ID register format variations.

## References

- `/sys/arch/acorn32/stand/boot32/boot32.c` - Main bootloader
- `/sys/arch/acorn32/stand/boot32/start.S` - Assembly startup and relocation
- `/sys/arch/acorn32/stand/boot32/rmheader.S` - RISC OS module header
- `/sys/arch/acorn32/include/bootconfig.h` - Boot configuration structure
- ARM Architecture Reference Manual - MMU and cache specifications
- RISC OS 3 Programmer's Reference Manual - SWI interfaces
