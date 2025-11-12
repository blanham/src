# NetBSD/sun2 Boot Documentation

## Platform Overview

NetBSD/sun2 supports Sun Microsystems Sun-2 workstations, the company's second generation of workstations based on the Motorola 68010 processor.

**Supported Models:**
- Sun 2/120 - Desktop workstation
- Sun 2/170 - Deskside workstation
- Sun 2/50 - Diskless workstation
- Sun 2/160 - Server

**CPU:** MC68010 @ 10 MHz

**Key Limitation:** 68010 has no MMU, requiring special memory management through Sun-2 MMU hardware.

## Boot Method

Sun-2 systems boot from OpenBoot PROM (older version) that provides comprehensive boot and diagnostic capabilities.

### Boot Chain

1. **PROM Monitor** - Sun boot PROM
2. **Bootloader** - Loaded from disk/tape/network
3. **NetBSD Kernel** - `/netbsd`

### PROM Monitor

**Access:** Press Stop-A or L1-A during boot

**Commands:**
```
> b              Boot from default device
> b sd(0,0,0)    Boot from SCSI disk 0
> b le()         Boot from Ethernet
> b st()         Boot from tape
```

**PROM Variables:**
```
> k              Print environment
> bootdev=sd(0,0,0)
> bootfile=netbsd
```

## 68010 Memory Management

### Sun-2 MMU

The Sun-2 uses custom MMU hardware (not 68010 internal):

**Characteristics:**
- Segment-based memory management
- 2KB segments
- Context registers for process isolation
- Page Map Entries (PME)
- Segment Map Entries (SME)

**Memory Management:**
```
Context Register → Segment Map → Page Map → Physical Address
```

**Setup in locore.s:**
```assembly
| Sun-2 MMU initialization
moveq   #FC_CONTROL,%d0
movc    %d0,%sfc
movc    %d0,%dfc

| Set context zero
moveq   #0,%d0
movsb   %d0,CONTEXT_REG
movsb   %d0,SCONTEXT_REG

| Setup segment mappings
| (Sun-2 specific segment table setup)
```

**Differences from Other m68k:**
- No standard PMU
- Segment-based not page-based
- Sun-specific hardware
- More limited virtual addressing

## Boot Process Stages

### Stage 1: PROM Monitor

**Functions:**
- Hardware initialization
- Memory test
- Device probing
- Boot device selection
- Network configuration (for netboot)

**Boot Device Selection:**
```
> b              # Boot default
> b sd(0,0,0)    # SCSI disk
> b le()         # Ethernet
> b st()         # Tape
```

### Stage 2: Bootloader

**NetBSD doesn't have a sophisticated bootloader for sun2** - The PROM directly loads the kernel.

**Boot Process:**
1. PROM loads kernel from boot device
2. Kernel initializes itself
3. No intermediate bootloader stage

### Stage 3: Kernel Entry

**Location:** `/sys/arch/sun2/sun2/locore.s`

**Entry Point:** `start`

**Initial Code:**
```assembly
    .text
GLOBAL(kernel_text)

ASGLOBAL(tmpstk)
ASGLOBAL(start)

| Sun-2 boot entry
    movw    #PSL_HIGHIPL,%sr    | no interrupts
    moveq   #FC_CONTROL,%d0     | make movs access "control"
    movc    %d0,%sfc            | space where sun2 designers
    movc    %d0,%dfc            | put all the "useful" stuff

| Set context zero
    moveq   #0,%d0
    movsb   %d0,CONTEXT_REG
    movsb   %d0,SCONTEXT_REG

| Jump to high code
    jra     L_high_code
```

**g0/g4 Entry Points:**
```assembly
| These entry points are in low memory
| for "g0" and "g4" PROM commands
ENTRY(g0_entry)
    jra     _C_LABEL(g0_handler)
ENTRY(g4_entry)
    jra     _C_LABEL(g4_handler)
```

### Stage 4: Bootstrap

**Location:** `/sys/arch/sun2/sun2/machdep.c`

**Actions:**
1. Complete MMU setup
2. Initialize Sun-2 specific hardware
3. Setup interrupt system
4. Configure devices (SCSI, Ethernet, serial)
5. Mount root filesystem

## Memory Map

```
Physical Address Space:

0x00000000 - 0x00FFFFFF    Main RAM (typically 2-8MB)
  0x00000000 - 0x00001FFF    PROM vectors (mapped from ROM)
  0x00002000 - 0x000FFFFF    Kernel

0x00700000 - 0x007FFFFF    Multibus memory space

0x00E00000 - 0x00FFFFFF    I/O Space
  - VME bus devices
  - Multibus devices

0x00FE0000 - 0x00FEFFFF    Onboard devices
  0x00FE0000               Z8530 SCC (serial)
  0x00FE2000               Clock/Counter
  0x00FE4000               EEPROM/IDPROM
  0x00FE6000               Ethernet (Intel)
  0x00FEE000               System enable register

0x00FF0000 - 0x00FFFFFF    Boot PROM (64KB)
```

**Virtual Address Layout:**
```
0x00000000 - 0x000007FF    Unmapped (NULL trap)
0x00000800 - ...           Kernel
...                        Kernel heap
0x0E000000 - 0x0FFFFFFF    I/O mappings
```

## Build and Installation

### Building

```bash
# Kernel
cd /usr/src/sys/arch/sun2/conf
config GENERIC
cd ../compile/GENERIC
make depend && make
```

**Note:** Sun-2 uses direct kernel boot from PROM, no separate bootloader to build.

### Installation

**Install kernel:**
```bash
# Copy kernel to bootable media
cp netbsd /export/sun2/root/netbsd
```

**For diskless (NFS root):**
```bash
# Setup NFS exports on server
# Configure RARP/BOOTPARAMS on server
# Configure TFTP with kernel image
```

### Network Boot Setup

**Server Configuration:**
```bash
# /etc/ethers (RARP)
8:0:20:xx:xx:xx sun2client

# /etc/bootparams (BOOTPARAMS)
sun2client root=server:/export/sun2/root

# /tftpboot/
# Copy kernel as boot.sun2
```

**Boot from network:**
```
> b le()
```

## Debugging

### Serial Console

**Port:** ttya (Z8530 SCC)

**Configuration:** 9600 8N1

**Enable in PROM:**
```
> output-device=ttya
> input-device=ttya
```

### PROM Debugger

**Enter:**
```
Press Stop-A (or L1-A)
```

**Commands:**
```
> g0             Call g0 handler
> g4             Call g4 handler
> c              Continue
> e [addr]       Examine memory
> d [addr]       Deposit to memory
```

### DDB

**Enable:**
```
options DDB
```

**Note:** Limited on sun2 due to 68010 constraints.

## Hardware Support

**Working:**
- 68010 CPU @ 10 MHz
- Sun-2 MMU
- RAM (2-8MB typical)
- Z8530 SCC serial ports
- Intel Ethernet (ie driver)
- NCR 5380 SCSI (si driver)
- VME bus (basic)
- Multibus (basic)

**Limitations:**
- No FPU (68010 doesn't support 68881/2)
- Limited memory (68010 24-bit addressing)
- No virtual memory (segment-based MMU)
- Slower than 68020+ systems

**Not Supported:**
- Some VME cards
- Some Multibus cards
- Obscure peripherals

## Common Issues

**"Machine type not supported"**
- **Cause:** Rare Sun-2 variant
- **Solution:** Check model number

**"MMU fault"**
- **Cause:** Bad memory or MMU hardware
- **Solution:** Test RAM, check MMU chips

**Boot hangs**
- **Cause:** SCSI issues, bad PROM, hardware fault
- **Solution:**
  - Check SCSI termination
  - Test with network boot
  - Verify PROM version

## References

**Source Files:**
- `/sys/arch/sun2/sun2/locore.s` - Kernel entry
- `/sys/arch/sun2/sun2/machdep.c` - Machine init
- `/sys/arch/sun2/sun2/pmap.c` - Sun-2 MMU management

**Documentation:**
- Sun-2 Architecture Manual
- Sun-2 System Maintenance Manual
- 68010 User's Manual (Motorola)
- Sun PROM Monitor Manual

**Hardware:**
- Sun-2/120, 2/170 service manuals
- VMEbus Specification
- Multibus I Specification
- Z8530 SCC Technical Manual

## Historical Note

Sun-2 was Sun's second generation workstation (after Sun-1):
- Introduced around 1983
- Used in universities and research
- Predecessor to popular Sun-3
- Relatively rare today
- Historical significance in Unix workstation evolution
