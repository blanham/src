# NetBSD/evbarm Boot Documentation

$NetBSD$

## Platform Overview

**Architecture**: ARM (32-bit) and AArch64 (64-bit)
**Port Date**: 2001-09-05
**Boot Method**: Highly variable by board (firmware, U-Boot, gzboot, board-specific)
**Firmware**: U-Boot, UEFI, board-specific ROM
**MMU Requirements**: ARM MMU initialization, cache setup, often FDT (Device Tree) based

## Hardware Support

evbarm is a meta-platform supporting **84+ ARM evaluation boards** spanning:
- ARMv4, ARMv5, ARMv6, ARMv7-A, ARMv8-A (AArch64)
- Various SoCs: Allwinner, Amlogic, Broadcom, Marvell, Nvidia, NXP, Qualcomm, Rockchip, Samsung, Texas Instruments, Xilinx

## Boot Process

### Overview

evbarm boot process varies significantly by board. Common patterns:

1. **ROM Boot** → **U-Boot** → **Kernel** (most common)
2. **ROM Boot** → **gzboot** → **Kernel** (compressed kernel bootloader)
3. **UEFI** → **UEFI bootloader** → **Kernel** (modern ARM servers)
4. **ROM Boot** → **Kernel** (direct boot, simple boards)

### Device Tree (FDT) Boot

**Modern approach** for most evbarm boards:

**Source**: `/sys/arch/evbarm/fdt/`

**Boot sequence**:
1. **Firmware/U-Boot** loads device tree blob (DTB)
2. **DTB passed to kernel** in register r2 (ARM convention)
3. **Kernel parses FDT** for:
   - Memory layout (`/memory` nodes)
   - CPU configuration (`/cpus` nodes)
   - Device bindings (via `compatible` strings)
   - Boot arguments (`/chosen/bootargs`)
   - Interrupt controller, clocks, etc.

**U-Boot commands**:
```
fatload mmc 0:1 ${kernel_addr_r} netbsd.ub
fatload mmc 0:1 ${fdt_addr_r} board.dtb
bootm ${kernel_addr_r} - ${fdt_addr_r}
```

### gzboot - Compressed Kernel Bootloader

**Location**: `/sys/arch/evbarm/stand/gzboot/`

**Purpose**: Decompress gzipped kernel and boot it

**Features**:
- **Small size**: Fits in limited boot flash/ROM
- **zlib decompression**: Inflate gzip-compressed kernels
- **MMU initialization**: Set up ARM MMU before decompression
- **Cache management**: Flush caches appropriately

**Build**:
```sh
cd /sys/arch/evbarm/stand/gzboot
make
# Creates gzboot_{platform}.bin
```

**Usage**:
1. Compress kernel: `gzip -9 netbsd` → `netbsd.gz`
2. Link with gzboot: Creates bootable image
3. Flash to board or load via U-Boot

**MMU Setup in gzboot** (`srtbegin.S`):
```assembly
; 1. Disable MMU
mrc p15, 0, r2, c1, c0, 0      ; Read CP15 control register
bic r2, r2, #0x01              ; Clear MMU enable bit
mcr p15, 0, r2, c1, c0, 0      ; Write back

; 2. Memory relocation (if in ROM)
; Check if running from ROM, copy to RAM if needed

; 3. Clear BSS

; 4. Setup stack
add sp, r1, #STACKSIZE

; 5. Call main() - decompress and boot kernel
bl main
```

## MMU Initialization

### ARM MMU Overview

ARM MMU provides:
- **Virtual to physical** address translation
- **Access permissions** (read/write, user/supervisor)
- **Cache policies** (cacheable, bufferable)
- **Execute Never (XN)** protection

### Page Table Formats

#### ARMv7 Short Descriptor (32-bit)

**Two-level page tables**:
- **L1 Table** (Translation Table): 4096 entries × 4 bytes = 16KB
  - Each entry covers 1MB (section) or points to L2 table
- **L2 Table** (Page Table): 256 entries × 4 bytes = 1KB
  - Each entry covers 4KB (small page)

**L1 Section Entry (1MB pages)**:
```
[31:20] - Section base address
[19]    - NS (Non-Secure)
[18]    - 0 (section)
[17]    - nG (not Global)
[16]    - S (Shareable)
[15]    - APX (Access Permission extension)
[14:12] - TEX (Type Extension)
[11:10] - AP (Access Permission)
[9]     - IMP (Implementation defined)
[8:5]   - Domain
[4]     - XN (Execute Never)
[3]     - C (Cacheable)
[2]     - B (Bufferable)
[1]     - 1 (section marker)
[0]     - PXN (Privileged Execute Never, ARMv7)
```

#### AArch64 Long Descriptor (64-bit)

**Four-level page tables** (similar to amd64):
- L0 (512GB blocks)
- L1 (1GB blocks)
- L2 (2MB blocks)
- L3 (4KB pages)

### CP15 Coprocessor Registers

**Critical ARM MMU control registers**:

**c1 - Control Register**:
```c
#define CPU_CONTROL_MMU_ENABLE      0x00000001  /* M: Enable MMU */
#define CPU_CONTROL_AFLT_ENABLE     0x00000002  /* A: Alignment fault */
#define CPU_CONTROL_DC_ENABLE       0x00000004  /* C: Data cache */
#define CPU_CONTROL_WBUF_ENABLE     0x00000008  /* W: Write buffer */
#define CPU_CONTROL_32BP_ENABLE     0x00000010  /* P: 32-bit exception */
#define CPU_CONTROL_32BD_ENABLE     0x00000020  /* D: 32-bit addressing */
#define CPU_CONTROL_LABT_ENABLE     0x00000040  /* L: Late abort */
#define CPU_CONTROL_IC_ENABLE       0x00001000  /* I: Instruction cache */
#define CPU_CONTROL_V_ENABLE        0x00002000  /* V: High vectors */
```

**c2 - Translation Table Base Register (TTBR)**:
- TTBR0: User space page tables
- TTBR1: Kernel space page tables

**c3 - Domain Access Control Register**:
- 16 domains, 2 bits each
- 00 = No access, 01 = Client, 11 = Manager

**c5 - Fault Status Registers**:
- DFSR: Data Fault Status
- IFSR: Instruction Fault Status

**c6 - Fault Address Register**:
- DFAR: Data Fault Address
- IFAR: Instruction Fault Address

**c7 - Cache Operations**:
- Invalidate: `mcr p15, 0, r0, c7, c7, 0`
- Clean: `mcr p15, 0, r0, c7, c10, 0`
- Clean & Invalidate: `mcr p15, 0, r0, c7, c14, 0`

**c8 - TLB Operations**:
- Invalidate entire TLB: `mcr p15, 0, r0, c8, c7, 0`
- Invalidate by MVA: `mcr p15, 0, r1, c8, c7, 1`

### Boot-Time MMU Setup Sequence

**Typical sequence for ARM bootloader**:

```assembly
; 1. Detect CPU version
mrc p15, 0, r0, c0, c0, 0      ; Read Main ID Register
; Parse to determine ARM version

; 2. Disable MMU and caches
mrc p15, 0, r1, c1, c0, 0      ; Read control register
bic r1, r1, #0x1               ; Clear M bit (MMU)
bic r1, r1, #0x4               ; Clear C bit (D-cache)
bic r1, r1, #0x1000            ; Clear I bit (I-cache)
mcr p15, 0, r1, c1, c0, 0      ; Write back

; 3. Invalidate caches
mov r0, #0
mcr p15, 0, r0, c7, c7, 0      ; Invalidate both caches
mcr p15, 0, r0, c8, c7, 0      ; Invalidate TLB

; 4. Drain write buffer (ARMv4+)
mcr p15, 0, r0, c7, c10, 4

; 5. Set up page tables
ldr r0, =page_table_address
mcr p15, 0, r0, c2, c0, 0      ; Set TTBR0

; 6. Set domain access (domain 0 = manager)
mvn r0, #0                     ; All 1s
mcr p15, 0, r0, c3, c0, 0      ; Set domain access

; 7. Enable MMU
mrc p15, 0, r0, c1, c0, 0
orr r0, r0, #0x1               ; Set M bit
orr r0, r0, #0x4               ; Set C bit (D-cache)
orr r0, r0, #0x1000            ; Set I bit (I-cache)
mcr p15, 0, r0, c1, c0, 0

; 8. Jump to virtual address (if needed)
ldr pc, =virtual_address
```

### Cache Coherency

**Critical for correct operation**:

1. **Before MMU enable**: Invalidate all caches
2. **After DMA**: Clean/invalidate affected cache lines
3. **Code modification**: Clean D-cache, invalidate I-cache
4. **Multiprocessor**: Use cache-coherent memory attributes

**Cache maintenance operations**:
```c
/* Clean D-cache line by MVA */
mcr p15, 0, r0, c7, c10, 1

/* Invalidate I-cache line by MVA */
mcr p15, 0, r0, c7, c5, 1

/* Data Synchronization Barrier */
mcr p15, 0, r0, c7, c10, 4  /* DSB */

/* Instruction Synchronization Barrier */
mcr p15, 0, r0, c7, c5, 4   /* ISB */
```

## Board-Specific Bootloaders

### Samsung S3C2440 (SMDK2440, Mini2440)

**Location**: `/sys/arch/evbarm/stand/boot2440/`

**Boot sequence**:
1. **S3C2440 ROM** loads first 4KB from NAND to internal SRAM
2. **Primary bootloader** (in NAND) initializes SDRAM
3. **Load boot2440** from NAND to SDRAM
4. **boot2440** loads kernel from SD card or network
5. **Kernel** boots

**Hardware initialization**:
- Clock configuration (FCLK, HCLK, PCLK)
- SDRAM controller setup
- UART initialization
- Network (DM9000) or SD/MMC

**Memory layout**:
```
0x30000000 - SDRAM start (64MB typical)
0x31500000 - Boot info structure
0x32000000 - Kernel load address
```

### i.MX23 (Freescale/NXP)

**Location**: `/sys/arch/evbarm/stand/bootimx23/`

**Features**:
- Support for i.MX23 SoC
- Power management initialization
- Clock tree configuration
- EMI (External Memory Interface) setup
- Pin multiplexing

## Common Board Support

**Location**: `/sys/arch/evbarm/stand/board/`

**Board abstraction layer**:

```c
/* board.h - Board support interface */
void board_init(void);    /* Initialize board resources */
void board_fini(void);    /* Finalize before kernel boot */
void cons_init(void);     /* Console initialization */
void mem_init(void);      /* Memory heap setup */
```

**Implementations** (36 board support files):

**Console drivers**:
- `ns16550.c` - Standard 16550 UART
- `epcom.c` - Samsung S3C UART
- `sscom.c` - S3C serial controller

**Memory controllers**:
- `ixp425_mem.c` - Intel IXP425
- `i80312_mem.c` - i80312 companion chip
- `gemini_mem.c` - Gemini SoC

**Network support**:
- `dm9000.c` - DM9000 Ethernet PHY
- `netif.c` - Network interface abstraction

## Boot Configuration (Legacy)

**Location**: `/sys/arch/evbarm/include/bootconfig.h`

**Structure**:
```c
struct bootconfig {
    uint32_t magic;              /* BOOTCONFIG_MAGIC */
    uint32_t version;            /* BOOTCONFIG_VERSION */
    uint8_t machine_id[4];       /* Machine ID */
    char kernelname[80];         /* Kernel path */
    char args[512];              /* Boot arguments */
    uint32_t ksym_start;         /* Kernel symbols start */
    uint32_t ksym_end;           /* Kernel symbols end */
    struct boot_physmem dram[2]; /* Physical memory descriptors */
};
```

**Physical memory descriptor**:
```c
struct boot_physmem {
    paddr_t bp_start;      /* Starting PFN (not address!) */
    psize_t bp_pages;      /* Number of pages */
    uint32_t bp_freelist;  /* VM_FREELIST_* */
    uint32_t bp_flags;     /* BOOT_PHYSMEM_CAN_DMA, etc. */
};
```

**Modern FDT approach** has largely replaced this for new boards.

## U-Boot Integration

### Loading NetBSD from U-Boot

**Prepare kernel**:
```sh
# Create U-Boot image
mkubootimage -A arm -C none -O netbsd -a 0x8000 -e 0x8000 \
    -n "NetBSD/evbarm" netbsd netbsd.ub
```

**U-Boot commands**:
```
# Load from SD card
fatload mmc 0:1 0x81000000 netbsd.ub
fatload mmc 0:1 0x82000000 board.dtb
bootm 0x81000000 - 0x82000000

# Or network boot
dhcp
tftp 0x81000000 netbsd.ub
tftp 0x82000000 board.dtb
bootm 0x81000000 - 0x82000000

# Or set up automatic boot
setenv bootcmd 'fatload mmc 0:1 0x81000000 netbsd.ub; fatload mmc 0:1 0x82000000 board.dtb; bootm 0x81000000 - 0x82000000'
saveenv
```

### U-Boot Environment Variables

```
bootargs=root=ld0a console=fb
bootdelay=3
baudrate=115200
```

## UEFI Boot (ARM Server)

**Modern ARM servers** (e.g., 96boards, ARM Server Base System Architecture):

1. **UEFI firmware** initializes hardware
2. **Loads** `BOOTAA64.EFI` from ESP partition
3. **UEFI bootloader** uses UEFI protocols for disk/network
4. **Loads kernel** with FDT
5. **Exit Boot Services**
6. **Kernel** takes control

## Build System

### Kernel Configuration

**Example kernel configs** (over 84 variants):
- `GENERIC` - Generic ARMv7 multi-platform
- `GENERIC64` - Generic AArch64 multi-platform
- `RPI` - Raspberry Pi
- `RPI2` - Raspberry Pi 2
- `NITROGEN6X` - Boundary Devices Nitrogen6X
- `BEAGLEBONE` - BeagleBone/BeagleBone Black
- Many more board-specific configs

### Building Bootloaders

**gzboot**:
```sh
cd /sys/arch/evbarm/stand/gzboot
make MACHINE=evbarm MACHINE_ARCH=armv7
```

**board-specific**:
```sh
cd /sys/arch/evbarm/stand/boot2440
make
```

## Supported Boards (Selection)

**Allwinner**: A10, A20, A31, A64, A80, H3, H5, H6
**Amlogic**: S805, S905
**Broadcom**: BCM2835 (Pi), BCM2836 (Pi 2), BCM2837 (Pi 3), BCM2711 (Pi 4)
**Marvell**: Armada XP, 370, 38x
**NXP/Freescale**: i.MX6, i.MX8
**Nvidia**: Tegra K1, X1
**Qualcomm**: Snapdragon
**Rockchip**: RK3328, RK3399
**Samsung**: Exynos5
**Texas Instruments**: OMAP3, OMAP4, AM335x
**Xilinx**: Zynq-7000, ZynqMP

## Installation

### SD Card Installation

```sh
# 1. Partition SD card
gpt create sd0
gpt add -t efi -l EFI -s 256m sd0
gpt add -t ffs -l NetBSD -s 8g sd0

# 2. Format partitions
newfs_msdos /dev/rsd0a
newfs /dev/rsd0b

# 3. Install bootloader/firmware (board-specific)
mount -t msdos /dev/sd0a /mnt
cp bootloader_files/* /mnt/
cp netbsd.ub /mnt/
cp board.dtb /mnt/
umount /mnt

# 4. Install system
mount /dev/sd0b /mnt
cd /mnt
tar xzpf /path/to/arm.tgz
tar xzpf /path/to/base.tgz
# ... extract sets
umount /mnt
```

## Debugging

### Serial Console

**Most boards**: 115200 8N1 on debug UART

**U-Boot access**: Press key during boot countdown

**NetBSD console**: Configured in kernel or via boot args

### Common Issues

**No output**: Check serial connection, baud rate, correct UART
**Hangs early**: MMU setup issue, check page tables
**Device not found**: FDT issue, verify DTB loaded correctly
**Kernel panic**: Memory layout mismatch, check load addresses

### QEMU Testing

```sh
# ARM Versatile PB
qemu-system-arm -M versatilepb -m 256M -kernel netbsd-GENERIC \
    -dtb vexpress-v2p-ca9.dtb -serial stdio

# AArch64 virt
qemu-system-aarch64 -M virt -cpu cortex-a57 -m 1G \
    -kernel netbsd-GENERIC64 -serial stdio
```

## References

### Source Files

**FDT support**:
- `/sys/arch/evbarm/fdt/fdt_machdep.c` - FDT initialization
- `/sys/arch/arm/arm/bootconfig.c` - Boot argument parsing

**Bootloaders**:
- `/sys/arch/evbarm/stand/gzboot/` - gzboot source
- `/sys/arch/evbarm/stand/board/` - Board support library

**ARM architecture**:
- `/sys/arch/arm/include/armreg.h` - CP15 register definitions
- `/sys/arch/arm/include/pmap.h` - MMU management

### Man Pages

- `boot(8)` - General boot procedures
- `installboot(8)` - Install bootloader

### External Documentation

- ARM Architecture Reference Manual
- U-Boot documentation
- Board-specific technical reference manuals
- Device Tree Specification

---

*Last Updated: 2025-11-12*
*Architecture Maintainer: NetBSD/evbarm Port*
