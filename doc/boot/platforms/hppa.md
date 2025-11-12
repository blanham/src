# NetBSD/hppa Boot Documentation

$NetBSD$

## Platform Overview

**Architecture**: HP PA-RISC (Precision Architecture)
**Port Date**: 2002-06-05
**Boot Method**: Multi-stage via PDC firmware (LIF boot → IPL → boot → kernel)
**Firmware**: PDC (Processor Dependent Code)
**MMU Requirements**: Hash-based TLB, space IDs, virtual addressing at boot

## Hardware Support

- **CPU**: HP PA-RISC 1.0, 1.1, 2.0 (PA-7xxx, PA-8xxx series)
- **Memory**: 32-bit addressing (PA-RISC 1.x), 64-bit addressing (PA-RISC 2.0)
- **Boot Devices**: SCSI disk, IDE disk, CD-ROM, network, tape (sequential media)
- **Firmware**: PDC (Processor Dependent Code) - HP's boot firmware
- **Systems**: HP 9000 series (700, 800), HP visualize workstations

### Supported Systems

**HP 9000/700 Series** (Workstations):
- **712**: PA-7100LC single CPU workstation
- **715**: PA-7100, PA-7150 workstation
- **725**: PA-7100LC high-end workstation
- **735**: PA-7150 workstation
- **743**: PA-7100 workstation
- **744**: PA-8000, PA-8200 workstation

**HP 9000/800 Series** (Servers):
- Various server models with PA-RISC processors

**HP Visualize Workstations**:
- B, C, and J class workstations

## Boot Process

### Stage 0: PDC Firmware

HP PA-RISC systems use PDC (Processor Dependent Code) firmware:

1. **Power-On Self Test (POST)** - Hardware initialization
2. **PDC initialization** - Firmware services setup
3. **Boot path selection** - Choose boot device
4. **LIF volume search** - Find Logical Interchange Format bootloader
5. **IPL load** - Load Initial Program Loader

**PDC Firmware Functions**:
- Device I/O operations (`boot_input`, `boot_output`)
- Console I/O
- Memory management
- Interrupt handling
- Boot device selection

### Stage 1: IPL (Initial Program Loader) in LIF Format

**Location**: `/sys/arch/hppa/stand/xxboot/`
**Format**: LIF (Logical Interchange Format) - HP's bootable file format
**Size**: ~7KB (three parts: 4KB + 1KB + 1.5KB)
**Purpose**: Load secondary bootloader from filesystem

#### LIF Header Format

From `/sys/arch/hppa/stand/xxboot/start.S`:

```asm
lifhdr:
    .byte   0x80, 0x00      ; LIF magic number
    .string "NetBSD"        ; Volume label (6 chars)
    ...
lif_ipl_addr:
    .word   top-lifhdr      ; IPL start (4KB offset, 2KB aligned)
lif_ipl_size:
    .word   0x00001000      ; IPL size (4KB, 2KB aligned)
lif_ipl_entry:
    .word   $START$-top     ; Entry point offset
```

**LIF Structure**:
- **Part 1**: First 4KB at offset 4096 in LIF volume
- **Part 2**: Next 1KB, copied from disk to memory
- **Part 3**: Final 1.5KB, copied from disk to memory

#### IPL Operation

From `/sys/arch/hppa/stand/xxboot/main.c`:

1. **Entry from PDC**:
   - `arg0`: Interactive flag (1 = interactive boot, 0 = auto boot)
   - `arg1`: Address past end of IPL part 1
   - Stack initialized by PDC

2. **Determine architecture**:
   ```c
   // Check PSW (Program Status Word) bit 27
   // 0x08000000 = 64-bit mode (PA-RISC 2.0)
   // Otherwise = 32-bit mode (PA-RISC 1.x)
   ```

3. **Read disklabel**:
   - Load BSD disklabel from disk
   - Parse partition table
   - Determine partition offsets

4. **User interaction** (if interactive):
   - Prompt for boot partition (`a`-`h`)
   - Default: partition `a` (root)

5. **Load secondary bootloader**:
   - Read filesystem (FFS or LFS)
   - Locate `/boot` file
   - Load to memory
   - Transfer control

#### Sequential Media Support

HP-PA firmware supports tape drives and other sequential media:

```c
// Check device class from PDC
if ((*(unsigned *)(PZ_MEM_BOOT+DEV_CLASS) & DEV_CL_MASK) == DEV_CL_SEQU) {
    // Sequential device - must read forward or rewind
    // Cannot seek backwards
}
```

**Sequential Media Handling**:
- Read blocks sequentially
- Rewind if earlier block needed
- Cache recent reads to avoid repositioning

### Stage 2: Secondary Bootstrap (boot)

**Location**: `/sys/arch/hppa/stand/boot/`
**Purpose**: Load and execute kernel with full filesystem support

#### Features

From `/sys/arch/hppa/stand/boot/boot.c`:

1. **Full filesystem support**:
   - FFS (Fast File System)
   - LFS (Log-structured File System)
   - CD9660 (ISO 9660)
   - NFS (network boot)

2. **Interactive boot menu**:
   ```
   Boot: [[[dk0a:]netbsd][-a][-c][-d][-s][-v][-q]] :-
   ```

3. **Boot flags**:
   - `-a`: Ask for root device
   - `-c`: User kernel configuration
   - `-d`: Drop to kernel debugger
   - `-s`: Single-user mode
   - `-v`: Verbose boot
   - `-q`: Quiet boot

4. **Kernel search order**:
   ```c
   char *names[] = {
       "netbsd",       "netbsd.gz",
       "netbsd.bak",   "netbsd.bak.gz",
       "netbsd.old",   "netbsd.old.gz",
       "onetbsd",      "onetbsd.gz",
       NULL
   };
   ```

5. **Reset command**: Type `reset` to reboot system

#### Bootinfo Structure

The bootloader passes system information to the kernel via bootinfo:

```c
struct btinfo_common {
    int type;           // Info type identifier
    int next;           // Offset to next item
};

// Bootinfo items:
// BTINFO_KERNELFILE - Kernel filename
// BTINFO_CONSOLE - Console device
// BTINFO_BOOTDEV - Boot device
// BTINFO_HOWTO - Boot flags
```

### Stage 3: Kernel

**Entry Point**: `exec_hppa()` function transfers control
**Format**: ELF32 or ELF64
**Load Address**: Kernel virtual address (varies by PA-RISC version)

#### Kernel Entry Parameters

From `exec_hppa()` in boot code:

The kernel receives:
- **arg0**: Interactive flag
- **arg1**: Reserved/version
- **arg2**: End of bootloader memory
- **arg3**: Boot partition number
- **arg4-arg5**: Reserved (cleared to zero)
- **Bootinfo pointer**: Passed via reserved register

## MMU Setup

### PA-RISC Virtual Memory Architecture

**Page Size**: 4KB (4096 bytes) typical
**Virtual Address**: 32-bit (PA-1.x) or 64-bit (PA-2.0)
**Physical Address**: 32-bit (PA-1.x) or 64-bit (PA-2.0)
**TLB**: Hash-based, software-managed
**Address Spaces**: Space IDs for multiple address spaces

### Space ID and Offset Addressing

PA-RISC uses a unique addressing model with **space IDs**:

```
Virtual Address = (Space ID, Offset)
- Space ID: Identifies address space (0-7 on PA-1.x)
- Offset: 32-bit or 64-bit offset within space
```

**Address Space Usage**:
- **Space 0**: Kernel space
- **Space 1-7**: User spaces (process address spaces)

### TLB (Translation Lookaside Buffer)

**PA-RISC 1.x TLB**:
- **Size**: 96-256 entries (varies by processor)
- **Type**: Fully associative
- **Management**: Software-managed via traps

**PA-RISC 2.0 TLB**:
- **Size**: 160+ entries
- **Type**: Fully associative
- **Features**: Separate I-TLB and D-TLB

### Page Table Structure

PA-RISC uses a **hash-based page table** (HPT):

```
Hash Function:
    hash = (space_id XOR virtual_page_number) & hash_mask

Hash Table Entry (HPTE):
    - Space ID
    - Virtual page number
    - Physical page number
    - Protection bits
    - Valid bit
```

**TLB Miss Handling**:
1. TLB miss trap occurs
2. Software computes hash
3. Search hash table
4. If found, insert into TLB
5. If not found, page fault

### Protection Bits

**PA-RISC Page Protection** (PL0-PL3 privilege levels):

```
Page Protection Field (7 bits):
- Bit 0-1: PL0 access (kernel)
- Bit 2-3: PL1 access
- Bit 4-5: PL2 access
- Bit 6:   PL3 access (user)

Access Types:
00 = No access
01 = Read
10 = Read/Write
11 = Read/Write/Execute
```

**Type Field** (T bit):
- **0**: Normal cacheable page
- **1**: Uncacheable I/O page

**Dirty (D) and Reference (R) bits**:
- Maintained by hardware
- Used for page replacement algorithms

### Kernel Virtual Address Space

**PA-RISC 1.x (32-bit)**:
```
0x00000000 - 0x3FFFFFFF : User space (space 1-7)
0x40000000 - 0xFFFFFFFF : Kernel space (space 0)
```

**PA-RISC 2.0 (64-bit)**:
```
0x0000000000000000 - 0x00000FFFFFFFFFFF : User space
0x0000100000000000 - 0xFFFFFFFFFFFFFFFF : Kernel space
```

**Kernel Segments**:
```
Kernel text:   Read-only, executable (PL0)
Kernel rodata: Read-only, non-executable (PL0)
Kernel data:   Read-write, non-executable (PL0)
Kernel BSS:    Read-write, non-executable (PL0)
```

### Cache Management

**PA-RISC Cache Types**:
- **I-cache**: Instruction cache (16-64KB)
- **D-cache**: Data cache (16-64KB)
- **Cache line size**: 16-64 bytes (typically 16 or 32)

**Cache Operations**:
- `fdc`: Flush data cache line
- `fic`: Flush instruction cache line
- `sync`: Synchronize I/O

**Cache Coherency**:
```asm
fdc  0(0,%r1)       ; Flush data cache at r1
fic,m %r20(0,%r1)   ; Flush I-cache, increment r1
sync                ; Ensure completion
```

## Memory Map

### Physical Memory Layout

```
0x00000000 - 0x00000FFF : PDC reserved (vectors, etc.)
0x00001000 - 0x003CFFFF : PDC firmware
0x003D0000 - 0x003FFFFF : PDC memory boot area (PZ_MEM_BOOT)
0x00400000 - ...        : Available RAM
  IPL loaded here
  Boot program loaded here
  Kernel loaded here
```

### LIF Volume Structure

**LIF (Logical Interchange Format)** - HP's bootable volume format:

```
Offset 0x000: LIF header (256 bytes)
    Magic: 0x8000
    Volume label: "NetBSD" or similar (6 chars)
    ...
Offset 0x0F0: IPL address, size, entry point

Offset 0x800 (2KB): Directory area
    File entries for bootable files

Offset 0x1000 (4KB): IPL part 1 (first 4KB)
Offset 0x2000 (8KB): Data area
    Contains IPL parts 2 and 3
    Contains boot program
    Contains kernel (optional)
```

## Build System

### Source Locations

- **IPL**: `/sys/arch/hppa/stand/xxboot/`
- **Boot**: `/sys/arch/hppa/stand/boot/`
- **CD boot**: `/sys/arch/hppa/stand/cdboot/`
- **Common code**: `/sys/arch/hppa/stand/common/`
- **Utilities**: `/sys/arch/hppa/stand/mkboot/`

### Building Bootloaders

```sh
# Build all bootloaders
cd /sys/arch/hppa/stand
make

# Build IPL
cd /sys/arch/hppa/stand/xxboot
make

# Build secondary boot
cd /sys/arch/hppa/stand/boot
make
```

### Key Makefiles

**`Makefile.buildboot`**: Common build rules for bootloaders
**`Makefile.inc`**: Include paths and compiler flags

**Compiler Flags** (PA-RISC specific):
```makefile
CFLAGS += -mpa-risc-1-0         # PA-RISC 1.0 compatibility
CFLAGS += -fno-stack-protector  # No stack protection
CFLAGS += -ffreestanding        # Freestanding environment
```

## Installation

### Creating LIF Bootable Volume

**Method 1: Using `mkboot` utility**:

```sh
# Create LIF volume on disk
/usr/mdec/mkboot -v /usr/mdec/xxboot /dev/rsd0a

# Install boot program
cp /usr/mdec/boot /boot
```

**Method 2: Manual LIF creation**:

```sh
# Write LIF header and IPL
dd if=/usr/mdec/xxboot of=/dev/rsd0c bs=8k

# Mount filesystem and install boot
mount /dev/sd0a /mnt
cp /usr/mdec/boot /mnt/boot
umount /mnt
```

### Installing on CD-ROM

```sh
# Use cdboot for bootable CD
cp /usr/mdec/cdboot /tmp/

# Create ISO with bootable LIF volume
makefs -t cd9660 -o 'bootimage=hppa;/tmp/cdboot' netbsd.iso /cdroot
```

### Network Boot Setup

**BOOTP Configuration** (`/etc/bootptab`):

```
client:\
    :ht=ether:\
    :ha=080009123456:\
    :ip=192.168.1.100:\
    :sm=255.255.255.0:\
    :gw=192.168.1.1:\
    :bf=/tftpboot/netbsd.hppa:
```

**Boot from network**:
```
Boot from: lan
```

## Debugging

### PDC Console Commands

**Access PDC console**: Press ESC or STOP-A during boot

**Common PDC commands**:
```
Main Menu: Enter command > boot
Main Menu: Enter command > boot pri                    # Primary boot path
Main Menu: Enter command > search                      # Search for boot devices
Main Menu: Enter command > information                 # System information
```

**Boot paths**:
- `pri`: Primary boot path
- `alt`: Alternate boot path
- `lan`: Network boot
- Custom path: e.g., `scsi.5.0`

### Boot Flags

**Interactive boot**:
```
Boot: dk0a:netbsd -s          # Single-user mode
Boot: dk0a:netbsd -d          # Drop to debugger
Boot: dk0a:netbsd -av         # Ask root, verbose
```

### Serial Console

**Configure PDC for serial**:
```
Main Menu: Enter command > path console
Current console path: graphics
New console path: rs232
```

**Serial parameters**: 9600 baud, 8N1 (default)

### Common Boot Issues

**Problem**: "Cannot open boot device"
**Solution**: Check boot path, verify disk is accessible via PDC

**Problem**: "Bad magic number in disklabel"
**Solution**: Recreate disklabel with `disklabel(8)`

**Problem**: "Cannot find /boot"
**Solution**: Copy `/usr/mdec/boot` to root filesystem

**Problem**: IPL checksum error
**Solution**: Rebuild and reinstall xxboot

**Problem**: Sequential media boot slow
**Solution**: Normal for tape, use disk when possible

### Debugging IPL

**Enable verbose output**:
The IPL prints minimal messages. To debug:

1. Check PDC messages before IPL
2. Verify LIF header is correct
3. Use `iplsum` to verify checksum:
   ```sh
   /usr/mdec/iplsum xxboot
   ```

## Platform-Specific Considerations

### PA-RISC Versions

**PA-RISC 1.0** (Early systems):
- 32-bit architecture
- Basic instruction set
- Used in: HP 9000/700 early models

**PA-RISC 1.1** (Most common):
- 32-bit architecture
- Enhanced instructions
- Used in: HP 9000/700 series, HP 9000/800 series
- PA-7100, PA-7150, PA-7200, PA-7300

**PA-RISC 2.0** (High-end):
- 64-bit architecture
- Superscalar, out-of-order execution
- Used in: HP 9000/800 high-end, HP Superdome
- PA-8000, PA-8200, PA-8500, PA-8600, PA-8700

### 32-bit vs 64-bit Mode

**Detection at boot**:
```c
// Check PSW bit 27
if (psw & 0x08000000) {
    // 64-bit mode (PA-RISC 2.0)
} else {
    // 32-bit mode (PA-RISC 1.x)
}
```

**Boot compatibility**:
- 32-bit bootloader can boot on PA-2.0
- Kernel must match architecture

### Byte Order

**Endianness**: Big-endian
**Alignment**: Strict alignment required
**Structure packing**: Natural alignment

### Stack Requirements

**Stack alignment**: 64-byte alignment required
**Stack direction**: Grows towards lower addresses
**Minimum stack**: 64KB for bootloader

### LIF Constraints

**Alignment requirements**:
- IPL start: 2KB aligned
- IPL size: Multiple of 2KB
- IPL entry: Relative to IPL start

**Maximum sizes**:
- IPL: ~16KB total (three parts)
- Boot: Limited by available memory

### Device Naming

**PDC boot paths**:
```
Format: class.target.lun
Example: scsi.5.0  (SCSI target 5, LUN 0)
```

**NetBSD device names**:
```
dk0 = First disk  (SD or SCSI)
dk1 = Second disk
...
```

### Cache Considerations

**Self-modifying code**: Must flush caches
**DMA operations**: Flush D-cache before DMA

**Example** (from IPL):
```asm
fdc  0(0,%r1)           ; Flush data cache
fic,m %r20(0,%r1)       ; Flush instruction cache
sync                    ; Wait for completion
```

### Privilege Levels

PA-RISC has 4 privilege levels (PL0-PL3):
- **PL0**: Kernel mode (most privileged)
- **PL1-PL2**: Unused by NetBSD
- **PL3**: User mode (least privileged)

**Transitions**:
- System calls: PL3 → PL0
- Return: PL0 → PL3

## References

### Source Files

**IPL (xxboot)**:
- `/sys/arch/hppa/stand/xxboot/start.S` - Assembly entry point (LIF header, init)
- `/sys/arch/hppa/stand/xxboot/main.c` - IPL main program
- `/sys/arch/hppa/stand/xxboot/readufs.c` - Filesystem reading code

**Secondary boot**:
- `/sys/arch/hppa/stand/boot/boot.c` - Main boot program
- `/sys/arch/hppa/stand/common/dev_hppa.c` - Device drivers

**Headers**:
- `/sys/arch/hppa/include/pdc.h` - PDC interface definitions
- `/sys/arch/hppa/include/vmparam.h` - Virtual memory parameters

### Man Pages

- `boot(8)` - General boot procedures
- `installboot(8)` - Install bootloader
- `mkboot(8)` - Create LIF boot volume
- `disklabel(8)` - Disk partitioning

### External Documentation

- PA-RISC 1.1 Architecture and Instruction Set Reference Manual
- PA-RISC 2.0 Architecture
- HP-UX documentation (PDC, LIF format)

---

*Last Updated: 2025-11-12*
*Architecture Maintainer: NetBSD/hppa Port*
