# NetBSD/iyonix Boot Process Documentation

## Platform Overview

NetBSD/iyonix supports the Castle Technology Iyonix, a high-performance ARM workstation featuring:
- Intel XScale 80321 processor (I/O Processor, 600 MHz)
- Intel i80321 chipset with integrated PCI-X support
- Up to 2GB DDR SDRAM
- AGP graphics slot
- PCI-X expansion slots
- SATA and IDE storage
- Gigabit Ethernet
- USB 2.0
- Standard PC peripherals (keyboard, mouse)

The Iyonix was designed as a successor to RISC OS machines, providing a modern ARM workstation platform with standard PC connectivity and expansion.

## Boot Method

### RedBoot Firmware

NetBSD/iyonix boots using **RedBoot**, an open-source embedded boot firmware based on eCos. RedBoot provides:

- Hardware initialization
- Boot menu and console
- Network boot support (TFTP, NFS)
- Flash memory management
- ELF kernel loading
- Debug and diagnostic features

**Note:** There is **no standalone NetBSD bootloader** in `/sys/arch/iyonix/stand/`. The system relies entirely on RedBoot firmware to load and start the NetBSD kernel.

### RedBoot Capabilities

RedBoot provides a command-line interface with:
- Disk boot from IDE/SATA
- Network boot (BOOTP/DHCP, TFTP)
- Flash management
- Memory examination and modification
- Direct kernel loading
- Boot script execution

## Boot Process Stages

### Stage 1: RedBoot Firmware Initialization

On power-on or reset, RedBoot:

1. **Hardware Initialization**
   - Initialize XScale 80321 processor
   - Configure memory controller
   - Detect and size SDRAM
   - Initialize PCI-X bus
   - Configure i80321 ATU (Address Translation Unit)
   - Setup interrupt controller

2. **Peripheral Initialization**
   - Initialize serial console (115200 baud default)
   - Setup Ethernet controller
   - Initialize IDE/SATA controllers
   - Configure USB controller
   - Initialize AGP bridge

3. **Display Banner**
   ```
   RedBoot(tm) bootstrap and debug environment [ROM]
   Platform: Iyonix (XScale IOP80321)
   Copyright (C) 2000, 2001, 2002, Red Hat, Inc.

   RAM: 0x00000000-0x10000000, [...]
   FLASH: 0x00000000 - 0x00800000, 256 blocks of 0x00004000 bytes each.
   ```

4. **Network Configuration** (if enabled)
   - Obtain IP address via BOOTP/DHCP
   - Display network configuration
   - Wait for boot interrupt

### Stage 2: Boot Menu and Configuration

**Interactive Boot Menu:**

```
RedBoot> boot

Or press Ctrl-C for command prompt:

RedBoot> help
Manage aliases kept in FLASH memory
      alias [-l|-d] [<name> [<value>]]
Display/switch console channel
      channel [-1|-2|<channel number>]
Execute a command
      exec [-w <timeout>] [-c] <addr> [<length>]
Manage FLASH images
      fis {cmds}
Load a file
      load [-r] [-v] [-h <host>] [-m <varies>] [-c <channel>] filename [addr]
Run image from memory
      go [-w <timeout>] [-c] [addr]
Set/Query the system console baud rate
      baudrate [-b <rate>]
Display RedBoot version information
      version
Display help for commands
      help [<topic>]
```

**Common Boot Commands:**
```bash
# Boot from flash
RedBoot> fis load netbsd
RedBoot> go

# Boot from IDE disk
RedBoot> load -r wd0a:netbsd
RedBoot> go

# Boot from network
RedBoot> load -r -h 192.168.1.1 netbsd
RedBoot> go

# Boot with arguments
RedBoot> go -w 5 0x00200000 -s -v
```

### Stage 3: Kernel Loading

RedBoot loads the NetBSD kernel:

1. **Locate Kernel**
   - From Flash Image System (FIS)
   - From disk partition
   - From network via TFTP
   - Direct memory load

2. **Parse ELF Format**
   ```c
   // RedBoot parses ELF header
   // Validates magic number
   // Reads program headers
   // Determines load addresses
   ```

3. **Load Kernel Segments**
   - Loads to physical memory (typically 0x00200000)
   - Each program segment loaded to specified address
   - Relocates if necessary
   - Validates checksums

4. **Symbol Table** (optional)
   - Load kernel symbols for debugging
   - Used by DDB (kernel debugger)

### Stage 4: MMU and Memory Setup

RedBoot prepares memory:

1. **Physical Memory Layout**
   ```
   0x00000000 - 0x000FFFFF : RedBoot (1MB)
   0x00100000 - 0x001FFFFF : Reserved (1MB)
   0x00200000 - 0xXXXXXXXX : Kernel and available memory
   ```

2. **MMU Configuration**
   - RedBoot typically disables MMU before kernel entry
   - Or provides 1:1 mapping
   - Kernel sets up its own page tables

3. **Cache State**
   - I-cache and D-cache may be enabled or disabled
   - Write buffer drained
   - Kernel handles cache initialization

### Stage 5: Kernel Entry

RedBoot transfers control to kernel:

1. **Processor State**
   ```
   Mode:      SVC32 (Supervisor mode)
   MMU:       Disabled or 1:1 mapped
   Caches:    Enabled or disabled (kernel handles)
   Interrupts: Disabled
   ```

2. **Register State**
   ```
   r0:        0 (or bootinfo pointer if available)
   r1:        Machine type ID (or 0)
   r2:        Physical address of parameter structure (or 0)
   PC:        Kernel entry point
   ```

   **Note:** RedBoot doesn't use a comprehensive bootinfo structure like some other platforms. The kernel must detect hardware independently.

3. **Jump to Kernel**
   ```
   go [address]
   // Branches to kernel entry point
   ```

### Stage 6: Kernel Initialization

**File: `sys/arch/iyonix/iyonix/iyonix_machdep.c:initarm()`**

The kernel initializes:

1. **Hardware Detection**
   - Probe for i80321 chipset
   - Detect memory size and layout
   - Identify installed devices
   - Configure interrupt controller

2. **Memory Configuration**
   ```c
   // Typically:
   // Physical DRAM: 0x00000000 - 0x...... (up to 2GB)
   // Kernel loaded at: 0x00200000
   ```

3. **Create Bootstrap Page Tables**
   - L1 page table (16KB)
   - Map kernel to virtual address (0xC0000000+)
   - Map devices (i80321 registers, PCI space)
   - Enable MMU

4. **Device Mapping**
   ```c
   // Map i80321 registers
   0xFE800000: Memory Controller
   0xFFE00000: ATU (Address Translation Unit)
   0xFFE01000: Interrupt Controller
   0xFFE02000: Timers
   ```

5. **Continue Boot**
   - Switch to virtual memory
   - Initialize devices
   - Mount root filesystem
   - Start init

## ARM MMU Setup Requirements

### Intel XScale 80321

**Control Register (CP15 c1):**
```
Bit 0:  M - MMU enable
Bit 2:  C - Data cache enable
Bit 3:  W - Write buffer enable
Bit 7:  B - Big-endian
Bit 9:  R - ROM protection
Bit 11: Z - Branch prediction enable
Bit 12: I - Instruction cache enable
Bit 13: V - High vectors (0xFFFF0000)
Bit 14: RR - Round-robin cache replacement
Bit 15: L4 - ARMv5 mode (vs ARMv4)
```

**XScale 80321 Features:**
- ARMv5TE architecture
- 32KB instruction cache
- 32KB data cache (write-back)
- 2KB mini data cache
- 128-entry DTLB
- 32-entry ITLB

**Cache Operations:**
```
Clean entire D-cache:
    MCR p15, 0, rd, c7, c10, 0

Invalidate I-cache:
    MCR p15, 0, rd, c7, c5, 0

Drain write buffer:
    MCR p15, 0, rd, c7, c10, 4

Flush prefetch buffer:
    MCR p15, 0, rd, c7, c5, 4

Invalidate TLB:
    MCR p15, 0, rd, c8, c7, 0
```

### Page Table Setup

**L1 Table Requirements:**
- 16KB size
- 16KB alignment
- 4096 entries (1MB sections)

**L2 Tables:**
- 1KB per table
- 1KB alignment
- 256 entries (4KB pages)

**Typical Mappings:**
```
Physical         Virtual          Size    Type
0x00000000    -> 0xC0000000      256MB   DRAM (cached, buffered)
0xFE800000    -> 0xFE800000      1MB     i80321 MCU (uncached)
0xFFE00000    -> 0xFFE00000      1MB     i80321 ATU (uncached)
0xF0000000    -> 0xF0000000      256MB   PCI memory (uncached)
```

## Memory Map

### Physical Memory Layout

```
0x00000000 - 0x000FFFFF : Boot ROM / RedBoot (1MB)
0x00100000 - 0x001FFFFF : Reserved (1MB)
0x00200000 - 0x7FFFFFFF : DRAM (kernel and user space)
0xFE800000 - 0xFE8FFFFF : i80321 Memory Controller
0xFFE00000 - 0xFFE00FFF : i80321 ATU (Address Translation Unit)
0xFFE01000 - 0xFFE01FFF : i80321 Interrupt Controller
0xFFE02000 - 0xFFE02FFF : i80321 Timers and DMA
0xF0000000 - 0xFFFFFFFF : PCI-X memory space
```

### Virtual Memory Layout

```
0xC0000000 - 0xCFFFFFFF : Kernel text/data/BSS
0xD0000000 - 0xDFFFFFFF : Kernel VM space
0xE0000000 - 0xEFFFFFFF : Reserved
0xF0000000 - 0xF0FFFFFF : PCI I/O and memory space
0xFE800000 - 0xFE8FFFFF : i80321 MCU registers
0xFFE00000 - 0xFFEFFFFF : i80321 peripheral registers
0xFFF00000 - 0xFFFFFFFF : High vectors and misc
```

### i80321 Register Spaces

**Memory Controller Unit (MCU):**
```
Base: 0xFE800000
- SDRAM Configuration Registers
- ECC Control
- Scrub Control
```

**Address Translation Unit (ATU):**
```
Base: 0xFFE00000
- PCI-X configuration
- Inbound/Outbound translation
- PCI-X to SDRAM mapping
```

**Interrupt Controller:**
```
Base: 0xFFE01000
- Interrupt Status
- Interrupt Enable
- FIQ Control
```

**Timers:**
```
Base: 0xFFE02000
- Timer 0/1 registers
- Watchdog timer
```

## Build and Installation

### Building Kernel

```bash
cd /sys/arch/iyonix/conf
config GENERIC
cd ../compile/GENERIC
make depend && make
```

Output: `netbsd` (ELF kernel)

### Installation Methods

#### Method 1: Flash Installation

**Copy kernel to Flash Image System:**

```bash
# From RedBoot prompt:
RedBoot> load -r -h 192.168.1.10 -m tftp netbsd
Loading netbsd...
Raw file loaded 0x00200000-0x00543210, assumed entry at 0x00200000

# Write to flash
RedBoot> fis create -b 0x00200000 -l 0x00343210 netbsd
... Erase from 0x... to 0x...
... Program from 0x... to 0x...
... Erase from 0x... to 0x...

# Set boot script
RedBoot> fis load netbsd
RedBoot> fconfig boot_script true
RedBoot> fconfig
Boot script:
Enter script, terminate with empty line
>> fis load netbsd
>> go
>>
```

#### Method 2: Disk Installation

**Install to IDE/SATA disk:**

1. **Partition Disk**
   ```bash
   # From NetBSD
   fdisk -u wd0
   disklabel -e wd0
   ```

2. **Install Kernel**
   ```bash
   # Create FFS filesystem
   newfs /dev/rwd0a
   mount /dev/wd0a /mnt
   cp netbsd /mnt/
   umount /mnt
   ```

3. **Boot from Disk**
   ```bash
   # From RedBoot
   RedBoot> load -r -b 0x00200000 wd0a:netbsd
   RedBoot> go
   ```

#### Method 3: Network Boot

**Setup TFTP server:**

1. **On boot server:**
   ```bash
   # Copy kernel to TFTP directory
   cp netbsd /tftpboot/

   # Start TFTP server
   in.tftpd -l -s /tftpboot
   ```

2. **Configure RedBoot:**
   ```bash
   RedBoot> ip_address -h 192.168.1.10 -l 192.168.1.100
   RedBoot> load -r -h 192.168.1.10 -m tftp netbsd
   RedBoot> go
   ```

### RedBoot Configuration

**View configuration:**
```bash
RedBoot> fconfig -l
boot_script: true
boot_script_data:
    fis load netbsd
    go
console_baud_rate: 115200
gdb_port: 9000
net_debug: false
```

**Modify settings:**
```bash
RedBoot> fconfig boot_script_timeout 10
RedBoot> fconfig console_baud_rate 38400
```

## Debugging

### Serial Console

**Connection:**
- Serial port: 115200 baud, 8N1, no flow control
- Standard DB9 null modem cable
- Connect to PC COM port

**Accessing RedBoot:**
```bash
# On PC:
cu -l /dev/ttyS0 -s 115200
# or
screen /dev/ttyS0 115200
# or
minicom -D /dev/ttyS0
```

**RedBoot Commands:**

```bash
# Memory examination
RedBoot> dump -b 0x00200000 -l 256

# Memory modification
RedBoot> mfill -b 0x00300000 -l 0x1000 -p 0x00

# Register display
RedBoot> regs

# Help
RedBoot> help
```

### Common Issues

**Issue: "Can't load 'netbsd'"**
- Cause: File not found or path incorrect
- Solution: Verify disk partition, check FIS directory
- Command: `fis list` to see available images

**Issue: Kernel loads but doesn't start**
- Cause: Wrong load address or corrupted kernel
- Solution: Verify ELF format, check load address
- Try: `load -v` for verbose output

**Issue: "Uncompression error"**
- Cause: Corrupted kernel or wrong format
- Solution: Rebuild kernel, verify transfer

**Issue: Network boot fails**
- Cause: Network configuration or TFTP server issue
- Solution: Check IP configuration, test TFTP manually
- Debug: `ip_address` to verify network settings

**Issue: System hangs after "go"**
- Cause: MMU setup error, hardware conflict
- Solution: Check kernel configuration, verify memory detection
- Try: Boot with verbose flags

### Kernel Debugging

**Enable Debugging:**

1. **Build debug kernel:**
   ```bash
   # In kernel config:
   options     DEBUG
   options     DDB
   options     DIAGNOSTIC
   makeoptions DEBUG="-g"
   ```

2. **Remote Debugging via GDB:**
   ```bash
   # In RedBoot:
   RedBoot> load -r -h 192.168.1.10 netbsd
   RedBoot> gdb
   GDB: waiting for connection on port 9000

   # On development machine:
   arm-netbsd-gdb netbsd
   (gdb) target remote 192.168.1.100:9000
   (gdb) continue
   ```

### Boot Verbosity

**Verbose Boot:**
```bash
# From RedBoot:
RedBoot> go 0x00200000 -v
```

**Kernel Messages:**
```
Copyright (c) 1996, 1997, 1998, 1999, 2000, 2001, 2002, 2003, 2004, 2005,
    2006, 2007, 2008, 2009, 2010, 2011, 2012, 2013, 2014, 2015, 2016, 2017,
    2018, 2019, 2020, 2021, 2022, 2023, 2024, 2025
    The NetBSD Foundation, Inc.  All rights reserved.
Copyright (c) 1982, 1986, 1989, 1991, 1993
    The Regents of the University of California.  All rights reserved.

NetBSD 10.0 (GENERIC) #0: ...
total memory = 512 MB
avail memory = 495 MB
```

## Technical Notes

### RedBoot Architecture

RedBoot is based on eCos RTOS:
- Modular command structure
- Flash Image System (FIS) for storage
- Network stack (BOOTP, DHCP, TFTP, NFS)
- Configurable via `fconfig`
- Scriptable boot sequences

### Flash Image System (FIS)

**FIS Directory Structure:**
```
RedBoot> fis list
Name              FLASH addr  Mem addr    Length    Entry point
RedBoot           0x00000000  0x00000000  0x00040000  0x00000000
RedBoot config    0x007E0000  0x007E0000  0x00010000  0x00000000
FIS directory     0x007F0000  0x007F0000  0x00010000  0x00000000
netbsd            0x00100000  0x00200000  0x00400000  0x00200000
```

### i80321 Integration

**Key Features:**
- 600 MHz XScale core
- 64-bit PCI-X bus at 133 MHz
- Integrated Ethernet (GigE)
- Memory controller supports DDR SDRAM
- Address Translation Unit for PCI-host bridging

**ATU Windows:**
- Inbound: PCI to SDRAM
- Outbound: SDRAM to PCI
- Configurable address translation
- Memory-mapped and I/O space support

### Console Options

**Primary Console:**
- Serial (default): 115200 baud
- VGA (if installed): via PCI graphics card
- Framebuffer console via `genfb(4)`

**Switching Consoles:**
```bash
# In kernel config:
options CONSPEED=115200
options CONSDEVNAME="\"com\"",CONADDR=0xfe800000
# or
options CONSDEVNAME="\"genfb\""
```

### PCI-X Support

Iyonix supports:
- PCI-X slots for high-speed peripherals
- AGP slot for graphics cards
- Standard PCI compatibility mode
- DMA and interrupt routing via i80321

### Networking

**Onboard Ethernet:**
- Intel i82544 GigE controller
- Integrated into i80321
- Uses `wm(4)` driver

**Network Boot:**
- BOOTP/DHCP for IP configuration
- TFTP for kernel loading
- NFS for root filesystem (optional)

## References

- `/sys/arch/iyonix/iyonix/iyonix_machdep.c` - Kernel initialization
- `/sys/arch/arm/xscale/i80321*` - i80321 chipset support
- `/sys/arch/iyonix/conf/GENERIC` - Kernel configuration
- Intel 80321 I/O Processor Developer's Manual
- RedBoot User's Guide
- eCos Documentation
- Castle Technology Iyonix Hardware Manual
