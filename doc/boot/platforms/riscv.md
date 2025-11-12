# NetBSD/riscv Boot Documentation

$NetBSD$

## Platform Overview

**Architecture**: RISC-V (RV32, RV64, RV128)
**Port Date**: 2014-09-19 (initial), significant updates through 2020s
**Boot Method**: External bootloader → Kernel (U-Boot, OpenSBI, or direct firmware)
**Firmware**: OpenSBI (Supervisor Binary Interface), U-Boot, or vendor-specific firmware
**MMU Requirements**: Sv32 (RV32), Sv39/Sv48 (RV64), page-based MMU

## Hardware Support

- **CPU**: RISC-V RV32GC, RV64GC processors
- **Memory**: 32-bit (RV32) or 64-bit (RV64) addressing
- **Boot Devices**: SD card, eMMC, SPI flash, network, UART (varies by board)
- **Firmware**: OpenSBI + U-Boot (most common), vendor firmware
- **Systems**: SiFive boards, StarFive VisionFive, QEMU virt machine, various FPGA implementations

### Supported Hardware

**SiFive Boards**:
- **HiFive Unmatched**: SiFive Freedom U740 (RV64GC, 5 cores)
- **HiFive Unleashed**: SiFive Freedom U540 (RV64GC, 5 cores)

**StarFive Boards**:
- **VisionFive**: JH7100 SoC
- **VisionFive 2**: JH7110 SoC (RV64GC)

**QEMU**:
- **virt machine**: Generic RISC-V virtual platform
- **sifive_u**: SiFive Unleashed emulation

**Other**:
- Various FPGA implementations
- Custom RISC-V designs

## Boot Process

### Stage 0: Reset and Firmware

RISC-V systems start in Machine mode (M-mode), the highest privilege level:

1. **Reset vector**: CPU starts at implementation-defined address
2. **M-mode firmware**: First-stage bootloader (ROM or flash)
3. **Load next stage**: Typically OpenSBI firmware

### Stage 1: OpenSBI (Supervisor Binary Interface)

**OpenSBI** provides a standard interface between firmware and OS:

**Responsibilities**:
- Initialize hardware
- Set up M-mode trap handlers
- Provide SBI services to Supervisor mode
- Load and execute next stage (U-Boot or kernel)

**SBI Services** (used by NetBSD kernel):
```c
// Console I/O
SBI_CONSOLE_PUTCHAR
SBI_CONSOLE_GETCHAR

// Timer
SBI_SET_TIMER

// Inter-processor interrupts
SBI_SEND_IPI
SBI_CLEAR_IPI

// Remote fence operations
SBI_REMOTE_FENCE_I
SBI_REMOTE_SFENCE_VMA

// System operations
SBI_SHUTDOWN
```

### Stage 2: U-Boot (Typical Configuration)

Most RISC-V boards use **U-Boot** as secondary bootloader:

**U-Boot Boot Sequence**:

1. **Loaded by OpenSBI** or M-mode firmware
2. **Initialize devices**:
   - Console (UART)
   - Storage (SD, eMMC, SPI)
   - Network (Ethernet)
3. **Load environment**: Boot configuration from storage
4. **Boot menu** or autoboot countdown
5. **Load kernel**: From storage or network
6. **Transfer control**: Jump to kernel entry point

**U-Boot Commands**:
```
=> printenv              # Show environment variables
=> setenv bootargs "console=ttyS0,115200"
=> load mmc 0:1 ${kernel_addr_r} /netbsd
=> booti ${kernel_addr_r} - ${fdt_addr_r}

# Or simplified
=> boot
```

### Stage 3: Direct Kernel Boot

NetBSD/riscv kernel can boot directly from firmware:

**From `/sys/arch/riscv/riscv/locore.S`**:

```asm
ENTRY_NP(start)
    csrw    sie, zero           // Disable interrupts
    csrw    sip, zero           // Clear pending interrupts

    li      s0, SR_FS
    csrc    sstatus, s0         // Disable FP

    mv      s10, a0             // Save hartid (hardware thread ID)
    mv      s11, a1             // Save DTB pointer (physical address)

    PTR_LA  t0, bootstk
    mv      sp, t0              // Set stack pointer

    // Calculate VA to PA offset
    PTR_LA  t0, start
    PTR_L   s8, .Lstart
    sub     s8, s8, t0          // s8 = vtopdiff

    // Detect CPU vendor for quirks
    li      a7, SBI_EID_BASE
    li      a6, SBI_FID_BASE_GETMVENDORID
    ecall                       // SBI call to get vendor ID
    // Handle T-Head XMAE extension if needed

    // Build initial page tables
    // ... (page table setup code)

    // Enable MMU
    // ... (MMU enable code)

    // Jump to kernel main
```

**Entry Arguments** (from bootloader):
- **a0**: Hart ID (hardware thread ID, typically 0 for boot hart)
- **a1**: Device Tree Blob (DTB) physical address

### No NetBSD-Specific Bootloader

NetBSD/riscv **does not include** its own bootloader:

**Why?**
1. **Ecosystem standard**: RISC-V ecosystem uses OpenSBI + U-Boot
2. **Hardware diversity**: Many board vendors provide firmware
3. **Flexibility**: Different boot methods for different platforms
4. **Maintenance**: Leverage existing, well-maintained bootloaders

**Result**: `/sys/arch/riscv/stand/` contains only Makefiles for installation, no bootloader code.

## MMU Setup

### RISC-V Virtual Memory

**RV32**:
- **Sv32**: 2-level page tables, 32-bit virtual addresses

**RV64**:
- **Sv39**: 3-level page tables, 39-bit virtual addresses (most common)
- **Sv48**: 4-level page tables, 48-bit virtual addresses
- **Sv57**: 5-level page tables, 57-bit virtual addresses (rare)

### Sv39 Page Tables (RV64 most common)

```
Virtual Address (39 bits used of 64):
[63:39] - Sign extension (must match bit 38)
[38:30] - Level 2 index (VPN[2], 9 bits, 512 entries)
[29:21] - Level 1 index (VPN[1], 9 bits, 512 entries)
[20:12] - Level 0 index (VPN[0], 9 bits, 512 entries)
[11:0]  - Page offset (12 bits, 4KB page)
```

### Page Table Entry (PTE) Format

**64-bit PTE** (Sv39/Sv48):

```
Bits [63:54] - Reserved (for OS use, Svpbmt, Svnapot extensions)
Bits [53:28] - PPN[2] (Physical Page Number high)
Bits [27:19] - PPN[1]
Bits [18:10] - PPN[0] (26 bits total = 56-bit physical address)
Bits [9:8]   - RSW (Reserved for Software)
Bit  [7]     - D (Dirty)
Bit  [6]     - A (Accessed)
Bit  [5]     - G (Global)
Bit  [4]     - U (User)
Bit  [3]     - X (eXecute)
Bit  [2]     - W (Write)
Bit  [1]     - R (Read)
Bit  [0]     - V (Valid)
```

**Permission Bits**:
- `V=0`: Invalid entry (page fault)
- `V=1, R=0, W=0, X=0`: Pointer to next level page table
- `V=1, R=1`: Readable page
- `V=1, W=1`: Writable page (must also have R=1)
- `V=1, X=1`: Executable page

**Additional Bits**:
- **U**: User accessible (0 = supervisor only)
- **G**: Global mapping (not flushed by ASID change)
- **A**: Accessed (set by hardware on access)
- **D**: Dirty (set by hardware on write)

### Initial Page Table Setup

From `locore.S`, kernel creates identity mapping and kernel mapping:

1. **Identity mapping**: PA 0 → VA 0 (for early boot)
2. **Kernel mapping**: Kernel PA → Kernel VA (typically high memory)

```
// Typical memory layout
Physical: 0x80000000 - 0x8FFFFFFF (256MB example)
Virtual:  0xFFFFFFFFC0000000+ (kernel high memory, RV64)
```

### Enabling MMU

**satp Register** (Supervisor Address Translation and Protection):

```
RV64 satp format:
Bits [63:60] - MODE
  0000 = Bare (no translation)
  1000 = Sv39
  1001 = Sv48
  1010 = Sv57
Bits [59:44] - ASID (Address Space ID, 16 bits)
Bits [43:0]  - PPN (Physical Page Number of root page table)
```

**Enable MMU** (from locore.S):
```asm
    // Construct satp value
    li      t0, SATP_MODE_SV39  // or SATP_MODE_SV48
    slli    t0, t0, 60
    or      t0, t0, s9          // s9 = root page table PPN

    sfence.vma                  // Flush TLB
    csrw    satp, t0            // Enable MMU
    sfence.vma                  // Flush again
```

### TLB Management

RISC-V TLB is managed via:

**Instructions**:
- `sfence.vma`: Flush TLB (all or specific)
- `sfence.vma rs1, rs2`: Flush specific VA and ASID

**Usage**:
```asm
sfence.vma zero, zero   # Flush all TLB entries
sfence.vma a0, zero     # Flush entries for VA in a0
sfence.vma zero, a1     # Flush entries for ASID in a1
sfence.vma a0, a1       # Flush specific VA and ASID
```

### Supervisor Mode

RISC-V privilege levels:
- **M-mode**: Machine mode (firmware, OpenSBI)
- **S-mode**: Supervisor mode (kernel)
- **U-mode**: User mode (applications)

NetBSD kernel runs in S-mode, relies on M-mode (OpenSBI) for:
- Timer interrupts
- Inter-processor interrupts
- Hardware control

## Memory Map

### Physical Memory (Board-Dependent)

**Typical RISC-V SoC** (e.g., SiFive U740):
```
0x00000000 - 0x000000FF : Debug area
0x00001000 - 0x00001FFF : ROM (boot code)
0x00100000 - 0x00100FFF : CLINT (timer)
0x0C000000 - 0x0FFFFFFF : PLIC (interrupt controller)
0x10000000 - 0x10000FFF : UART0
0x10010000 - 0x10010FFF : UART1
...
0x80000000 - 0xFFFFFFFF : RAM (2GB+, varies by board)
```

### Virtual Address Space

**RV64 (Sv39)**:
```
0x0000000000000000 - 0x0000003FFFFFFFFF : User space (256GB)
0x0000004000000000 - 0xFFFFFFBFFFFFFFFF : (non-canonical, invalid)
0xFFFFFFC000000000 - 0xFFFFFFFFFFFFFFFF : Kernel space (256GB)
```

**Kernel Layout**:
```
0xFFFFFFFFC0000000 : Kernel text (loaded here typically)
                   : Kernel rodata
                   : Kernel data
                   : Kernel BSS
...                : Dynamic allocations
```

## Build System

### Source Locations

- **Kernel**: `/sys/arch/riscv/riscv/`
- **Board support**: `/sys/arch/riscv/fdt/`, `/sys/arch/riscv/sifive/`, `/sys/arch/riscv/starfive/`
- **Device tree**: `/sys/arch/riscv/dts/`
- **Stand**: `/sys/arch/riscv/stand/` (minimal, for installation only)

### Building Kernel

```sh
# Build RV64 kernel
cd /usr/src
./build.sh -m riscv -a riscv64 kernel=GENERIC64

# Result: sys/arch/riscv/compile/GENERIC64/netbsd
```

### Kernel Configurations

- **GENERIC64**: RV64 generic kernel
- **RPI**: Raspberry Pi (future)
- **QEMU**: QEMU virt machine

## Installation

### Preparing Boot Media

**On target board or another system**:

```sh
# Partition SD card (example)
gpt create sd0
gpt add -t efi -s 128m sd0     # For U-Boot/firmware
gpt add -t ffs -s 8g sd0       # For root filesystem

# Create filesystems
newfs /dev/rsd0b               # Root partition

# Mount and extract
mount /dev/sd0b /mnt
cd /mnt
tar xzpf /path/to/base.tgz
tar xzpf /path/to/etc.tgz
# ... other sets

# Copy kernel
cp /path/to/netbsd /mnt/

# Unmount
umount /mnt
```

### U-Boot Configuration

**Create `/boot/boot.scr.txt`** on boot partition:
```
setenv bootargs "root=ld0a console=ttyS0,115200"
fatload mmc 0:1 ${kernel_addr_r} /netbsd
booti ${kernel_addr_r} - ${fdt_addr_r}
```

**Compile boot script**:
```sh
mkimage -T script -C none -d boot.scr.txt boot.scr
```

### Direct Kernel Boot (OpenSBI)

Some systems can boot kernel directly from OpenSBI:

```
FW_PAYLOAD_PATH=/path/to/netbsd make
```

## Debugging

### QEMU

**Run NetBSD/riscv in QEMU**:

```sh
# RV64
qemu-system-riscv64 \
    -machine virt \
    -smp 4 \
    -m 2G \
    -bios /usr/share/qemu/opensbi-riscv64-generic-fw_dynamic.bin \
    -kernel /path/to/netbsd \
    -append "root=ld0a console=ttyS0" \
    -drive file=disk.img,format=raw,id=hd0 \
    -device virtio-blk-device,drive=hd0 \
    -netdev user,id=net0 \
    -device virtio-net-device,netdev=net0 \
    -serial stdio
```

### Serial Console

**Enable serial console**:

**U-Boot**:
```
setenv bootargs "root=ld0a console=ttyS0,115200"
```

**Serial parameters**: 115200 baud, 8N1 (typical)

### Debugging with GDB

**QEMU with GDB**:
```sh
qemu-system-riscv64 -s -S ... # -s = gdbserver on :1234, -S = wait
```

**Connect GDB**:
```sh
riscv64-unknown-elf-gdb /path/to/netbsd
(gdb) target remote :1234
(gdb) break start
(gdb) continue
```

### Common Boot Issues

**Problem**: Kernel doesn't start
**Solution**: Check DTB is passed correctly, verify OpenSBI version

**Problem**: "Unhandled exception"
**Solution**: Ensure MMU setup is correct, check page table entries

**Problem**: SBI call failures
**Solution**: Update OpenSBI firmware, check M-mode firmware

**Problem**: Device not detected
**Solution**: Verify device tree, check driver support

## Platform-Specific Considerations

### RISC-V ISA Extensions

**Base ISA**:
- **I**: Integer base
- **M**: Multiplication/division
- **A**: Atomic operations
- **F**: Single-precision floating-point
- **D**: Double-precision floating-point
- **C**: Compressed instructions

**Common combinations**:
- **RV64GC**: RV64IMAFD + C (general-purpose 64-bit)
- **RV32GC**: RV32IMAFD + C (general-purpose 32-bit)

**Optional extensions**:
- **Zicsr**: CSR instructions (standard now)
- **Zifencei**: Instruction fence (standard now)
- **Svpbmt**: Page-based memory types
- **Svnapot**: NAPOT translation contiguity
- **Sstc**: Supervisor-mode timer interrupts

### Vendor-Specific Extensions

**T-Head XMAE** (Memory Attribute Extension):

From `locore.S`, NetBSD detects and handles:
```c
// T-Head XMAE extension
PTE_XMAE_PMA     // Regular memory
PTE_XMAE_IO      // I/O devices
```

This allows finer-grained memory attribute control.

### Hart (Hardware Thread) Support

RISC-V calls CPU cores/threads "harts":

**Boot process**:
1. Hart 0 (boot hart) starts first
2. Builds page tables, initializes kernel
3. Wakes secondary harts via SBI

**SMP Support**:
```c
SBI_SEND_IPI      // Wake up hart
SBI_REMOTE_FENCE  // Synchronize TLBs
```

### Byte Order

**Endianness**: Little-endian (standard)
**Alignment**: Misaligned access supported (may be slow)

### Cache Management

**I-cache flush**: `fence.i` instruction
**D-cache**: Usually coherent, but some platforms need manual flush

## Ecosystem Integration

### Device Tree (FDT)

RISC-V uses Flattened Device Tree for hardware description:

**Passed to kernel**:
- In `a1` register at entry
- Contains memory map, devices, clocks, etc.

**NetBSD parses**:
- Memory regions
- UART for console
- Timers, interrupt controllers
- Device drivers

### SBI (Supervisor Binary Interface)

**Version**: SBI 0.2+ (extensible)

**Mandatory extensions**:
- Base
- Timer
- IPI
- RFENCE
- HSM (Hart State Management)

**Optional extensions**:
- PMU (Performance monitoring)
- DBCN (Debug console)

## References

### NetBSD Source

**Core files**:
- `/sys/arch/riscv/riscv/locore.S` - Boot entry, MMU setup
- `/sys/arch/riscv/riscv/pmap.c` - Memory management
- `/sys/arch/riscv/riscv/trap.c` - Exception handling
- `/sys/arch/riscv/riscv/cpu.c` - CPU initialization

**Board support**:
- `/sys/arch/riscv/fdt/` - FDT-based platform support
- `/sys/arch/riscv/sifive/` - SiFive board support
- `/sys/arch/riscv/starfive/` - StarFive board support

### External Documentation

**RISC-V Specifications**:
- RISC-V ISA Manual (Unprivileged and Privileged)
- RISC-V SBI Specification
- RISC-V Platform Specification

**Bootloaders**:
- OpenSBI documentation: https://github.com/riscv/opensbi
- U-Boot RISC-V: https://u-boot.readthedocs.io

**Hardware**:
- SiFive HiFive Unleashed/Unmatched documentation
- StarFive VisionFive documentation
- QEMU RISC-V: https://www.qemu.org/docs/master/system/target-riscv.html

### Man Pages

- `boot(8)` - General boot procedures
- `fdt(4)` - Flattened Device Tree

---

*Last Updated: 2025-11-12*
*Architecture Maintainer: NetBSD/riscv Port*
