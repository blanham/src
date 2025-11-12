# NetBSD/cobalt Boot Documentation

$NetBSD$

## Platform Overview

**Architecture**: MIPS 32-bit (little-endian)
**Port Date**: 2000-03-19
**Boot Method**: Firmware → boot program → kernel
**Firmware**: Cobalt ROM Monitor
**MMU Requirements**: MIPS TLB, basic initialization by firmware

## Hardware Support

NetBSD/cobalt supports Cobalt Networks MIPS-based microservers:

- **CPU**: QED RM5200 (R5000-class), 150-250 MHz
- **Memory**: 16MB to 256MB SDRAM
- **Boot Devices**: IDE disk, network (NFS)
- **Firmware**: Cobalt ROM Monitor (simple firmware)
- **Console**: Serial port (16550 UART) - no VGA

### Supported Models

**Cobalt Qube 2700** (Generation I):
- CPU: RM5230A (R5000), 150 MHz
- Memory: 16-32MB
- Storage: 2.5" IDE disk
- Network: DEC 21143 Tulip Ethernet
- Form factor: Cube

**Cobalt RaQ** (Generation I):
- CPU: RM5230, 150 MHz
- Memory: 16-64MB
- Storage: 3.5" IDE disk
- Network: Tulip Ethernet
- Form factor: 1U rackmount

**Cobalt Qube 2** (Generation II):
- CPU: RM5261, 250 MHz
- Memory: 32-128MB
- Storage: 2.5" IDE laptop drive
- Network: VIA Rhine Ethernet
- LCD front panel

**Cobalt RaQ 2** (Generation II):
- CPU: RM5261, 250 MHz
- Memory: 64-256MB
- Storage: 3.5" IDE disk
- Network: VIA Rhine Ethernet
- Form factor: 1U rackmount

## Boot Process

### Overview

Cobalt uses a **two-stage boot process**: simple ROM firmware loads a bootloader from disk, which then loads the kernel.

### Stage 1: Cobalt ROM Firmware

**Functions**:
- Minimal hardware initialization
- Memory controller setup
- PCI bus initialization
- Load boot sector from IDE disk

**Boot Sequence**:
1. **Power-On**: ROM executes at 0xBFC00000
2. **Hardware Init**: Memory, PCI, serial console
3. **Load Boot Sector**: Read sector 0 of IDE disk
4. **Load Bootloader**: Read subsequent sectors for full bootloader
5. **Transfer Control**: Jump to bootloader entry point

**Firmware Limitations**:
- **No interactive shell**: Very minimal firmware
- **No BIOS services**: Bootloader must handle all I/O
- **Fixed boot device**: Always boots from first IDE device (wd0)
- **No network boot from firmware**: Must boot from disk first

### Stage 2: Bootloader (boot)

**Location**: `/sys/arch/cobalt/stand/boot/`
**Size**: ~64KB (bootloader + minimal filesystem support)
**Install Location**: Disk sectors 0-31 (16KB special boot area)

**Source Files**:
- `start.S` - Assembly entry point (97 lines)
- `boot.c` - Main bootloader logic (450+ lines)
- `devopen.c` - Device open/parse (200+ lines)
- `disk.c` - IDE disk I/O (no firmware services)
- `cons.c` - Console I/O (direct 16550 UART)
- `bootinfo.c` - Boot information structures

**Entry Point** (`start.S`):
```assembly
start:
    .set  noreorder
    .set  mips3

    # Set up stack
    la    sp, start - CALLFRAME_SIZ
    sw    zero, CALLFRAME_RA(sp)
    sw    zero, CALLFRAME_SP(sp)

    # Save arguments
    move  s0, a0                # argc
    move  s1, a1                # argv

    # Flush caches
    jal   flushcache
    nop

    # Clear BSS
    la    a0, edata
    move  a1, zero
    la    a2, end
    jal   memset
    subu  a2, a2, a0

    # Call main(argc, argv)
    move  a0, s0
    jal   main
    move  a1, s1
```

**Cache Flushing**:
- Manually flush 32KB I-cache and D-cache
- Use cache instructions directly (CACHE_R4K_I, CACHE_R4K_D)
- Necessary because firmware doesn't provide services

**Main Function** (`boot.c`):
```c
void main(unsigned int argc, char **argv)
{
    /* Initialize console */
    cninit();

    /* Display banner */
    printf("\n>> NetBSD/cobalt " NETBSD_VERS " Boot, Revision %s\n",
           bootprog_rev);

    /* Parse boot arguments */
    parseargs(argc, argv);

    /* Initialize Galileo GT-64111 system controller */
    gt_init();

    /* Initialize block device (IDE disk) */
    devinit();

    /* Try to load kernel */
    for (i = 0; kernelnames[i]; i++) {
        printf("Loading: %s\n", kernelnames[i]);
        if (loadfile(kernelnames[i], marks, LOAD_KERNEL) == 0)
            break;
    }

    /* Prepare boot information */
    bi_init();
    bi_add_bootinfo();
    bi_add_symtab(marks);

    /* Execute kernel */
    entry = (void *)marks[MARK_ENTRY];
    printf("Starting at 0x%x\n", (u_int)entry);
    (*entry)(argc, argv, DL_GETENV(DL_BOOTINFO));
}
```

**Kernel Names Tried** (in order):
1. `netbsd`
2. `netbsd.gz`
3. `onetbsd`
4. `onetbsd.gz`
5. `netbsd.bak`
6. `netbsd.bak.gz`
7. `netbsd.old`
8. `netbsd.old.gz`
9. `netbsd.cobalt`
10. `netbsd.cobalt.gz`
11. `netbsd.elf`
12. `netbsd.elf.gz`

**Device Naming**:
- `wd0`: First IDE disk
- `wd0a`: First partition (root)
- `nfs`: Network filesystem (after kernel boot)

### Stage 3: Kernel

**Entry Point**: `mach_init()`

**Arguments**:
- `a0`: Memory size (from firmware)
- `a1`: Boot information magic
- `a2`: Boot information pointer (physical address)

**Initialization Sequence**:
```c
void mach_init(int32_t memsize32, u_int bim, int32_t bip32)
{
    /* Convert arguments to proper types */
    memsize = (uint32_t)memsize32;
    bootinfo_ptr = (void *)(intptr_t)bip32;

    /* Read Cobalt board ID from system controller */
    cobalt_id = read_board_id();

    /* Clear BSS */
    kernend = (void *)mips_round_page(end);
    memset(edata, 0, (char *)kernend - (char *)edata);

    /* Copy exception handlers to proper locations */
    mips_vector_init(NULL, false);

    /* Determine memory size */
    /* Firmware passes it in a0, or detect via GT-64111 */
    physmem = btoc(memsize);

    /* Parse boot information */
    if (bim == BOOTINFO_MAGIC) {
        memcpy(bootstring, bootinfo_ptr, sizeof(bootstring));
        decode_bootstring();
    }

    /* Identify platform */
    printf("%s\n", cobalt_model[cobalt_id]);

    /* Initialize bus space tags */
    mainbus_bus_mem_init(&cobalt_bs, NULL);

    /* Initialize console */
    consinit();

    /* Bootstrap UVM */
    pmap_bootstrap();
}
```

**Board ID Detection**:
```c
/* GT-64111 board revision register */
#define GT_BASE         0x14000000
#define GT_BOARD_REV    0x00000c44

u_int read_board_id(void)
{
    volatile uint32_t *board_rev;
    uint32_t rev;

    board_rev = (uint32_t *)MIPS_PHYS_TO_KSEG1(GT_BASE + GT_BOARD_REV);
    rev = *board_rev;

    if (rev == 0)
        return COBALT_ID_QUBE2700;  /* Qube 2700 */
    else if (rev == 1)
        return COBALT_ID_RAQ;       /* RaQ */
    else if ((rev >= 2) && (rev <= 5))
        return COBALT_ID_QUBE2;     /* Qube 2 */
    else
        return COBALT_ID_RAQ2;      /* RaQ 2 */
}
```

## Hardware Details

### Galileo GT-64111 System Controller

**Base Address**: 0x14000000 (physical)
**Function**: System controller, PCI bridge, memory controller, interrupt controller

**Key Registers**:
- **CPU Interface**: 0x000-0x0ff
- **SDRAM**: 0x400-0x4ff
- **Device Bus**: 0x460-0x4ff
- **PCI**: 0xc00-0xcff
- **Interrupts**: 0xc18-0xc1c

**Memory Mapping**:
```
0x00000000 - 0x0FFFFFFF : SDRAM (up to 256MB)
0x10000000 - 0x13FFFFFF : PCI memory space (64MB)
0x14000000 - 0x17FFFFFF : GT-64111 registers
0x1C000000 - 0x1FFFFFFF : PCI I/O space (64MB)
```

### IDE Controller

**Chipset**: VIA VT82C586B (integrated in GT-64111)
**Interface**: PCI IDE (both PATA channels)
**Addressing**: PCI I/O space

**Primary Channel** (used for boot):
- Command Block: 0x1F0-0x1F7 (compat mode)
- Control Block: 0x3F6

**Boot Disk Layout**:
```
Sector 0-31    : Bootloader (16KB boot area)
Sector 32+     : NetBSD root filesystem (FFS)
```

**Important**: Cobalt uses a special boot area, **NOT** a standard MBR. The bootloader is installed directly to the first 16KB of the disk.

### Serial Console

**UART**: National Semiconductor 16550-compatible
**Base Address**: 0x1C800000 (I/O mapped)
**Parameters**: 115200 baud, 8N1 (8 data, no parity, 1 stop)

**Registers** (standard 16550):
- 0x00: RBR/THR (Receive/Transmit Buffer)
- 0x01: IER (Interrupt Enable)
- 0x02: IIR/FCR (Interrupt ID / FIFO Control)
- 0x03: LCR (Line Control)
- 0x04: MCR (Modem Control)
- 0x05: LSR (Line Status)
- 0x06: MSR (Modem Status)
- 0x07: SCR (Scratch)

## Memory Map

### Virtual Address Space

Standard MIPS32 segments:

```
0x00000000 - 0x7FFFFFFF : useg   (user, TLB-mapped)
0x80000000 - 0x9FFFFFFF : kseg0  (kernel, cached, unmapped)
0xA0000000 - 0xBFFFFFFF : kseg1  (kernel, uncached, unmapped)
0xC0000000 - 0xFFFFFFFF : kseg2  (kernel, TLB-mapped)
```

### Physical Address Space

```
0x00000000 - 0x0FFFFFFF : Main RAM (16MB to 256MB)
                          - 0x00000000: Exception vectors
                          - 0x00001000: Start of usable RAM
                          - Kernel loaded at 0x80000000 (kseg0)

0x10000000 - 0x13FFFFFF : PCI memory space (64MB)
                          - Mapped devices (framebuffer, etc.)

0x14000000 - 0x17FFFFFF : GT-64111 internal registers (64MB)
                          - 0x14000000: GT-64111 base

0x18000000 - 0x1BFFFFFF : Reserved

0x1C000000 - 0x1C7FFFFF : PCI I/O space (8MB)
                          - 0x1C000000: Legacy I/O base
                          - 0x1C0001F0: IDE primary
                          - 0x1C800000: Serial console

0x1C800000 - 0x1FFFFFFF : Extended PCI I/O

0xBFC00000 - 0xBFFFFFFF : Boot ROM (4MB)
                          - Reset vector at 0xBFC00000
```

## Build Instructions

### Building the Bootloader

```sh
cd /sys/arch/cobalt/stand/boot
make

# Or using build.sh
./build.sh -m cobalt tools
./build.sh -m cobalt distribution
```

**Output**: `boot` - ELF executable bootloader (~64KB)

### Building the Kernel

```sh
# Using build.sh (recommended)
./build.sh -m cobalt kernel=GENERIC

# Or manually
cd /sys/arch/cobalt/conf
config GENERIC
cd ../compile/GENERIC
make depend
make
```

**Output**: `netbsd` - ELF kernel

**Kernel Config Options**:
```
makeoptions CPUFLAGS="-march=vr5000"  # QED RM52xx CPU
```

## Installation

### Disk Preparation

**Important**: Cobalt uses a special boot area, **not MBR**.

**Disk Layout**:
```
Sectors 0-31    : Boot area (16KB) - bootloader installed here
Sectors 32+     : NetBSD disklabel and root filesystem
```

### Installing the Bootloader

**From another NetBSD system**:

```sh
# Copy bootloader to boot area (first 16KB)
dd if=boot of=/dev/rwd0d bs=512 seek=0 count=32

# Or use installboot (if supported)
installboot -v /dev/rwd0a /usr/mdec/boot
```

**From the Cobalt itself** (after initial setup):

```sh
# Install bootloader
installboot /dev/rwd0a /usr/mdec/boot
```

### Disk Partitioning

**Create NetBSD disklabel**:

```sh
# Write disklabel
disklabel -e wd0

# Example layout:
#  a: /        (root)     - 1GB
#  b: swap                - 256MB
#  d: whole disk
#  e: /usr                - remaining space
```

**Important**: Partition 'a' (root) must start at sector 32 or later to avoid overwriting the boot area.

### Installing System

```sh
# Create filesystems
newfs /dev/rwd0a
newfs /dev/rwd0e

# Mount
mount /dev/wd0a /mnt
mkdir /mnt/usr
mount /dev/wd0e /mnt/usr

# Extract sets
cd /mnt
tar xzpf /path/to/base.tgz
tar xzpf /path/to/etc.tgz
tar xzpf /path/to/comp.tgz
# ... other sets

# Copy kernel
cp /path/to/netbsd /mnt/netbsd

# Unmount
umount /mnt/usr /mnt
```

### Network Boot Setup

Cobalt firmware cannot network boot directly, but the bootloader supports NFS root:

**Setup NFS server**:

```sh
# Export root filesystem
echo "/export/cobalt-root -maproot=root cobalt.local" >> /etc/exports
exportfs -a

# Extract NetBSD sets to NFS root
cd /export/cobalt-root
tar xzpf /path/to/base.tgz
# ... other sets
```

**Configure bootloader** (from disk):

Edit `/boot.cfg` or kernel command line to specify NFS root.

## Boot Configuration

### Boot Arguments

**From bootloader prompt**:

```
>> boot netbsd           # Boot default kernel
>> boot netbsd -s        # Single-user mode
>> boot netbsd -a        # Ask for root device
>> boot netbsd -d        # Enter debugger
>> boot netbsd -v        # Verbose boot
```

**Kernel Boot String**:

The bootloader passes boot arguments via bootinfo structure. Common arguments:

- `-s`: Single-user mode
- `-a`: Ask for root device
- `-d`: Drop to DDB debugger
- `-v`: Verbose boot
- `root=wd0a`: Specify root device
- `nfsroot=192.168.1.1:/export/root`: NFS root

### Boot Configuration File

Cobalt bootloader does not use `/boot.cfg` by default. Boot arguments are passed from firmware/bootloader.

## Debugging

### Serial Console

**Required**: Cobalt has **no VGA**; serial console is mandatory.

**Connection**:
- Cable: DB9 null modem or Cisco console cable
- Settings: 115200 baud, 8N1, no flow control

**Terminal Program**:
```sh
cu -l /dev/ttyU0 -s 115200
```

or

```sh
minicom -D /dev/ttyU0 -b 115200
```

### DDB Kernel Debugger

**Enable in kernel**:
```
options     DDB
makeoptions COPY_SYMTAB=1
```

**Enter DDB**:
- At boot: `boot netbsd -d`
- From serial console: `Ctrl-Alt-Esc` or break signal
- On panic: Automatic

**Commands**:
```
db> trace               # Stack trace
db> ps                  # Process list
db> show registers      # CPU registers
db> x/x addr,count     # Examine memory
db> break function      # Set breakpoint
db> continue            # Resume execution
db> reboot              # Reboot system
```

### Common Issues

**"Cannot load kernel"**:
- Verify `netbsd` exists in root directory
- Check kernel is valid ELF executable
- Try compressed kernel: `netbsd.gz`

**"IDE error" or disk read failures**:
- Bad disk or cable
- Bootloader not properly installed
- Try: `dd if=/usr/mdec/boot of=/dev/rwd0d bs=512 count=32`

**Serial console garbled**:
- Baud rate mismatch (must be 115200)
- Wrong cable type (need null modem)
- Flow control issue (disable XON/XOFF)

**System doesn't boot**:
- Bootloader not in boot area (sectors 0-31)
- Kernel not in root directory
- Filesystem corruption: run `fsck`

**Hangs at "Starting at 0x..."**:
- Exception during kernel initialization
- Memory size detection issue
- Try verbose boot: `-v`

## Platform Notes

### Cobalt Qube/RaQ Differences

**Qube**:
- Blue cube form factor
- LCD front panel (Qube 2)
- 2.5" laptop IDE drive
- Smaller RAM capacity

**RaQ**:
- 1U rackmount form factor
- No LCD
- 3.5" desktop IDE drive
- Higher RAM capacity
- Often used as web servers

### Historical Context

- **Cobalt Networks** (1996-2000): Created affordable MIPS-based servers
- **Sun Microsystems** acquired Cobalt in 2000
- **End of Production**: ~2003
- **Legacy**: Pioneered low-cost, Linux/BSD-based appliance servers
- **Hardware Availability**: Units still found on surplus/eBay market

## Source Code Reference

### Bootloader Source

- `/sys/arch/cobalt/stand/boot/start.S` - Entry point (97 lines)
- `/sys/arch/cobalt/stand/boot/boot.c` - Main logic (450 lines)
- `/sys/arch/cobalt/stand/boot/devopen.c` - Device handling (200 lines)
- `/sys/arch/cobalt/stand/boot/disk.c` - Disk I/O (180 lines)
- `/sys/arch/cobalt/stand/boot/cons.c` - Console I/O (150 lines)

### Kernel Source

- `/sys/arch/cobalt/cobalt/machdep.c` - Machine-dependent (800+ lines)
- `/sys/arch/cobalt/cobalt/autoconf.c` - Device autoconfiguration
- `/sys/arch/cobalt/dev/gt.c` - GT-64111 driver (1000+ lines)

### Include Files

- `/sys/arch/cobalt/include/bootinfo.h` - Boot info structures
- `/sys/arch/cobalt/dev/gtreg.h` - GT-64111 registers
- `/sys/arch/cobalt/include/autoconf.h` - Autoconfiguration

### Man Pages

- `boot(8)` - Boot procedures
- `installboot(8)` - Install bootloader
- `intro(4)` - Hardware support overview

---

*Last Updated: 2025-11-12*
*Architecture Maintainer: NetBSD/cobalt Port*
