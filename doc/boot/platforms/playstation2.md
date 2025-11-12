# NetBSD/playstation2 Boot Documentation

$NetBSD$

## Platform Overview

**Architecture**: MIPS 64-bit (little-endian, modified ISA)
**Port Date**: 2001-12-02
**Boot Method**: PS2 BIOS → MC bootloader or USB → kernel
**Firmware**: Sony PlayStation 2 BIOS
**MMU Requirements**: Emotion Engine MMU (non-standard)

## Hardware Support

NetBSD/playstation2 supports the **Sony PlayStation 2** game console:

- **CPU**: Emotion Engine (EE) - Custom MIPS-based CPU
- **Clock**: 294.912 MHz (NTSC) or 300 MHz (PAL)
- **Memory**: 32MB RDRAM
- **Boot Devices**: Memory Card, USB, network (rare), hard disk (with adapter)
- **Graphics**: Graphics Synthesizer (GS)
- **Console**: Framebuffer or serial (requires mod)

### PlayStation 2 Hardware

**Emotion Engine (EE)**:
- Custom 64-bit MIPS core (MIPS III-based with extensions)
- 128-bit SIMD vector units (VU0, VU1)
- 16KB I-cache, 8KB D-cache
- 16KB scratchpad RAM
- Non-standard MMU (simplified)

**Memory Configuration**:
- **Main RAM**: 32MB RDRAM
- **IOP RAM**: 2MB (for I/O processor)
- **GS RAM**: 4MB (for graphics)

**I/O Processor (IOP)**:
- MIPS R3000A-based
- Handles I/O operations
- Runs PS1-compatible code

## Boot Process

### Boot Methods

NetBSD on PlayStation 2 can boot via:

1. **Memory Card**: Boot from MC exploit
2. **USB**: Boot from USB storage
3. **Hard Disk**: Boot from HDD (with network adapter)
4. **Network**: Network boot (very rare, requires modchip)

### Memory Card Boot (Most Common)

**Boot Sequence**:
```
PS2 BIOS → Memory Card Exploit → netbsd.elf
```

**Requirements**:
- Memory card with exploit save game
- NetBSD kernel on memory card or USB

**Process**:
1. **Insert Memory Card** with exploit
2. **Power On PlayStation 2**
3. **Browser**: Navigate to memory card
4. **Load Save**: Load exploit save game
5. **Exploit Runs**: Loads NetBSD kernel from MC or USB
6. **NetBSD Boots**

### Exploit Loaders

**Common Loaders**:
- **Independence**: Early MC exploit
- **uLaunchELF**: ELF loader from memory card
- **Free MC Boot (FMCB)**: Persistent softmod
- **Various Others**: Community-developed

**Functionality**:
- Read ELF executables from storage
- Load into memory
- Transfer control to entry point

### Direct Kernel Boot

**No Traditional Bootloader**: NetBSD kernel boots directly from exploit loader.

**Kernel Format**: ELF executable
**Load Address**: 0x80001000 (varies)

**Entry Point**: `mach_init()`

### Stage: Kernel

**Entry Point**: `mach_init()`
**Source**: `/sys/arch/playstation2/playstation2/machdep.c`

**Initialization**:
```c
void mach_init(void)
{
    /* Initialize Emotion Engine */
    ee_init();

    /* Clear BSS */
    memset(edata, 0, end - edata);

    /* Initialize exception vectors */
    mips_vector_init(NULL, false);

    /* Initialize memory (32MB fixed) */
    physmem = btoc(32 * 1024 * 1024);

    /* Initialize console */
    consinit();  /* Framebuffer or serial */

    /* Bootstrap VM */
    pmap_bootstrap();
}
```

## Emotion Engine Architecture

### Non-Standard MIPS

**Deviations from MIPS III**:
- **128-bit integer instructions**: Multimedia operations
- **Simplified TLB**: 48 entries, simplified structure
- **Scratchpad RAM**: 16KB fast local memory
- **Vector Units**: VU0 (in EE), VU1 (co-processor)

### Memory Segments

**Modified MIPS segments**:
```
0x00000000 - 0x01FFFFFF : Main RAM (32MB)
0x10000000 - 0x10003FFF : Scratchpad (16KB)
0x11000000 - 0x11003FFF : VU0 code memory (4KB)
0x11004000 - 0x11004FFF : VU0 data memory (4KB)
0x11008000 - 0x1100FFFF : VU1 code memory (16KB)
0x1C000000 - 0x1FFFFFFF : IOP memory (2MB)
0x70000000 - 0x70003FFF : Scratchpad (uncached alias)
0x80000000 - 0x81FFFFFF : KSEG0 (cached alias of main RAM)
0xA0000000 - 0xA1FFFFFF : KSEG1 (uncached alias of main RAM)
0xB0000000 - 0xBFFFFFFF : I/O registers
```

### I/O Registers

**Graphics Synthesizer (GS)**: 0x12000000
**GIF (Graphics Interface)**: 0x10003000
**VIF0/VIF1 (VU Interface)**: 0x10003800, 0x10003C00
**IPU (Image Processing Unit)**: 0x10002000
**DMAC (DMA Controller)**: 0x10008000
**IOP Registers**: Various addresses

## PlayStation 2 Specifics

### Framebuffer Console

**Default Console**: Framebuffer output to TV

**Resolution**:
- 640x448 (NTSC) or 640x512 (PAL)
- Interlaced output

**Configuration**: Automatically detected from PS2 BIOS settings

### Controller Input

**DualShock 2**: Can be used for basic input (limited)
**USB Keyboard**: Preferred for console input (requires USB adapter)

### Storage Options

**Memory Card** (8MB):
- Slow I/O
- Limited space
- Suitable for kernel only

**USB Storage**:
- Faster than memory card
- Can hold full root filesystem
- Requires USB driver support

**Hard Disk** (with Network Adapter):
- 40GB or 80GB drives
- Fastest option
- Requires PlayStation 2 Network Adapter

**Network (rare)**:
- Network Adapter provides Ethernet
- NFS root possible
- Requires modchip or exploit

### Graphics

**Graphics Synthesizer (GS)**:
- Hardware 3D acceleration
- Programmable via GIF/VIF/VU
- NetBSD uses basic framebuffer mode

**wscons**: NetBSD virtual console on framebuffer

## Build Instructions

### Building Kernel

```sh
./build.sh -m playstation2 kernel=GENERIC

# Or manually
cd /sys/arch/playstation2/conf
config GENERIC
cd ../compile/GENERIC
make depend
make
```

**Output**: `netbsd` - ELF executable kernel

**Load Address**: Kernel must be loaded at correct address for exploit

### Creating Bootable Image

**Memory Card Format**:
1. Format memory card with PS2 format
2. Copy kernel as ELF file
3. Install exploit loader (e.g., FMCB)

**USB Format**:
```sh
# Format USB drive with FAT32 or UFS
# Copy kernel
cp netbsd /media/usb/netbsd.elf
```

## Installation

### Prerequisites

1. **PlayStation 2 Console** (any model: SCPH-10000 to SCPH-90000)
2. **Memory Card** (8MB Sony memory card)
3. **Exploit** (Free MC Boot, Independence, etc.)
4. **USB Drive** or **Hard Disk** (for root filesystem)
5. **USB Keyboard** (for console interaction)

### Installation Steps

**1. Install Exploit to Memory Card**:

Follow exploit-specific installation instructions (e.g., FMCB installer)

**2. Prepare Root Filesystem**:

```sh
# Format USB drive or HDD with FFS
newfs /dev/sd0a

# Mount and extract NetBSD sets
mount /dev/sd0a /mnt
cd /mnt
tar xzpf /path/to/base.tgz
# ... other sets
umount /mnt
```

**3. Copy Kernel**:

Copy `netbsd` to memory card or USB drive where exploit can find it.

**4. Configure Exploit Loader**:

Configure loader to:
- Load `netbsd.elf` from storage
- Set load address correctly
- Transfer control to kernel

**5. Boot**:

Power on PS2, exploit runs, NetBSD boots.

## Debugging

### Serial Console (Requires Hardware Mod)

**Modification**: Solder serial port to PS2 motherboard
**Connection**: TTL serial (3.3V)
**Baud**: 57600 or 115200

**Enable in Kernel**:
```
options CONSPEED=57600
```

**Note**: Most users use framebuffer console (no serial mod needed).

### DDB Kernel Debugger

```
options     DDB
makeoptions COPY_SYMTAB=1
```

**Enter**: Boot with `-d` flag (requires kernel modification or button combo)

### Debugging Tips

- **Framebuffer Output**: Primary debugging method
- **printf Debugging**: Add debug prints to kernel
- **Emulators**: PCSX2 emulator partially supports NetBSD (experimental)

## Common Issues

### Kernel Doesn't Load

- **Wrong Format**: Ensure kernel is ELF, not compressed
- **Load Address**: Check exploit's expected load address
- **Corrupt File**: Re-copy kernel

### Hangs at Boot

- **Memory Size**: PS2 has fixed 32MB, kernel must not exceed
- **I/O Initialization**: Emotion Engine I/O setup issues
- **Console**: Try `-h` for serial console (if available)

### No Display

- **Video Mode**: PS2 auto-detects from BIOS (NTSC/PAL)
- **GS Initialization**: Graphics Synthesizer init failure
- **Cable**: Check A/V cable connection

### USB Issues

- **Driver Support**: Limited USB driver support in NetBSD/PS2
- **Device Compatibility**: Not all USB devices work
- **Timing**: USB on PS2 can be finicky

## Platform Notes

- **Unconventional Platform**: PlayStation 2 is not a traditional computer
- **Limited Resources**: 32MB RAM, custom hardware
- **Game Console**: Not designed for general-purpose computing
- **Homebrew Community**: Active PS2 homebrew scene
- **Emulation**: PCSX2 emulator can run some NetBSD/PS2
- **Historical Interest**: Demonstrates NetBSD's portability
- **Learning Platform**: Good for learning low-level MIPS programming

## Source Code Reference

**Platform-Specific**:
- `/sys/arch/playstation2/playstation2/machdep.c` - Machine-dependent (600 lines)
- `/sys/arch/playstation2/playstation2/autoconf.c` - Autoconfiguration
- `/sys/arch/playstation2/ee/` - Emotion Engine support

**Include Files**:
- `/sys/arch/playstation2/include/ee.h` - Emotion Engine definitions
- `/sys/arch/playstation2/include/gs.h` - Graphics Synthesizer
- `/sys/arch/playstation2/include/intr.h` - Interrupts

**Drivers**:
- `/sys/arch/playstation2/dev/` - PlayStation 2 devices
- `/sys/arch/playstation2/dev/gs.c` - Graphics Synthesizer driver
- `/sys/arch/playstation2/dev/sbus.c` - System bus

## External Documentation

- **PlayStation 2 Linux Kit**: Sony's official Linux for PS2 (discontinued)
- **PS2DEV**: PlayStation 2 development community
- **Homebrew Documentation**: Various community resources
- **Emotion Engine Manual**: Technical documentation (if available)

---

*Last Updated: 2025-11-12*
*Architecture Maintainer: NetBSD/playstation2 Port*
