# NetBSD/landisk Boot Documentation

## Platform Overview

NetBSD/landisk provides support for I-O DATA's line of SH4-based Network Attached Storage (NAS) appliances. These are small, embedded systems designed for home and small office file serving, featuring multiple hard drive bays and network connectivity.

### Supported Devices

- **USL-5P**: 1-bay NAS (also known as HDL-U)
- **HDL-U Series**: USL-5P rebrand
- **HDL-G Series**: Multi-bay NAS systems (HDL-G, HDL-GX, HDL-GW, HDL-GZ)
- **HDL-F Series**: Fanless NAS systems
- **HDL-W Series**: Wireless NAS systems
- **LANDISK Home**: Consumer NAS product line
- Other I-O DATA LANDISK variants

### Hardware Specifications (Typical)

- **CPU**: Renesas SH7751R (SH4) @ 240 MHz
- **Architecture**: Little-endian SH4
- **Memory**: 64 MB SDRAM (IOM_RAM_SIZE=0x04000000)
- **ROM**: 512 KB Flash (IOM_ROM_SIZE=0x00080000)
- **Storage**: 1-4 IDE/SATA hard drives (depending on model)
- **Network**: RTL8110S/RTL8169S Gigabit Ethernet
- **Interfaces**: USB 2.0 ports, serial console (internal)
- **Power**: External power adapter (typically 12V)

## Boot Method

NetBSD/landisk uses a two-stage bootloader system similar to the i386 architecture:

1. **Primary Bootstrap (bootxx)**: Loaded from MBR or partition boot sector
2. **Secondary Bootstrap (boot)**: Full-featured bootloader with interactive menu

### Bootloader Architecture

**Location**: `/sys/arch/landisk/stand/`

**Components**:
- **mbr/**: Master Boot Record code
- **bootxx/**: Primary bootstrap (stage 1)
- **boot/**: Secondary bootstrap (stage 2)

This design allows booting from various disk configurations including MBR partitions, NetBSD disklabel partitions, and RAID volumes.

## Boot Process Stages

### Stage 0: Hardware Bootstrap (ROM)

1. **Power-On Reset**:
   - SH7751R CPU starts at reset vector
   - ROM BIOS initializes hardware
   - Sets up memory controller for 64 MB RAM
   - Initializes PCI bus and devices

2. **Boot Device Selection**:
   - Checks boot mode settings (DIP switches on some models)
   - Default: Boot from IDE/SATA drive
   - Alternative: Network boot (if configured)
   - Serial console available on internal header

3. **MBR Loading**:
   - ROM reads sector 0 (MBR) from boot drive
   - Loads 512-byte MBR to memory
   - Transfers control to MBR code

### Stage 1: Primary Bootstrap (bootxx)

The primary bootstrap is installed in either the MBR or a partition boot sector.

**Load Address**: Varies (typically loaded by ROM/MBR)
**Target Address**: **0x8c201000** (PRIMARY_LOAD_ADDRESS)

#### bootxx Variants

1. **bootxx_ffsv1**: For FFSv1 filesystems
2. **bootxx_ffsv2**: For FFSv2 filesystems
3. **bootxx_ustarfs**: For tar-based filesystems

#### bootxx Execution Flow

**File**: `/sys/arch/landisk/stand/bootxx/boot1.c`

1. **Initialization** (`boot1()`):
   ```
   NetBSD/landisk [FFSv2] Primary Bootstrap
   ```

2. **Locate Secondary Bootstrap**:
   - Attempts to open file `/boot` from filesystem
   - First try: Start of MBR partition
   - Second try: With RAID frame offset (RF_PROTECTED_SECTORS)
   - Third try: Partition 'a' from disklabel

3. **Load Secondary Bootstrap**:
   - Read `/boot` file from disk
   - Maximum size check
   - Load to address: **0x8ff00000** (SECONDARY_LOAD_ADDRESS)

4. **Validate Secondary Bootstrap**:
   - Check magic number: LANDISK_BOOT_MAGIC_2
   - Verify file format

5. **Transfer Control**:
   - Jump to secondary bootstrap at 0x8ff00000
   - Pass boot sector number

**Key Addresses**:
```c
#define PRIMARY_LOAD_ADDRESS    0x8c201000
#define SECONDARY_LOAD_ADDRESS  0x8ff00000
```

### Stage 2: Secondary Bootstrap (boot)

The secondary bootstrap provides an interactive boot environment.

**Load Address**: **0x8ff00000**
**Execution**: boot2() function

#### boot2 Execution Flow

**File**: `/sys/arch/landisk/stand/boot/boot2.c`

1. **Hardware Initialization**:
   - `tick_init()`: Initialize timer
   - `cninit()`: Initialize console (SCIF serial port)
   - Console device from boot_params.bp_consdev

2. **Display Banner**:
   ```
   >> NetBSD/landisk Boot, Revision 1.x
   ```

3. **Boot Device Detection**:
   - `bios2dev()`: Convert BIOS device to NetBSD device
   - Sets default device (e.g., wd0a)
   - Determines boot partition from MBR

4. **Interactive Boot Menu**:
   ```
   Press return to boot now, any other key for boot menu
   booting wd0a:netbsd - starting in 5
   ```

5. **Auto-Boot Sequence**:
   - Timeout: 5 seconds (configurable via boot_params.bp_timeout)
   - Tries kernel names in order:
     - netbsd
     - netbsd.gz
     - onetbsd
     - onetbsd.gz
     - netbsd.old
     - netbsd.old.gz

6. **Boot Menu Commands**:
   ```
   > help
   commands are:
   boot [device:][filename] [-acdqsv]
        (ex. "wd0a:netbsd.old -s")
   ls [path]
   help|?
   halt
   quit
   ! (monitor)
   ```

7. **Kernel Loading** (`exec_netbsd()`):
   - Parse boot specification
   - Open kernel file
   - `loadfile()`: Load kernel into memory
   - Kernel loaded at address 0 (start of RAM)
   - Default load: 0x8c000000 (P1 cached mapping)

8. **Transfer to Kernel**:
   - Flush and disable caches
   - Prepare bootinfo structure
   - Jump to kernel entry point
   - Pass howto flags and bootinfo

**Boot Parameters Structure**:
```c
struct landisk_boot_params {
    uint32_t bp_consdev;    // Console device
    uint32_t bp_timeout;    // Boot timeout
};
```

### Stage 3: Kernel Initialization

**Entry Point**: Kernel entry point from ELF header
**Text Address**: **0x8c001000** (DEFTEXTADDR)

1. **Early Bootstrap** (`locore.s`):
   - Save bootinfo pointer
   - Set up kernel stack
   - Clear BSS segment
   - Install exception vectors
   - SH4-specific initialization

2. **MMU and Cache Setup**:
   - Initialize SH4 MMU
   - Flush TLB (64 entries)
   - Set up kernel page tables
   - Enable virtual memory translation
   - Configure cache policies

3. **Console Initialization**:
   - Set up SCIF serial console
   - Baud rate: 9600 bps (default)
   - Can be changed in kernel config

4. **Machine-Dependent Init** (`machdep.c`):
   - Parse bootinfo
   - Initialize interrupt controller (SH7751R INTC)
   - Set up timers (TMU)
   - Probe PCI bus
   - Detect network and disk controllers

5. **Device Autoconfiguration**:
   - Attach SH bus devices
   - PCI device enumeration
   - IDE/SATA controllers (wdc/wd)
   - Network controller (re - Realtek)
   - USB controllers (ohci/ehci)

## SH3 vs SH4 MMU Differences

The landisk platform exclusively uses the SH4 (SH7751R) processor, but it's useful to understand the differences from SH3 for context.

### SH4 MMU (SH7751R on landisk)

1. **TLB Structure**:
   - **UTLB**: 64 entries, 4-way set associative
   - **ITLB**: 4 entries, fully associative
   - Separate instruction and data paths

2. **Address Space**:
   - 8-bit ASID (256 address space identifiers)
   - 29-bit physical address (512 MB max)
   - 32-bit virtual address

3. **Page Sizes**:
   - Supported: 1 KB, 4 KB, 64 KB, 1 MB
   - NetBSD uses: 4 KB pages (PAGE_SIZE = 4096)

4. **TLB Management**:
   - MMUCR: MMU Control Register
     - AT: Address translation enable
     - TI: TLB invalidate
     - URB: Wired entry boundary
     - SQMD: Store queue mode
   - Hardware-assisted miss handling
   - Software TLB replacement

5. **Wired Entries**:
   - MMUCR.URB field (bits 18-23)
   - Reserves TLB entries for critical mappings
   - landisk kernel: Reserves entries for u-area
   - Formula: (SH4_UTLB_ENTRY - UPAGES) << SH4_MMUCR_URB_SHIFT

6. **Store Queues**:
   - Two 64-byte store queues
   - Write-combining buffer
   - Improves write performance to devices
   - SQMD bit controls user-mode access

### Key SH4 Features Used by landisk

1. **Cache Configuration**:
   - Instruction cache: 8 KB, 2-way
   - Data cache: 16 KB, 2-way
   - Write-back or write-through modes
   - Cache line: 32 bytes

2. **Exception Handling**:
   - Vector Base Register (VBR)
   - Multiple exception types (TLB miss, page fault, etc.)
   - Efficient exception processing

3. **PCI Bus Interface**:
   - SH7751R has integrated PCI host controller
   - 32-bit/33MHz PCI bus
   - Memory-mapped PCI configuration space

### SH3 vs SH4 Quick Reference

| Feature | SH3 | SH4 (landisk) |
|---------|-----|---------------|
| UTLB | 32 entries | 64 entries |
| ITLB | None | 4 entries |
| ASID | 4 bits (16) | 8 bits (256) |
| I-Cache | 4-8 KB | 8 KB |
| D-Cache | 8-16 KB | 16 KB |
| Store Queues | No | Yes (2×64 bytes) |
| Wired Entries | No | Yes (MMUCR.URB) |
| Clock | Up to 100 MHz | 200+ MHz |
| PCI | External | Integrated |

## Memory Map

### Physical Memory Layout

```
0x00000000 - 0x0007ffff : Boot Flash ROM (512 KB)
0x04000000 - 0x040fffff : PCI I/O space
0x04100000 - 0x041fffff : PCI memory space (low)
0x08000000 - 0x0bffffff : PCI memory space (high, 64 MB)
0x0c000000 - 0x0fffffff : Main SDRAM (64 MB)
0xfe000000 - 0xfeffffff : On-chip peripheral registers (SH7751R)
0xff000000 - 0xff0fffff : PCI configuration space
```

### Virtual Memory Segmentation (SH4)

```
P0: 0x00000000 - 0x7fffffff : User space (2 GB, TLB-translated)
P1: 0x80000000 - 0x9fffffff : Kernel cached (512 MB, direct-mapped)
P2: 0xa0000000 - 0xbfffffff : Kernel uncached (512 MB, direct-mapped)
P3: 0xc0000000 - 0xdfffffff : Kernel virtual (512 MB, TLB-translated)
P4: 0xe0000000 - 0xffffffff : Control/device registers (512 MB)
```

### NetBSD Virtual Memory Layout

```
0x00000000 - 0x7ffff000 : User address space (VM_MAXUSER_ADDRESS)
0x80000000 - 0x9fffffff : Kernel P1 (cached, direct physical 0x00000000)
0xa0000000 - 0xbfffffff : Kernel P2 (uncached, direct physical 0x00000000)
0xc0000000 - 0xdfffffff : Kernel P3 (TLB-translated virtual memory)
0xe0000000 - 0xffffffff : Memory-mapped I/O, control registers
```

### Bootloader Memory Usage

```
0x8c000000 : Physical RAM base (via P1)
0x8c001000 : Kernel text address (DEFTEXTADDR)
0x8c201000 : Primary bootstrap load address
0x8ff00000 : Secondary bootstrap load address
```

### Kernel Memory Layout

```
0x8c001000 : Kernel text (.text)
0x8c0xxxxx : Kernel read-only data (.rodata)
0x8c1xxxxx : Kernel data (.data)
0x8c2xxxxx : Kernel BSS (.bss)
0x8c3xxxxx : Kernel heap and dynamic memory
0x8fxxxxxx : End of 64 MB RAM (0x0c000000 + 0x04000000)
```

### Device Registers (P4 Space)

```
SH7751R On-Chip Peripherals:
0xffc00000 - 0xffc000ff : Cache control
0xffc00100 - 0xffc001ff : MMU (PTEH, PTEL, TTB, TEA, MMUCR, PTEA)
0xffe00000 - 0xffe003ff : DMAC (DMA controller)
0xffe80000 - 0xffe800ff : TMU (Timer unit)
0xffe80400 - 0xffe804ff : RTC (Real-time clock)
0xffec0000 - 0xffec00ff : SCIF (Serial port)
0xfff00000 - 0xfff003ff : INTC (Interrupt controller)

PCI Configuration:
0xff000000 - 0xff0fffff : PCI configuration space
0xff200000 - 0xff2fffff : SH7751 PCI controller registers
```

### PCI Device Memory

```
PCI I/O Space:
0x04000000 - 0x040fffff : Mapped PCI I/O (1 MB)

PCI Memory Space:
0x08000000 - 0x0bffffff : PCI memory window (64 MB)
  - Realtek network controller registers
  - USB controller registers
  - Other PCI device memory
```

## Build and Installation

### Building Bootloaders

```bash
# Build landisk tools
cd /usr/src
./build.sh -U -m landisk tools

# Build bootloaders
cd /usr/src/sys/arch/landisk/stand
make

# Results:
# mbr/mbr               - Master Boot Record
# bootxx/bootxx_ffsv2   - Primary bootstrap for FFSv2
# boot/boot             - Secondary bootstrap
```

### Building Kernel

```bash
# Build GENERIC kernel
cd /usr/src
./build.sh -U -m landisk kernel=GENERIC

# Build INSTALL kernel (for installation)
./build.sh -U -m landisk kernel=INSTALL

# Results:
# /usr/obj/sys/arch/landisk/compile/GENERIC/netbsd
# /usr/obj/sys/arch/landisk/compile/INSTALL/netbsd
```

### Kernel Configuration

Default: `/usr/src/sys/arch/landisk/conf/GENERIC`

Key options:
```
machine landisk sh3
include "arch/sh3/conf/std.sh3el"

# CPU configuration
options     SH4                      # SH4 CPU
options     SH7751R                  # Renesas SH7751R
options     PCLOCK=60000000          # 60 MHz peripheral clock

# Memory configuration
options     IOM_ROM_BEGIN=0x00000000
options     IOM_ROM_SIZE=0x00080000  # 512 KB
options     IOM_RAM_BEGIN=0x0c000000
options     IOM_RAM_SIZE=0x04000000  # 64 MB

# Link address
makeoptions DEFTEXTADDR="0x8c001000"
makeoptions ENDIAN="-EL"             # Little-endian
```

### Installation Procedure

#### Method 1: Direct Installation (Serial Console)

1. **Connect Serial Console**:
   - Locate internal serial header on LANDISK board
   - Connect 3.3V TTL serial adapter
   - Settings: 9600 baud, 8N1

2. **Boot Installation Kernel**:
   - Prepare USB drive or CF card with INSTALL kernel
   - Boot from installation media

3. **Run sysinst**:
   ```bash
   # NetBSD installation program will start automatically
   # Or run manually:
   sysinst
   ```

4. **Install Bootloader**:
   ```bash
   # After installation
   /usr/mdec/installboot -v /dev/rwd0a /usr/mdec/bootxx_ffsv2
   ```

#### Method 2: Prepare Disk on Another System

1. **Create Disk on NetBSD System**:
   ```bash
   # Assuming disk is sd0

   # Create MBR
   fdisk -u -0 -s 169/1/1 sd0

   # Create disklabel
   disklabel -e sd0
   # Add partitions:
   #  a: root (FFS)
   #  b: swap
   #  d: /usr (FFS)

   # Create filesystems
   newfs /dev/rsd0a
   newfs /dev/rsd0d

   # Mount and extract sets
   mount /dev/sd0a /mnt
   cd /mnt
   tar xzpf /path/to/base.tgz
   tar xzpf /path/to/etc.tgz
   # ... other sets

   # Install bootloader
   /usr/mdec/installboot -v /dev/rsd0a /usr/mdec/bootxx_ffsv2

   # Unmount
   umount /mnt
   ```

2. **Install Disk in LANDISK**:
   - Power off LANDISK
   - Install prepared disk
   - Power on and boot

#### Method 3: Network Installation

1. **Set up TFTP/NFS Server**:
   ```bash
   # On server
   # Configure TFTP for kernel
   cp netbsd.INSTALL /tftpboot/

   # Configure NFS for sets
   # Export NetBSD distribution directory
   ```

2. **Configure LANDISK for Network Boot**:
   - Enter boot ROM setup (device-specific)
   - Configure network parameters
   - Set boot device to network

### Post-Installation

1. **Configure Serial Console** (if using):
   ```bash
   # Edit /etc/ttys
   console "/usr/libexec/getty std.9600" vt100 on secure
   ```

2. **Configure Network**:
   ```bash
   # Edit /etc/rc.conf
   hostname="landisk"
   defaultroute="192.168.1.1"
   ifconfig_re0="inet 192.168.1.10 netmask 255.255.255.0"
   ```

3. **Enable Services**:
   ```bash
   # For NAS functionality
   sshd=YES
   samba=YES   # If using SMB/CIFS
   nfs_server=YES  # If using NFS
   ```

## Debugging

### Serial Console Access

The LANDISK devices have an internal serial console header.

**Hardware Connection**:
- **Warning**: Uses 3.3V TTL levels (not RS-232!)
- Requires USB-to-TTL serial adapter (3.3V)
- Common adapters: FTDI, CP2102, PL2303
- Pin locations vary by model (check device wiki)

**Typical Pinout**:
```
1. GND
2. RX (to adapter TX)
3. TX (to adapter RX)
4. VCC (3.3V - don't connect!)
```

**Terminal Settings**:
```
Baud rate: 9600 (default), 115200 (if configured)
Data bits: 8
Parity: None
Stop bits: 1
Flow control: None
```

**Terminal Program**:
```bash
# Using cu
cu -l /dev/ttyU0 -s 9600

# Using screen
screen /dev/ttyU0 9600

# Using minicom
minicom -D /dev/ttyU0 -b 9600
```

### Bootloader Debugging

Enable verbose output in boot loader:

**Boot Prompt**:
```
> boot -v netbsd
```

**Debug Boot Process**:
```
> boot -d netbsd  # Boot with DDB
> boot -a netbsd  # Ask for root device
> boot -s netbsd  # Single user mode
```

### DDB (In-Kernel Debugger)

Enable in kernel configuration:
```
options     DDB
options     DDB_HISTORY_SIZE=512
options     DDB_ONPANIC=1
makeoptions DEBUG="-g"
makeoptions COPY_SYMTAB=1
```

**Entering DDB**:
- Send break on serial console
- Kernel panic drops into DDB automatically
- Press Ctrl+Alt+Esc (if keyboard attached)

**Useful DDB Commands**:
```
trace                  # Stack backtrace
ps                     # Process list
show registers         # CPU registers
show pcpu              # Per-CPU info
machine tlb            # Display TLB entries
machine cache          # Cache status
x/i <addr>             # Disassemble instructions
x/x <addr>             # Examine memory (hex)
```

### KGDB (Remote Kernel Debugging)

Configure kernel for remote debugging:
```
options     KGDB
options     "KGDB_DEVNAME=\"scif\""
options     "KGDB_DEVRATE=115200"
```

**Remote Debugging Session**:
```bash
# On development machine
cd /usr/obj/sys/arch/landisk/compile/GENERIC
shle-netbsd-gdb netbsd

(gdb) target remote /dev/ttyU0
(gdb) break main
(gdb) continue
(gdb) backtrace
```

### Common Issues

1. **Bootloader Not Found**:
   - Verify bootloader installed: `installboot -v /dev/rwd0a /usr/mdec/bootxx_ffsv2`
   - Check disk partitioning (MBR and disklabel)
   - Verify boot partition is marked active

2. **Kernel Load Failure**:
   - Check filesystem integrity: `fsck /dev/rwd0a`
   - Verify kernel file exists: `ls -l /netbsd`
   - Check permissions on kernel file
   - Try booting with full path: `boot wd0a:netbsd`

3. **Serial Console Not Working**:
   - Verify cable connections
   - Check TTL voltage levels (3.3V not 5V!)
   - Try different baud rates
   - Confirm correct SCIF port
   - Check kernel console configuration

4. **Network Controller Not Detected**:
   - Verify PCI bus initialization
   - Check PCI device enumeration in dmesg
   - Ensure kernel has `re` driver
   - Check physical connection/link lights

5. **Disk Not Detected**:
   - Check IDE/SATA cable connections
   - Verify drive power
   - Check if drive is recognized by BIOS
   - Try different drive

6. **Random Crashes/Hangs**:
   - Check memory with diagnostic tools
   - Verify power supply is adequate
   - Check for overheating
   - Look for hardware conflicts in dmesg

### Performance Tuning

**For NAS Use**:
```bash
# In /etc/sysctl.conf
kern.maxfiles=8192
kern.maxvnodes=8192
vm.filemax=75        # Percentage of RAM for file cache
```

**Network Performance**:
```bash
# Enable jumbo frames (if supported)
ifconfig re0 mtu 9000
```

## Hardware Monitoring

Check system status:
```bash
# Temperature (if sensor available)
envstat

# Disk status
atactl wd0 identify
atactl wd0 smart status

# Network statistics
netstat -i
```

## References

- NetBSD/landisk Homepage: https://www.netbsd.org/ports/landisk/
- SH7751R Hardware Manual (Renesas)
- SH-4 CPU Core Architecture (Renesas)
- I-O DATA LANDISK Product Information
- NetBSD Source: `/usr/src/sys/arch/landisk/`
- Bootloader Source: `/usr/src/sys/arch/landisk/stand/`
- Installation Guide: https://www.netbsd.org/ports/landisk/install.html

## Technical Contacts

For questions about NetBSD/landisk:
- NetBSD Port Maintainers: port-sh3@netbsd.org
- General NetBSD Questions: netbsd-help@netbsd.org
- Hardware-Specific Issues: netbsd-users@netbsd.org

## Revision History

- Initial documentation based on NetBSD source analysis
- Bootloader architecture from `/sys/arch/landisk/stand/`
- Boot process derived from bootxx and boot source code
- Memory maps from kernel configuration and SH7751R manual
- Installation procedures from NetBSD installation documentation
