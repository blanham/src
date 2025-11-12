# NetBSD/sbmips Boot Documentation

$NetBSD$

## Platform Overview

**Architecture**: MIPS 32-bit and 64-bit (big-endian)
**Port Date**: 2001-07-07 (part of evbmips)
**Boot Method**: CFE firmware → boot → kernel
**Firmware**: Common Firmware Environment (CFE)
**MMU Requirements**: MIPS64 TLB

## Hardware Support

NetBSD/sbmips supports Broadcom SiByte evaluation boards:

- **CPU**: Broadcom SiByte SB-1, SB-1A processors (MIPS64)
- **Memory**: 128MB to 1GB
- **Boot Devices**: IDE disk, CompactFlash, network (TFTP)
- **Firmware**: CFE (Common Firmware Environment)
- **Console**: Serial port (16550 UART)

### Supported Boards

**SWARM** (BCM91250A):
- SB-1 or SB-1A CPU (600-1000 MHz)
- Dual Gigabit Ethernet
- IDE and USB
- CompactFlash socket

**LittleSur** (BCM91250C):
- SB-1A CPU
- Compact form factor

**Sentosa** (BCM91480B):
- SB-1A CPU (700-1000 MHz)
- PCI-X support

**Rhone** (BCM91125E):
- SB-1250 CPU
- Evaluation board

## Boot Process

### Overview

SiByte boards use **CFE** (Common Firmware Environment), a lightweight, open-source firmware developed by Broadcom.

### Boot Stages

1. **CFE firmware**: Hardware init, boot services
2. **boot**: NetBSD bootloader (ELF executable)
3. **Kernel**: NetBSD kernel

### Stage 1: CFE Firmware

**Functions**:
- CPU and memory initialization
- Device enumeration
- Network stack (TFTP, DHCP)
- Boot device selection
- Interactive shell

**CFE Commands**:
```
CFE> boot -elf tftp://192.168.1.1/netbsd    # Network boot
CFE> boot -elf disk:netbsd                  # Boot from disk
CFE> boot -elf cf:netbsd                    # Boot from CompactFlash
CFE> show devices                           # List devices
CFE> setenv bootfile cf:netbsd              # Set default boot
CFE> printenv                               # Show variables
CFE> help                                   # Show help
```

**Device Naming**:
- `disk:` - IDE disk
- `cf:` - CompactFlash
- `tftp:` - TFTP network boot
- `flash:` - Flash memory

### Stage 2: Bootloader (boot)

**Location**: `/sys/arch/evbmips/stand/sbmips/`
**Format**: ELF executable (loaded directly by CFE)
**Size**: ~128KB

**Features**:
- Full filesystem support (FFS, FFS v2)
- ELF kernel loading
- Gzip compression support
- CFE firmware callbacks
- Boot parameter passing

**Source Files**:
- `boot/boot.c` - Main bootloader (400 lines)
- `netboot/netboot.c` - Network bootloader variant
- `common/` - Shared SiByte code

**Functionality**:
```c
void main(long fwhandle, long evector, long fwentry, long fwseal)
{
    /* Initialize CFE interface */
    cfe_init(fwhandle, fwentry);

    /* Initialize console */
    cninit();

    /* Parse boot arguments */
    parseargs();

    /* Try to load kernel */
    for (i = 0; kernelnames[i]; i++) {
        if (loadfile(kernelnames[i], marks, LOAD_KERNEL) == 0)
            break;
    }

    /* Prepare boot info */
    bi_init();
    bi_add();

    /* Execute kernel */
    entry = (void *)marks[MARK_ENTRY];
    (*entry)(argc, argv, envp);
}
```

**No bootxx Stage**: CFE loads the bootloader directly (unlike many other MIPS platforms).

### Stage 3: Kernel

**Entry Point**: `mach_init()`
**Source**: `/sys/arch/evbmips/sbmips/machdep.c`

**Kernel Initialization**:
- Parse bootinfo
- Initialize SiByte system controller
- Setup interrupt controller
- Configure caches
- Bootstrap VM

## Memory Map

### Virtual Address Space (64-bit)

```
0x0000000000000000 - 0x000000FFFFFFFFFF : xuseg  (User, TLB-mapped)
0x4000000000000000 - 0x400000FFFFFFFFFF : xsseg  (Supervisor)
0x8000000000000000 - 0x800000FFFFFFFFFF : xkphys uncached (1TB)
0x9000000000000000 - 0x900000FFFFFFFFFF : xkphys cached (1TB)
0xC000000000000000 - 0xFFFFFFFFFFFFFFFF : xkseg  (Kernel, TLB-mapped)
```

### Physical Memory (SWARM)

```
0x0000000000 - 0x3FFFFFFF : Main RAM (up to 1GB)
0x0010000000 - 0x001FFFFFFF : SiByte system controller (256MB)
0x0040000000 - 0x007FFFFFFF : PCI memory space (1GB)
0x00DE000000 - 0x00DFFFFFFF : PCI I/O space (32MB)
```

## SiByte SB-1 Processor

### Features

- **Architecture**: MIPS64 (MIPS III + extensions)
- **Clock**: 600-1000 MHz
- **Cores**: 1 or 2 (SB-1250 has 2)
- **L1 Cache**: 32KB I-cache, 32KB D-cache per core
- **L2 Cache**: 512KB to 2MB (shared)
- **TLB**: 64 entries

### SiByte System Controller

**Base Address**: 0x0010000000

**Key Units**:
- **Interrupt Mapper**: Route interrupts to CPUs
- **DMA Engines**: High-speed data movement
- **MAC Controllers**: Dual Gigabit Ethernet
- **Serial Controllers**: UARTs
- **Timers**: Watchdog and general-purpose
- **GPIO**: General-purpose I/O
- **PCI/HyperTransport**: Bus interfaces

## Build Instructions

### Building Bootloader

```sh
cd /sys/arch/evbmips/stand/sbmips
make

# Or using build.sh
./build.sh -m evbmips tools
./build.sh -m evbmips distribution
```

**Output**: `boot` - ELF bootloader for CFE

### Building Kernel

```sh
./build.sh -m evbmips kernel=SBMIPS

# Or manually
cd /sys/arch/evbmips/conf
config SBMIPS
cd ../compile/SBMIPS
make depend
make
```

**Kernel Config**: `/sys/arch/evbmips/conf/SBMIPS`

## Installation

### CompactFlash Installation

**Most common** for SiByte boards:

```sh
# Partition CF card
gpt create sd0
gpt add -t ffs -s 4g sd0

# Create filesystem
newfs /dev/rsd0a

# Mount and install
mount /dev/sd0a /mnt
cd /mnt
tar xzpf /path/to/base.tgz
# ... other sets

# Copy bootloader and kernel
cp /usr/mdec/boot /mnt/boot
cp netbsd /mnt/netbsd

umount /mnt
```

### Network Boot Installation

**TFTP Boot**:
```
CFE> ifconfig eth0 -addr=192.168.1.2 -mask=255.255.255.0
CFE> boot -elf tftp://192.168.1.1/boot
```

**NFS Root**:
```
CFE> setenv -p BOOT_CONSOLE serial
CFE> setenv -p bootfile tftp://192.168.1.1/netbsd
CFE> setenv -p nfsroot 192.168.1.1:/export/sbmips
CFE> boot
```

### CFE Environment Variables

**Set variables**:
```
CFE> setenv bootfile cf:netbsd
CFE> setenv -p bootfile cf:netbsd      # Persistent
CFE> saveenv                            # Save to NVRAM
```

**Common variables**:
- `bootfile` - Default boot file
- `BOOT_CONSOLE` - Console device (serial/vga)
- `nfsroot` - NFS root path

## Debugging

### Serial Console

**Default**: Serial console at 115200 baud, 8N1

**CFE**:
```
CFE> setenv BOOT_CONSOLE serial
CFE> saveenv
```

**Connection**: Standard DB9 serial port

### DDB Kernel Debugger

```
options     DDB
makeoptions COPY_SYMTAB=1
```

**Enter DDB**: `Ctrl-Alt-Esc` or `break` signal

### CFE Debugging

```
CFE> show cpu               # CPU info
CFE> show memory            # Memory map
CFE> show devices           # Device list
CFE> test device            # Test device
```

### SiByte Debug Features

**Performance Counters**: Hardware performance monitoring
**Trace Buffer**: Instruction trace buffer (debug builds)
**EJTAG**: JTAG debugging interface (requires hardware debugger)

## Source Code Reference

**Bootloader**:
- `/sys/arch/evbmips/stand/sbmips/boot/` - Main bootloader (400 lines)
- `/sys/arch/evbmips/stand/sbmips/netboot/` - Network boot variant
- `/sys/arch/evbmips/stand/sbmips/common/` - Shared code (500 lines)

**Kernel**:
- `/sys/arch/evbmips/sbmips/machdep.c` - Machine-dependent (800 lines)
- `/sys/arch/evbmips/sbmips/autoconf.c` - Autoconfiguration
- `/sys/arch/mips/sibyte/` - SiByte system controller drivers (5000+ lines)

**Include**:
- `/sys/arch/mips/sibyte/include/` - SiByte register definitions
- `/sys/arch/evbmips/include/bootinfo.h` - Boot information

**CFE**:
- CFE source is available separately from Broadcom

## Platform Notes

- **Big-Endian**: SiByte processors run in big-endian mode
- **64-bit**: SB-1 is a MIPS64 processor
- **Evaluation Boards**: Primarily used for development/evaluation
- **CFE**: Open-source firmware, well-documented
- **Performance**: High-performance processors for embedded/networking
- **Limited Production**: Mainly evaluation and development boards

---

*Last Updated: 2025-11-12*
*Architecture Maintainer: NetBSD/evbmips (sbmips) Port*
