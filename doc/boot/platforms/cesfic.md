# NetBSD/cesfic Boot Documentation

## Platform Overview

NetBSD/cesfic is a port to the FIC8234 VME processor board manufactured by CES (Concurrent Engineering Systems) in Geneva, Switzerland. These boards were popular in high-energy physics data acquisition systems at facilities like CERN.

**Hardware Specifications:**
- CPU: Motorola 68040 at 25 MHz
- Optional dual-processor configuration
- RAM: 8MB or 32MB
- Serial: 2 ports on Zilog Z85C30
- Ethernet: 79c900 (ILACC)
- SCSI: NCR 53C710 (not yet supported in NetBSD)
- VME bus interface

**Current Status:**
- Basic support functional
- Diskless (NFS root) or ramdisk required
- SCSI support not implemented
- Multi-user and self-hosting capable

## Boot Method

The cesfic platform uses a unique boot method: it boots from a running OS-9 system. There is no traditional bootloader; instead, the NetBSD kernel is loaded as a binary image by OS-9 and then executed.

### Boot Process

1. **OS-9 Operating System** - System boots into OS-9
2. **Binary Kernel Image** - NetBSD kernel loaded by OS-9
3. **Direct Execution** - Jump to kernel entry point

**No traditional bootloader exists** - The simplicity of this approach means there's no complex multi-stage boot process.

## Creating Boot Image

### Build Kernel

```bash
cd /usr/src/sys/arch/cesfic/conf
config GENERIC
cd ../compile/GENERIC
make depend
make
```

### Create Binary Image

The kernel must be converted from ELF format to a raw binary that OS-9 can load:

```bash
objcopy --output-target=binary netbsd netbsd.bin
```

**This produces a flat binary** with no headers, suitable for loading at a specific physical address.

## 68040 MMU Setup

The FIC8234 uses a Motorola 68040 processor with its integrated MMU.

### 68040 MMU Characteristics

- Four-level page tables
- 4KB page size
- Separate instruction and data TLBs (64 entries each)
- Address Translation Cache (ATC)
- Transparent Translation Registers (TTR0/TTR1)
- URP (User Root Pointer) and SRP (Supervisor Root Pointer)
- Integrated FPU

### MMU Setup in locore.s

```assembly
| 68040 MMU initialization
.long   0x4e7b1807              | movc d1,srp
.word   0xf518                  | pflusha
movl    #MMU40_TCR_BITS,%d0     | TC: E=1, P=1, 4K pages
.long   0x4e7b0003              | movc d0,tc
```

### Cache Control

**68040 Cache Structure:**
- 4KB instruction cache
- 4KB data cache
- 16-byte cache lines
- Write-back or write-through modes
- CINVA/CPUSHA instructions for maintenance

**Cache Initialization:**
```assembly
movl    #CACHE40_OFF,%d0        | Disable caches initially
movc    %d0,%cacr
.word   0xf4f8                  | cpusha bc (push and invalidate all)
```

## Boot Process Stages

### Stage 1: OS-9 Environment

**Prerequisites:**
- FIC8234 board running OS-9
- Serial console connected
- Network configured (for NFS root)

**Loading Process:**
1. Transfer `netbsd.bin` to OS-9 filesystem
2. Use OS-9 memory management to allocate space
3. Load binary image to physical address 0x20100000

**Memory Layout in OS-9:**
```
0x20000000          Start of RAM
0x20100000          Kernel load address (RAM + 1MB)
0x20100400          Kernel entry point (RAM + 1MB + 1KB)
```

### Stage 2: Kernel Entry

**Location:** `/sys/arch/cesfic/cesfic/locore.s`

**Entry Point:** `start` at 0x20100400 (offset 0x400 in loaded image)

**Initial State:**
- 68040 supervisor mode
- MMU disabled
- Caches disabled
- Running at load address (0x20100000)
- Interrupts disabled

**Execution:**
```bash
# From OS-9 prompt
load netbsd.bin 0x20100000
jump 0x20100400
```

**Entry Code:**
```assembly
.text
ASENTRY_NOPROFILE(start)
    movw    #PSL_HIGHIPL,%sr    | disable all interrupts

    | We're linked to run at a different address
    | Calculate relocation offset
    lea     %pc@(Lstart),%a0
    lea     _ASM_LABEL(start),%a1
    subl    %a1,%a0
    movl    %a0,_C_LABEL(reloc)

    | Clear BSS
    lea     _C_LABEL(edata),%a0
    addl    %reloc,%a0
    lea     _C_LABEL(end),%a1
    addl    %reloc,%a1
Lbss:
    clrl    %a0@+
    cmpl    %a0,%a1
    bhi     Lbss

    | Setup temporary stack
    lea     _ASM_LABEL(tmpstk),%sp
    addl    %reloc,%sp

    | Initialize 68040 MMU
    bsr     _C_LABEL(init_mmu)

    | Jump to C code
    jbsr    _C_LABEL(_bootstrap)
```

### Stage 3: Bootstrap Initialization

**Location:** `/sys/arch/cesfic/cesfic/machdep.c`

**Function:** `_bootstrap()` and `cpu_startup()`

**Actions:**
1. Initialize memory management
   - Setup page tables
   - Configure RAM (8MB or 32MB)
   - Reserve kernel memory
2. Initialize console (Z85C30 serial)
3. Setup interrupt system
4. Configure VME bus interface
5. Initialize Ethernet (ILACC 79c900)
6. Mount NFS root or ramdisk
7. Start init

**Memory Configuration:**
```c
// RAM starts at physical 0x20000000
// Kernel at 0x20100000 (offset 1MB)
physmem_start = 0x20100000;
physmem_end = 0x20000000 + (8 * 1024 * 1024);  // 8MB or 32MB

// Setup kernel virtual address space
virtual_avail = VM_MIN_KERNEL_ADDRESS;
virtual_end = VM_MAX_KERNEL_ADDRESS;
```

## Memory Map

### Physical Memory Layout

```
Physical Address Space:

0x00000000 - 0x1FFFFFFF    VME Bus Address Space
                            (accessible via VME interface)

0x20000000 - 0x21FFFFFF    Main RAM (8MB or 32MB)
  0x20000000 - 0x200FFFFF    First 1MB (reserved/OS-9 compatible)
  0x20100000 - ...           NetBSD kernel load address
  0x20100400 - ...           Kernel entry point
  ...                        Kernel text/data/bss
  ...                        Kernel heap
  ...                        Free memory

0xF0000000 - 0xFFFFFFFF    I/O Space
  0xFE000000 - ...           VME A24 address space
  0xFF000000 - ...           VME A16 address space
  0xFFF00000 - ...           Local I/O devices
    - Z85C30 Serial ports
    - 79c900 Ethernet
    - NCR 53C710 SCSI
    - VME interface registers
```

### Kernel Virtual Address Space

```
Virtual Address Space:

0x00000000 - 0x000007FF    Unmapped (NULL pointer trap)
0x00000800 - ...           Kernel text (read-only, executable)
...                        Kernel data/bss
...                        Kernel heap
...                        Page tables

0x80000000 - 0xBFFFFFFF    User process space (1GB)

0xF0000000 - 0xFFFFFFFF    I/O device mappings
```

## Build and Installation

### Prerequisites

- Cross-compilation environment for m68k
- OS-9 system running on FIC8234
- Network connection for NFS
- Serial console connection

### Building

```bash
# Build toolchain
cd /usr/src
./build.sh -m cesfic tools

# Build kernel
cd /usr/src/sys/arch/cesfic/conf
/usr/src/tooldir/bin/nbconfig GENERIC
cd ../compile/GENERIC
/usr/src/tooldir/bin/nbmake-cesfic depend
/usr/src/tooldir/bin/nbmake-cesfic
```

### Creating Bootable Image

```bash
# Convert ELF kernel to binary
objcopy --output-target=binary netbsd netbsd.bin

# Transfer to FIC8234 via serial or network
# Example using Kermit:
kermit
set line /dev/ttyS0
set speed 19200
send netbsd.bin
```

### Network Boot Setup

**NFS Server Configuration:**
```bash
# On NFS server, export root filesystem
# /etc/exports:
/export/cesfic/root -maproot=root -network 192.168.1.0 -mask 255.255.255.0

# Setup root filesystem
cd /export/cesfic/root
tar xzf base.tgz
tar xzf etc.tgz

# Configure /etc/fstab for NFS root
nfsserver:/export/cesfic/root / nfs rw 0 0
```

**Kernel Configuration for NFS Root:**
```
# In kernel config
options NFS_BOOT_DHCP
options NFS_BOOT_BOOTPARAM

# Or compile-time root specification
config netbsd root on le0 type nfs
```

### Booting

**From OS-9:**
```
$ load netbsd.bin 0x20100000
Loaded 2048KB at 0x20100000
$ jump 0x20100400

NetBSD/cesfic 9.0 (GENERIC)
Copyright (c) 1996, 1997, 1998, 1999, 2000, 2001, 2002, 2003, 2004, 2005,
    2006, 2007, 2008, 2009, 2010, 2011, 2012, 2013, 2014, 2015, 2016, 2017,
    2018, 2019, 2020 The NetBSD Foundation, Inc...
```

## Debugging

### Serial Console

**Hardware:**
- Z85C30 dual serial controller
- Port 0: Console (19200 8N1 default)
- Port 1: Available for other uses

**Console Configuration:**
```c
// In machdep.c
#define CONSOLE_SPEED 19200
```

**DDB Access:**
- Build kernel with `options DDB`
- Break on serial line or panic

### Common Issues

#### "Cannot mount root"
- **Cause:** NFS configuration problem or network issue
- **Solution:**
  - Verify NFS exports on server
  - Check network connectivity
  - Ensure DHCP/BOOTPARAMS configured
  - Try ramdisk root instead

#### "MMU fault at startup"
- **Cause:** Bad memory or incorrect load address
- **Solution:**
  - Verify RAM size (8MB vs 32MB)
  - Check load address (must be 0x20100000)
  - Test memory from OS-9

#### System hangs after "jump"
- **Cause:** Incorrect binary format or entry point
- **Solution:**
  - Verify objcopy command
  - Check entry point is 0x400 into image
  - Ensure interrupts disabled in OS-9 before jump

### Memory Debugging

**Check memory size:**
```c
// In early boot code
printf("physmem: %d pages (%d MB)\n",
    physmem, physmem / 256);
```

**DDB memory commands:**
```
db> show page
db> show map
db> x/i 0x20100400    # Disassemble entry point
```

## Hardware Support Status

### Working

- 68040 CPU at 25 MHz
- 8MB/32MB RAM
- Z85C30 serial ports (console)
- 79c900 Ethernet (le driver)
- VME bus interface (basic)
- NFS root filesystem

### Not Working / Not Tested

- NCR 53C710 SCSI controller
- Dual-processor support
- VME slave mode
- All VME A24/A32 functionality

### Limitations

- No local disk support (SCSI not implemented)
- Must boot from NFS or ramdisk
- No bootloader (must use OS-9 to load)
- Limited VME bus support

## Development Notes

### Adding SCSI Support

The NCR 53C710 SCSI controller hardware exists but driver support is incomplete.

**Required Work:**
1. Port siop(4) driver from other platforms
2. Add SCSI probe code to autoconf
3. Test with actual SCSI devices
4. Implement disk boot support

**Reference:** See mvme68k or amiga ncr53c710 drivers

### Bootloader Implementation

Currently there is no bootloader. A future enhancement would be:

1. **Stage 1:** Small OS-9 program that:
   - Reads kernel from SCSI disk
   - Loads to memory
   - Jumps to entry point

2. **Stage 2:** Native bootloader that:
   - Provides boot menu
   - Supports FFS filesystem
   - Allows kernel selection
   - Passes boot parameters

## References

**Source Files:**
- `/sys/arch/cesfic/cesfic/locore.s` - Kernel entry and startup
- `/sys/arch/cesfic/cesfic/machdep.c` - Machine-dependent code
- `/sys/arch/cesfic/cesfic/pmap_bootstrap.c` - MMU initialization
- `/sys/arch/cesfic/conf/GENERIC` - Generic kernel config
- `/sys/arch/cesfic/README` - Platform README

**Hardware Documentation:**
- FIC8234 Hardware Manual (CES)
- 68040 User's Manual (Motorola)
- NCR 53C710 SCSI Processor Technical Manual
- Z85C30 Serial Communications Controller Manual
- Am79C900 ILACC Ethernet Controller Manual

**Related Platforms:**
- mvme68k - Similar VME-based 68040 system
- hp300 - Similar architecture
- sun3 - 68030 with similar boot concepts

## Appendix: FIC8234 Technical Details

### Board Layout

**Processor:**
- MC68040RC25 running at 25 MHz
- Optional second 68040 for multiprocessing
- No FPU emulation needed (040 has built-in FPU)

**Memory:**
- 8MB or 32MB DRAM
- 70ns or faster DRAM recommended
- Parity checking optional
- Memory controller integrated

**I/O Controllers:**
- Z85C30 SCC (Serial Communications Controller)
  - Channel A: Console
  - Channel B: Auxiliary
- Am79C900 ILACC (Integrated Local Area Communication Controller)
  - 10BASE-T/10BASE-2 Ethernet
- NCR 53C710 SCSI I/O Processor
  - SCSI-2 compliant
  - Fast SCSI (10 MB/s)

**VME Bus Interface:**
- VME A16, A24, A32 address spaces
- D8, D16, D32 data transfers
- Bus master and slave modes
- Interrupt handler for VME IRQ1-7

### Memory Controller

**DRAM Configuration:**
- Four banks of DRAM
- Page mode or static column mode
- Refresh controller integrated
- Memory interleaving supported

**Cache Configuration:**
- 4KB instruction cache
- 4KB data cache
- Copyback or writethrough mode
- Cache coherency for VME access

### Interrupt System

**Interrupt Sources:**
- Serial ports (SCC)
- Ethernet (ILACC)
- SCSI (53C710)
- VME interrupts (IRQ1-7)
- Timers
- Software interrupts

**Interrupt Priority:**
```
Level 7: NMI (Non-Maskable Interrupt)
Level 6: Serial ports
Level 5: Ethernet
Level 4: SCSI
Level 3-1: VME interrupts
```

### Boot Process Detail

**OS-9 Load Procedure:**
```
1. Allocate memory at 0x20100000
2. Read kernel binary from OS-9 file
3. Copy to allocated memory
4. Set up environment:
   - Disable interrupts
   - Clear caches
   - Reset MMU
5. Jump to entry point (0x20100400)
```

**Why offset 0x400?**
- First page (0x000-0x3FF) reserved for vectors
- Allows VA=PA mapping during early boot
- Compatible with other m68k platforms
- Traps NULL pointer dereferences

**Load Address Choice (0x20100000):**
- RAM base is 0x20000000
- First 1MB reserved for compatibility
- Provides space for boot structures
- Aligned on 1MB boundary for MMU

### Future Enhancements

**Potential improvements:**
1. SCSI driver implementation
2. Native bootloader (no OS-9 dependency)
3. Multiprocessor support
4. Enhanced VME bus support
5. DMA improvements
6. Real-time clock support
7. Watchdog timer support

**Community Contributions Welcome:**
- SCSI testing and debugging
- Hardware documentation
- Driver development
- Performance optimization
