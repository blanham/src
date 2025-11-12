# NetBSD/news68k Boot Documentation

## Platform Overview

NetBSD/news68k supports Sony NEWS (Network Engineering WorkStation) m68k-based workstations, popular Unix workstations in Japan during the late 1980s and early 1990s.

**Supported Models:**
- NEWS-1200, NEWS-1250, NEWS-1400, NEWS-1450
- NEWS-1500, NEWS-1520, NEWS-1550, NEWS-1700, NEWS-1750
- NEWS-1800, NEWS-1830, NEWS-1850, NEWS-1860, NEWS-1900,NEWS-1950

**CPUs:**
- 68020 @ 12.5 MHz (NEWS-12xx)
- 68030 @ 20/25 MHz (NEWS-14xx, 15xx, 17xx)
- 68040 @ 25/33 MHz (NEWS-18xx, 19xx)

**Features:**
- Sony proprietary graphics
- Lance Ethernet
- NCR 53C90 SCSI
- Multiple serial ports
- News-OS native OS

## Boot Method

NEWS systems boot from ROM firmware that loads the NetBSD bootloader from disk or network.

### Boot Chain

1. **ROM Firmware** - Sony boot ROM
2. **Secondary Boot** - `/boot` from disk/network
3. **NetBSD Kernel** - `/netbsd`

### ROM Monitor

**Commands:**
- `b` - Boot from default device
- `b device` - Boot from specified device
- `halt` - Halt system

**Boot Syntax:**
```
> b
> b sd(0,0,0)
> b en(0,0,0)  # Ethernet boot
```

## 68020/68030/68040 MMU Setup

### 68020 with 68851 (NEWS-12xx)

- External PMMU
- Standard pmove instructions

### 68030 (NEWS-14xx, 15xx, 17xx)

- Integrated MMU
- Transparent translation

```assembly
pmove   %a0@,%crp
pmove   %a1@,%tc
pflusha
```

### 68040 (NEWS-18xx, 19xx)

- Different MMU architecture
- movc instructions

```assembly
.long   0x4e7b1807      | movc d1,srp
.word   0xf518          | pflusha
```

## Boot Process Stages

### Stage 1: ROM Firmware

**Functions:**
- Hardware initialization
- Device probing
- Boot device selection
- Load secondary boot

### Stage 2: Secondary Boot

**Location:** `/sys/arch/news68k/stand/boot/`

**Features:**
- FFS filesystem support
- Interactive prompt
- Kernel loading

**Boot Prompt:**
```
NetBSD/news68k Secondary Boot

boot: [device:]kernel [-flags]
```

### Stage 3: Kernel Entry

**Location:** `/sys/arch/news68k/news68k/locore.s`

**Register Convention:**
```
d4 = maxmem
d5 = bootname pointer
d6 = bootdev
d7 = boothowto
```

**Entry:**
```assembly
ASENTRY_NOPROFILE(start)
    movw    #PSL_HIGHIPL,%sr
    | Save boot parameters
    | Detect CPU
    | Initialize MMU
    | Call main()
```

## Memory Map

```
Physical Address Space:

0x00000000 - 0x0FFFFFFF    RAM (varies by model)

0x60000000 - 0x6FFFFFFF    Device space
  - SCSI controller
  - Ethernet
  - Serial ports

0xE0000000 - 0xEFFFFFFF    Graphics framebuffer

0xF0000000 - 0xFFFFFFFF    ROM and I/O
```

## Build and Installation

```bash
# Bootloader
cd /usr/src/sys/arch/news68k/stand
make depend && make

# Kernel
cd /usr/src/sys/arch/news68k/conf
config GENERIC
cd ../compile/GENERIC
make depend && make
```

## Debugging

**Serial Console:**
- Default 9600 8N1
- Enable with `options SERCONSOLE`

**DDB:**
```
options DDB
```

## References

- `/sys/arch/news68k/` - Source tree
- Sony NEWS Technical Documentation (Japanese)
- NCR 53C90 SCSI Controller Manual
