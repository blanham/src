# NetBSD/dreamcast Boot Documentation

## Platform Overview

The SEGA Dreamcast is a home video game console released in 1998, powered by a Hitachi SH-4 processor (SH7750) running at 200 MHz. NetBSD/dreamcast provides support for this platform with a specialized boot process that leverages the Dreamcast's unique CD-ROM boot system.

### Hardware Specifications

- **CPU**: Hitachi SH-4 SH7750 @ 200 MHz
- **Architecture**: Little-endian SuperH SH4
- **Memory**: 16 MB main RAM (0x0c000000-0x0cffffff)
- **ROM**: 1 MB boot ROM (0x00000000-0x000fffff)
- **System Clock**: PCLOCK = 49.9 MHz (50 MHz)

## Boot Method

The Dreamcast uses a unique boot method that does not require a traditional bootloader in the NetBSD tree. Instead, it relies on the Dreamcast's built-in boot ROM and the standard GD-ROM/CD-ROM boot process.

### Boot ROM System

The Dreamcast has a 2 MB mask ROM at physical address 0x00000000 that contains:
- Initial Program Loader (IPL)
- BIOS routines for hardware initialization
- CD-ROM boot support (IP.BIN loader)

### Boot Process Stages

#### Stage 1: Hardware Bootstrap (Mask ROM)

1. **Power-On Reset**: CPU starts execution at 0xa0000000 (P2 cached area)
2. **ROM IPL Execution**:
   - Initializes SH-4 CPU and MMU
   - Sets up exception vectors
   - Initializes hardware (graphics, sound, GD-ROM drive)
   - Disables instruction and data caches initially

3. **CD-ROM Detection**:
   - Checks for disc in drive
   - Reads the ISO9660 filesystem header
   - Looks for IP.BIN (Initial Program) in the root directory

#### Stage 2: IP.BIN Loading

The Dreamcast boot ROM loads and executes IP.BIN from the CD-ROM:

1. **IP.BIN Structure**:
   - Located at LBA 0 of the CD (first 32KB)
   - Contains hardware identification
   - Bootstrap code to load the main executable
   - Product information and metadata

2. **IP.BIN Execution**:
   - Loaded to address 0x8c008000
   - Performs additional hardware setup
   - Contains game/OS-specific initialization
   - Loads the main kernel binary

#### Stage 3: NetBSD Kernel Loading

The IP.BIN bootstrap code loads the NetBSD kernel:

1. **Kernel Location**:
   - Kernel must be in ELF or COFF format
   - Typically named "netbsd" or "netbsd.coff"
   - Located in the CD-ROM filesystem (ISO9660 or custom)

2. **Load Address**:
   - Default text address: **0x8c001000** (P1 cached area)
   - Kernel loaded directly to RAM
   - No relocation required

3. **Kernel Entry**:
   - IP.BIN transfers control to kernel entry point
   - Kernel starts in privileged mode
   - MMU is disabled initially
   - Caches may be enabled or disabled

#### Stage 4: Kernel Initialization

Once the NetBSD kernel gains control:

1. **Early Initialization** (`locore.s`):
   - Set up stack pointer
   - Initialize BSS segment
   - Set up exception vectors
   - Configure SH4-specific registers

2. **MMU Setup**:
   - Initialize TLB (Translation Lookaside Buffer)
   - Set up kernel page tables
   - Enable virtual memory translation
   - Configure cache policies

3. **Machine-Dependent Initialization** (`machdep.c`):
   - Detect memory size
   - Initialize console (framebuffer or serial)
   - Set up interrupt controllers
   - Initialize timers
   - Probe and attach devices

## SH3 vs SH4 MMU Differences

The Dreamcast uses the SH4 processor, which has significant MMU improvements over SH3:

### SH4 MMU Features (Dreamcast)

1. **TLB Structure**:
   - **UTLB** (Unified TLB): 64 entries, 4-way set associative
   - **ITLB** (Instruction TLB): 4 entries, fully associative
   - Separate instruction and data TLBs for better performance

2. **Page Sizes**:
   - 1 KB, 4 KB, 64 KB, 1 MB pages supported
   - NetBSD uses 4 KB pages (PAGE_SIZE = 4096)

3. **Address Space**:
   - 32-bit virtual address space
   - 29-bit physical address space (512 MB addressable)

4. **TLB Management**:
   - Hardware TLB miss handling
   - Software TLB replacement
   - ASID (Address Space Identifier): 8 bits (256 processes)

5. **Cache Coherency**:
   - Store queues (SQ) for write combining
   - Write-back and write-through modes
   - Hardware cache management

### Key Differences from SH3

| Feature | SH3 | SH4 (Dreamcast) |
|---------|-----|-----------------|
| TLB Entries | 32-entry UTLB, 4-way | 64-entry UTLB, 4-way + 4-entry ITLB |
| ASID Bits | 4 bits (16 contexts) | 8 bits (256 contexts) |
| Cache | 4 KB/8 KB I/D | 8 KB/16 KB I/D |
| Store Queues | No | Yes (64 bytes × 2) |
| FPU | Optional | Integrated |
| MMU Registers | SH3_PTEH, SH3_PTEL, SH3_TTB, SH3_TEA | SH4_PTEH, SH4_PTEL, SH4_TTB, SH4_TEA, SH4_PTEA |

## Memory Map

The Dreamcast uses the standard SH4 memory segmentation:

### Physical Memory Layout

```
0x00000000 - 0x000fffff : Boot ROM (1 MB)
0x00200000 - 0x0021ffff : Flash ROM (128 KB)
0x005f6800 - 0x005f7fff : System Control Registers
0x005f8000 - 0x005f9fff : Maple Bus Registers
0x00600000 - 0x0067ffff : G1 Bus Registers
0x00700000 - 0x00710fff : AICA (Sound) Memory
0x00800000 - 0x009fffff : G2 Bus Devices
0x04000000 - 0x04ffffff : Video RAM (8 MB)
0x0c000000 - 0x0cffffff : Main RAM (16 MB)
0x10000000 - 0x107fffff : TA (Tile Accelerator) FIFO
0x11000000 - 0x117fffff : TA Yuv FIFO
```

### Virtual Memory Layout (SH4 Segmentation)

The SH4 divides the 32-bit address space into multiple segments:

```
P0: 0x00000000 - 0x7fffffff : User space (2 GB, TLB translated)
P1: 0x80000000 - 0x9fffffff : Kernel cached (512 MB, direct map to 0x00000000)
P2: 0xa0000000 - 0xbfffffff : Kernel uncached (512 MB, direct map to 0x00000000)
P3: 0xc0000000 - 0xdfffffff : Kernel space (512 MB, TLB translated)
P4: 0xe0000000 - 0xffffffff : Control registers (512 MB)
```

### NetBSD Virtual Memory Layout

```
0x00000000 - 0x7ffff000 : User virtual memory (VM_MAXUSER_ADDRESS)
0x80000000 - 0x9fffffff : P1 - Direct-mapped cached RAM (KSEG0)
0xa0000000 - 0xbfffffff : P2 - Direct-mapped uncached RAM (KSEG1)
0xc0000000 - 0xdfffffff : Kernel virtual memory (VM_MIN_KERNEL_ADDRESS to VM_MAX_KERNEL_ADDRESS)
0xe0000000 - 0xffffffff : P4 - Memory-mapped I/O and control registers
```

### Kernel Memory Usage

```
0x8c000000 : Physical RAM base (via P1 mapping)
0x8c001000 : Kernel text segment (DEFTEXTADDR)
0x8c00xxxx : Kernel data and BSS
0x8cxxxxxx : Kernel heap and buffers
0x8cffffff : End of 16 MB RAM
```

## Build and Installation

### Building the Kernel

```bash
# Configure for Dreamcast
cd /usr/src
./build.sh -U -m dreamcast tools

# Build the kernel
./build.sh -U -m dreamcast kernel=GENERIC

# Result: /usr/obj/sys/arch/dreamcast/compile/GENERIC/netbsd
```

### Creating a Bootable CD-ROM

To boot NetBSD on the Dreamcast, you need to create a bootable CD with:

1. **IP.BIN**: The Initial Program binary
   - Contains Dreamcast-specific bootstrap code
   - Loads and executes the kernel
   - Must be generated with Dreamcast SDK tools or use existing templates

2. **Kernel Binary**: NetBSD kernel in COFF or ELF format
   ```bash
   # Convert ELF to COFF if needed
   objcopy -O coff-sh netbsd netbsd.coff
   ```

3. **ISO9660 Filesystem**:
   ```bash
   # Create CD image
   mkisofs -o netbsd-dreamcast.iso \
           -V "NetBSD_DC" \
           -G IP.BIN \
           -r -l \
           cdroot/
   ```

4. **Burn CD-R**:
   ```bash
   cdrecord -v speed=4 dev=1,0,0 netbsd-dreamcast.iso
   ```

### Installation Methods

1. **CD-ROM Boot** (Primary Method):
   - Burn NetBSD installation CD
   - Insert into Dreamcast
   - Power on console
   - System will auto-boot from CD

2. **Serial Console**:
   - SCIF serial port available (115200 bps)
   - Connect serial adapter to Dreamcast
   - Use terminal emulator for console access

3. **Network Boot**:
   - Requires BBA (Broadband Adapter)
   - Can load kernel over network
   - Not commonly used

### Kernel Configuration

Default kernel configuration: `/usr/src/sys/arch/dreamcast/conf/GENERIC`

Key options:
```
options     SH4              # SH4 CPU support
options     SH7750           # SH7750 CPU variant
options     PCLOCK=49900000  # Peripheral clock frequency
options     IOM_ROM_BEGIN=0x00000000
options     IOM_ROM_SIZE=0x00100000
options     IOM_RAM_BEGIN=0x0c000000
options     IOM_RAM_SIZE=0x01000000
makeoptions DEFTEXTADDR="0x8c001000"
makeoptions ENDIAN="-EL"     # Little-endian
```

## Debugging

### Serial Console

The Dreamcast has an SCIF (Serial Communication Interface with FIFO) port:

**Configuration**:
- Port: SCIF
- Baud rate: 115200 bps
- Data bits: 8
- Parity: None
- Stop bits: 1
- Flow control: None

**Kernel Configuration**:
```
options     CONSPEED=115200
options     SCIFCONSOLE
options     SCIFCN_SPEED=115200
```

### DDB (In-Kernel Debugger)

Enable DDB in kernel configuration:
```
options     DDB              # In-kernel debugger
options     DDB_HISTORY_SIZE=512
makeoptions COPY_SYMTAB=1    # Include symbol table
```

**Access DDB**:
- Press Ctrl+Alt+Esc on keyboard (if supported)
- System panic drops into DDB
- Use `trace` command for stack backtrace
- Use `ps` to show process list
- Use `machine` for SH4-specific commands

### Remote Debugging with KGDB

Enable KGDB support:
```
options     KGDB
options     KGDB_DEVNAME="\"scif\""
options     KGDB_DEVRATE=115200
```

**Remote debugging session**:
```bash
# On development host
cd /usr/obj/sys/arch/dreamcast/compile/GENERIC
sh4-netbsd-gdb netbsd
(gdb) target remote /dev/ttyU0
(gdb) continue
```

### Common Issues

1. **No Video Output**:
   - Check cable connection (VGA or composite)
   - Verify kernel framebuffer support
   - Try serial console instead

2. **CD Won't Boot**:
   - Verify IP.BIN is correct for Dreamcast
   - Check ISO9660 filesystem structure
   - Use CD-R media (not CD-RW)
   - Burn at slow speed (4x recommended)

3. **Kernel Panics**:
   - Check memory configuration
   - Verify kernel is built for SH4
   - Enable DDB for panic analysis
   - Check serial console for error messages

4. **MMU Faults**:
   - TLB miss exceptions indicate page table issues
   - Check kernel page table setup
   - Verify virtual address mappings
   - Use DDB to inspect MMU state

### Debug Tools

1. **Register Dumps**:
   ```c
   // In kernel code
   printf("PC=%p SR=%08x\n", (void*)regs->tf_spc, regs->tf_ssr);
   printf("PR=%p GBR=%08x\n", (void*)regs->tf_pr, regs->tf_gbr);
   ```

2. **MMU State**:
   ```
   # In DDB
   machine tlb        # Show TLB entries
   machine cache      # Show cache status
   machine reg        # Show SH4 registers
   ```

3. **Memory Dumps**:
   ```
   # In DDB
   x/x 0x8c001000     # Examine kernel text
   x/x 0x0c000000     # Examine physical RAM
   ```

## References

- NetBSD/dreamcast Homepage: https://www.netbsd.org/ports/dreamcast/
- SH-4 CPU Core Architecture Manual (Renesas)
- SH7750 Hardware Manual (Hitachi/Renesas)
- Dreamcast Programming Documentation (Marcus Comstedt)
- NetBSD Source: `/usr/src/sys/arch/dreamcast/`
- SH3/SH4 Common Code: `/usr/src/sys/arch/sh3/`

## Technical Contacts

For questions about NetBSD/dreamcast:
- NetBSD Port Maintainers: port-dreamcast@netbsd.org
- General NetBSD Questions: netbsd-help@netbsd.org

## Revision History

- Initial documentation based on NetBSD source analysis
- Kernel configuration from `sys/arch/dreamcast/conf/`
- Boot process derived from SH4 architecture documentation
