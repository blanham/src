# NetBSD/amd64 Boot Documentation

$NetBSD$

## Platform Overview

**Architecture**: AMD x86-64 (Intel 64, x86_64)
**Port Date**: 2001-06-19
**Boot Method**: Multi-stage with prekern (BIOS: MBR → bootxx → boot → prekern → kernel; UEFI: bootx64.efi → prekern → kernel)
**Firmware**: BIOS (legacy), UEFI 64-bit
**MMU Requirements**: Real mode → Protected mode → Long mode transition, 4-level page tables

## Hardware Support

- **CPU**: AMD64, Intel 64 (EM64T) processors
- **Memory**: Full 64-bit address space (48-bit virtual, 52-bit physical on modern CPUs)
- **Boot Devices**: Hard disk, SSD, USB, CD/DVD, network (PXE)
- **Firmware**: IBM PC BIOS (legacy), UEFI 64-bit

## Boot Process

### Stage 0: BIOS/UEFI Firmware

**BIOS Mode**:
1. Power-On Self Test (POST)
2. Boot device selection
3. Load MBR (512 bytes) to 0x7C00
4. Transfer control to MBR

**UEFI Mode**:
1. UEFI firmware initialization
2. Boot manager loads `/EFI/BOOT/bootx64.efi` from ESP
3. EFI boot services provide disk, network, graphics access
4. Transfer control to bootx64.efi

### Stage 1-2: BIOS Boot Chain (if legacy boot)

**Identical to i386** (see i386.md):
- MBR variants (mbr, gptmbr, etc.)
- bootxx filesystem loaders
- Secondary boot program

The amd64 platform reuses i386 bootloaders for BIOS boot mode since the CPU starts in real mode (16-bit) which is identical to i386.

### Stage 3: Prekern (Kernel Preloader)

**Location**: `/sys/arch/amd64/stand/prekern/`
**Purpose**: Transition to 64-bit long mode and set up kernel environment
**Load Address**: 0x100000 (1MB physical)

The prekern is the critical component that differentiates amd64 from i386 boot. It performs:

1. **Mode transition**: 32-bit protected → 64-bit long mode
2. **MMU initialization**: Create 4-level page tables
3. **Kernel relocation**: Support for KASLR (Kernel Address Space Layout Randomization)
4. **ELF loading**: Parse and relocate 64-bit kernel ELF file
5. **Memory management**: Set up virtual address space

#### Prekern Components

**Assembly initialization** (`locore.S` - 640 lines):
- Initial entry point from bootloader
- GDT setup for long mode
- Page table construction
- CPU feature detection (NX bit, RDRAND, RDSEED)
- Mode switching sequence
- Exception handler setup

**Main coordinator** (`prekern.c` - 320 lines):
- API version checking (PREKERN_API_VERSION = 2)
- Bootinfo structure parsing
- Module loading coordination
- Kernel loading and relocation
- Final handoff to kernel

**Memory management** (`mm.c` - 502 lines):
- Physical memory map processing
- Virtual address space layout
- Page table allocation and management
- Identity mapping setup
- Kernel VA randomization (KASLR)
- Segment protection (NX, RO, RW)

**ELF loader** (`elf.c` - 400+ lines):
- ELF64 header parsing
- Program header processing
- Section relocation
- Symbol table handling
- Support for 7 relocation types

**Console output** (`console.c` - 138 lines):
- VGA text mode (80x25)
- Basic character output for debugging

**PRNG for KASLR** (`prng.c` - 229 lines):
- SHA-512 based random number generation
- Entropy sources: RDSEED, RDRAND, entropy file, RDTSC

**Exception handling** (`trap.S` - 196 lines):
- IDT setup
- Exception handlers for debugging prekern issues

### Stage 4: Kernel

**Virtual Load Address**: 0xFFFFFFFF80000000 + random offset (KASLR)
**Entry Point**: Kernel `start` symbol

## MMU Setup

### Initial State (Real Mode)

When BIOS loads the boot sector, CPU is in:
- **16-bit real mode**
- **20-bit addressing** (1MB limit)
- **Segmented memory**
- **MMU disabled**

### Protected Mode (32-bit)

Bootloader transitions to 32-bit protected mode (same as i386):
- **32-bit instructions**
- **4GB address space**
- **Flat memory model** (segments cover full 4GB)
- **GDT** with code/data segments

### Long Mode (64-bit) - Prekern Responsibility

#### Prerequisites

Before entering long mode, prekern must:

1. **Enable PAE** (Physical Address Extension)
   ```
   Set CR4.PAE (bit 5)
   ```

2. **Load CR3** with PML4 (top-level page table)
   ```
   mov cr3, [pml4_address]
   ```

3. **Enable Long Mode** in EFER MSR
   ```
   rdmsr (ECX = 0xC0000080)  ; Read EFER
   or eax, 0x100             ; Set LME (Long Mode Enable)
   wrmsr                     ; Write EFER
   ```

4. **Enable Paging** (CR0.PG) to activate long mode
   ```
   mov eax, cr0
   or eax, 0x80000000        ; Set PG bit
   mov cr0, eax
   ```

5. **Far jump** to 64-bit code segment
   ```
   jmp 0x08:longmode64       ; 0x08 = 64-bit code selector
   ```

#### Page Table Structure

**4-Level Paging** (48-bit virtual address):

```
Virtual Address (48 bits used):
[47:39] - PML4 index (9 bits, 512 entries)
[38:30] - PDPT index (9 bits, 512 entries)
[29:21] - PDT index  (9 bits, 512 entries)
[20:12] - PT index   (9 bits, 512 entries)
[11:0]  - Page offset (12 bits, 4KB page)
```

**Page Table Entry Format** (64-bit):

```
Bit 0:    P   (Present)
Bit 1:    RW  (Read/Write)
Bit 2:    US  (User/Supervisor)
Bit 3:    PWT (Page Write-Through)
Bit 4:    PCD (Page Cache Disable)
Bit 5:    A   (Accessed)
Bit 6:    D   (Dirty, only in PT entries)
Bit 7:    PS  (Page Size, 2MB pages if set in PDT)
Bit 8:    G   (Global)
Bits 9-11:    Available for OS use
Bits 12-51:   Physical address (40 bits = 1TB)
Bits 52-62:   Available
Bit 63:   XD  (Execute Disable / NX bit)
```

#### Memory Regions Created by Prekern

**Identity Mapping** (first 1GB):
```
Virtual 0x00000000 - 0x40000000 → Physical 0x00000000 - 0x40000000
Purpose: Access physical memory during early boot
Permissions: RW (no execute)
```

**Kernel Mapping** (high canonical address):
```
Virtual 0xFFFFFFFF80000000 + offset → Physical (kernel load location)
Purpose: Kernel text, data, bss
Permissions: Text = RX, Rodata = RO, Data = RW
Offset: Random (KASLR), 2GB window
```

**Bootstrap Page Tables**:
- ~60 pages allocated (~245 KB)
- Covers: PML4, PDPTs, PDTs, PTs for identity and kernel regions

#### KASLR (Kernel Address Space Layout Randomization)

**Randomization Window**: 2GB (0xFFFFFFFF80000000 - 0x0000000100000000)

**Entropy Sources**:
1. **RDSEED** instruction (if CPU supports)
2. **RDRAND** instruction (if CPU supports)
3. **Entropy file** (from bootloader)
4. **RDTSC** (Time Stamp Counter as fallback)

**PRNG Algorithm**: SHA-512 based
- Collects entropy from all available sources
- Produces cryptographically strong random offset
- Aligns to 2MB boundary for large pages

**Security Benefits**:
- Prevents exploitation of kernel memory corruption bugs
- Makes ROP (Return-Oriented Programming) attacks harder
- Each boot has different kernel addresses

#### Page Attributes

**NX/XD Bit Support**:
- Checked via CPUID extended features
- If available, data pages marked non-executable
- Text pages executable, data/stack not executable

**Segment Protection**:
```
Kernel text:    Present, Read-only, Execute (NX clear)
Kernel rodata:  Present, Read-only, No-execute (NX set)
Kernel data:    Present, Read-write, No-execute (NX set)
Kernel bss:     Present, Read-write, No-execute (NX set)
```

### GDT for Long Mode

**Location**: `/sys/arch/amd64/stand/prekern/locore.S`

```c
gdt:
    .quad 0x0000000000000000  /* Null descriptor */
    .quad 0x00AF9A000000FFFF  /* Code segment (0x08) */
    .quad 0x00CF92000000FFFF  /* Data segment (0x10) */
```

**Code Segment (0x08)**:
- L bit set (64-bit code segment)
- D bit clear (required for 64-bit)
- Base = 0, Limit = ignored in long mode

**Data Segment (0x10)**:
- Base = 0, Limit = ignored in long mode
- Segmentation not used in 64-bit mode (flat model)

## ELF Kernel Loading

### Supported Relocation Types

**From `/sys/arch/amd64/stand/prekern/elf.c`**:

1. **R_X86_64_NONE** (0) - No relocation
2. **R_X86_64_64** (1) - 64-bit absolute address
   ```
   *location = symbol_value + addend
   ```

3. **R_X86_64_PC32** (2) - 32-bit PC-relative
   ```
   *location = symbol_value + addend - location_address
   ```

4. **R_X86_64_32** (10) - Zero-extended 32-bit
   ```
   *location = (symbol_value + addend) & 0xFFFFFFFF
   ```

5. **R_X86_64_GLOB_DAT** (6) - GOT entry for global data
   ```
   *location = symbol_value
   ```

6. **R_X86_64_JUMP_SLOT** (7) - PLT entry for functions
   ```
   *location = symbol_value
   ```

7. **R_X86_64_RELATIVE** (8) - Relative to base address
   ```
   *location = base_address + addend
   ```

### Kernel Load Sequence

1. **Parse ELF header** - Verify magic, class, machine type
2. **Process program headers** - Identify PT_LOAD segments
3. **Allocate memory** - Reserve VA space at randomized base
4. **Copy segments** - Load from ELF file to memory
5. **Apply relocations** - Fix up addresses for new base
6. **Set protections** - Mark text RX, rodata RO, data RW

## Bootinfo Structure

**Passed from bootloader to prekern to kernel**:

```c
struct bootinfo {
    uint32_t bi_version;      /* Version number */
    uint32_t bi_kern_start;   /* Kernel start (physical) */
    uint32_t bi_kern_size;    /* Kernel size */
    uint32_t bi_root_start;   /* Ramdisk start */
    uint32_t bi_root_size;    /* Ramdisk size */
    uint32_t bi_memmap_start; /* Memory map pointer */
    uint32_t bi_memmap_size;  /* Memory map size */
    uint32_t bi_modules;      /* Loaded modules */
    uint32_t bi_sym_start;    /* Symbol table start */
    uint32_t bi_sym_size;     /* Symbol table size */
    /* Additional fields for amd64 */
    uint64_t bi_flags;        /* Feature flags */
    uint64_t bi_kern_virt;    /* Kernel virtual address */
};
```

**API Version**: PREKERN_API_VERSION = 2

## Differences from i386

| Feature | i386 | amd64 |
|---------|------|-------|
| Address Space | 32-bit (4GB) | 64-bit (256TB virtual) |
| Kernel Address | Fixed 0xC0100000 | Randomized 0xFFFFFFFF80000000 + offset |
| Page Tables | 2-level (PAE: 3-level) | 4-level mandatory |
| KASLR | Limited, optional | Full, 2GB window |
| NX Bit | Optional (PAE) | Standard (if CPU supports) |
| Prekern | Not used | Required |
| Boot Stages | 3 (MBR, bootxx, boot) | 4 (MBR, bootxx, boot, prekern) |
| Relocations | Minimal | Full kernel relocation |
| Mode Switches | Real → Protected | Real → Protected → Long |

## Memory Map

### Physical Memory

```
0x00000000 - 0x000003FF : Interrupt Vector Table (IVT)
0x00000400 - 0x000004FF : BIOS Data Area
0x00000500 - 0x00007BFF : Free low memory
0x00007C00 - 0x00007DFF : Boot sector (MBR)
0x00008800 - 0x00008FFF : PBR
0x00001000 - 0x00001FFF : bootxx
0x00010000 - 0x0003FFFF : boot (secondary bootloader)
0x00040000 - 0x00070000 : boot heap
0x00080000 - 0x0009FFFF : Extended BIOS Data Area
0x000A0000 - 0x000BFFFF : VGA memory
0x000C0000 - 0x000FFFFF : BIOS ROM
0x00100000 - ...        : Prekern load area (1MB+)
Kernel loaded:           : Kernel ELF file
```

### Virtual Address Space (after prekern)

```
0x0000000000000000 - 0x00007FFFFFFFFFFF : User space (128TB)
0x0000800000000000 - 0xFFFF7FFFFFFFFFFF : Non-canonical (causes #GP)
0xFFFF800000000000 - 0xFFFFFFFF7FFFFFFF : Kernel space (128TB)
0xFFFFFFFF80000000 - 0xFFFFFFFFFFFFFFFF : Direct map + kernel (2GB)
```

**Kernel Regions**:
```
0xFFFFFFFF80000000 + KASLR_offset : Kernel text (RX)
                                   : Kernel rodata (RO)
                                   : Kernel data (RW)
                                   : Kernel BSS (RW)
```

## UEFI Boot

**Location**: `/sys/arch/amd64/stand/efiboot/`

### UEFI Boot Flow

1. **UEFI Firmware** loads `bootx64.efi` from ESP partition
2. **EFI Boot Services**:
   - `LocateProtocol()` - Find required protocols (disk, network, graphics)
   - `LoadImage()` - Load prekern
   - `GetMemoryMap()` - Obtain memory layout
3. **Exit Boot Services** - Transfer control to prekern
4. **Prekern** - Standard prekern flow
5. **Kernel** - Boot continues

### UEFI Advantages

- **No BIOS limitations**: LBA48+ disk support, GPT native
- **Modern graphics**: GOP (Graphics Output Protocol) for framebuffer
- **Network boot**: HTTP, HTTPS boot support
- **Secure Boot**: Code signature verification (if enabled)
- **Faster**: Direct filesystem access without BIOS interrupts

### UEFI Memory Map

UEFI provides detailed memory map with types:
- **EfiConventionalMemory** - Available RAM
- **EfiLoaderCode** - Bootloader code
- **EfiLoaderData** - Bootloader data
- **EfiBootServicesCode** - Firmware code (reclaimable)
- **EfiBootServicesData** - Firmware data (reclaimable)
- **EfiRuntimeServicesCode** - Runtime firmware code (persistent)
- **EfiRuntimeServicesData** - Runtime firmware data (persistent)
- **EfiACPIReclaimMemory** - ACPI tables (reclaimable)
- **EfiACPIMemoryNVS** - ACPI NVS (persistent)
- **EfiReserved** - Reserved memory

## Build System

### Source Locations

- **Prekern**: `/sys/arch/amd64/stand/prekern/`
- **BIOS boot**: `/sys/arch/i386/stand/` (reused)
- **UEFI boot**: `/sys/arch/amd64/stand/efiboot/`
- **Shared libraries**: `/sys/lib/libsa/`, `/sys/lib/libkern/`

### Compilation

**Prekern Makefile**: `/sys/arch/amd64/stand/prekern/Makefile`

```makefile
PROG=prekern
SRCS=locore.S prekern.c mm.c elf.c console.c prng.c trap.S
CFLAGS+=-ffreestanding -fno-stack-protector
CFLAGS+=-mno-red-zone  # No red zone (128-byte stack offset)
CFLAGS+=-mcmodel=large # Large code model for high addresses
LDFLAGS+=-Wl,-T,prekern.ldscript
```

**Linker Script**: Places prekern at 0x100000 (1MB)

## Installation

### BIOS Boot

```sh
# Install bootloader (same as i386)
installboot /dev/rwd0a /usr/mdec/bootxx_ffsv2
cp /usr/mdec/boot /boot
```

### UEFI Boot

```sh
# Create EFI System Partition (ESP)
gpt create wd0
gpt add -t efi -s 256m wd0      # Create ESP

# Format as FAT32
newfs_msdos /dev/rwd0a

# Install bootloader
mount -t msdos /dev/wd0a /mnt
mkdir -p /mnt/EFI/BOOT
cp /usr/mdec/bootx64.efi /mnt/EFI/BOOT/
umount /mnt
```

## Debugging

### Serial Console

**Enable in boot.cfg**:
```
consdev com0
```

**Kernel console**:
```
boot -h netbsd
```

**Serial Parameters**: 115200 8N1, COM1 (0x3F8)

### Prekern Debugging

**Enable verbose output**: Modify `prekern.c`, rebuild

**Common issues**:
- **Hang after bootloader**: Page table setup error
- **Triple fault**: Invalid GDT or IDT
- **#GP (General Protection)**: Non-canonical address access
- **#PF (Page Fault)**: Incorrect page table mappings

**Debugging via QEMU**:
```sh
qemu-system-x86_64 -s -S -hda disk.img
gdb /path/to/prekern
(gdb) target remote :1234
(gdb) break prekern_entry
(gdb) continue
```

## Troubleshooting

### UEFI vs BIOS Mode

**Check boot mode**:
```sh
dmesg | grep -i "boot"
```

Output will show "BIOS boot" or "UEFI boot"

### Secure Boot Issues

If Secure Boot is enabled:
- NetBSD bootloader must be signed
- Or disable Secure Boot in firmware settings

### Memory Issues

**Insufficient memory**:
- Minimum: 32MB (with BIOS)
- Recommended: 512MB+
- Large kernels may require more memory for prekern

### KASLR Problems

**Disable KASLR** (for debugging):
```
boot -o nokaslr netbsd
```

## Platform-Specific Considerations

### CPU Requirements

- **Minimum**: AMD K8 (Athlon 64), Intel Core 2
- **Features**:
  - Long mode (64-bit) support
  - NX/XD bit (recommended)
  - RDRAND/RDSEED (for better KASLR entropy)

### Large Memory Systems

- **Physical Address Extension**: Up to 52-bit physical addresses
- **5-level paging**: Support planned (Intel Ice Lake+, 57-bit VA)
- **Memory above 4GB**: Fully supported

### Virtualization

**Works in**:
- VMware ESXi, Workstation, Fusion
- VirtualBox
- QEMU/KVM
- Hyper-V
- Xen (both HVM and PVH modes)

**UEFI boot recommended** for modern VMs

## References

### Source Files

**Critical prekern files**:
- `/sys/arch/amd64/stand/prekern/locore.S` - Assembly init (640 lines)
- `/sys/arch/amd64/stand/prekern/prekern.c` - Main coordinator (320 lines)
- `/sys/arch/amd64/stand/prekern/mm.c` - Memory management (502 lines)
- `/sys/arch/amd64/stand/prekern/elf.c` - ELF loader (400+ lines)
- `/sys/arch/amd64/stand/prekern/prng.c` - KASLR PRNG (229 lines)

### Man Pages

- `boot(8)` - General boot procedures
- `boot_amd64(8)` - amd64-specific boot documentation
- `installboot(8)` - Install bootloader
- `gpt(8)` - GUID Partition Table editor

### External Documentation

- AMD64 Architecture Programmer's Manual
- Intel 64 and IA-32 Architectures Software Developer Manuals
- UEFI Specification v2.8+

---

*Last Updated: 2025-11-12*
*Architecture Maintainer: NetBSD/amd64 Port*
