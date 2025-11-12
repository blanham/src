# NetBSD/next68k Boot Documentation

## Platform Overview

NetBSD/next68k supports NeXT Computer workstations (68030/68040-based models), including cubes and slabs. These were innovative Unix workstations known for pioneering object-oriented development (NeXTSTEP OS).

**Supported Models:**
- NeXT Computer (Original Cube) - 68030 @ 25 MHz
- NeXTcube - 68040 @ 25 MHz
- NeXTstation - 68040 @ 25 MHz (slab/pizza box)
- NeXTstation Color - 68040 @ 25 MHz
- NeXTstation Turbo - 68040 @ 33 MHz
- NeXTcube Turbo - 68040 @ 33 MHz

**Key Features:**
- MegaPixel Display (monochrome 1120x832)
- Color display (NeXTstation Color)
- Optical Magneto-Optical (MO) drive
- SCSI hard drive
- Ethernet
- NeXT soundbox (DSP-based audio)
- Motorola 56001 DSP

## Boot Method

NeXT systems boot from ROM monitor that loads the NetBSD bootloader from disk or network.

### Boot Chain

1. **ROM Monitor** - NeXT ROM firmware
2. **Secondary Boot** - `/boot` (boot loader)
3. **NetBSD Kernel** - `/netbsd`

### ROM Monitor

**Access:** Press Command key during power-on

**Commands:**
```
> boot sd
> boot en
> boot od      # Optical disk
> setenv boot_dev sd
```

**Boot Device Syntax:**
```
sd(controller,target,lun)    # SCSI disk
en(controller,target,lun)    # Ethernet
od(controller,target,lun)    # Optical disk
```

## 68030/68040 MMU Differences

### 68030 (Original NeXT Computer)

**Setup:**
```assembly
| 68030 MMU
pmove   %a0@,%crp       | CPU root pointer
pmove   %a1@,%tc        | Translation control
pflusha                 | Flush TLB
pmove   %a2@,%tt0       | Transparent translation
```

**Features:**
- Integrated MMU
- 25 MHz operation
- Standard 68030 paging

### 68040 (NeXTcube, NeXTstation)

**Setup:**
```assembly
| 68040 MMU
.long   0x4e7b1807      | movc d1,srp
.word   0xf4f8          | cpusha bc
.word   0xf518          | pflusha
movl    #MMU40_TCR,%d0
.long   0x4e7b0003      | movc d0,tc
```

**Features:**
- Integrated FPU
- 25 or 33 MHz (Turbo)
- Enhanced caching

**NeXT 68040 Specifics:**
- Some boards have buggy early 68040s
- Requires workarounds for mask set issues
- Turbo models more reliable

## Boot Process Stages

### Stage 1: ROM Monitor

**Functions:**
- Hardware initialization
- Memory test
- Device probing (SCSI, Ethernet, MO)
- Boot device selection
- Load bootloader

**ROM Variables:**
```
boot_dev     Default boot device
boot_file    Default kernel
console      Console device
```

### Stage 2: Secondary Boot

**Location:** `/sys/arch/next68k/stand/boot/`

**Features:**
- Minimal FFS support
- Interactive prompt
- Kernel loading
- Boot flags

**Boot Prompt:**
```
>> NetBSD BOOT [build]
boot: [device:]kernel [-flags]
```

**Examples:**
```
boot: netbsd
boot: netbsd -s
boot: sd(0,0,0):netbsd.old
```

### Stage 3: Kernel Entry

**Location:** `/sys/arch/next68k/next68k/locore.s`

**Entry Point:** `start`

**Boot Parameters (from ROM):**
```c
#define NEXT_RAMBASE 0x04000000
char *boot_arg;   // Kernel path from ROM
```

**Entry Code:**
```assembly
ASENTRY_NOPROFILE(start)
    movw    #PSL_HIGHIPL,%sr    | Disable interrupts

    | Determine CPU type (68030 or 68040)
    | Save boot arguments
    | Initialize MMU
    | Setup caches
    | Call main()
```

**CPU Detection:**
```assembly
| Detect 68030 vs 68040
movl    #CACHE_OFF,%d0
movc    %d0,%cacr
| Test specific bits to determine CPU
```

### Stage 4: Machine Init

**Location:** `/sys/arch/next68k/next68k/machdep.c`

**Initialization:**
1. Parse ROM boot arguments
2. Detect machine type (cube vs. slab, color vs. mono)
3. Initialize console (serial or framebuffer)
4. Setup memory (starts at 0x04000000)
5. Initialize SCSI (NCR 53C90)
6. Initialize Ethernet (MB8795)
7. Initialize MO drive (if present)
8. Configure DSP 56001 (if needed)
9. Mount root filesystem

**Machine Detection:**
```c
// Read from Gestalt Manager equivalent
machine = MON(char, MG_machine_type);
switch (machine) {
case NeXT_CUBE:
case NeXT_WARP9:  // 68040 cube
case NeXT_TURBO_MONO:
case NeXT_TURBO_COLOR:
    // Configure accordingly
}
```

## Memory Map

```
Physical Address Space:

0x00000000 - 0x03FFFFFF    (unmapped/reserved)

0x04000000 - 0x0FFFFFFF    Main RAM (4-64MB typical)
                            RAM base at 64MB offset

0x02000000 - 0x020FFFFF    MO (Magneto-Optical) drive

0x0C000000 - 0x0DFFFFFF    Framebuffer
  Mono: 1120x832x2 (1-bit)
  Color: 1120x832x12 or 16-bit

0x0F000000 - 0x0FFFFFFF    I/O Space
  0x0F000000               SCSI (NCR 53C90)
  0x0F100000               Ethernet (MB8795)
  0x0F118000               Serial (SCC 8530)
  0x0F120000               DSP 56001
  0x0F140000               DMA controller
  0x0F160000               Event counter
  0x0F180000               RTC (MC68HC68T1)

0x10000000 - 0x1FFFFFFF    ROM and NeXT Monitor
```

**Virtual Address Layout:**
```
0x00000000 - 0x000007FF    Unmapped (NULL trap)
0x00000800 - 0x03FFFFFF    Unmapped
0x04000000 - ...           Kernel (at NEXT_RAMBASE)
...                        Kernel heap
0x80000000 - ...           User space
```

## Build and Installation

### Building

```bash
# Bootloader
cd /usr/src/sys/arch/next68k/stand
make depend && make

# Kernel
cd /usr/src/sys/arch/next68k/conf
config GENERIC
cd ../compile/GENERIC
make depend && make
```

### Installation

**Install bootloader:**
```bash
cp /usr/mdec/boot /targetroot/boot
```

**Install kernel:**
```bash
cp netbsd /targetroot/netbsd
```

**Set ROM variables:**
```
> setenv boot_dev sd
> setenv boot_file netbsd
```

## Debugging

### Serial Console

**Hardware:**
- DB-9 serial port on back
- Standard RS-232 (9600 8N1)

**Enable:**
```c
// In kernel config
options SERCONSOLE
```

**Or set ROM:**
```
> setenv console s
```

### DDB

**Enable:**
```
options DDB
```

**Break:**
- Press BREAK on serial
- NMI button (if accessible)
- Panic or programmed breakpoint

### Common Issues

**"Boot device not found"**
- Check SCSI termination
- Verify disk is ID 0
- Check ROM boot_dev setting

**"Cannot load kernel"**
- Verify /boot exists
- Check filesystem integrity
- Ensure kernel at correct path

**Graphics Issues**
- NeXT display requires specific signals
- Check if monochrome vs. color detected correctly
- Try serial console

**MO Drive Problems**
- MO drives can be temperamental
- Boot from SCSI hard drive instead
- MO support is basic

## Hardware Support

**Working:**
- 68030 @ 25 MHz (Original)
- 68040 @ 25/33 MHz (Cube/Station)
- Memory (4-64MB typical)
- MegaPixel Display
- Serial (SCC 8530)
- SCSI (NCR 53C90)
- Ethernet (MB8795)
- RTC (MC68HC68T1)

**Partial:**
- MO drive (basic support)
- Keyboard/Mouse
- Printer port

**Not Supported:**
- DSP 56001 audio
- NeXT Dimension graphics board
- Sound output
- NeXT Laser Printer

## References

**Source Files:**
- `/sys/arch/next68k/next68k/locore.s` - Kernel entry
- `/sys/arch/next68k/next68k/machdep.c` - Machine init
- `/sys/arch/next68k/stand/boot/` - Bootloader

**Documentation:**
- NeXT Technical Summaries
- NeXT Hardware Reference
- 68030/68040 User's Manuals
- NCR 53C90 SCSI Controller
- Motorola 56001 DSP Manual (for reference)

**Notes:**
- NeXT used proprietary NeXTSTEP OS (Mach kernel + BSD)
- NetBSD provides pure BSD Unix alternative
- Many NeXT innovations later used in Mac OS X
- Steve Jobs's company between Apple stints
