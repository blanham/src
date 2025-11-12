# NetBSD/mvme68k Boot Documentation

## Platform Overview

NetBSD/mvme68k supports Motorola VMEbus Single Board Computers (SBCs) based on 68k processors. These industrial-grade boards were popular in embedded systems, telecommunications, and aerospace applications.

**Supported Boards:**
- MVME147: 68030 @ 33 MHz, onboard SCSI/Ethernet
- MVME162: 68040 @ 25 MHz, IP module architecture
- MVME167: 68040 @ 33 MHz, improved MVME162
- MVME172: 68060 @ 50 MHz, high performance
- MVME177: 68060 @ 66 MHz, fastest variant

**Common Features:**
- VMEbus interface (master/slave)
- Onboard SCSI (NCR 53C710)
- Onboard Ethernet (Intel 82596 or AMD 79C940)
- Multiple serial ports
- Battery-backed NVRAM
- Real-time clock
- Industrial temperature range

## Boot Method

Motorola VME boards boot from Bug monitor (Built-in User Generated software), a sophisticated ROM-based firmware that provides comprehensive system management and boot capabilities.

### Boot Chain

1. **Bug Monitor** - ROM firmware
2. **Bootloader** - `bootst`, `bootxx`, or network boot
3. **NetBSD Kernel** - `/netbsd`

### Bug Monitor

**Features:**
- Self-test diagnostics
- Memory test
- Disk boot
- Network boot (TFTP/BOOTP)
- Tape boot
- Debugger
- Environment variables
- System configuration

**Common Bug Commands:**
```
Bug> env               List environment
Bug> bo                Boot from default device
Bug> niot              Network I/O test
Bug> iot               I/O system test
Bug> bug               Return to Bug monitor
```

## Boot Methods

### 1. Disk Boot

**From SCSI Disk:**
```
Bug> bo                # Boot from default (configured via env)
Bug> bo 0,0            # Boot from SCSI ID 0, LUN 0
```

**Boot Sequence:**
1. Bug loads bootloader from disk
2. Bootloader loads kernel
3. Kernel executes

### 2. Network Boot

**Setup BOOTP/TFTP:**
```
Bug> nbo 0,0           # Network boot (Ethernet controller 0, unit 0)
```

**Server Requirements:**
- BOOTP/DHCP server
- TFTP server with kernel image
- NFS server for root filesystem (optional)

**Configure Bug environment:**
```
Bug> env
Network Boot File Server IP Address? 192.168.1.1
Boot File Name? netbsd.mvme68k
Bug> end
Bug> nbo
```

### 3. Tape Boot

**For installation:**
```
Bug> iot               # Display tape devices
Bug> bo st0            # Boot from tape SCSI ID 0
```

Tape contains:
- Miniroot filesystem
- Kernel
- Distribution sets

## 68030/68040/68060 MMU Differences

### 68030 (MVME147)

**Characteristics:**
- Integrated PMMU
- pmove instructions
- Three-level page tables

**Setup:**
```assembly
| 68030 MMU
pmove   %a0@,%crp       | Load root pointer
pmove   %a1@,%tc        | Load TC
pflusha                 | Flush TLB
```

### 68040 (MVME162, MVME167)

**Characteristics:**
- Different MMU architecture
- movc instructions replace pmove
- Four-level page tables
- Separate instruction/data TLBs

**Setup:**
```assembly
| 68040 MMU
.long   0x4e7b1807      | movc d1,srp
.word   0xf518          | pflusha
movl    #MMU40_TCR,%d0
.long   0x4e7b0003      | movc d0,tc
```

### 68060 (MVME172, MVME177)

**Similar to 68040 with enhancements:**
- Superscalar execution
- Branch prediction
- Enhanced caches
- PCR (Processor Control Register)

**Setup:**
```assembly
| 68060-specific
movl    #1,%d0
.long   0x4e7b0808      | movc d0,pcr (enable superscalar)
movl    #0x80f04445,%d0 | TTR for I/O
.long   0x4e7b0004      | movc d0,itt0
.long   0x4e7b0005      | movc d0,dtt0
.word   0xf4f8          | cpusha bc
.word   0xf518          | pflusha
```

## Boot Process Stages

### Stage 1: Bug Monitor

**Power-On:**
1. Bug ROM executes from 0xFF800000
2. Hardware initialization
3. Memory test (if enabled)
4. Check boot configuration
5. Auto-boot (if configured) or prompt

**Bug Environment Variables:**
```
Bug> env
    Network Enable? Y
    BOOTP Enable? Y
    Auto Boot Enable? Y
    Auto Boot at power-up? Y
    Auto Boot Controller LUN? 00
    Auto Boot Device? 00
```

**Manual Boot:**
```
Bug> bo 0,0            # SCSI boot
Bug> nbo 0,0           # Network boot
```

### Stage 2: Bootloader

**Location:** `/sys/arch/mvme68k/stand/`

**Bootloaders:**
- `bootst` - Tape bootloader
- `bootxx` - Disk bootloader
- `netboot` - Network bootloader (built-in Bug)

**Functions:**
- Minimal filesystem support
- Load kernel from disk/tape/network
- Pass parameters to kernel
- Interactive commands (minimal)

### Stage 3: Kernel Entry

**Location:** `/sys/arch/mvme68k/mvme68k/locore.s`

**Entry Point:** `start`

**Bug Parameters (in bugargs structure):**
```c
struct bugargs {
    u_int   cputyp;     // CPU type (147/162/167/etc.)
    u_int   ctrlun;     // Boot controller/LUN
    u_int   devlun;     // Boot device/LUN
    u_int   brdtyp;     // Board type
    u_long  arg_start;  // Argument string start
    u_long  arg_end;    // Argument string end
};
```

**Initial Code:**
```assembly
ASENTRY_NOPROFILE(start)
    movw    #PSL_HIGHIPL,%sr    | Disable interrupts

    | Save Bug arguments
    lea     _C_LABEL(bugargs),%a0
    movl    %d0,%a0@(0)         | cputyp
    movl    %d1,%a0@(4)         | ctrlun
    movl    %d2,%a0@(8)         | devlun

    | Detect CPU type
    | Setup MMU for detected CPU
    | Initialize caches
    | Jump to C code
    jbsr    _C_LABEL(_bootstrap)
```

### Stage 4: Machine Init

**Location:** `/sys/arch/mvme68k/mvme68k/machdep.c`

**Actions:**
1. Determine board type from Bug
2. Initialize onboard devices:
   - NCR 53C710 SCSI
   - Intel 82596/AMD 79C940 Ethernet
   - Z8530 SCC serial ports
   - MK48T08 NVRAM/RTC
3. Setup VMEbus interface
4. Configure interrupts (MC68230 PIT, PCCchip2, etc.)
5. Mount root filesystem
6. Start init

## Memory Map

### MVME147 (68030)

```
Physical Address Space:

0x00000000 - 0x00FFFFFF    RAM (4-16MB)
0xFF800000 - 0xFFFFFFFF    I/O and ROM
  0xFF800000 - 0xFF9FFFFF    Bug ROM (2MB)
  0xFFC00000 - 0xFFFFFFFF    I/O devices
    0xFFC00000               VMEChip (VLSI)
    0xFFC10000               PCCchip (peripheral controller)
    0xFFCC0000               SCC (Z8530 serial)
    0xFFCC8000               NVRAM/RTC (MK48T02)
```

### MVME162/167 (68040)

```
Physical Address Space:

0x00000000 - 0x07FFFFFF    RAM (up to 128MB)
0xFF000000 - 0xFFFFFFFF    I/O and ROM
  0xFF800000 - 0xFF8FFFFF    Bug ROM (1MB)
  0xFFF00000 - 0xFFFFFFFF    I/O space
    0xFFF00000               MCchip (Memory Controller)
    0xFFF40000               PCCchip2
    0xFFF45000               SCC (Z8530)
    0xFFF48000               NCR 53C710 SCSI
    0xFFF60000               Ethernet (82596 or 79C940)
```

### MVME172/177 (68060)

**Similar to MVME167 with:**
- More RAM capacity (up to 256MB)
- Faster CPU and buses
- Enhanced peripheral controllers

### VMEbus Address Space

```
A16 Space: 0xFFFF0000 - 0xFFFFFFFF (64KB)
A24 Space: 0xFF000000 - 0xFFFFFFFF (16MB)
A32 Space: 0x00000000 - 0xFFFFFFFF (4GB)
```

## Build and Installation

### Building

```bash
# Bootloaders
cd /usr/src/sys/arch/mvme68k/stand
make depend && make

# Kernel
cd /usr/src/sys/arch/mvme68k/conf
config GENERIC
cd ../compile/GENERIC
make depend && make
```

### Installation to Disk

**Install bootblock:**
```bash
cd /usr/mdec
installboot /dev/rsd0a bootxx bootst
```

**Install kernel:**
```bash
cp netbsd /targetroot/netbsd
```

**Configure Bug for auto-boot:**
```
Bug> env
Auto Boot Enable? Y
Auto Boot at power-up? Y
Auto Boot Controller LUN? 00
Auto Boot Device? 00
Bug> end
```

### Network Installation

**TFTP Server Setup:**
```bash
# Copy kernel to tftp directory
cp netbsd /tftpboot/netbsd.mvme68k

# Configure dhcpd
host mvme {
    hardware ethernet 08:00:3e:xx:xx:xx;
    fixed-address 192.168.1.100;
    filename "netbsd.mvme68k";
}
```

**Boot from network:**
```
Bug> nbo 0,0
```

## Debugging

### Bug Debugger

**Enter debugger:**
```
Bug> bug
or press ABORT switch
```

**Commands:**
- `md <addr>` - Memory display
- `mm <addr>` - Memory modify
- `rd` - Register display
- `rm` - Register modify
- `go <addr>` - Execute at address
- `br <addr>` - Set breakpoint
- `trace` - Single step

### Serial Console

**Ports:**
- Port 0: Console (9600 8N1 default)
- Port 1-3: Additional ports

**Configure Bug:**
```
Bug> env
Console Port? 0
Bug> end
```

### DDB

**Enable:**
```
options DDB
```

**Break:**
- Press ABORT button
- Serial BREAK
- Programmed breakpoint

## Hardware Support

**Working:**
- All supported boards (147/162/167/172/177)
- NCR 53C710 SCSI
- Intel 82596 Ethernet (ie driver)
- AMD 79C940 Ethernet (le driver)
- Z8530 SCC serial ports
- MK48T08 NVRAM/RTC
- VMEbus interface (basic)

**Limited:**
- VMEbus slave mode
- Some VME cards
- Multiprocessor (not supported)

## References

**Source:**
- `/sys/arch/mvme68k/` - Source tree
- `/sys/arch/mvme68k/stand/` - Bootloaders

**Documentation:**
- MVME147 Single Board Computer User's Manual (Motorola)
- MVME162 Embedded Controller User's Manual
- MVME167 Single Board Computer User's Manual
- 68030/68040/68060 User's Manuals
- Bug Monitor User's Manual

**Hardware:**
- NCR 53C710 SCSI Processor Manual
- Intel 82596 LAN Coprocessor Manual
- Z8530 SCC Technical Manual
- VMEbus Specification Rev. C
