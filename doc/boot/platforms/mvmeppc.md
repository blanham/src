# NetBSD/mvmeppc Boot Documentation

## Platform Overview

NetBSD/mvmeppc is the port of NetBSD to Motorola MVME (Motorola VMEbus Evaluation) PowerPC boards. These are industrial-grade VMEbus single-board computers designed for embedded and real-time applications.

Supported boards:
- MVME2100 (MPC8240, 603e core)
- MVME2400/2401/2403/2404/2406 (MPC750/7400)
- MVME2600/2700 (MPC7400/7410/7450/7455)
- Other compatible MVME PowerPC boards

These are typically used in industrial control, telecommunications, and military applications.

## Boot Method

**Primary Boot Method:** PPCBUG Firmware

MVME boards use Motorola's PPCBUG firmware:
- ROM-based debug monitor
- Similar to MVMEBUG on 68k boards
- Provides low-level diagnostics
- Can boot from VMEbus, SCSI, Ethernet, or Flash
- No OpenFirmware

### Boot Sequence

1. **Power-On** → PPCBUG initialization
2. **PPCBUG** → Hardware diagnostics
3. **PPCBUG** → Loads bootloader from specified device
4. **Bootloader** → Loads NetBSD kernel with bootinfo
5. **Kernel** → Initializes using bootinfo structure

## Boot Loader Implementation

### Primary Bootloader: boot (via PPCBUG)

**Location:** `/sys/arch/mvmeppc/stand/boot/`

The mvmeppc bootloader:
- Loaded by PPCBUG firmware
- Uses PPCBUG system calls for I/O
- Provides bootinfo structure to kernel
- Supports SCSI and network boot
- Simple, reliable design

**Key Files:**
- `boot/boot.c` - Bootloader main code
- `libsa/` - Standalone library

## BAT Register Setup

**Source:** `/sys/arch/mvmeppc/mvmeppc/machdep.c:initppc()`

### BAT Configuration

MVME boards use PReP-style initialization with platform-specific BATs:

```c
void initppc(u_long startkernel, u_long endkernel, void *btinfo)
{
    /* Copy bootinfo */
    memcpy(&bootinfo, btinfo, sizeof(bootinfo));

    /* Identify platform from bootinfo */
    ident_platform();
    if (platform == NULL)
        panic("Unsupported model: MVME%04x", bootinfo.bi_modelnumber);

    /* Set memory regions from bootinfo */
    physmemr[0].start = 0;
    physmemr[0].size = bootinfo.bi_memsize & ~PGOFSET;
    availmemr[0].start = (endkernel + PGOFSET) & ~PGOFSET;
    availmemr[0].size = bootinfo.bi_memsize - availmemr[0].start;

    /* Set CPU clock from bootinfo */
    ticks_per_sec = bootinfo.bi_clocktps;
    ns_per_tick = 1000000000 / ticks_per_sec;

    /* Common PReP initialization (sets up BATs) */
    prep_initppc(startkernel, endkernel, boothowto, 0);

    /* Platform-specific PIC setup */
    (*platform->pic_setup)();
}
```

The BAT setup is done by `prep_initppc()` and varies by board model, typically mapping:
- PCI memory space
- PCI I/O space
- VMEbus space
- On-board device registers

## Boot Process Stages

### Stage 1: PPCBUG Firmware

PPCBUG performs:
1. POST (Power-On Self Test)
2. Memory sizing and testing
3. PCI bus enumeration
4. VMEbus initialization
5. Boot device selection

### Stage 2: Assembly Entry Point

**Source:** `/sys/arch/mvmeppc/mvmeppc/locore.S:__start`

```assembly
__start:
    /* Disable FPU/MMU/exceptions */
    li      0, 0
    mtmsr   0
    isync

    /* Compute end of kernel memory */
#if NKSYMS || defined(DDB) || defined(MODULAR)
    lis     7, _C_LABEL(startsym)@ha
    addi    7, 7, _C_LABEL(startsym)@l
    stw     3, 0(7)
    lis     7, _C_LABEL(endsym)@ha
    addi    7, 7, _C_LABEL(endsym)@l
    stw     4, 0(7)
#else
    lis     4, _C_LABEL(end)@ha
    addi    4, 4, _C_LABEL(end)@l
#endif

    /* Initialize CPU info */
    INIT_CPUINFO(4, 1, 9, 0)

    /* Call C initialization */
    lis     3, __start@ha
    addi    3, 3, __start@l
    bl      _C_LABEL(initppc)
    bl      _C_LABEL(main)
```

### Stage 3: Platform Identification

```c
void ident_platform(void)
{
    /* Identify board from model number in bootinfo */
    switch (bootinfo.bi_modelnumber) {
    case 0x2100:
        platform = &platform_mvme2100;
        break;
    case 0x2400:
    case 0x2401:
    case 0x2403:
    case 0x2404:
    case 0x2406:
        platform = &platform_mvme2400;
        break;
    case 0x2600:
    case 0x2700:
        platform = &platform_mvme2600;
        break;
    default:
        platform = NULL;
    }
}
```

### Stage 4: Interrupt Controller Setup

Each platform has specific interrupt controller initialization:

```c
void cpu_startup(void)
{
    /* Map PReP interrupt vector register */
    prep_intr_reg = (vaddr_t) mapiodev(MVMEPPC_INTR_REG, PAGE_SIZE, false);
    if (!prep_intr_reg)
        panic("startup: no room for interrupt register");

    /* Common OEA startup */
    snprintf(modelbuf, sizeof(modelbuf),
        "%s\nCore Speed: %dMHz, Bus Speed: %dMHz\n",
        platform->model,
        bootinfo.bi_mpuspeed/1000000,
        bootinfo.bi_busspeed/1000000);
    oea_startup(modelbuf);

    /* Enable hardware interrupts */
    splraise(-1);
    __asm volatile ("mfmsr %0; ori %0,%0,%1; mtmsr %0"
                  : "=r"(msr) : "K"(PSL_EE));

    bus_space_mallocok();
}
```

## MMU Requirements

### Initial State

- **MMU:** Disabled during locore.S
- **Caches:** Enabled after initialization
- **Real Mode:** Early boot runs in real mode

### Supported Processors

- **MPC8240:** 603e core, 200-400 MHz
- **MPC750:** G3, 300-500 MHz
- **MPC7400/7410:** G4, 400-700 MHz
- **MPC7450/7455:** G4+, 600-1.4 GHz

### Cache Configuration

Varies by processor:
- **603e:** 16KB I-cache, 16KB D-cache
- **750:** 32KB I-cache, 32KB D-cache
- **7400+:** 32KB I-cache, 32KB D-cache, optional L2/L3

## Memory Map

### Physical Memory Layout

```
0x00000000 - 0x00003fff    Exception vectors
0x00004000 - RAM_END       Main memory (varies: 64MB-2GB)
0x80000000 - 0xbfffffff    PCI memory space
0xc0000000 - 0xdfffffff    VMEbus A32 space (on some models)
0xf0000000 - 0xfeffffff    I/O and device registers
0xff000000 - 0xffffffff    Flash/ROM
```

### Board-Specific Registers

Each MVME model has specific register layouts for:
- PCI bridge configuration
- VMEbus interface
- On-board devices
- Interrupt controller
- NVRAM

## Build and Installation

### Building the Bootloader

```bash
cd /sys/arch/mvmeppc/stand
make cleandir
make depend
make
```

### Installing via PPCBUG

From PPCBUG prompt:

```
PPCBUG> niot
# Configure network boot

PPCBUG> bo 0 0
# Boot from network

# Or boot from SCSI:
PPCBUG> bo 0 4
# SCSI controller 0, ID 4
```

### Network Boot Setup

1. Configure BOOTP/DHCP server
2. Configure TFTP server
3. Use PPCBUG to configure network parameters
4. Boot via `niot` and `bo` commands

## Debugging

### PPCBUG Debugger

PPCBUG provides extensive debugging:

```
PPCBUG> md <address>        ; Memory display
PPCBUG> mm <address>        ; Memory modify
PPCBUG> rd                  ; Register display
PPCBUG> t                   ; Trace
PPCBUG> br <address>        ; Breakpoint
```

### Serial Console

MVME boards typically use serial console:
- Default: 9600 baud, 8N1
- EIA-232 on front panel
- Can be configured in PPCBUG

### Kernel Debugging

Enable DDB in kernel:
```
options DDB
options DDB_HISTORY_SIZE=512
```

### Common Boot Issues

**Problem:** Boot device not found
- **Cause:** Incorrect PPCBUG configuration
- **Solution:** Use PPCBUG `env` command to set boot device

**Problem:** Wrong board detected
- **Cause:** Bootinfo structure incorrect
- **Solution:** Update bootloader

**Problem:** VMEbus devices not working
- **Cause:** VMEbus not initialized
- **Solution:** Platform-specific VMEbus setup may be needed

## Platform-Specific Notes

### MVME2100

- Entry-level PowerPC VMEbus SBC
- MPC8240 (603e core)
- 100Mbps Ethernet
- SCSI and IDE
- Up to 256MB RAM

### MVME2400 Series

- Mid-range G3-based boards
- MPC750 processor
- Enhanced I/O capabilities
- Better performance than 2100
- Up to 512MB RAM

### MVME2600/2700

- High-end G4-based boards
- AltiVec support
- Higher clock speeds
- More memory capacity
- Advanced features

### VMEbus Support

VMEbus functionality:
- A16, A24, A32 address spaces
- D8, D16, D32, D64 data transfers
- Master and slave capabilities
- Interrupt handling

## References

### Source Files

- `/sys/arch/mvmeppc/mvmeppc/locore.S` - Assembly boot code
- `/sys/arch/mvmeppc/mvmeppc/machdep.c` - Platform initialization
- `/sys/arch/mvmeppc/stand/boot/` - Bootloader
- `/sys/arch/mvmeppc/include/bootinfo.h` - Bootinfo structure

### Documentation

- Motorola MVME PowerPC board manuals
- PPCBUG Debug Monitor User's Manual
- VMEbus Specification (IEEE 1014)
- PowerPC processor documentation

### Bootinfo Structure

The bootinfo passed from bootloader contains:
```c
struct mvmeppc_bootinfo {
    uint32_t bi_boothowto;      /* Boot flags */
    uint32_t bi_bootaddr;       /* Boot device address */
    uint32_t bi_modelnumber;    /* Board model number */
    uint32_t bi_memsize;        /* Memory size */
    uint32_t bi_mpuspeed;       /* CPU speed in Hz */
    uint32_t bi_busspeed;       /* Bus speed in Hz */
    uint32_t bi_clocktps;       /* Decrementer ticks per second */
    /* ... */
};
```
