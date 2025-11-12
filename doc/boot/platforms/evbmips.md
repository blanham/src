# NetBSD/evbmips Boot Documentation

$NetBSD$

## Platform Overview

**Architecture**: MIPS 32-bit and 64-bit (both endiannesses)
**Port Date**: 2001-05-29
**Boot Method**: Varies by board (firmware → bootloader → kernel)
**Firmware**: Board-specific (U-Boot, CFE, YAMON, PMON, etc.)
**MMU Requirements**: MIPS TLB (board-dependent)

## Hardware Support

NetBSD/evbmips is a **meta-platform** supporting **54+ MIPS evaluation boards** from multiple vendors:

### Supported Board Families

**Alchemy (AMD)**:
- Alchemy Semiconductor Au1x00 series
- DBAu1000, DBAu1100, DBAu1500, DBAu1550

**Atheros**:
- AR531x, AR71xx, AR724x, AR933x, AR934x wireless SoC boards

**Broadcom SiByte** (see also: sbmips):
- SWARM (BCM91250A), LittleSur, Sentosa, Rhone
- CFE firmware

**Cavium**:
- Octeon and Octeon Plus (MIPS64)
- CN38xx, CN50xx, CN52xx, CN56xx, CN58xx, CN61xx, CN63xx, CN66xx, CN68xx, CN70xx, CN73xx, CN78xx

**Ingenic**:
- JZ4780 (MIPS32r2)
- Creator CI20 board

**Loongson**:
- Loongson 2E, 2F, 3A (MIPS64)
- Lemote Fuloong, Yeeloong, Lynloong

**MALTA** (MIPS Technologies):
- CoreLV, CoreFPGA, CoreFPGA2, CoreFPGA3, CoreFPGA4, CoreFPGA5, CoreFPGA6
- YAMON firmware

**Meraki**:
- Meraki MR18 (Atheros AR9344)

**Mikrotik**:
- RouterBOARD series (Atheros)

**Netgear**:
- Various router/AP models (Atheros)

**Ralink**:
- RT3052, RT3883, MT7620, MT7621, MT7628
- Wireless router SoCs

**And many more** from various vendors.

## Boot Methods by Board Family

### 1. U-Boot Boards (Common)

**Examples**: Many embedded boards, routers

**Boot Sequence**:
```
ROM → U-Boot → kernel (no NetBSD bootloader)
```

**U-Boot Commands**:
```
=> tftpboot 0x80100000 netbsd-BOARD.ub
=> bootm 0x80100000
```

**Or with DTB**:
```
=> tftpboot ${kernel_addr} netbsd.ub
=> tftpboot ${fdt_addr} board.dtb
=> bootm ${kernel_addr} - ${fdt_addr}
```

**Direct Boot**: U-Boot loads kernel directly (ELF or U-Boot image format)

### 2. CFE Boards (Broadcom SiByte)

See separate **sbmips** documentation for detailed CFE boot process.

**Boot Sequence**:
```
ROM → CFE → boot → kernel
```

**CFE Commands**:
```
CFE> boot -elf disk:netbsd
CFE> boot -elf tftp://server/netbsd
```

### 3. YAMON Boards (MIPS MALTA)

**Boot Sequence**:
```
ROM → YAMON → kernel (direct or via loader)
```

**YAMON Commands**:
```
YAMON> load tftp://192.168.1.1/netbsd
YAMON> go
```

### 4. PMON Boards (Loongson)

**Boot Sequence**:
```
ROM → PMON → kernel
```

**PMON Commands**:
```
PMON> load /dev/fs/ext2@wd0/netbsd
PMON> g
```

### 5. Proprietary Firmware

Many boards have vendor-specific firmware:
- Custom boot loaders
- Minimal boot ROMs
- Direct kernel loading

## Board-Specific Boot Examples

### MALTA (YAMON)

**Location**: `/sys/arch/evbmips/malta/`

**Boot**:
```
YAMON> setenv ipaddr 192.168.1.2
YAMON> setenv serverip 192.168.1.1
YAMON> setenv image netbsd-MALTA
YAMON> run netboot
```

### Cavium Octeon (U-Boot)

**Location**: `/sys/arch/evbmips/octeon/`

**Boot**:
```
Octane# tftpboot netbsd-OCTEON
Octane# bootoctlinux
```

### Alchemy (YAMON)

**Location**: `/sys/arch/evbmips/alchemy/`

**Boot**:
```
YAMON> load tftp://server/netbsd-DBAU1500
YAMON> go
```

### Ingenic (U-Boot)

**Location**: `/sys/arch/evbmips/ingenic/`

**Boot**:
```
ci20# fatload mmc 0:1 ${kernel_addr} netbsd-GENERIC.ub
ci20# bootm ${kernel_addr}
```

### Atheros (U-Boot or RedBoot)

**Location**: `/sys/arch/evbmips/atheros/`

**Boot** (varies by board):
```
RedBoot> load -r -b 0x80100000 netbsd
RedBoot> go 0x80100000
```

## Memory Maps

Memory maps vary significantly by board. Examples:

### MALTA

```
0x00000000 - 0x0FFFFFFF : SDRAM (256MB)
0x18000000 - 0x1BFFFFFF : PCI I/O
0x1FC00000 - 0x1FFFFFFF : Boot Flash
```

### Cavium Octeon

```
0x0000000000000000 - 0x00000007FFFFFFFF : Physical memory (32GB max)
0x0001180000000000 - 0x000118FFFFFFFFFF : I/O space
0x000011F000000000 - 0x000011FFFFFFFFFF : Boot bus
```

### Alchemy

```
0x00000000 - 0x0FFFFFFF : SDRAM (up to 256MB)
0x10000000 - 0x1FFFFFFF : PCMCIA
0x14000000 - 0x17FFFFFF : PCI memory
0x1FC00000 - 0x1FFFFFFF : Boot ROM
```

## Device Tree (FDT/DTB) Support

Many modern evbmips boards use **Flattened Device Tree** (FDT):

**Usage**:
- Firmware passes DTB to kernel
- Kernel probes devices from DTB
- Eliminates hard-coded hardware configurations

**Boot with DTB**:
```
U-Boot> tftpboot ${kernel_addr} netbsd.ub
U-Boot> tftpboot ${fdt_addr} board.dtb
U-Boot> bootm ${kernel_addr} - ${fdt_addr}
```

**Kernel Support**:
```
options FDT
makeoptions DTS="board.dts"
```

## Build Instructions

### Building Kernel

**Select appropriate kernel config** (varies by board):

```sh
# Examples
./build.sh -m evbmips kernel=MALTA
./build.sh -m evbmips kernel=OCTEON
./build.sh -m evbmips kernel=ALCHEMY
./build.sh -m evbmips kernel=LOONGSON
./build.sh -m evbmips kernel=ATHEROS
./build.sh -m evbmips kernel=CI20
./build.sh -m evbmips kernel=RB153
# ... many more
```

**All kernel configs**: `/sys/arch/evbmips/conf/`

**Examples**:
- `MALTA` - MIPS Malta board
- `MALTA64` - MIPS Malta (64-bit)
- `OCTEON` - Cavium Octeon
- `ALCHEMY` - Alchemy boards
- `DB120` - Atheros DB120
- `LOONGSON` - Loongson 2F
- `CI20` - Ingenic CI20

### Creating U-Boot Images

Many boards require U-Boot image format:

```sh
# Create U-Boot image
mkubootimage -A mips -C none -O netbsd \
    -a 0x80100000 -e 0x80100000 \
    -n "NetBSD/evbmips" netbsd netbsd.ub
```

### Building Bootloaders

**SiByte/CFE boards only**:
```sh
cd /sys/arch/evbmips/stand/sbmips
make
```

**Most other boards**: No NetBSD bootloader (use firmware directly)

## Installation

Installation varies significantly by board:

### General Steps

1. **Prepare Boot Media**:
   - SD card, CompactFlash, USB, or network

2. **Install Firmware** (if needed):
   - Flash U-Boot, CFE, or other firmware

3. **Copy Kernel**:
   - Transfer `netbsd` or `netbsd.ub` to boot device

4. **Configure Boot**:
   - Set firmware environment variables

5. **Create Root Filesystem**:
   - Format and populate root partition

### Example: SD Card Installation

```sh
# Format SD card
gpt create sd0
gpt add -t efi -s 256m sd0
gpt add -t ffs -s 4g sd0

newfs_msdos /dev/rsd0a
newfs /dev/rsd0b

# Copy boot files to FAT partition
mount -t msdos /dev/sd0a /mnt
cp netbsd.ub /mnt/
cp board.dtb /mnt/
umount /mnt

# Extract system to FFS partition
mount /dev/sd0b /mnt
cd /mnt
tar xzpf /path/to/base.tgz
# ... other sets
umount /mnt
```

## Kernel Configuration Options

**Common options**:

```
# Endianness
makeoptions LDFLAGS="-EB"      # Big-endian
makeoptions LDFLAGS="-EL"      # Little-endian

# CPU architecture
makeoptions CPUFLAGS="-march=mips32"
makeoptions CPUFLAGS="-march=mips64"
makeoptions CPUFLAGS="-march=octeon"

# Device Tree
options FDT
makeoptions DTS="board.dts"
```

## Debugging

### Serial Console

**Most boards**: Serial console default
- Baud: 115200 (common) or 9600
- Data: 8N1

**Configure**:
```
options CONSPEED=115200
```

### DDB Kernel Debugger

```
options     DDB
makeoptions COPY_SYMTAB=1
```

### Firmware Debugging

**U-Boot**:
```
=> md 0x80000000       # Memory dump
=> bootm 0x80100000    # Boot from memory
```

**CFE**:
```
CFE> show memory
CFE> show devices
```

**YAMON**:
```
YAMON> dump 0x80000000
```

## Common Issues

### Endianness Mismatch

**Problem**: Kernel built with wrong endianness
**Solution**: Check board's native endianness, rebuild kernel

### Load Address Conflicts

**Problem**: Kernel loaded at wrong address
**Solution**: Check firmware's expected load address, use mkubootimage correctly

### Missing Device Tree

**Problem**: Kernel can't find devices
**Solution**: Ensure DTB is loaded and passed to kernel

### Firmware Version

**Problem**: Old firmware incompatible
**Solution**: Upgrade firmware (U-Boot, CFE, etc.)

## Source Code Reference

**Board-Specific Code**:
- `/sys/arch/evbmips/malta/` - MALTA support
- `/sys/arch/evbmips/octeon/` - Cavium Octeon
- `/sys/arch/evbmips/alchemy/` - Alchemy
- `/sys/arch/evbmips/atheros/` - Atheros
- `/sys/arch/evbmips/ingenic/` - Ingenic
- `/sys/arch/evbmips/loongson/` - Loongson
- `/sys/arch/evbmips/sbmips/` - SiByte (see sbmips.md)

**SoC Support**:
- `/sys/arch/mips/alchemy/` - Alchemy SoC drivers
- `/sys/arch/mips/atheros/` - Atheros SoC drivers
- `/sys/arch/mips/cavium/` - Cavium Octeon drivers
- `/sys/arch/mips/ingenic/` - Ingenic SoC drivers
- `/sys/arch/mips/ralink/` - Ralink SoC drivers
- `/sys/arch/mips/sibyte/` - SiByte drivers

**Bootloaders**:
- `/sys/arch/evbmips/stand/sbmips/` - SiByte bootloader (only evbmips bootloader)

**Kernel Configs**:
- `/sys/arch/evbmips/conf/` - All 54+ kernel configurations

## Platform Notes

- **Meta-Platform**: evbmips is not a single platform but a collection
- **Board Diversity**: Extreme variation in hardware, firmware, boot methods
- **Evaluation Focus**: Most boards are evaluation/development boards
- **Embedded Use**: Many boards used in routers, APs, embedded systems
- **Active Development**: New boards added regularly
- **Documentation**: Refer to board-specific documentation for details
- **Community**: Often requires board-specific knowledge from community

---

*Last Updated: 2025-11-12*
*Architecture Maintainer: NetBSD/evbmips Port*
