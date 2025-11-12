# NetBSD/luna68k Boot Documentation

## Platform Overview

NetBSD/luna68k supports OMRON LUNA workstations, Japanese-made Unix workstations from the late 1980s.

**Supported Models:**
- LUNA (original) - 68020 @ 20 MHz
- LUNA-II - 68030 @ 25 MHz

**Hardware:**
- CPU: MC68020 or MC68030 with MMU
- RAM: 16MB - 64MB
- Graphics: Monochrome or 8-bit color
- Network: AMD Lance Ethernet (le)
- SCSI: MB89352 SCSI controller (spc)
- Serial: 4 serial ports via Hitachi HD64570 SCC

## Boot Method

LUNA systems boot from ROM firmware that provides a boot monitor and loads the NetBSD bootloader.

### Boot Chain

1. **ROM Monitor** - OMRON firmware
2. **Secondary Boot** - `/boot` loaded from disk
3. **NetBSD Kernel** - `/netbsd`

### Boot Process

**From ROM Monitor:**
```
LUNA> b
or
LUNA> bo sd(0,0,0)netbsd
```

The ROM monitor:
1. Initializes hardware
2. Probes SCSI devices
3. Loads `/boot` from root partition
4. Executes boot loader

## 68020/68030 MMU Setup

### 68020 with 68851 PMMU

**LUNA (original):**
- MC68020 with external MC68851 PMMU
- Two-level page tables
- Standard pmove instructions

### 68030 Integrated MMU

**LUNA-II:**
- MC68030 with integrated MMU
- Compatible with 68851
- Faster MMU operations
- Transparent translation support

**Setup:**
```assembly
| 68030 MMU initialization
lea     _C_LABEL(Sysseg_pa),%a0
movl    %a0@,%d0
pmove   %d0,%crp                | Load root pointer
pflusha                         | Flush TLB
movl    _C_LABEL(protott),%d0
pmove   %d0,%tt0                | Setup transparent translation
lea     _C_LABEL(protocrp),%a0
pmove   %a0@,%tc                | Enable MMU
```

## Boot Process Stages

### Stage 1: ROM Monitor

**Features:**
- Hardware diagnostics
- SCSI boot support
- Network boot (TFTP)
- Memory test
- System configuration

**Commands:**
```
b [device]      Boot from device
g [addr]        Go to address
d [addr]        Display memory
m [addr]        Modify memory
t               Run diagnostics
```

### Stage 2: Secondary Boot

**Location:** `/sys/arch/luna68k/stand/boot/`

**Functions:**
- FFS filesystem support
- Kernel loading
- Boot parameter passing
- Interactive prompt

**Boot Syntax:**
```
boot: [device:]kernel [-flags]
```

**Examples:**
```
boot: netbsd -s          # Single user
boot: netbsd.old         # Alternate kernel
boot: sd(0,0)netbsd     # Specify SCSI device
```

### Stage 3: Kernel Entry

**Location:** `/sys/arch/luna68k/luna68k/locore.s`

**Process:**
1. Receive boot parameters in registers:
   - d6: boot device
   - d7: boot flags (howto)
2. Detect CPU type (68020 vs 68030)
3. Initialize MMU
4. Setup caches
5. Call `main()`

## Memory Map

```
Physical Address Space:

0x00000000 - 0x01FFFFFF    Main RAM (16-64MB)
  0x00000000 - 0x00001FFF    ROM vectors (mapped from ROM)
  0x00002000 - ...           Kernel

0x40000000 - 0x4FFFFFFF    Expansion bus

0x60000000 - 0x6FFFFFFF    Device space
  0x61000000               HD64570 SCC (serial)
  0x63000000               MB89352 SCSI controller
  0x64000000               AMD Lance Ethernet

0xA0000000 - 0xAFFFFFFF    Framebuffer
  0xB0000000               Graphics control registers

0xF0000000 - 0xFFFFFFFF    ROM and I/O
  0xF0000000               System ROM (1MB)
```

## Build and Installation

### Building

```bash
# Bootloader
cd /usr/src/sys/arch/luna68k/stand
make depend && make

# Kernel
cd /usr/src/sys/arch/luna68k/conf
config GENERIC
cd ../compile/GENERIC
make depend && make
```

### Installation

```bash
# Install bootloader
cp /usr/mdec/boot /targetroot/boot

# Install kernel
cp netbsd /targetroot/netbsd
```

## Debugging

### Serial Console

**Configuration:**
- Port 0: Console (9600 8N1)
- Ports 1-3: Available

**Enable in kernel:**
```
options CONSPEED=9600
```

### DDB

```
options DDB
```

**Enter debugger:** Press BREAK on console or trigger panic

## Hardware Support

**Working:**
- CPU: 68020, 68030
- Memory: 16-64MB
- Serial: HD64570 (4 ports)
- SCSI: MB89352 controller
- Ethernet: AMD Lance
- Graphics: Monochrome/8-bit color

**Limitations:**
- No sound support
- Limited graphics acceleration
- No VME bus support (if present)

## References

- `/sys/arch/luna68k/` - Source tree
- OMRON LUNA Technical Reference (Japanese)
- MB89352 SCSI Controller Manual
- HD64570 SCC Technical Manual
