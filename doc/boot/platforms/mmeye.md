# NetBSD/mmeye Boot Documentation

## Platform Overview

NetBSD/mmeye provides support for the Hitachi MMEYE (Multimedia Eye) series of embedded camera controller boards based on SuperH SH3 processors. These boards were designed for embedded vision systems, digital cameras, and image processing applications.

### Supported Boards

- **MMEYE Original**: Early SH7708-based board
- **MMEYE NEW**: Updated board with enhanced features
- **MMTA-XXX Series**: Hitachi multimedia terminal adapters
- Custom derivatives based on MMEYE reference design

### Hardware Specifications (Typical)

- **CPU**: Hitachi SH7708R @ 100 MHz (SH3 core)
- **Architecture**: Big-endian or Little-endian (configurable)
- **Memory**: 16 MB SDRAM (can be 8/16/32/64 MB depending on configuration)
- **ROM**: Flash ROM for bootloader
- **Storage**: IDE hard drive or CompactFlash
- **Network**: Various network adapters (board-dependent)
- **Serial**: SCIF serial port (console)
- **Video**: Video input/output hardware (camera interface)
- **Clock**: PCLOCK = 33.33 MHz

### Board Variants

The MMEYE platform has several interrupt controller configurations:

```c
options MMEYE_NEW_INT=0xb000000e    // New interrupt controller location
```

## Boot Method

NetBSD/mmeye supports three different boot methods depending on the board configuration and desired boot source:

1. **boot**: Standard bootloader for IDE/ATA devices
2. **bootcoff**: COFF format bootloader
3. **bootelf**: ELF format bootloader

### Bootloader Architecture

**Location**: `/sys/arch/mmeye/stand/`

**Components**:
- **boot/**: Main bootloader for IDE/ATA boot
- **bootcoff/**: COFF format bootloader
- **bootelf/**: ELF format bootloader

All three bootloaders provide similar functionality but differ in the executable format they load.

## Boot Process Stages

### Stage 0: Hardware Bootstrap (ROM)

1. **Power-On Reset**:
   - SH7708R CPU starts at reset vector
   - Typically 0x00000000 (ROM) or 0xa0000000 (P2 space)
   - ROM BIOS performs minimal hardware initialization

2. **ROM Initialization**:
   - Set up memory controller
   - Configure SDRAM timing
   - Initialize serial port for diagnostics
   - Set up exception vectors

3. **Boot Source Selection**:
   - Check DIP switches or jumpers
   - Default: Boot from Flash ROM
   - Alternative: Boot from IDE/CF device
   - Some boards support network boot

### Stage 1: Bootloader Execution

The MMEYE bootloader is loaded from ROM or Flash and provides an interactive boot environment.

**Load Address**: Varies by configuration (typically in P1 cached space)

#### Bootloader Initialization

**File**: `/sys/arch/mmeye/stand/boot/boot.c`

1. **Hardware Setup**:
   - Initialize timer (`tmu_init()` for SH4, not used on SH3)
   - Initialize console (`cninit()`)
   - Set up serial port (SCIF)

2. **Display Banner**:
   ```
   >> NetBSD/mmeye Bootloader, Revision 1.x [@address]
   ```

3. **Boot Device Detection**:
   - Default device: DEFBOOTDEV (typically "wd0a" for IDE)
   - Default kernel: DEFKERNELNAME (typically "netbsd")

4. **Boot Prompt**:
   ```
   Boot [wd0a:netbsd]:
   ```

5. **User Input Processing**:
   - Timeout: Wait for user input
   - Parse boot specification: `[device:][kernel] [-flags]`
   - Examples:
     - `netbsd` - Boot default kernel
     - `wd0a:netbsd.old` - Boot old kernel from wd0a
     - `netbsd -s` - Boot single-user mode
     - `-a` - Ask for root device

6. **Kernel Names**:
   The bootloader tries multiple kernel names in sequence:
   ```c
   char *kernelnames[] = {
       "netbsd",
       "netbsd.gz",
       "onetbsd",
       "onetbsd.gz",
       NULL
   };
   ```

### Stage 2: Kernel Loading

**File**: `/sys/arch/mmeye/stand/boot/boot.c` (main function)

1. **Parse Boot Parameters**:
   - Extract device name (e.g., "wd0a")
   - Extract kernel name (e.g., "netbsd")
   - Parse boot flags (e.g., `-s` for single-user)

2. **Open Kernel File**:
   - Construct full path: `device:kernel`
   - Example: `wd0a:netbsd`
   - Use libsa `loadfile()` function

3. **Load Kernel into Memory**:
   - Read ELF headers
   - Allocate memory for segments
   - Load text, data, BSS segments
   - Default load address from ELF header
   - Typical text address: **0x8c010000**

4. **Prepare Boot Information**:
   - Create bootinfo structure
   - `bi_bootdev`: Boot device information
   - `bi_bootpath`: Kernel path
   - `bi_howto`: Boot flags (howto)
   - Initialize bootinfo: `bi_init()`
   - Add boot device: `bi_add(&bi_bdev, BTINFO_BOOTDEV, sizeof(bi_bdev))`
   - Add boot path: `bi_add(&bi_bpath, BTINFO_BOOTPATH, sizeof(bi_bpath))`
   - Add howto flags: `bi_add(&bi_howto, BTINFO_HOWTO, sizeof(bi_howto))`

5. **Display Load Information**:
   ```
   Loading: wd0a:netbsd (howto 0x0)
   Starting at 0x8c010000
   ```

6. **Transfer Control to Kernel**:
   - Entry point from ELF: `marks[MARK_ENTRY]`
   - Pass bootinfo magic: `BOOTINFO_MAGIC`
   - Pass bootinfo address: `bi_addr`
   - Delay for console output: `delay(10000)`
   - Jump to kernel: `(*entry)(BOOTINFO_MAGIC, bi_addr)`

### Stage 3: Kernel Initialization

**Entry Point**: 0x8c010000 (DEFTEXTADDR)

1. **Early Bootstrap** (`locore.s`):
   - Receive bootinfo parameters (magic, address)
   - Validate BOOTINFO_MAGIC
   - Save bootinfo pointer
   - Set up kernel stack
   - Clear BSS segment
   - Install exception vector table (VBR)

2. **CPU Identification**:
   - Detect SH3 vs SH4 (mmeye is SH3)
   - Identify specific CPU model (SH7708R)
   - Read CPU version registers
   - Configure CPU-specific features

3. **MMU Initialization**:
   - Flush TLB entries
   - Set ASID to 0 (kernel)
   - Set up initial page tables
   - Enable address translation (MMUCR.AT)
   - Configure cache policies

4. **Cache Setup**:
   - SH7708R has 8 KB instruction cache
   - 8 KB data cache (or 16KB depending on model)
   - Enable instruction cache
   - Configure data cache (write-back or write-through)
   - Set cache-enable bits

5. **Console Initialization**:
   - Initialize SCIF serial port
   - Baud rate: 38400 or 115200 (configurable)
   - Display boot messages

6. **Machine-Dependent Initialization** (`machdep.c`):
   - Parse bootinfo structure
   - Extract boot device and path
   - Configure interrupt controller
   - MMEYE_NEW_INT address if configured
   - Set up timers (TMU)
   - Initialize clock

7. **Device Autoconfiguration**:
   - Probe SH bus devices
   - Initialize IDE/ATA controller (WDC)
   - Attach disk drives (wd)
   - Initialize network controller (if present)
   - Initialize serial ports (com, scif)
   - Initialize parallel port (if present)

8. **Root Filesystem Mounting**:
   - Use boot device from bootinfo
   - Mount root filesystem
   - Continue to multi-user boot

## SH3 vs SH4 MMU Differences

The MMEYE platform uses SH3 (SH7708R), but understanding the differences from SH4 is useful for context.

### SH3 MMU (SH7708R on mmeye)

1. **TLB Structure**:
   - **UTLB**: 32 entries, 4-way set associative
   - Single unified TLB for instructions and data
   - No separate ITLB

2. **Address Translation**:
   - 4-bit ASID (16 address space identifiers)
   - Supports 16 simultaneous address spaces
   - 29-bit physical address space (512 MB)

3. **Page Sizes**:
   - 1 KB pages
   - 4 KB pages (used by NetBSD)
   - PAGE_SIZE = 4096 bytes

4. **MMU Registers**:
   Located at 0xffffxx00 (P4 space):
   - **PTEH** (0xffffff00): Page Table Entry High
     - VPN (Virtual Page Number)
     - ASID (Address Space ID)
   - **PTEL** (0xffffff04): Page Table Entry Low
     - PPN (Physical Page Number)
     - Protection bits (V, PR, D, C, SH)
   - **TTB** (0xffffff08): Translation Table Base
   - **TEA** (0xffffff0c): TLB Exception Address
   - **MMUCR** (0xffffff10): MMU Control Register
     - AT: Address Translation enable
     - TF: TLB Flush

5. **TLB Operations**:
   - **LDTLB** instruction: Load TLB entry
   - Software TLB miss handler required
   - Manual TLB entry management
   - Associative writes to TLB arrays

6. **Address Array Access**:
   - Address array: 0xf2000000-0xf2ffffff
   - Data array: 0xf3000000-0xf3ffffff
   - Direct TLB manipulation possible

### SH4 MMU (For Comparison)

| Feature | SH3 (mmeye) | SH4 |
|---------|-------------|-----|
| TLB Type | Unified (UTLB only) | Split (UTLB + ITLB) |
| UTLB Entries | 32, 4-way | 64, 4-way |
| ITLB Entries | None | 4, fully associative |
| ASID Size | 4 bits (16 contexts) | 8 bits (256 contexts) |
| Page Sizes | 1KB, 4KB | 1KB, 4KB, 64KB, 1MB |
| Wired Entries | No | Yes (MMUCR.URB) |
| Store Queues | No | Yes (2×64 bytes) |
| PTEA Register | No | Yes (extended attributes) |
| Cache | 8KB I + 8KB D | 8KB I + 16KB D |

### SH3 Cache Configuration

1. **Instruction Cache**:
   - Size: 8 KB
   - 2-way set associative
   - 256 lines × 16 bytes/line
   - Cache control via CCR

2. **Data Cache**:
   - Size: 8 KB (or 16 KB on some variants)
   - 2-way set associative
   - Write-back or write-through
   - Cache control via CCR

3. **Cache Control Register (CCR)**:
   - Located at 0xffffffec
   - CE: Cache Enable
   - WT: Write-Through mode
   - CB: Copy-Back (Write-Back) mode
   - CF: Cache Flush
   - ORA: Operating Register Array mode

## Memory Map

### Physical Memory Layout

```
0x00000000 - 0x001fffff : ROM (Flash, 2 MB typical)
0x04000000 - 0x043fffff : I/O devices (4 MB window)
0x06000000 - 0x06ffffff : Video/Camera hardware
0x0a000000 - 0x0affffff : Network/peripheral devices
0x0c000000 - 0x0cffffff : Main SDRAM (16 MB)
0x0c000000 - 0x0dffffff : Extended RAM configurations (up to 32/64 MB)
```

### Virtual Memory Segmentation (SH3)

```
P0: 0x00000000 - 0x7fffffff : User space (2 GB, TLB-translated)
P1: 0x80000000 - 0x9fffffff : Kernel cached (512 MB, direct-mapped to 0x00000000)
P2: 0xa0000000 - 0xbfffffff : Kernel uncached (512 MB, direct-mapped to 0x00000000)
P3: 0xc0000000 - 0xdfffffff : Kernel virtual (512 MB, TLB-translated)
P4: 0xe0000000 - 0xffffffff : Control registers (512 MB, device registers)
```

### NetBSD Memory Layout

```
User Space:
0x00000000 - 0x7ffff000 : User virtual address space (VM_MAXUSER_ADDRESS)

Kernel Space:
0x80000000 - 0x9fffffff : P1 - Direct-mapped cached (physical 0x00000000+)
0xa0000000 - 0xbfffffff : P2 - Direct-mapped uncached (physical 0x00000000+)
0xc0000000 - 0xdfffffff : P3 - TLB-translated kernel virtual memory
0xe0000000 - 0xffffffff : P4 - Memory-mapped I/O and control registers
```

### Kernel Memory Usage

```
Physical RAM: 0x0c000000 - 0x0cffffff (16 MB)

Via P1 Mapping (cached):
0x8c000000 : RAM base
0x8c010000 : Kernel text address (DEFTEXTADDR)
0x8c0xxxxx : Kernel .text segment
0x8c1xxxxx : Kernel .rodata segment
0x8c2xxxxx : Kernel .data segment
0x8c3xxxxx : Kernel .bss segment
0x8c4xxxxx : Kernel heap
0x8cxxxxxx : Dynamic kernel memory
0x8cffffff : End of 16 MB RAM

Via P2 Mapping (uncached):
0xac000000 - 0xacffffff : Uncached access to same physical RAM
```

### Device and I/O Memory

```
I/O Space (via P2 uncached):
0xa4000000 - 0xa43fffff : Peripheral devices
0xa6000000 - 0xa6ffffff : Video/Camera interface
0xaa000000 - 0xaaffffff : Network devices

MMEYE NEW Interrupt Controller:
0xb000000e : Interrupt status/control register (if MMEYE_NEW_INT defined)
```

### SH7708R On-Chip Peripherals (P4 Space)

```
0xfffe0000 - 0xfffeffff : On-chip peripheral module space
0xffff0000 - 0xffff1fff : Cache and TLB control
  0xffffffec : CCR (Cache Control Register)
  0xffffff00 : PTEH (Page Table Entry High)
  0xffffff04 : PTEL (Page Table Entry Low)
  0xffffff08 : TTB (Translation Table Base)
  0xffffff0c : TEA (TLB Exception Address)
  0xffffff10 : MMUCR (MMU Control Register)
0xffff8000 - 0xffffbfff : Interrupt controller (INTC)
  0xfffffee0 : ICR (Interrupt Control Register)
  0xfffffee2 : IPRA-IPRF (Interrupt Priority Registers)
0xffffc000 - 0xffffc0ff : Timer unit (TMU)
  0xffffc000 : TSTR (Timer Start Register)
  0xffffc008 : TCOR0/TCNT0 (Timer 0)
  0xffffc014 : TCOR1/TCNT1 (Timer 1)
  0xffffc020 : TCOR2/TCNT2 (Timer 2)
0xffffc800 - 0xffffc8ff : SCIF (Serial Communication Interface with FIFO)
  0xffffc800 : SCSMR (Serial Mode Register)
  0xffffc804 : SCSCR (Serial Control Register)
  0xffffc808 : SCFTDR (Transmit FIFO Data Register)
  0xffffc80c : SCFSR (Serial Status Register)
  0xffffc810 : SCFRDR (Receive FIFO Data Register)
  0xffffc814 : SCFCR (FIFO Control Register)
  0xffffc818 : SCFDR (FIFO Data Count Register)
0xffffd000 - 0xffffd0ff : RTC (Real-Time Clock)
```

## Build and Installation

### Building Bootloaders

```bash
# Navigate to mmeye stand directory
cd /usr/src/sys/arch/mmeye/stand

# Build all bootloaders
make

# Or build specific bootloader
cd boot && make
cd bootcoff && make
cd bootelf && make

# Results:
# boot/boot           - Standard bootloader
# bootcoff/bootcoff   - COFF format bootloader
# bootelf/bootelf     - ELF format bootloader
```

### Building Kernel

```bash
# Build tools
cd /usr/src
./build.sh -U -m mmeye tools

# Build GENERIC kernel
./build.sh -U -m mmeye kernel=GENERIC

# Result: /usr/obj/sys/arch/mmeye/compile/GENERIC/netbsd
```

### Kernel Configuration

Default configuration: `/usr/src/sys/arch/mmeye/conf/GENERIC`

Key options:
```
include "arch/mmeye/conf/std.mmeye"

# CPU support
options     SH3                      # SH3 CPU family
options     SH7708R                  # SH7708R CPU @ 100MHz
options     MMEYE                    # MMEYE board support
options     MMEYE_NEW_INT=0xb000000e # New interrupt controller
#options    MMEYE_NO_CACHE           # Disable cache (for debugging)

# Clocks
options     PCLOCK=33330000          # 33.33MHz peripheral clock
options     INITTODR_ALWAYS_USE_RTC  # Always use RTC for time

# Memory
options     IOM_RAM_SIZE=0x01000000  # 16MB
options     IOM_RAM_BEGIN=0x0c000000

# Link address
makeoptions DEFTEXTADDR="0x8c010000"
```

### Installation Methods

#### Method 1: IDE/ATA Installation

1. **Prepare Disk**:
   ```bash
   # On another NetBSD system with disk attached

   # Partition disk
   fdisk -u sd0

   # Create disklabel
   disklabel -e sd0

   # Create filesystems
   newfs -O 2 /dev/rsd0a  # Root
   newfs -O 2 /dev/rsd0d  # /usr

   # Mount and extract
   mount /dev/sd0a /mnt
   mkdir /mnt/usr
   mount /dev/sd0d /mnt/usr

   cd /mnt
   tar xzpf /path/to/base.tgz
   tar xzpf /path/to/etc.tgz
   tar xzpf /path/to/comp.tgz
   # ... other sets

   # Copy kernel
   cp /path/to/netbsd /mnt/

   # Unmount
   umount /mnt/usr
   umount /mnt
   ```

2. **Install Bootloader**:
   ```bash
   # Copy bootloader to disk
   # Method depends on ROM/Flash configuration
   # Consult board documentation
   ```

3. **Install Disk in MMEYE**:
   - Power off board
   - Connect IDE/ATA disk
   - Power on and boot

#### Method 2: CompactFlash Boot

1. **Format CompactFlash**:
   ```bash
   # Similar to IDE method
   # Use CF card adapter
   newfs -O 2 /dev/rsd0a
   ```

2. **Extract System**:
   ```bash
   mount /dev/sd0a /mnt
   cd /mnt
   # Extract distribution sets
   ```

3. **Configure Boot**:
   - Some MMEYE boards can boot from CF
   - Configure DIP switches/jumpers
   - Install bootloader to CF

#### Method 3: ROM/Flash Boot

1. **Build ROM Image**:
   ```bash
   # Combine bootloader and kernel
   objcopy -O binary boot boot.bin
   objcopy -O binary netbsd netbsd.bin

   # Create ROM image (board-specific)
   cat boot.bin netbsd.bin > romimage.bin
   ```

2. **Program Flash ROM**:
   ```bash
   # Use appropriate Flash programmer
   # Method varies by board and programmer
   flashrom -p <programmer> -w romimage.bin
   ```

### Serial Console Configuration

Configure serial console in kernel:
```
options     SCIFCONSOLE              # Use SCIF for console
options     CONSPEED=38400           # Or 115200
options     "SCIFCN_SPEED=38400"
```

Configure getty in `/etc/ttys`:
```
console "/usr/libexec/getty std.38400" vt100 on secure
```

## Debugging

### Serial Console

**Hardware**:
- SCIF serial port on MMEYE board
- RS-232 levels or TTL (check board specs)
- DB-9 or header connector

**Settings**:
```
Baud rate: 38400 or 115200
Data bits: 8
Parity: None
Stop bits: 1
Flow control: None
```

**Terminal Program**:
```bash
# Using cu
cu -l /dev/ttyU0 -s 38400

# Using screen
screen /dev/ttyU0 38400

# Using tip
tip com0
```

### DDB (In-Kernel Debugger)

Enable DDB:
```
options     DDB
options     DDB_HISTORY_SIZE=512
options     DDB_ONPANIC=1
makeoptions DEBUG="-g"
makeoptions COPY_SYMTAB=1
```

**Entering DDB**:
- Send break signal on serial port
- Kernel panic (if DDB_ONPANIC=1)
- Call `Debugger()` in kernel code

**DDB Commands**:
```
trace                  # Stack backtrace
ps                     # Process list
show registers         # Display CPU registers
show tlb               # TLB entries (if implemented)
x/x <addr>             # Examine memory
x/i <addr>             # Disassemble instructions
break <addr>           # Set breakpoint
continue               # Continue execution
```

### KGDB (Remote Debugging)

Configure kernel:
```
options     KGDB
options     "KGDB_DEVNAME=\"scif\""
options     "KGDB_DEVRATE=38400"
```

**Remote Session**:
```bash
# On development machine
cd /usr/obj/sys/arch/mmeye/compile/GENERIC
sh3eb-netbsd-gdb netbsd    # For big-endian
# or
sh3el-netbsd-gdb netbsd    # For little-endian

(gdb) target remote /dev/ttyU0
(gdb) continue
```

### Cache Debugging

Disable cache for debugging:
```
options     MMEYE_NO_CACHE
```

This can help isolate cache-related issues but will significantly reduce performance.

### Common Issues

1. **Boot Failure**:
   - Check ROM/Flash programming
   - Verify bootloader location
   - Check DIP switch settings
   - Verify power supply

2. **Console Not Working**:
   - Check serial cable
   - Verify baud rate matches kernel config
   - Ensure SCIF port enabled
   - Check signal levels (RS-232 vs TTL)

3. **IDE/ATA Not Detected**:
   - Check cable connections
   - Verify power to drive
   - Check jumper settings (master/slave)
   - Try different drive

4. **Interrupt Issues**:
   - Verify MMEYE_NEW_INT setting
   - Check interrupt controller address
   - Different board revisions use different addresses

5. **Memory Errors**:
   - Test RAM with diagnostic tools
   - Check memory configuration options
   - Verify RAM size matches kernel config

### Performance Tuning

**Cache Configuration**:
```c
// Enable both caches for best performance
CCR |= (CCR_CE | CCR_CB);  // Cache enable, copy-back mode
```

**Compiler Optimization**:
```bash
# In kernel config
makeoptions COPTS="-O2"
```

## Hardware-Specific Notes

### Interrupt Controller

The MMEYE board has different interrupt controller configurations:

**Original MMEYE**:
- Standard SH7708R interrupt controller
- Registers at standard SH3 addresses

**MMEYE NEW**:
- Custom interrupt controller
- Define location: `MMEYE_NEW_INT=0xb000000e`

### WDC (Western Digital Controller)

IDE/ATA interface configuration:
```c
wdc0    at shb?                      # On-board IDE
wd*     at wdc? channel ? drive ?    # IDE disks
```

### Network Controllers

Various network adapters supported:
```c
ne*     at pcmcia? function ?        # NE2000-compatible
```

## References

- NetBSD/mmeye Information: https://www.netbsd.org/ports/mmeye/
- Hitachi SH7708R Hardware Manual
- SH-3 CPU Core Architecture (Hitachi/Renesas)
- MMEYE Board Documentation (Hitachi)
- NetBSD Source: `/usr/src/sys/arch/mmeye/`
- Bootloader Source: `/usr/src/sys/arch/mmeye/stand/`

## Technical Contacts

For questions about NetBSD/mmeye:
- NetBSD Port Maintainers: port-sh3@netbsd.org
- General NetBSD Questions: netbsd-help@netbsd.org
- Embedded SH3 Discussion: netbsd-users@netbsd.org

## Revision History

- Initial documentation based on NetBSD source analysis
- Bootloader architecture from `/sys/arch/mmeye/stand/`
- Boot process derived from boot.c source code
- Memory maps from SH7708R hardware manual and kernel configuration
- Installation procedures adapted from general NetBSD installation documentation
