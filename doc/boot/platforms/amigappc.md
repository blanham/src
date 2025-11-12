# NetBSD/amigappc Boot Documentation

## Platform Overview

NetBSD/amigappc is the port of NetBSD to Phase 5 PowerPC-based Amiga systems. These are expansion boards that replace the 68k processor in classic Amiga systems with PowerPC processors while maintaining compatibility with Amiga custom chips and hardware.

Supported systems:
- Phase 5 Blizzard PPC boards (603e/604e)
- Phase 5 CyberStorm PPC boards (604e)
- Other PowerPC accelerator cards for Amiga

These boards allow Amiga systems to run PowerPC code while retaining access to Amiga custom chips (Paula, Denise, etc.) and existing hardware.

## Boot Method

**Primary Boot Method:** Amiga Kickstart + PowerPC Boot ROM

The boot process is unique, combining:
- Amiga Kickstart ROM (68k firmware)
- PowerPC boot ROM on accelerator card
- Hybrid 68k/PowerPC environment
- Direct kernel loading from AmigaDOS or bootloader

### Boot Sequence

1. **Power-On** → Amiga Kickstart (68k) initialization
2. **Kickstart** → Initializes Amiga custom chips
3. **PPC ROM** → PowerPC processor initialization
4. **Bootloader/AmigaDOS** → Loads NetBSD kernel
5. **Kernel** → Runs on PowerPC, accesses Amiga hardware via BATs

## Boot Loader Implementation

### No Standalone Bootloader

Unlike other PowerPC platforms, amigappc typically does not use a standalone bootloader in `/sys/arch/amigappc/stand/`. Instead:

- Kernel can be loaded directly from AmigaDOS
- May use AmigaOS-based boot tools
- Kernel receives minimal boot parameters
- Most hardware detection is done by kernel

The kernel is loaded as an Amiga executable and started by the PowerPC ROM.

## BAT Register Setup

**Source:** `/sys/arch/amigappc/amigappc/machdep.c`

### Complex BAT Configuration

amigappc has one of the most complex BAT setups due to the hybrid nature of the platform:

```c
void amigappc_batinit(...)
{
    /*
     * Setup BAT registers for Amiga hardware access
     * Must map:
     * - Amiga custom chip registers
     * - Chip RAM (used by custom chips)
     * - Zorro bus I/O space
     * - PowerPC local memory
     * - ROM space
     */

    /* Detect board type (BlizzardPPC or CyberStormPPC) */
    if (/* BlizzardPPC */) {
        amigappc_batinit(
            0x00000000, BAT_BL_16M, BAT_I|BAT_G,    /* Chip RAM */
            (startkernel & 0xf0000000), BAT_BL_256M, 0,  /* Local RAM */
            0xfff00000, BAT_BL_512K, 0,              /* ROM */
            0);
    } else { /* CyberStormPPC */
        amigappc_batinit(
            0x00000000, BAT_BL_16M, BAT_I|BAT_G,    /* Chip RAM */
            (startkernel & 0xf8000000), BAT_BL_128M, 0,  /* Local RAM */
            0xfff00000, BAT_BL_512K, 0,              /* ROM */
            0x40000000, BAT_BL_256M, BAT_I|BAT_G,    /* Additional I/O */
            0);
    }
}
```

### amigappc BAT Layout (BlizzardPPC)

```
DBAT0: 0x00000000 - 16MB   (Amiga Chip RAM, cache-inhibited)
DBAT1: 0x40000000 - 256MB  (PowerPC local RAM, cacheable)
DBAT2: 0xfff00000 - 512KB  (ROM)
DBAT3: Available for expansion
```

### amigappc BAT Layout (CyberStormPPC)

```
DBAT0: 0x00000000 - 16MB   (Amiga Chip RAM, cache-inhibited)
DBAT1: 0x08000000 - 128MB  (PowerPC local RAM, cacheable)
DBAT2: 0xfff00000 - 512KB  (ROM)
DBAT3: 0x40000000 - 256MB  (Zorro/I/O space, cache-inhibited)
```

### Special BAT Considerations

The Chip RAM mapping at 0x00000000 is critical:
- Must be cache-inhibited (BAT_I|BAT_G)
- Accessed by Amiga custom chips via DMA
- Contains Amiga interrupt vectors (68k compatibility)
- Graphics data for Amiga video hardware

## Boot Process Stages

### Stage 1: Amiga Kickstart

The 68k Kickstart ROM:
1. Initializes Amiga custom chips
2. Sets up Paula (audio/floppy DMA)
3. Initializes CIA chips (timers/ports)
4. Prepares for PowerPC transition

### Stage 2: PowerPC ROM Activation

The PowerPC accelerator ROM:
1. Detects PowerPC processor type
2. Initializes PowerPC-specific hardware
3. Sets up initial BAT mappings
4. Prepares environment for kernel

### Stage 3: No Assembly Locore Entry

**Source:** `/sys/arch/amigappc/amigappc/locore.S`

Unlike other PowerPC ports, amigappc has minimal assembly startup code. Most initialization happens in C code in machdep.c due to the complex hybrid environment.

### Stage 4: C Initialization (initppc)

The initppc() function performs extensive setup:

```c
void initppc(u_int startkernel, u_int endkernel, u_int args)
{
    /* Detect memory configuration */
    /* amigappc can have multiple memory regions:
     * - Chip RAM (0-16MB)
     * - Fast RAM on PowerPC board
     * - Zorro expansion RAM
     */

    /* Set up BAT registers */
    amigappc_batinit(...);

    /* Initialize Amiga-specific hardware */
    /* - Custom chips (via BAT mappings)
     * - CIA chips
     * - Zorro bus
     */

    /* Common OEA initialization */
    oea_initppc(startkernel, endkernel);
}
```

### Stage 5: Amiga Hardware Integration

The kernel sets up interrupt handling for Amiga hardware:

```c
/* Amiga interrupt system */
struct isr *isr_ports;   /* Level 2 (ports) interrupt chain */
struct isr *isr_exter;   /* Level 6 (external) interrupt chain */

/* Amiga CIA interrupts */
void add_isr(struct isr *isr);
void remove_isr(struct isr *isr);

/* Custom chip interrupt handlers */
int lev1_intr(void *);   /* TBE, DSKBLK, SOFTINT */
int ports_intr(void *);  /* Keyboard, parallel port */
int exter_intr(void *);  /* Serial, external interrupts */
```

## MMU Requirements

### Initial State

- **MMU:** May be partially set up by PowerPC ROM
- **68k Compatibility:** Some 68k addressing may be active
- **Hybrid Mode:** Transitioning from 68k to pure PowerPC

### Supported Processors

- **PowerPC 603e:** Common on BlizzardPPC
- **PowerPC 604e:** CyberStormPPC and high-end boards

### Cache Configuration

- **603e:** 16KB I-cache, 16KB D-cache
- **604e:** 32KB I-cache, 32KB D-cache

Critical: Chip RAM must bypass cache for DMA coherency.

### Memory Translation Complexity

The amigappc MMU setup is complex because:
- Chip RAM at 0x00000000 (Amiga addressing)
- PowerPC RAM at different physical address
- Zorro I/O space
- ROM at high addresses
- Must maintain 68k vector table access

## Memory Map

### Physical Memory Layout

```
0x00000000 - 0x00ffffff    Amiga Chip RAM (up to 16MB)
0x01000000 - 0x07ffffff    Unused/Reserved
0x08000000 - 0x0fffffff    PowerPC Fast RAM (CyberStormPPC)
0x40000000 - 0x4fffffff    PowerPC Fast RAM (BlizzardPPC, varies)
0xd80000 - 0xdfffff        Amiga custom chip registers
0xe80000 - 0xefffff        Zorro II I/O space
0xf00000 - 0xffffff        Amiga system ROM (Kickstart)
0xfff00000 - 0xffffffff    PowerPC boot ROM
```

### Amiga Custom Chip Registers

Critical register regions (accessed via BAT0):
```
0xbfe001 - CIA-A (keyboard, timers)
0xbfd000 - CIA-B (serial, timers)
0xdff000 - Custom chips (Paula, Denise, etc.)
```

### Zorro Bus Space

Zorro II and III expansion:
```
0x00e80000 - 0x00efffff   Zorro II I/O
0x00f00000 - 0x00ffffff   ROM/Autoconfig
0x10000000 - ...          Zorro III space (if present)
```

## Build and Installation

### Building the Kernel

```bash
cd /sys/arch/amigappc/conf
config GENERIC
cd ../compile/GENERIC
make depend
make
```

Produces:
- `netbsd` - ELF kernel
- May need conversion for Amiga executable format

### Installing NetBSD

This is complex due to the hybrid environment:

1. **From AmigaOS:**
   - Partition hard disk
   - Create AmigaDOS and NetBSD partitions
   - Install NetBSD sets from AmigaOS
   - Configure bootloader

2. **From Existing NetBSD:**
   - Use standard installation procedures
   - Install to RDB (Rigid Disk Block) partitions

### Boot Configuration

Boot methods vary:
- Direct kernel load from AmigaDOS
- Use of loadbsd tool (68k program that loads NetBSD)
- May require specific partition types in RDB

## Debugging

### Serial Console

amigappc typically uses Amiga serial port:
- Located at 0xbfd000 (CIA-B)
- Standard Amiga serial port settings
- May use hardware handshaking

```c
#if NSER > 0
extern void ser_outintr(void);
extern void ser_fastint(void);
#endif
```

### Kernel Debugging

Enable DDB:
```
options DDB
options DDB_HISTORY_SIZE=512
```

Access via:
- Serial console
- Amiga keyboard (if console)

### Amiga-Specific Debugging

Custom chip access debugging:

```c
/* Access custom chips for debugging */
#include <amiga/amiga/custom.h>

/* LED debugging via power LED */
custom.intena = INTF_SETCLR | INTF_PORTS;
```

### Common Boot Issues

**Problem:** Chip RAM not accessible
- **Cause:** BAT0 not set up correctly
- **Solution:** Verify BAT_I|BAT_G flags for cache inhibit

**Problem:** DMA doesn't work
- **Cause:** Chip RAM caching enabled
- **Solution:** Ensure Chip RAM BAT is cache-inhibited

**Problem:** Interrupts not working
- **Cause:** Amiga interrupt chain not initialized
- **Solution:** Check custom.intena register setup

**Problem:** Graphics corruption
- **Cause:** Cache coherency issue with Chip RAM
- **Solution:** Verify cache settings, may need explicit cache flushes

## Platform-Specific Notes

### Phase 5 Boards

Phase 5 produced several PowerPC accelerators:
- **BlizzardPPC:** For Amiga 1200, budget option
- **CyberStormPPC:** For Amiga 3000/4000, high-end

### Memory Configuration

Unusual memory layout:
- Chip RAM: Shared with Amiga custom chips
- Fast RAM: PowerPC-only, much faster
- Some systems: Zorro expansion RAM

### Custom Chip Integration

NetBSD/amigappc maintains access to:
- **Paula:** Audio, floppy, serial
- **Denise:** Video (some compatibility)
- **CIA-A/B:** Timers, keyboard, mouse, ports

### Interrupt Handling

Amiga interrupt levels:
- Level 1: TBE (serial transmit), DSKBLK (floppy DMA)
- Level 2: PORTS (keyboard, parallel)
- Level 6: EXTER (serial receive, external)

Mapped to PowerPC interrupt system.

### Serial Port

Amiga serial port unique features:
- Uses CIA-B for baud generation
- Hardware in/out buffers
- RTS/CTS handshaking

### Floppy Drive

If floppy support enabled:
```c
#if NFD > 0
extern void fdintr(int);
#endif
```

Amiga floppy controller (Paula) accessed via DMA to Chip RAM.

## References

### Source Files

- `/sys/arch/amigappc/amigappc/machdep.c` - Platform initialization
- `/sys/arch/amigappc/amigappc/locore.S` - Minimal assembly startup
- `/sys/arch/amiga/amiga/` - Shared Amiga hardware code
- `/sys/arch/amiga/include/` - Amiga hardware definitions

### Hardware Documentation

- Phase 5 BlizzardPPC/CyberStormPPC manuals
- Amiga Hardware Reference Manual
- PowerPC 603e/604e User's Manuals
- Commodore Amiga technical documentation

### Amiga-Specific Resources

- Custom chip registers (Paula, Denise, etc.)
- CIA timer chips
- Zorro bus specification
- RDB (Rigid Disk Block) format

### Historical Notes

Phase 5 PowerPC accelerators were an attempt to extend the life of classic Amiga systems into the PowerPC era. NetBSD/amigappc preserves the ability to run modern software on these unique hybrid systems, bridging the gap between classic Amiga hardware and modern PowerPC processing.

The port is challenging because it must:
- Respect Amiga custom chip limitations
- Handle hybrid 68k/PowerPC environment
- Maintain DMA coherency with Chip RAM
- Support legacy Amiga peripherals
- Provide modern PowerPC performance

This makes amigappc one of the most architecturally interesting PowerPC ports.
