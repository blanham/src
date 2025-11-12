# NetBSD/vax Boot Documentation

$NetBSD$

## Platform Overview

**Architecture**: Digital VAX (Virtual Address eXtension)
**Port Date**: 1994-08-02 (one of the earliest NetBSD ports)
**Boot Method**: Multi-stage via console ROM/firmware (ROM → bootxx → boot → kernel)
**Firmware**: Console ROM (varies by VAX model)
**MMU Requirements**: VAX virtual memory with page tables, system/user space separation

## Hardware Support

- **CPU**: Digital VAX processors (VAX-11/750, VAX-11/780, MicroVAX, VAXstation, VAX 6000, VAX 8xxx series)
- **Memory**: 32-bit addressing, 512MB maximum (architecture limit)
- **Boot Devices**: RA/RD disks (MSCP), HP disks (MASSBUS), TK tapes, CD-ROM, network (MOP)
- **Firmware**: Console ROM (model-specific, some with console emulators)
- **Systems**: VAX-11/750, VAX-11/780, VAX 8200, VAX 8600, MicroVAX II/III, VAXstation 2000/3100/4000, VS4000, VAX 6000

### VAX Models Supported

**Classic VAX**:
- **VAX-11/750**: First widely-deployed VAX
- **VAX-11/780**: "The VAX", flagship model
- **VAX 8200, 8250, 8300, 8350**: BI-bus systems
- **VAX 8600, 8650**: High-end systems

**MicroVAX/VAXstation**:
- **MicroVAX II, III, 3300, 3400, 3500, 3600, 3800, 3900**: Q-bus desktop systems
- **VAXstation 2000, 3100, 3200, 3500, 4000**: Workstations with graphics
- **VS4000/60, VS4000/90**: Later workstations

**VAX 6000 Series**:
- VAX 6000-200, 6000-300, 6000-400, 6000-500, 6000-600 series

## Boot Process

### Stage 0: Console ROM/Firmware

VAX systems have console firmware in ROM:

**Power-On Sequence**:
1. **Self-test**: Hardware diagnostics
2. **Console prompt**: `>>>` prompt appears
3. **User interaction**: Commands to boot or configure

**Console Commands** (typical):
```
>>> BOOT DUA0         # Boot from disk
>>> BOOT ESA0         # Boot from Ethernet (MOP)
>>> BOOT/R5:xx DUA0   # Boot with flags
>>> SHOW DEVICE       # List devices
>>> SET BOOT DUA0     # Set default boot device
```

**Boot Flags** (R5 register):
- `RB_ASKNAME` (0x01): Ask for root device
- `RB_SINGLE` (0x02): Single-user mode
- `RB_KDB` (0x04): Enter kernel debugger
- `RB_HALT` (0x08): Halt before mount
- Many others

### Stage 1: Primary Bootstrap (Console ROM I/O)

Different VAX models use different mechanisms:

**VAX-11/750 Method**:
- ROM code loads block 0 from disk
- Executes first-stage bootloader

**MicroVAX/VAXstation Method** (VMB - Virtual Memory Bootstrap):
- ROM loads VMB (Virtual Memory Bootstrap)
- VMB loads bootxx from disk
- More sophisticated than 750

**VAX 8200 Method**:
- BI-bus based
- Uses adapter-specific ROM code

### Stage 2: Secondary Bootstrap (bootxx)

**Location**: `/sys/arch/vax/boot/xxboot/`
**Purpose**: Load `/boot` or `/boot.vax` from filesystem
**Size**: Fits in boot blocks (typically 15 sectors)

#### bootxx Operation

From `/sys/arch/vax/boot/xxboot/bootxx.c`:

```c
void Xmain(void)
{
    union {
        struct exec aout;
        Elf32_Ehdr elf;
    } hdr;
    int io;
    u_long entry;

    vax_cputype = (mfpr(PR_SID) >> 24) & 0xFF;  // Get CPU type

    // Set up RPB (Restart Parameter Block)
    rpb = (void *)0xf0000;
    bqo = (void *)0xf1000;  // Boot Qio structure

    if (from == FROMMV) {
        // MicroVAX: copy RPB from ROM
        bcopy((void *)bootregs[11], rpb, sizeof(struct rpb));
        bcopy((void*)rpb->iovec, bqo, rpb->iovecsz);
    } else {
        // VAX-11/750 and others: construct RPB
        memset(rpb, 0, sizeof(struct rpb));
        rpb->devtyp = bootregs[0];   // Device type
        rpb->unit = bootregs[3];     // Unit number
        rpb->rpb_bootr5 = bootregs[5];  // Boot flags
        rpb->csrphy = bootregs[2];   // CSR physical address
        rpb->adpphy = bootregs[1];   // Adapter physical address
    }

    // Try to open /boot.vax or /boot
    io = open("/boot.vax", 0);
    if (io < 0)
        io = open("/boot", 0);
    if (io < 0)
        __asm("halt");

    // Load boot program (a.out or ELF)
    read(io, (void *)&hdr.aout, sizeof(hdr.aout));

    if (N_GETMAGIC(hdr.aout) == OMAGIC && N_GETMID(hdr.aout) == MID_VAX) {
        // Load a.out format
        entry = hdr.aout.a_entry;
        read(io, (void *)entry, hdr.aout.a_text + hdr.aout.a_data);
        memset((void *)(entry + hdr.aout.a_text + hdr.aout.a_data),
               0, hdr.aout.a_bss);
    } else if (memcmp(hdr.elf.e_ident, ELFMAG, SELFMAG) == 0) {
        // Load ELF format
        // ... ELF loading code
    }

    // Jump to boot program
    __asm("movl %0, r11" :: "g"(rpb));  // Pass RPB in R11
    __asm("jmp (%0)" :: "r"(entry));
}
```

**Supported Boot Sources**:
- **FROM750**: VAX-11/750 (simple ROM boot)
- **FROMMV**: MicroVAX (VMB-based boot)
- **FROMVMB**: VAX 8200 and others (BI-bus systems)

### Stage 3: Tertiary Bootstrap (boot)

**Location**: `/sys/arch/vax/boot/boot/`
**Purpose**: Interactive bootloader with full filesystem support

#### Features

From `/sys/arch/vax/boot/boot/boot.c`:

1. **Autoboot with countdown**:
   ```
   >> NetBSD/vax boot [version] <<
   >> Press any key to abort autoboot  5
   ```

2. **Kernel search list**:
   ```c
   static struct {
       char name[12];
       int quiet;
   } filelist[] = {
       { "netbsd.vax", 1 },   // Tried first, silently
       { "netbsd", 0 },       // Standard name
       { "netbsd.gz", 0 },    // Compressed
       { "netbsd.old", 0 },   // Backup
       { "gennetbsd", 0 },    // Generic
       { "", 0 },
   };
   ```

3. **Interactive commands**:
   - `boot [kernel]` - Boot specified kernel
   - `?` or `help` - Show help
   - `halt` - Halt the system

4. **Boot flags**:
   - `-a`: Ask for root device
   - `-s`: Single-user mode
   - `-d`: Drop to debugger
   - `-q`: Quiet boot
   - `-v`: Verbose boot

5. **Autoconf**: Detect boot device automatically

#### Kernel Loading

```c
for (fileindex = 0; filelist[fileindex].name[0] != '\0'; fileindex++) {
    errno = 0;
    if (!filelist[fileindex].quiet)
        printf("> boot %s\n", filelist[fileindex].name);

    marks[MARK_START] = 0;
    fd = loadfile(filelist[fileindex].name, marks,
                  LOAD_KERNEL|COUNT_KERNEL);

    if (fd >= 0) {
        close(fd);
        // Start kernel
        machdep_start((char *)marks[MARK_ENTRY],
                     marks[MARK_NSYM],
                     (void *)marks[MARK_START],
                     (void *)marks[MARK_SYM],
                     (void *)marks[MARK_END]);
    }
}
```

### Stage 4: Kernel

**Entry Point**: Kernel `start` symbol
**Format**: ELF32 or a.out (OMAGIC)
**Load Address**: Kernel virtual address (typically 0x80000000)

#### Kernel Entry

The kernel receives:
- **R11**: Pointer to RPB (Restart Parameter Block)
- **R10**: Boot flags (R5 from console)
- **R9**: Symbol table information

RPB contains:
- Boot device information (type, unit, CSR address)
- Memory size and layout
- Console I/O vectors
- Restart information

## MMU Setup

### VAX Virtual Memory Architecture

**Page Size**: 512 bytes (0x200)
**Virtual Address**: 32-bit (4GB virtual address space)
**Physical Address**: 30-bit (1GB physical, model-dependent)
**Page Tables**: Two-level page tables
**Address Space**: System (S0, S1) and User (P0, P1) spaces

### Virtual Address Format

```
Virtual Address (32 bits):
[31:30] - Address space selection:
  00 = P0 (user process space, grows up)
  01 = P1 (user control space, grows down)
  10 = S0 (system space)
  11 = S1 (system control space)
[29:9]  - Virtual page number (21 bits, 2M pages max)
[8:0]   - Page offset (9 bits, 512 byte page)
```

### Address Spaces

**P0 (Process space)**:
- Grows upward from 0x00000000
- User program text, data, heap
- Each process has separate P0

**P1 (Process control)**:
- Grows downward from 0x7FFFFFFF
- User stack, argument list
- Each process has separate P1

**S0 (System space)**:
- 0x80000000 - 0xBFFFFFFF
- Kernel text, data, buffer cache
- Shared by all processes

**S1 (System control)**:
- 0xC0000000 - 0xFFFFFFFF
- Per-process kernel stack
- System control structures

### Page Table Entry (PTE) Format

**32-bit PTE**:

```
Bit 31:    V    (Valid)
Bit 30:    PROT (Protection, 4 bits with 27-29)
Bits 29-27:     (Protection field)
  0000 = No access
  0001 = Reserved
  0010 = Kernel read
  0011 = Kernel read/write
  0100 = User read, kernel read/write
  0101 = User read, kernel read
  0110 = User read/write, kernel read/write
  0111 = User read/write, kernel read
  1xxx = Invalid combinations
Bit 26:    M    (Modify)
Bits 25-21:     Reserved
Bits 20-0:      PFN (Page Frame Number, 21 bits)
```

**Protection Bits**:
- No access (0)
- Kernel read-only
- Kernel read-write
- User read + kernel read-write
- User read + kernel read-only
- User read-write + kernel read-write
- User read-write + kernel read-only

### Page Table Registers

**System Page Table**:
- **SBR** (System Base Register): Physical address of system page table
- **SLR** (System Length Register): Length of system page table

**Process Page Table** (per-process):
- **P0BR** (P0 Base Register): Base of P0 page table
- **P0LR** (P0 Length Register): Length of P0 page table
- **P1BR** (P1 Base Register): Base of P1 page table
- **P1LR** (P1 Length Register): Length of P1 page table

### TLB (Translation Buffer)

VAX has a hardware TLB (Translation Buffer):
- **Size**: Model-dependent (typically 64-128 entries)
- **Management**: Hardware-managed with software control
- **Invalidation**: `mtpr` instruction to invalidate

**TLB Invalidate**:
```c
mtpr(TBIS, va);  // Invalidate single TLB entry
mtpr(TBIA, 0);   // Invalidate all TLB entries
```

### Memory Management Instructions

**Read page table**:
```asm
mfpr    $PR_SBR, r0    ; Read SBR
mfpr    $PR_SLR, r1    ; Read SLR
```

**Write page table**:
```asm
mtpr    r0, $PR_SBR    ; Write SBR
mtpr    r1, $PR_SLR    ; Write SLR
```

**Probe access**:
```asm
prober  $3, $1, (r0)   ; Probe read access (user mode)
probew  $3, $1, (r0)   ; Probe write access
```

## Memory Map

### Physical Memory Layout

```
0x00000000 - 0x000001FF : Interrupt vectors (low memory)
0x00000200 - 0x00001FFF : Console ROM scratch area
0x00002000 - ...        : Kernel and user memory
0x20000000 - 0x3FFFFFFF : I/O space (varies by model, e.g., UNIBUS)
```

**Note**: Physical memory layout varies significantly by VAX model.

### Virtual Address Space

```
0x00000000 - 0x3FFFFFFF : P0 (user process space, 1GB)
0x40000000 - 0x7FFFFFFF : P1 (user control space, 1GB)
0x80000000 - 0xBFFFFFFF : S0 (system space, 1GB)
0xC0000000 - 0xFFFFFFFF : S1 (system control, 1GB)
```

**Kernel typically loaded at**: 0x80000000 (S0 start)

## Build System

### Source Locations

- **Primary bootstrap**: `/sys/arch/vax/boot/xxboot/`
- **Secondary bootstrap**: `/sys/arch/vax/boot/boot/`
- **Common code**: `/sys/arch/vax/boot/common/`
- **PCS microcode**: `/sys/arch/vax/stand/pcs/` (for VAX 11/750 microcode patches)

### Building Bootloaders

```sh
# Build all bootloaders
cd /sys/arch/vax/boot
make

# Build xxboot
cd /sys/arch/vax/boot/xxboot
make

# Build boot
cd /sys/arch/vax/boot/boot
make
```

## Installation

### Installing Bootblocks

```sh
# Install bootxx in boot blocks
installboot /dev/rra0a /usr/mdec/xxboot

# Copy boot program to root
cp /usr/mdec/boot /boot
```

### Creating Bootable Media

**Tape**:
```sh
# Create bootable tape
mt -f /dev/rmt0 rewind
dd if=/usr/mdec/xxboot of=/dev/rmt0 bs=512
dd if=/usr/mdec/boot of=/dev/rmt0 bs=10k
dd if=/netbsd of=/dev/rmt0 bs=10k
mt -f /dev/rmt0 offline
```

**Diskette** (for systems with floppy):
```sh
dd if=/usr/mdec/xxboot of=/dev/rfd0a bs=512
# Copy boot and kernel
```

## Debugging

### Console Commands

**Boot with flags**:
```
>>> BOOT/R5:3 DUA0    # R5=3 = RB_ASKNAME | RB_SINGLE
```

**Boot from specific device**:
```
>>> BOOT DUA1         # Second disk
>>> BOOT MUA0         # Tape
>>> BOOT ESA0         # Ethernet (MOP)
```

**Examine/deposit**:
```
>>> EXAMINE 80000000  # Examine memory
>>> DEPOSIT 80000000 12345678
```

### Serial Console

Many VAX systems support serial console:
- Set console to serial in console settings
- Serial parameters: 9600 baud, 8N1 (typical)

### Common Boot Issues

**Problem**: "Can't open /boot"
**Solution**: Boot program not in root filesystem

**Problem**: "Bad magic number"
**Solution**: Corrupt boot or kernel file

**Problem**: System hangs at ">>>"
**Solution**: Check boot device exists and is accessible

**Problem**: "Bootxx too large"
**Solution**: Rebuild with smaller bootxx

### Kernel Debugging

**Drop to debugger**:
```
boot netbsd -d
```

**DDB commands**:
```
db> show registers
db> trace
db> continue
```

## Platform-Specific Considerations

### VAX Instruction Set

**CISC Architecture**:
- Complex instruction set
- Variable-length instructions (1-37 bytes)
- Many addressing modes
- String and decimal instructions

**Unique features**:
- `calls` and `ret` for procedure calls (automatic stack frame)
- `movc3`, `movc5` for block moves
- `index` instruction for array indexing
- Privileged instructions (require kernel mode)

### Byte Order

**Endianness**: Little-endian
**Alignment**: Natural alignment preferred but not required
**Bit numbering**: Bit 0 is LSB

### Floating Point

**F_floating**: 32-bit, VAX-specific format
**D_floating**: 64-bit, VAX-specific format
**G_floating**: 64-bit, extended range
**H_floating**: 128-bit (rare)

**Note**: VAX floating point is **different** from IEEE 754.

### Interrupts and Exceptions

**IPL (Interrupt Priority Level)**:
- 0-31 priority levels
- 0 = lowest, 31 = highest
- Controlled by PSL (Processor Status Longword)

**SCB (System Control Block)**:
- Interrupt/exception vectors
- Located at physical address 0
- 512 bytes (128 longword vectors)

### Models Differences

**VAX-11/750**:
- MASSBUS disks (HP)
- UNIBUS peripherals
- Microcode in WCS (Writable Control Store)

**MicroVAX**:
- Q-bus peripherals
- MSCP disks (RA series)
- Integrated console

**VAX 8200**:
- BI-bus (Backplane Interconnect)
- Different boot process
- High-speed bus

**VAXstation**:
- Graphics framebuffer
- Mouse and keyboard
- Workstation-oriented

### PCS (Patchable Control Store)

VAX-11/750 has patchable microcode:
- **Location**: `/sys/arch/vax/stand/pcs/`
- **Purpose**: Update CPU microcode
- **File**: `pcs750.bin` (uuencoded)
- **Loading**: Via `loadpcs()` in boot

## Historical Significance

The VAX architecture was historically important:

- **32-bit pioneer**: First widely-successful 32-bit architecture
- **Virtual memory**: Hardware support for virtual memory
- **VMS**: Designed for VAX/VMS operating system
- **BSD lineage**: 4BSD and descendants (including NetBSD) ran on VAX
- **Academic**: Widely used in universities

NetBSD/vax maintains compatibility with this important legacy system.

## References

### Source Files

**Bootloaders**:
- `/sys/arch/vax/boot/xxboot/bootxx.c` - Primary bootstrap
- `/sys/arch/vax/boot/boot/boot.c` - Secondary bootstrap
- `/sys/arch/vax/boot/common/` - Common boot code

**Kernel**:
- `/sys/arch/vax/vax/` - Machine-dependent kernel code
- `/sys/arch/vax/include/rpb.h` - RPB structure
- `/sys/arch/vax/include/pte.h` - Page table entry definitions

### Man Pages

- `boot(8)` - General boot procedures
- `installboot(8)` - Install bootloader
- `disklabel(8)` - Disk partitioning

### External Documentation

- VAX Architecture Reference Manual (Digital Equipment Corporation)
- VAX/VMS Internals and Data Structures
- MicroVAX system documentation
- Various VAX hardware manuals

### VAX Emulators

**SIMH**: Excellent VAX emulator
- Emulates VAX-11/780, VAX 8600, MicroVAX, others
- Runs NetBSD/vax
- http://simh.trailing-edge.com/

---

*Last Updated: 2025-11-12*
*Architecture Maintainer: NetBSD/vax Port*
