# NetBSD/mac68k Boot Documentation

## Platform Overview

NetBSD/mac68k supports Apple Macintosh computers with 68k processors, from the Macintosh II (1987) through the Quadra/Centris series (mid-1990s).

**Supported Models:**
- Macintosh II, IIx, IIcx, IIci, IIsi, IIfx
- Macintosh SE/30
- Macintosh LC series (LC, LC II, LC III, LC 475, LC 520, LC 550, LC 575, LC 580, LC 630)
- Macintosh Quadra/Centris (605, 610, 630, 650, 660AV, 700, 800, 840AV, 900, 950)
- Performa series (equivalent to LC/Quadra models)

**CPU Support:**
- 68020 @ 16 MHz (Mac II, LC)
- 68030 @ 16-50 MHz (IIx, IIcx, IIci, SE/30, IIsi, LC III)
- 68040 @ 25-40 MHz (Quadra/Centris series)

**Note:** 68000-based Macs (128k, 512k, Plus, SE, Classic) are NOT supported (no MMU).

## Boot Method

Mac68k boots from Mac OS (System 6 or 7), using a Mac OS application as the bootloader. There is no standalone bootloader.

### Boot Chain

1. **Mac ROM** - Mac Toolbox ROM
2. **Mac OS** - System 6.0.7 or later, System 7.x recommended
3. **BSD/Mac68k Booter** - Mac OS application
4. **NetBSD Kernel** - Loaded by booter

### Unique Characteristics

- **No standalone bootloader**: Must boot from Mac OS
- **Toolbox calls**: Booter uses Mac OS Toolbox APIs
- **Resource fork**: Mac-specific file format
- **Virtual memory**: Mac OS VM must be OFF
- **Memory Manager**: Booter allocates memory via Mac OS

## Boot Loaders

### BSD/Mac68k Booter

**Type:** Mac OS application (MPW Tool or Application)

**Versions:**
- Booter 1.x: Classic Mac OS application
- Booter 2.x: PowerPC-compatible with 68k emulation

**Location:** Any Mac OS volume, typically System Folder

**Features:**
- GUI interface for kernel selection
- Boot flag configuration
- Video mode selection
- Serial port setup
- Memory configuration
- Preferences file support

**Usage:**
1. Launch Booter from Mac OS
2. Select kernel file
3. Configure options (single-user, serial, etc.)
4. Click "Boot Now" button

**Options Window:**
- **Kernel:** Path to NetBSD kernel
- **Boot flags:** -s (single), -a (ask), -d (debug)
- **Video:** B&W, grayscale, or color
- **Serial:** Enable serial console
- **Memory:** ST RAM size to use

## 68020/68030/68040 MMU Differences

### 68020 with 68851 PMMU

**Early Macs (Mac II):**
- Some have 68020 with external 68851 PMMU
- Others have 68020 without PMMU (not supported)

**Check:** Only Mac II with PMMU is supported

### 68030 Integrated MMU

**Most supported Macs:**
- IIx, IIcx, IIci, IIsi, SE/30, LC series
- Full virtual memory support
- Transparent translation registers

**Setup:**
```assembly
| 68030 MMU init
pmove   %a0@,%crp       | CPU root pointer
pmove   %a1@,%tc        | Translation control
pflusha                 | Flush TLB
```

**Mac-specific considerations:**
- Mac OS may have enabled MMU
- Must save Mac OS MMU state
- Restore or replace with NetBSD tables

### 68040 Integrated MMU

**Quadra/Centris series:**
- Different MMU architecture
- Must detect and handle properly

**Setup:**
```assembly
| 68040 MMU init
.long   0x4e7b1807      | movc d1,srp
.word   0xf518          | pflusha
movl    #MMU40_TCR,%d0
.long   0x4e7b0003      | movc d0,tc
```

**Mac 68040 Issues:**
- Some Quadras have buggy 68040 (need errata handling)
- LC 040 has no FPU (use software emulation)

## Boot Process Stages

### Stage 1: Mac OS Launch

**Prerequisites:**
- Mac OS 6.0.7 or later (7.x recommended)
- Virtual memory OFF (in Memory control panel)
- 32-bit addressing ON (via Mode32 or built-in)
- No conflicting extensions

**Booter Application:**
1. User launches Booter from Mac OS
2. Booter parses preferences
3. Reads kernel file into memory
4. Configures boot parameters

**Memory Allocation:**
```c
// Booter uses Mac OS Memory Manager
kernelPtr = NewPtr(kernelSize);
if (!kernelPtr)
    Alert("Not enough memory");

// Read kernel from file
FSRead(kernelFile, &kernelSize, kernelPtr);
```

### Stage 2: Mac OS Environment Handoff

**Booter Actions:**
1. Parse kernel (ELF or a.out)
2. Allocate memory for kernel
3. Load kernel sections
4. Build Mac68k-specific boot info structure
5. Save Mac OS MMU state (CRP, TC, TT registers)
6. Disable interrupts
7. Call kernel entry point

**Boot Info Structure:**
```c
struct mac68k_bootinfo {
    u_long  video_addr;     // Framebuffer address
    u_long  video_len;      // Framebuffer size
    u_long  mac_model;      // Mac model ID
    u_long  serial_flags;   // Serial port config
    u_long  boottime;       // Boot time
    // Mac OS MMU state
    u_long  macos_crp1;     // CRP high
    u_long  macos_crp2;     // CRP low
    u_long  macos_tc;       // TC register
    u_long  macos_tt0;      // TT0
    u_long  macos_tt1;      // TT1
};
```

**Register Convention:**
```
a0 = bootinfo structure pointer
a1 = environment variables (optional)
d4 = maxmem (top of RAM)
d5 = netbsd kernel name (unused)
d6 = bootdev (unused on mac68k)
d7 = boothowto (boot flags)
```

### Stage 3: Kernel Entry (locore.s)

**Location:** `/sys/arch/mac68k/mac68k/locore.s`

**Entry Point:** `start`

**Initial State:**
- Running in Mac OS address space
- Mac OS MMU may be enabled
- Caches may be on
- Interrupts should be disabled

**Process:**
```assembly
ASENTRY_NOPROFILE(start)
    movw    #PSL_HIGHIPL,%sr    | Disable interrupts
    lea     _ASM_LABEL(tmpstk),%sp | Setup temp stack

    | Save boot parameters
    movl    %a0,_C_LABEL(mac68k_bootinfo)
    movl    %d4,_C_LABEL(maxmem)
    movl    %d7,_C_LABEL(boothowto)

    | Detect CPU type
    movl    #CACHE_OFF,%d0
    movc    %d0,%cacr
    | Test for 68030/040 as usual

    | Parse Mac OS environment
    jbsr    _C_LABEL(getenvvars)

    | Initialize MMU for detected CPU
    | (May need to switch from Mac OS page tables)

    | Jump to C code
    jbsr    _C_LABEL(main)
```

**CPU Detection:**
```assembly
| Determine CPU type empirically
movl    #0x200,%d0              | data freeze bit
movc    %d0,%cacr               | only on 68030
movc    %cacr,%d0
tstl    %d0
jeq     Lnot68030

| 68030 detected
movl    #CACHE_OFF,%d0
movc    %d0,%cacr
movl    #MMU_68030,_C_LABEL(mmutype)
movl    #CPU_68030,_C_LABEL(cputype)
jra     Lstart1

Lnot68030:
| Test for 68040
bset    #31,%d0                 | data cache enable
movc    %d0,%cacr
movc    %cacr,%d0
tstl    %d0
beq     Lis68020                | No, must be 68020

| 68040 detected
movl    #CACHE40_OFF,%d0
movc    %d0,%cacr
.word   0xf4f8                  | cpusha bc
movl    #MMU_68040,_C_LABEL(mmutype)
movl    #CPU_68040,_C_LABEL(cputype)
```

### Stage 4: Machine-Dependent Init

**Location:** `/sys/arch/mac68k/mac68k/machdep.c`

**Initialization:**
1. Parse boot info structure
2. Determine Mac model from Gestalt Manager or ROM
3. Setup video based on saved address
4. Initialize VIA (Versatile Interface Adapter)
5. Setup serial ports (SCC Z8530)
6. Configure ADB (Apple Desktop Bus)
7. Probe NuBus cards
8. Mount root filesystem

**Mac Model Detection:**
```c
// Use Gestalt Manager info or ROM check
switch (mac68k_machine.machineID) {
case MACH_MACII:
case MACH_MACIIX:
case MACH_MACIICX:
case MACH_MACIICI:
// ... configure for specific Mac model
}
```

## Memory Map

### Physical Address Space

```
Mac memory maps vary by model, but generally:

0x00000000 - 0x0FFFFFFF    RAM (up to 256MB on Quadras)
  0x00000000 - 0x000003FF    Exception vectors
  0x00000400 - ...           Mac OS low memory (preserved)
  ...                        Kernel

0x40000000 - 0x4FFFFFFF    ROM (varies by Mac model)
  Typically 0x40000000 - 0x400FFFFF for 1MB ROM
  Or 0x40800000 - 0x40BFFFFF for 4MB ROM

0x50000000 - 0x5FFFFFFF    NuBus slot space (Mac II series)
  0x50000000 - 0x50FFFFFF    Slot 9 (internal video)
  0x51000000 - 0x51FFFFFF    Slot A
  0x52000000 - 0x52FFFFFF    Slot B
  ...
  0x56000000 - 0x56FFFFFF    Slot E

0x60000000 - 0x6FFFFFFF    More NuBus space (Quadra)

0xF0000000 - 0xFFFFFFFF    I/O space
  0xF0000000               VIA1 (Versatile Interface Adapter)
  0xF0004000               VIA2
  0xF0006000               SCC (Zilog 8530 serial)
  0xF0010000               SCSI (NCR 5380 or 53C96)
  0xF0014000               Sound buffer
  ...
```

**Mac-specific regions:**
- **System heap:** Mac OS structures (avoid)
- **Screen buffer:** Framebuffer location
- **ROM:** Mac Toolbox ROM (read-only)
- **Parameter RAM:** 256 bytes NVRAM

### Kernel Virtual Address Space

```
0x00000000 - 0x000007FF    Unmapped (NULL trap)
0x00000800 - ...           Kernel text/data
...                        Kernel heap
...                        Page tables
0x80000000 - ...           User space
```

**Video Memory:**
- Mapped at boot from Mac OS configuration
- Address saved in bootinfo structure
- Can be internal or NuBus card

## Build and Installation

### Building Components

**Booter (requires MPW or CodeWarrior):**
```bash
# From Mac OS environment using MPW
# Or cross-compile (difficult, not recommended)
# Binaries typically provided in release
```

**Kernel:**
```bash
cd /usr/src/sys/arch/mac68k/conf
config GENERIC
cd ../compile/GENERIC
make depend && make
```

### Installation

**Prepare Mac OS Boot Disk:**
1. Format HFS or HFS+ volume in Mac OS
2. Install Mac OS 7.x
3. Turn OFF virtual memory
4. Turn ON 32-bit addressing

**Install NetBSD:**
1. Copy Booter application to Mac volume
2. Copy NetBSD kernel to Mac volume
3. Create Booter preferences

**Dual Boot Setup:**
- Keep Mac OS bootable
- Store NetBSD root on separate partition
- Use Apple_UNIX partition type

## Debugging

### Serial Console

**Hardware:**
- Modem port (external 8-pin mini-DIN)
- Printer port (external 8-pin mini-DIN)

**Pinout:** Use Mac serial cable or adapter

**Enable:**
- In Booter: Check "Serial Console" option
- In kernel: options SERCONSOLE

**Speed:** Typically 57600 bps (fast Macs) or 9600 bps

### DDB

**Enable:**
```
options DDB
```

**Break methods:**
- Keyboard: Command-Power (on ADB Macs)
- Serial: Send BREAK
- Panic or programmed breakpoint

### Mac-Specific Debugging

**Framebuffer Issues:**
- Check bootinfo video_addr
- Verify with MacsBug or ResEdit
- Some Macs have quirky video

**ADB Problems:**
- Keyboard/mouse use ADB
- Can hang if ADB not initialized
- Use serial console if ADB fails

**NuBus Cards:**
- Probed at boot
- Check declaration ROM
- May conflict with drivers

## Common Issues

### "Not enough memory"
- **Cause:** Mac OS using too much RAM
- **Solution:**
  - Disable extensions
  - Increase System heap
  - Remove INITs/cdevs

### "Can't find kernel"
- **Cause:** File not found or wrong path
- **Solution:**
  - Use full path (volume:folder:netbsd)
  - Check file permissions
  - Verify file not corrupted

### "Virtual memory is on"
- **Cause:** Mac OS VM conflicts with NetBSD
- **Solution:**
  - Turn OFF in Memory control panel
  - Restart Mac OS
  - Then run Booter

### "32-bit addressing required"
- **Cause:** Running in 24-bit mode
- **Solution:**
  - Install MODE32 (System 6)
  - Or upgrade to System 7
  - Check Memory control panel

### System hangs at "Starting NetBSD"
- **Cause:** Hardware conflict or bad kernel
- **Solution:**
  - Boot with -s (single user)
  - Disable ADB
  - Try serial console
  - Remove NuBus cards

### Video corruption
- **Cause:** Incorrect framebuffer address
- **Solution:**
  - Check Booter video settings
  - Try B&W mode
  - Update Booter version

## Hardware Support

**Well Supported:**
- Internal video (most models)
- SCC serial ports (Z8530)
- SCSI (NCR 5380, 53C96)
- ADB keyboard/mouse
- Internal floppy (SWIM)
- Ethernet (Sonic, MACE)

**Partially Supported:**
- NuBus Ethernet cards
- NuBus SCSI cards
- Some video cards

**Not Supported:**
- Sound (planned but incomplete)
- Floppy disk (IWM on older Macs)
- Some proprietary Apple hardware

## References

**Source Files:**
- `/sys/arch/mac68k/mac68k/locore.s` - Kernel entry
- `/sys/arch/mac68k/mac68k/machdep.c` - Machine init
- `/sys/arch/mac68k/mac68k/pmap_bootstrap.c` - MMU setup
- `/sys/arch/mac68k/dev/` - Device drivers

**Documentation:**
- Inside Macintosh (Apple, multiple volumes)
- Guide to the Macintosh Family Hardware (Apple)
- Mac68k FAQ (NetBSD wiki)
- Designing Cards and Drivers for the Macintosh Family

**Mac OS:**
- System 6.0.7 or later
- System 7.1 or 7.5 recommended
- Mac OS 8.x works but overkill

**Tools:**
- Macintosh Programmer's Workshop (MPW)
- ResEdit (resource editor)
- MacsBug (debugger)

## Appendix: Mac Models

**Mac II Series:**
- Mac II: 68020/68851, NuBus, open architecture
- Mac IIx/IIcx: 68030, faster, same NuBus
- Mac IIci: 68030, cache slot, integrated video option
- Mac IIsi: 68030, 1 NuBus slot, compact
- Mac IIfx: 68030 @ 40 MHz, fast SCSI, expensive

**Compact Macs:**
- SE/30: 68030, one expansion slot, popular

**LC Series:**
- LC, LC II, LC III: Budget Macs, limited expansion
- LC 040: 68040 but crippled bus (slow)

**Quadra/Centris:**
- 68040 at various speeds
- Fast SCSI-2
- More RAM capacity
- Professional/server use

**Quadra Special Models:**
- 840AV, 660AV: AV features (not all supported)
- Some have AAUI Ethernet (not standard)

**Note on PowerPC:**
- This is mac68k port (680x0 only)
- Power Macintosh use macppc port
- Hybrid Power Macs ran 68k code in emulation (not supported for booting NetBSD/mac68k)
