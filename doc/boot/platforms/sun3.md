# NetBSD/sun3 Boot Documentation

## Platform Overview

NetBSD/sun3 supports Sun Microsystems Sun-3 workstations, which were extremely popular Unix workstations in the mid-to-late 1980s based on the Motorola 68020 and 68030 processors.

**Supported Models:**

**Sun-3 (68020):**
- Sun 3/50 - Compact desktop
- Sun 3/60 - Desktop
- Sun 3/75 - Desktop with FPU
- Sun 3/110 - Desktop
- Sun 3/160 - Deskside
- Sun 3/180 - Server with VME

**Sun-3x (68030):**
- Sun 3/80 - Desktop with 68030
- Sun 3/470 - Server/multiuser system

**CPUs:**
- 68020 @ 16.67/20/25 MHz (Sun-3)
- 68030 @ 20/25/33 MHz (Sun-3x)

## Boot Method

Sun-3 systems boot from OpenBoot PROM (early version) that provides boot and diagnostic capabilities.

### Boot Chain

1. **PROM Monitor** - Sun boot PROM
2. **Bootloader** - Minimal (PROM can load kernel directly)
3. **NetBSD Kernel** - `/netbsd`

### PROM Monitor

**Access:** Press Stop-A (or L1-A on Type-4 keyboard)

**Commands:**
```
> b              Boot from default device
> b sd(0,0,0)    Boot from SCSI disk 0
> b le()         Boot from Ethernet
> boot           Boot default
```

**PROM Variables:**
```
> k              Display environment
bootdev          Boot device
bootfile         Kernel path (optional)
```

## 68020/68030 MMU Differences

### 68020 + 68851 PMMU (Sun-3)

**Configuration:**
- External 68851 Paged Memory Management Unit
- Two-level page tables
- CRP (CPU Root Pointer)
- SRP (Supervisor Root Pointer)
- TC (Translation Control)

**Setup:**
```assembly
| 68020/68851 MMU
pmove   %a0@,%crp       | Load CPU root pointer
pmove   %a1@,%tc        | Load translation control
pflusha                 | Flush TLB
```

**Sun-3 MMU Characteristics:**
- Segment-based architecture (like sun2 but with PMMU)
- Context registers
- Segment map and page map
- Function codes for different address spaces

### 68030 Integrated MMU (Sun-3x)

**Configuration:**
- Integrated PMMU (compatible with 68851)
- Three-level page tables
- Transparent translation registers (TT0/TT1)
- Faster MMU operations

**Setup:**
```assembly
| 68030 MMU
pmove   %a0@,%crp       | Load CPU root pointer
pmove   %a1@,%tc        | Load translation control
pflusha                 | Flush TLB
pmove   %a2@,%tt0       | Transparent translation
```

**Sun-3x Specifics:**
- Different memory map than Sun-3
- Enhanced performance
- More RAM capacity
- Some I/O address differences

## Boot Process Stages

### Stage 1: PROM Monitor

**Functions:**
- Power-on self-test (POST)
- Memory test
- Device probing (SCSI, Ethernet, VME)
- Network configuration (for netboot)
- Boot device selection

**Boot Options:**
```
> b                    # Boot default
> b sd(0,0,0)          # SCSI disk, controller 0, ID 0, LUN 0
> b le()               # Network boot
> boot -s              # Single-user mode
> boot -a              # Ask for root device
```

### Stage 2: Kernel Load

**Sun-3 uses direct kernel load:**
1. PROM reads kernel from boot device
2. PROM loads to memory
3. PROM jumps to kernel entry

**No separate bootloader** - PROM has built-in boot capability for:
- SCSI disks (sd)
- Ethernet (le)
- Tape (st)

### Stage 3: Kernel Entry (locore.s)

**Location:** `/sys/arch/sun3/sun3/locore.s`

**Entry Point:** `start`

**Initial State:**
- Loaded at low physical address
- Linked at high virtual address (KERNBASE3)
- MMU disabled
- Running physical==virtual initially

**Relocation Process:**
```assembly
    .text
GLOBAL(kernel_text)

ASGLOBAL(tmpstk)
ASGLOBAL(start)

| Position-independent code initially
    movw    #PSL_HIGHIPL,%sr    | no interrupts
    moveq   #FC_CONTROL,%d0     | make movs access "control"
    movc    %d0,%sfc            | space where sun3 designers
    movc    %d0,%dfc            | put all the "useful" stuff

| Set context zero
    moveq   #0,%d0
    movsb   %d0,CONTEXT_REG

| Copy segment mappings to high memory
| This maps kernel at KERNBASE3 (0xE000000)
    movl    #(SEGMAP_BASE+0),%a0        | src
    movl    #(SEGMAP_BASE+KERNBASE3),%a1| dst
    movl    #(0x400000/NBSG),%d0        | count

L_per_pmeg:
    movsb   %a0@,%d1            | copy segmap entry
    movsb   %d1,%a1@
    addl    #NBSG,%a0           | increment
    addl    #NBSG,%a1
    subql   #1,%d0
    bgt     L_per_pmeg

| Kernel now double-mapped at 0 and KERNBASE3
| Force long jump to high address
    movl    #IC_CLEAR,%d0
    movc    %d0,%cacr           | Flush I-cache
    jmp     L_high_code:l       | long jump

L_high_code:
| Now running at high addresses
| No longer position-independent

    | Setup temporary stack
    lea     _ASM_LABEL(tmpstk),%sp
    movl    #0,%a6              | Zero frame pointer

    | Call bootstrap code
    jsr     _C_LABEL(_bootstrap)

    | _bootstrap never returns
```

### Stage 4: Bootstrap and Init

**Location:** `/sys/arch/sun3/sun3/machdep.c`

**Functions: `_bootstrap()` and `cpu_startup()`**

**Actions:**
1. Complete MMU initialization
2. Parse PROM information (monitor, model, memory)
3. Initialize sun3-specific devices:
   - Z8530 SCC (serial ports)
   - AMD Lance Ethernet (le)
   - NCR 5380 SCSI (si)
   - VME bus interface
4. Setup interrupt system (vector base, autovector)
5. Initialize PROM callout vectors
6. Mount root filesystem (disk or NFS)
7. Start init process

**PROM Interface:**
```c
// Sun-3 can call back to PROM for services
extern struct sunromvec *romVectorPtr;

// PROM services:
romVectorPtr->v_putchar(c);     // Console output
romVectorPtr->v_getchar();      // Console input
romVectorPtr->v_romvec_version; // PROM version
```

## Memory Map

### Sun-3 (68020) Memory Map

```
Physical/Virtual Address Space:

0x00000000 - 0x0FFFFFFF    Main RAM (4-64MB typical, max 64MB)
  0x00000000 - 0x00001FFF    PROM vectors and low memory
  0x00002000 - ...           Available RAM

0x0F000000 - 0x0FFFFFFF    I/O Space
  0x0F000000               Onboard I/O
    0x0F000000 - 0x0F0003FF  System enable/diagnostics
    0x0F000800 - 0x0F000FFF  Ethernet (Lance)
    0x0F002000 - 0x0F002FFF  SCSI (NCR 5380)
    0x0F004000 - 0x0F004FFF  Z8530 SCC (serial)
    0x0F006000 - 0x0F006FFF  EEPROM/IDPROM
    0x0F008000 - 0x0F008FFF  Intersil 7170 clock

0x0FE00000 - 0x0FEFFFFF    VMEbus A24 address space

0x0FF00000 - 0x0FFFFFFF    VMEbus A16 address space

0xFE000000 - 0xFEFFFFFF    Boot PROM

Kernel Virtual Addresses:
0xE0000000 (KERNBASE3)      Kernel base
0xE0000000 - ...            Kernel text/data
...                         Kernel heap
0xF0000000 - 0xFFFFFFFF     I/O mappings
```

### Sun-3x (68030) Memory Map

**Similar but with differences:**
```
0x00000000 - 0x1FFFFFFF    Main RAM (up to 128MB)

I/O Space locations differ slightly
Boot PROM at different address
Enhanced VME support
```

## Build and Installation

### Building Kernel

```bash
cd /usr/src/sys/arch/sun3/conf
config GENERIC       # or GENERIC3X for Sun-3x
cd ../compile/GENERIC
make depend && make
```

**Note:** Use GENERIC for 68020 Sun-3, GENERIC3X for 68030 Sun-3x

### Installation

**Install to disk:**
```bash
# Assuming disk is sd0
mount /dev/sd0a /mnt
cp netbsd /mnt/netbsd
umount /mnt
```

**Network Boot Setup:**
```bash
# On boot server
# /etc/ethers
8:0:20:xx:xx:xx sun3client

# /etc/bootparams
sun3client root=bootserver:/export/sun3/root \
           swap=bootserver:/export/sun3/swap

# /tftpboot/
cp netbsd /tftpboot/netbsd.sun3
```

**Boot from network:**
```
> b le()
```

## Debugging

### Serial Console

**Hardware:**
- ttya: Console port (DB-25)
- ttyb: Second serial port

**Configuration:** 9600 8N1 (typical)

**Enable in PROM:**
```
> setenv output-device ttya
> setenv input-device ttya
```

### PROM Debugger

**Enter debugger:**
```
Press Stop-A (L1-A)
```

**Commands:**
```
> c              Continue
> go             Go to address
> dump           Dump memory
> 0x1000!        Store to memory
> 0x1000?        Examine memory
```

### DDB (Kernel Debugger)

**Enable:**
```
options DDB
options DDB_HISTORY_SIZE=512
```

**Break into DDB:**
- Stop-A from console
- Programmed breakpoint
- Panic

**Commands:**
```
db> ps           Process list
db> bt           Backtrace
db> reboot       Reboot system
db> sync         Sync disks
```

## Common Issues

### "Cannot mount root"
- **Cause:** Wrong root device or NFS problem
- **Solution:**
  - Check bootparams on server
  - Verify network connectivity
  - Try boot -a (ask for root device)

### "SCSI timeout"
- **Cause:** SCSI termination or cabling
- **Solution:**
  - Check SCSI termination (both ends)
  - Verify SCSI IDs unique
  - Check cables

### "Machine type not recognized"
- **Cause:** Uncommon Sun-3 variant
- **Solution:** Check model, use correct kernel

### System hangs at boot
- **Cause:** Bad RAM, VME card conflict
- **Solution:**
  - Remove VME cards
  - Test RAM
  - Try network boot

### "MMU fault" during boot
- **Cause:** Bad memory or MMU hardware
- **Solution:**
  - Test memory with PROM diagnostics
  - Try different SIMM configuration

## Hardware Support

**Well Supported:**
- 68020 @ 16.67/20/25 MHz (Sun-3)
- 68030 @ 20/25/33 MHz (Sun-3x)
- 68881/68882 FPU
- RAM (4-64MB Sun-3, up to 128MB Sun-3x)
- Z8530 SCC serial ports
- AMD Lance Ethernet (le)
- NCR 5380 SCSI (si)
- VMEbus interface
- Intersil 7170 RTC
- CG2/CG3/CG4/CG6 framebuffers

**Partially Supported:**
- Some VME cards
- Some SBUS cards (Sun-3x)

**Not Supported:**
- Some obscure peripherals
- Proprietary Sun hardware

## References

**Source Files:**
- `/sys/arch/sun3/sun3/locore.s` - Kernel entry (68020)
- `/sys/arch/sun3/sun3x/locore.s` - Kernel entry (68030)
- `/sys/arch/sun3/sun3/machdep.c` - Machine init
- `/sys/arch/sun3/sun3/pmap.c` - MMU management

**Documentation:**
- Sun-3 Architecture Manual
- Sun-3 System Maintenance Manual
- Sun-3x Hardware Manual
- 68020/68030 User's Manuals
- Sun PROM User's Manual

**Hardware:**
- Sun-3/50, 3/60, 3/75, 3/110, 3/160, 3/180 service manuals
- Sun-3/80, 3/470 service manuals
- VMEbus Specification
- NCR 5380 SCSI manual
- AMD Lance Ethernet manual

## Historical Significance

Sun-3 workstations were extremely important in Unix history:
- Dominant Unix workstation in mid-1980s
- Used in universities, research labs, companies
- Ran SunOS (BSD-based Unix)
- Popular for software development
- X11 Window System development platform
- Still collectible and usable today with NetBSD
