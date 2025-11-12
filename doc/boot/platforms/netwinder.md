# NetBSD/netwinder Boot Process Documentation

## Platform Overview

NetBSD/netwinder supports the Rebel NetWinder series of network computers:
- NetWinder DM (Digital Media)
- NetWinder OfficeServer
- Other NetWinder variants

**Hardware Specifications:**
- Digital/Intel StrongARM SA-110 processor (200-275 MHz)
- DC21285 (Footbridge) core logic chipset
- 16MB - 128MB SDRAM
- Integrated CyberPro 2010 graphics (IGS)
- On-board 10/100 Ethernet (via DC21143 Tulip)
- IDE storage
- PCI expansion bus
- PC/AT compatible I/O

The NetWinder was designed as a low-power network server and workstation running Linux, but it also runs NetBSD well.

## Boot Method

### NeTTrom Firmware

NetBSD/netwinder boots using **NeTTrom**, a proprietary boot firmware developed for the NetWinder. NeTTrom provides:

- Hardware initialization
- Boot menu and command interface
- Network boot support (TFTP, NFS)
- Disk boot from IDE
- Serial and VGA console
- Boot parameter configuration

**Note:** There is **no standalone NetBSD bootloader** in `/sys/arch/netwinder/stand/`. The system relies entirely on NeTTrom firmware to load and start the NetBSD kernel.

### NeTTrom Capabilities

NeTTrom firmware provides:
- Hardware POST and initialization
- Boot configuration menu
- Network protocols (BOOTP, TFTP, NFS)
- IDE disk access
- ELF kernel loading
- Boot parameter passing

## Boot Process Stages

### Stage 1: NeTTrom Firmware Initialization

On power-on or reset, NeTTrom:

1. **Hardware Initialization**
   - Initialize SA-110 processor
   - Configure DC21285 Footbridge chipset
   - Initialize SDRAM controller
   - Detect and size memory
   - Configure PCI bus
   - Initialize ISA bridge

2. **Peripheral Setup**
   - Initialize CyberPro graphics (if VGA console)
   - Setup serial console (115200 baud default)
   - Initialize keyboard controller
   - Configure IDE controller
   - Setup Ethernet interface

3. **Display Banner**
   ```
   NeTTrom version X.X.XX
   Copyright (C) Rebel.com

   NetWinder DM-XXXX
   StrongARM SA-110 @ 275 MHz
   Memory: 64 MB
   ```

4. **Network Configuration** (optional)
   - Attempt BOOTP/DHCP if configured
   - Display IP address and server info
   - Wait for boot interrupt

### Stage 2: Boot Configuration Menu

**Interactive Menu:**

Press the designated key during POST (typically ESC or SPACE) to enter menu:

```
NeTTrom Boot Menu:

1. Boot from IDE disk
2. Boot from network (TFTP)
3. Boot from NFS root
4. Configure boot parameters
5. System configuration
6. Network configuration
7. Drop to command prompt
8. Reset system

Select option:
```

**Auto-boot Mode:**
- If no key pressed, proceeds with default boot
- Typical timeout: 5 seconds
- Uses stored boot configuration

### Stage 3: Kernel Loading

NeTTrom loads the NetBSD kernel:

1. **IDE Disk Boot** (most common)
   ```
   Loading from IDE disk...
   Reading partition table...
   Loading /netbsd...
   ```

   Process:
   - Read partition table
   - Locate NetBSD partition (type 0xA9)
   - Mount filesystem (FFS or other)
   - Load kernel file from root

2. **Network Boot (TFTP)**
   ```
   Requesting file from TFTP server...
   Server: 192.168.1.1
   File: netbsd
   Loading: #####################
   ```

   Process:
   - Use BOOTP/DHCP for IP configuration
   - Contact TFTP server
   - Download kernel to memory
   - Verify transfer

3. **NFS Root Boot**
   - Similar to TFTP but mounts NFS root
   - Downloads kernel from NFS share
   - Can continue with NFS as root filesystem

### Stage 4: Boot Parameters

**Boot Parameter Structure:**

NeTTrom passes boot information to kernel via structure in memory:

```c
struct nwbootinfo {
    union {
        struct {
            unsigned long bp_pagesize;
            unsigned long bp_nrpages;
            unsigned long bp_ramdisk_size;
            unsigned long bp_flags;
            unsigned long bp_rootdev;
        } u1_bp;
        char filler1[256];
    } bi_u1;
    union {
        char paths[8][128];
        struct magic {
            unsigned long magic;
            char filler2[1024 - sizeof(unsigned long)];
        } u2_d;
    } bi_u2;
    char bi_cmdline[1024];
};
```

**Information Provided:**
- Page size (typically 4096)
- Number of pages (total memory)
- Root device
- Command line arguments
- Boot paths

### Stage 5: MMU and Memory Setup

NeTTrom prepares system:

1. **Memory Layout**
   ```
   0x00000000 - 0x03FFFFFF : SDRAM (64MB typical)
   0x04000000 - 0x07FFFFFF : Extended SDRAM (if present)
   ```

2. **Kernel Load Address**
   - Typically loaded at physical 0x00008000
   - NeTTrom loads kernel low in memory

3. **MMU State**
   - NeTTrom may enable or disable MMU
   - Usually provides identity mapping if enabled
   - Kernel reconfigures MMU during boot

4. **Cache Configuration**
   - Caches may be enabled or disabled
   - Kernel performs cache initialization

### Stage 6: Kernel Entry

NeTTrom transfers control:

1. **Processor State**
   ```
   Mode:      SVC32 (Supervisor mode)
   MMU:       Typically disabled or 1:1 mapped
   Caches:    May be enabled or disabled
   Interrupts: Disabled
   ```

2. **Register State**
   ```
   r0:        Pointer to nwbootinfo structure
   r1:        Architecture ID (optional)
   r2:        Page table pointer (if MMU enabled)
   PC:        Kernel entry point (typically 0x00008000)
   ```

3. **Jump to Kernel**
   ```
   Branch to kernel entry point
   ```

### Stage 7: Kernel Initialization

**File: `sys/arch/netwinder/netwinder/netwinder_machdep.c:initarm()`**

The kernel initializes:

1. **Process Boot Information**
   ```c
   struct nwbootinfo *bootinfo = (struct nwbootinfo *)r0;

   // Extract:
   // - Memory size (pagesize * nrpages)
   // - Boot command line
   // - Root device
   ```

2. **Hardware Detection**
   - Identify DC21285 Footbridge
   - Detect memory configuration
   - Configure interrupt controller
   - Initialize PCI bus

3. **Memory Configuration**
   ```c
   // Create bootconfig structure
   bootconfig.dramblocks = 1;
   bootconfig.dram[0].address = 0x00000000;
   bootconfig.dram[0].pages = bootinfo->bi_nrpages;
   ```

4. **Create Bootstrap Page Tables**
   - Allocate L1 table (16KB)
   - Map kernel to virtual address
   - Map DC21285 registers
   - Map PCI I/O and memory space
   - Enable MMU

5. **Device Mappings**
   ```
   0xFD000000: DC21285 CSR space
   0xFE000000: Cache flush region
   0xFF000000: PCI I/O space
   ```

6. **Continue Boot**
   - Switch to virtual addressing
   - Initialize devices
   - Mount root filesystem
   - Start init

## ARM MMU Setup Requirements

### StrongARM SA-110

**Control Register (CP15 c1):**
```
Bit 0:  M - MMU enable
Bit 2:  C - Data cache enable
Bit 3:  W - Write buffer enable
Bit 7:  B - Big-endian
Bit 9:  R - ROM protection
Bit 11: Z - Branch prediction enable
Bit 12: I - Instruction cache enable
Bit 13: V - High vectors
```

**SA-110 Specifications:**
- 16KB instruction cache (32-way)
- 16KB data cache (32-way, write-back)
- 8-entry write buffer
- 64-entry TLB (fully associative)

**Cache Operations:**
```
Flush I&D cache:      MCR p15, 0, r0, c7, c7, 0
Clean D-cache entry:  MCR p15, 0, r0, c7, c10, 1
Drain write buffer:   MCR p15, 0, r0, c7, c10, 4
Invalidate TLB:       MCR p15, 0, r0, c8, c7, 0
```

### Page Table Setup

**L1 Table:**
- 16KB, 16KB aligned
- 4096 entries (1MB sections)
- Domain 0

**Section Descriptors:**
```
Bits 31-20: Section base address
Bits 11-10: Access Permission (AP)
Bits 8-5:   Domain
Bit 4:      Should Be One (section marker)
Bit 3:      C (Cacheable)
Bit 2:      B (Bufferable)
Bits 1-0:   10 (section descriptor)
```

## Memory Map

### Physical Memory Layout

```
0x00000000 - 0x00007FFF : NeTTrom vectors and data
0x00008000 - 0x003FFFFF : Kernel and available memory
0x00400000 - 0x03FFFFFF : Extended DRAM (varies)
0x04000000 - 0x07FFFFFF : Additional DRAM banks (if present)
0x42000000 - 0x420FFFFF : DC21285 CSR registers
0x79000000 - 0x790FFFFF : Outbound PCI memory
0x7C000000 - 0x7CFFFFFF : PCI memory window (16MB)
0x80000000 - 0x8FFFFFFF : Outbound PCI memory (256MB)
```

### Virtual Memory Layout

```
0xF0000000 - 0xF0FFFFFF : Kernel text/data (16MB)
0xF1000000 - 0xFCFFFFFF : Kernel VM space (192MB)

Device Mappings:
0xFD000000 - 0xFD0FFFFF : DC21285 ARM CSR space
0xFE000000 - 0xFE0FFFFF : Cache flush region
0xFF000000 - 0xFF0FFFFF : PCI I/O space
0xFF100000 - 0xFF1FFFFF : PCI IACK space
```

### DC21285 Footbridge Registers

**Control and Status Registers:**
```
Base: 0x42000000
  +0x00: SA Control
  +0x04: SA Base Address Mask
  +0x08: SA Base Address Offset
  +0x10: SDRAM Configuration
  +0x14: SDRAM Timing
  +0x80: ROM Control
```

## Build and Installation

### Building Kernel

```bash
cd /sys/arch/netwinder/conf
config GENERIC
cd ../compile/GENERIC
make depend && make
```

Output: `netbsd` (ELF kernel)

### Installation Methods

#### Method 1: IDE Disk Installation

**Most Common Method:**

1. **Partition Disk**
   ```bash
   # From NetBSD
   fdisk -u wd0
   disklabel -e wd0
   ```

2. **Create Filesystem**
   ```bash
   newfs /dev/rwd0a
   mount /dev/wd0a /mnt
   ```

3. **Install Kernel**
   ```bash
   cp netbsd /mnt/
   # Also install bootblocks if needed
   installboot /dev/rwd0a /usr/mdec/bootxx_ffs
   ```

4. **Configure NeTTrom**
   ```
   Boot device: IDE disk
   Boot partition: 0 (or appropriate)
   Kernel path: /netbsd
   ```

#### Method 2: Network Boot (TFTP)

**Setup TFTP Server:**

1. **On boot server:**
   ```bash
   # Copy kernel to TFTP directory
   cp netbsd /tftpboot/

   # Start TFTP server
   in.tftpd -l -s /tftpboot

   # Configure DHCP/BOOTP
   # Add NetWinder MAC address to dhcpd.conf
   ```

2. **Configure NeTTrom:**
   ```
   Network Boot Configuration:
   - Enable network boot
   - Set TFTP server IP
   - Set kernel filename
   - Save configuration
   ```

3. **Boot Process:**
   ```
   NeTTrom obtains IP via BOOTP
   Connects to TFTP server
   Downloads netbsd
   Boots kernel
   ```

#### Method 3: NFS Root

**For diskless operation:**

1. **Setup NFS Server:**
   ```bash
   # Export root filesystem
   echo "/export/netwinder *(rw,no_root_squash)" >> /etc/exports
   exportfs -a

   # Install NetBSD system
   cd /export/netwinder
   tar xzpf sets.tgz
   ```

2. **Configure NeTTrom:**
   ```
   Boot method: NFS
   NFS server: 192.168.1.1
   NFS path: /export/netwinder
   Kernel path: /netbsd
   ```

### NeTTrom Configuration

**Accessing Configuration:**

1. Enter boot menu during POST
2. Select "Configure boot parameters"
3. Modify settings
4. Save to flash

**Common Settings:**
```
Boot Timeout:      5 seconds
Boot Device:       IDE / Network
Console:           Serial / VGA
Serial Speed:      115200 baud
Network:           DHCP / Static IP
Default Kernel:    /netbsd
Boot Arguments:    -s (single user), -v (verbose)
```

## Debugging

### Serial Console

**Connection:**
- Serial port: 115200 baud, 8N1, no flow control
- Standard DB9 cable
- Connect to PC COM port

**Accessing NeTTrom Console:**
```bash
# On PC:
cu -l /dev/ttyS0 -s 115200
# or
screen /dev/ttyS0 115200
# or
minicom -D /dev/ttyS0
```

**NeTTrom Commands:**

```
NeTTrom> help
Available commands:
  boot [device] [args] - Boot from device
  setenv var value     - Set environment variable
  printenv [var]       - Display environment
  reset                - Reset system
  mem                  - Display memory info
  pci                  - Show PCI devices
```

### VGA Console

**Graphics Console:**
- CyberPro 2010 (IGS) integrated graphics
- Standard VGA monitor support
- PC keyboard required
- Console driver: `igsfb(4)`

**Switching Consoles:**

In NeTTrom configuration:
```
Console Device: Serial
    or
Console Device: VGA
```

### Common Issues

**Issue: "Cannot load kernel from IDE"**
- Cause: Disk not detected or wrong partition
- Solution: Verify IDE connection, check partition table
- Try: Boot from network to test

**Issue: "BOOTP/DHCP timeout"**
- Cause: Network configuration or cable issue
- Solution: Check network cable, verify DHCP server
- Test: Use static IP configuration

**Issue: "Kernel load failed"**
- Cause: Corrupted kernel or wrong format
- Solution: Verify ELF format, rebuild kernel
- Check: File size and permissions

**Issue: Kernel loads but hangs**
- Cause: MMU setup error or hardware detection failure
- Solution: Try different kernel, enable verbose boot
- Debug: Use serial console for kernel messages

**Issue: Display issues with VGA**
- Cause: Unsupported monitor or graphics mode
- Solution: Use serial console, configure igsfb driver
- Alternative: Force specific video mode

### Kernel Debugging

**Enable Debugging:**

In kernel config:
```
options     DEBUG
options     DDB
options     DIAGNOSTIC
options     VERBOSE_INIT_ARM
```

**Verbose Boot:**
```
# From NeTTrom, pass -v flag:
boot ide /netbsd -v
```

**Kernel Messages:**
```
NetBSD/netwinder booting...
initarm: memory configuration...
initarm: Configuring DC21285...
pci0 at footbridge0
...
```

### Boot Parameter Debugging

**Verify Boot Info:**
```c
printf("nwbootinfo:\n");
printf("  pagesize:  %lu\n", nwbootinfo.bi_pagesize);
printf("  nrpages:   %lu\n", nwbootinfo.bi_nrpages);
printf("  rootdev:   0x%lx\n", nwbootinfo.bi_rootdev);
printf("  cmdline:   %s\n", nwbootinfo.bi_cmdline);
```

Expected values:
- pagesize: 4096
- nrpages: depends on RAM (16384 for 64MB)
- rootdev: device number or 0
- cmdline: boot arguments string

## Technical Notes

### NeTTrom Firmware

NeTTrom is proprietary firmware:
- Developed by Rebel.com for NetWinder
- Provides PC BIOS-like functionality
- Stores configuration in flash memory
- Supports Linux and NetBSD boot protocols

### DC21285 Integration

**Footbridge Chipset:**
- StrongARM host interface
- PCI bus master capability
- Integrated PCI-ISA bridge
- SDRAM controller
- Interrupt controller
- Timers and DMA

**Key Features:**
- 64-bit SDRAM interface
- 32-bit PCI bus
- Address translation for PCI
- Integrated peripherals

### Graphics Support

**CyberPro 2010 (IGS):**
- Integrated on NetWinder motherboard
- PCI-attached graphics
- 2-4MB video RAM
- 2D acceleration
- Supported by `igsfb(4)` driver

**Console Options:**
- Serial: Always available, reliable
- VGA: Graphics console via igsfb
- Both can be used simultaneously

### Network Hardware

**Ethernet:**
- DEC 21143 Tulip controller
- On-board 10/100 Ethernet
- Uses `tlp(4)` driver
- PCI-attached

**Network Boot:**
- Supports BOOTP and DHCP
- TFTP for kernel loading
- NFS root filesystem option

### Disk Support

**IDE Controller:**
- Standard PC-style IDE
- Supports 2 IDE devices
- UDMA support
- Works with `wd(4)` driver

**SCSI** (some models):
- Adaptec or NCR SCSI controller
- PCI-attached
- Optional on some NetWinder models

## Device Support

**Well-Supported:**
- DC21285 Footbridge core
- StrongARM SA-110 CPU
- CyberPro graphics
- DEC Tulip Ethernet
- IDE disk
- Serial ports
- PC keyboard/mouse

**Partially Supported:**
- Audio (depends on model)
- PCMCIA (if present)
- USB (limited)

**Not Supported:**
- Some model-specific peripherals
- Proprietary extensions

## References

- `/sys/arch/netwinder/netwinder/netwinder_machdep.c` - Kernel initialization
- `/sys/arch/netwinder/include/netwinder_boot.h` - Boot structure definition
- `/sys/arch/arm/footbridge/` - DC21285 drivers
- Digital Semiconductor 21285 Core Logic Technical Reference
- Intel StrongARM SA-110 Microprocessor Technical Reference
- NetWinder Hardware Documentation (Rebel.com)
