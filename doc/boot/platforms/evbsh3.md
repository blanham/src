# NetBSD/evbsh3 Boot Documentation

## Platform Overview

NetBSD/evbsh3 provides support for various Hitachi/Renesas SuperH SH3 and SH4 evaluation boards. This port targets development boards used for embedded systems development and testing, supporting multiple CPU variants and board configurations.

### Supported Boards

1. **NEXTVOD** - Set-top box development board
2. **T-SH7706LAN** - SH7706 evaluation board with Ethernet
3. **T-SH7706LSR** - SH7706 evaluation board with serial
4. **COMPUTEX7750** - SH7750 evaluation board
5. **COMPUTEXEVB** - General Computex evaluation board
6. **CQREEK** - CqREEK SH3 development board
7. **KZSH401** - KZ-SH4-01 evaluation board
8. **AP_MS104_SH4** - AP-MS104-SH4 development board

### Hardware Specifications (Typical)

- **CPU**: Various SH3/SH4 processors (SH7706, SH7708, SH7750, etc.)
- **Architecture**: Big-endian or Little-endian (board-dependent)
- **Memory**: 4-64 MB RAM (board-dependent)
- **ROM**: Flash ROM or EPROM for bootloader
- **Interfaces**: Serial, Ethernet, IDE (board-dependent)

## Boot Method

The evbsh3 platform uses different boot methods depending on the board configuration:

1. **ROM Boot**: Bootstrap code in Flash ROM or EPROM
2. **mesboot**: Minimal Embedded System Boot loader for some boards
3. **U-Boot**: External bootloader support (NEXTVOD)
4. **Direct Kernel Load**: For boards with external loaders

### Bootloader: mesboot

The mesboot (Minimal Embedded System Boot) loader is a simple bootloader used on some evbsh3 boards.

**Location**: `/sys/arch/evbsh3/stand/mesboot/`

**Features**:
- Minimal size for small ROM configurations
- Loads NetBSD kernel from MMC/SD card or Flash
- Simple command-line interface
- Memory detection and configuration
- Turbo mode support for faster operation

## Boot Process Stages

### Stage 1: Hardware Reset

1. **Power-On/Reset**:
   - CPU starts at reset vector (varies by board)
   - Typically 0x00000000 (ROM) or 0xa0000000 (P2 space)
   - Hardware is in a minimal initialized state

2. **ROM Bootstrap**:
   - Minimal CPU initialization
   - Set up stack pointer
   - Initialize critical hardware (memory controller, serial port)
   - Determine boot source (ROM, Flash, external)

### Stage 2: mesboot Execution (if present)

For boards using mesboot:

1. **mesboot Initialization**:
   ```
   NetBSD boot loader ver.0.2
   ```

2. **Memory Detection**:
   - Reads MCR (Memory Control Register)
   - Detects RAM size (8MB, 16MB, 32MB, or 64MB)
   - Configures memory subsystem

3. **Kernel Location**:
   - Default: `/mmc0/netbsd.bin` (from MMC card)
   - Can be specified as command-line argument
   - Loads kernel to address **0x8c002000**

4. **Kernel Parameters**:
   - Sets up kernel command line
   - Configures console (ttySC1, 115200 baud)
   - Sets root device (e.g., /dev/shmmc2)
   - Example: `mem=16M console=ttySC1,115200 root=/dev/shmmc2`

5. **Transfer Control**:
   - Copies parameters to 0x8c000000
   - Jumps to kernel entry point at 0x8c002000
   - Disables interrupts before jump

### Stage 3: U-Boot (NEXTVOD boards)

For NEXTVOD boards using U-Boot:

1. **U-Boot Initialization**:
   - Standard U-Boot boot sequence
   - Hardware initialization
   - Environment variable loading

2. **Kernel Loading**:
   - Load address: **0x80000000** (P1 address)
   - Text address: **0x80000040** (skips U-Boot image header)
   - Can load from network, Flash, or storage

3. **Boot Command**:
   ```
   bootm 0x80000000
   ```

### Stage 4: Direct ROM Boot

Some boards boot directly from ROM:

1. **ROM Image**:
   - Kernel linked at ROM address
   - Executes in place from ROM
   - May copy code to RAM for performance

2. **Memory Layout**:
   - Text address varies by board configuration
   - Common addresses:
     - Big-endian: 0x8c010000
     - Little-endian: 0x8c010000
     - NEXTVOD: 0x80000040

### Stage 5: Kernel Initialization

Once the kernel gains control:

1. **Early Bootstrap** (`locore.s`):
   - Identify CPU type (SH3 vs SH4)
   - Set up initial stack
   - Initialize BSS
   - Set up exception vectors
   - Configure cache

2. **MMU Initialization**:
   - Detect SH3 or SH4 MMU
   - Initialize TLB
   - Set up kernel page tables
   - Enable virtual memory

3. **Machine Initialization** (`machdep.c`):
   - Parse boot parameters
   - Initialize console
   - Detect memory size
   - Initialize devices
   - Mount root filesystem

## SH3 vs SH4 MMU Differences

The evbsh3 port supports both SH3 and SH4 processors, which have different MMU architectures:

### SH3 MMU Characteristics

1. **TLB Structure**:
   - **UTLB**: 32 entries, 4-way set associative
   - Single unified TLB for instructions and data
   - No separate ITLB

2. **Address Translation**:
   - 4-bit ASID (16 address spaces)
   - Page sizes: 1KB, 4KB
   - NetBSD uses 4KB pages

3. **MMU Registers** (at 0xffffxx00):
   - PTEH: Page Table Entry High
   - PTEL: Page Table Entry Low
   - TTB: Translation Table Base
   - TEA: TLB Exception Address
   - MMUCR: MMU Control Register

4. **TLB Operations**:
   - Software TLB miss handling
   - Manual TLB entry updates
   - LDTLB instruction loads entries
   - Address array manipulation for invalidation

### SH4 MMU Characteristics

1. **TLB Structure**:
   - **UTLB**: 64 entries, 4-way set associative
   - **ITLB**: 4 entries, fully associative
   - Separate instruction TLB for better performance

2. **Address Translation**:
   - 8-bit ASID (256 address spaces)
   - Page sizes: 1KB, 4KB, 64KB, 1MB
   - NetBSD uses 4KB pages
   - Extended addressing capabilities

3. **MMU Registers**:
   - All SH3 registers plus:
   - PTEA: Page Table Entry Assistance (extended attributes)
   - Enhanced MMUCR with wired entry support

4. **Advanced Features**:
   - Hardware page table walker (limited)
   - Store queues for write combining
   - Multiple simultaneous privileged regions
   - Enhanced cache control

### Comparison Table

| Feature | SH3 | SH4 |
|---------|-----|-----|
| UTLB Entries | 32, 4-way | 64, 4-way |
| ITLB | None | 4-entry |
| ASID Size | 4 bits (16 contexts) | 8 bits (256 contexts) |
| Page Sizes | 1KB, 4KB | 1KB, 4KB, 64KB, 1MB |
| Wired Entries | No | Yes (MMUCR.URB) |
| Store Queues | No | Yes |
| PTEA Register | No | Yes |
| Clock Speed | Up to 100 MHz | Up to 200+ MHz |

## Memory Map

### Generic SH3/SH4 Virtual Memory

```
P0: 0x00000000 - 0x7fffffff : User space (2GB, TLB-translated)
P1: 0x80000000 - 0x9fffffff : Kernel cached (512MB, direct physical map)
P2: 0xa0000000 - 0xbfffffff : Kernel uncached (512MB, direct physical map)
P3: 0xc0000000 - 0xdfffffff : Kernel virtual (512MB, TLB-translated)
P4: 0xe0000000 - 0xffffffff : Control registers (512MB, device registers)
```

### NetBSD Memory Layout

```
0x00000000 - 0x7ffff000 : User virtual address space
0x80000000 - 0x9fffffff : P1 cached kernel segment (direct-mapped)
0xa0000000 - 0xbfffffff : P2 uncached kernel segment (direct-mapped)
0xc0000000 - 0xdfffffff : P3 kernel virtual memory (TLB-mapped)
0xe0000000 - 0xffffffff : P4 memory-mapped I/O and control registers
```

### Board-Specific Memory Maps

#### NEXTVOD Configuration
```
Physical RAM: 0x0c000000 - 0x0fffffff (64 MB typical)
Load Address: 0x80000000 (P1 mapping)
Text Address: 0x80000040 (skip U-Boot header)
Kernel Space: 0x80000000 - 0x83ffffff
```

#### T-SH7706LAN Configuration
```
Physical RAM: 0x0c000000 - 0x0cffffff (16 MB)
Text Address: 0x8c010000
ROM: 0x00000000 - 0x001fffff (2 MB Flash)
```

#### Typical Evaluation Board
```
0x00000000 - 0x001fffff : Flash ROM (up to 2 MB)
0x04000000 - 0x043fffff : I/O space (varies by board)
0x0c000000 - 0x0cffffff : Main RAM (16 MB typical)
0x8c000000 : P1 cached mapping of RAM
0x8c010000 : Kernel text address (most boards)
0xac000000 : P2 uncached mapping of RAM
0xffff0000 - 0xffffffff : On-chip peripheral registers
```

### Peripheral Register Mapping (P4 Space)

```
0xfffe0000 - 0xfffeffff : On-chip peripherals
0xffff0000 - 0xffff00ff : Cache control
0xffff0100 - 0xffff01ff : MMU control
0xffff8000 - 0xffffbfff : Interrupt controller
0xffffc000 - 0xffffffff : Timer, serial, etc.
```

## Build and Installation

### Building for evbsh3

```bash
# Set up build environment
cd /usr/src

# Build tools for SH3
./build.sh -U -m evbsh3 tools

# Build kernel for specific board (example: COMPUTEX7750)
./build.sh -U -m evbsh3 kernel=COMPUTEX7750

# Result: /usr/obj/sys/arch/evbsh3/compile/COMPUTEX7750/netbsd
```

### Building mesboot

```bash
# Navigate to mesboot directory
cd /usr/src/sys/arch/evbsh3/stand/mesboot

# Build mesboot binary
make

# Result: mesboot/binary/mesboot (binary executable)
```

### Kernel Configurations

Available kernel configurations in `/sys/arch/evbsh3/conf/`:

- **NEXTVOD**: For NEXTVOD set-top box boards
- **T_SH7706LAN**: For T-SH7706 boards with Ethernet
- **T_SH7706LSR**: For T-SH7706 boards with serial
- **COMPUTEX7750**: For SH7750-based Computex boards
- **COMPUTEXEVB**: Generic Computex evaluation board
- **CQREEKSH3**: For CqREEK development boards
- **KZSH401**: For KZ-SH4-01 boards
- **AP_MS104_SH4**: For AP-MS104-SH4 boards

### Installation Methods

#### 1. Flash ROM Programming

For boards with Flash ROM:

```bash
# Convert kernel to appropriate format
objcopy -O binary netbsd netbsd.bin

# Program Flash using appropriate tool
# (Varies by board and programmer)
flashrom -p <programmer> -w netbsd.bin
```

#### 2. MMC/SD Card Boot (mesboot)

For boards using mesboot:

```bash
# Format SD card with FAT filesystem
newfs_msdos -F 16 /dev/rsd0e

# Mount and copy kernel
mount -t msdos /dev/sd0e /mnt
cp netbsd.bin /mnt/mmc0/
umount /mnt

# Insert SD card into board
# mesboot will load /mmc0/netbsd.bin
```

#### 3. Network Boot (U-Boot boards)

For boards with U-Boot and network support:

```bash
# On development host, set up TFTP server
# Copy kernel to TFTP directory
cp netbsd /tftpboot/

# In U-Boot console:
setenv serverip 192.168.1.10
setenv ipaddr 192.168.1.100
tftpboot 0x80000000 netbsd
bootm 0x80000000
```

#### 4. Serial Download

Some boards support serial download:

```bash
# Use kermit or similar tool
# Set up serial port: 115200 8N1
# Enter board's serial download mode
# Send kernel binary via serial
```

### Creating Installation Media

For installing a complete system:

1. **Prepare Root Filesystem**:
   ```bash
   # Create filesystem image
   dd if=/dev/zero of=root.img bs=1m count=64
   vnconfig vnd0 root.img
   newfs /dev/rvnd0a
   mount /dev/vnd0a /mnt

   # Extract sets
   cd /mnt
   tar xzpf /path/to/base.tgz
   tar xzpf /path/to/etc.tgz
   # ... other sets

   umount /mnt
   vnconfig -u vnd0
   ```

2. **Transfer to Board**:
   - Copy to Flash ROM
   - Write to SD card
   - Transfer via network

## Debugging

### Serial Console

Most evbsh3 boards provide serial console access:

**Configuration**:
- Port: SCIF (Serial Communication Interface with FIFO)
- Baud rate: 115200 bps (typical), can be 38400 or 57600
- Data bits: 8
- Parity: None
- Stop bits: 1
- Flow control: None (or hardware RTS/CTS if available)

**Kernel Configuration**:
```
options     CONSPEED=115200
options     SCIFCONSOLE
options     "SCIFCN_SPEED=115200"
```

### DDB (In-Kernel Debugger)

Enable DDB for debugging:

```
options     DDB
options     DDB_HISTORY_SIZE=512
options     DDB_ONPANIC=1
makeoptions DEBUG="-g"
makeoptions COPY_SYMTAB=1
```

**Common DDB Commands**:
```
trace              # Stack backtrace
ps                 # Process list
show registers     # CPU registers
x/x <addr>         # Examine memory
machine tlb        # Show TLB entries (if supported)
machine cache      # Cache status
```

### KGDB (Remote Debugging)

Enable KGDB for remote debugging:

```
options     KGDB
options     "KGDB_DEVNAME=\"scif\""
options     "KGDB_DEVRATE=115200"
```

**Remote Session**:
```bash
# On development machine
cd /usr/obj/sys/arch/evbsh3/compile/BOARDNAME
shle-netbsd-gdb netbsd          # For little-endian
# or
sh-netbsd-gdb netbsd            # For big-endian

(gdb) target remote /dev/ttyU0
(gdb) break main
(gdb) continue
```

### Board-Specific Debug Features

#### NEXTVOD Debugging

- U-Boot provides extensive debugging
- Network console available
- Built-in memory test commands

#### mesboot Debugging

Enable debug output in mesboot:
```bash
# Run with verbose output
mesboot -0 /mmc0/netbsd.bin  # Toggle turbo mode
```

### Common Issues and Solutions

1. **Board Won't Boot**:
   - Check power supply and reset
   - Verify Flash ROM programming
   - Check jumper settings on board
   - Verify serial console connection
   - Check boot mode switches

2. **Kernel Load Failures**:
   - Verify load address matches kernel link address
   - Check memory size configuration
   - Ensure bootloader and kernel endianness match
   - Verify kernel format (ELF vs binary)

3. **MMU Exceptions**:
   - Check TLB initialization
   - Verify page table setup
   - Check memory map configuration
   - Enable DDB for exception analysis

4. **Serial Console Issues**:
   - Verify baud rate matches kernel config
   - Check cable wiring (null modem vs straight)
   - Confirm SCIF port is enabled
   - Try different baud rates (115200, 57600, 38400)

5. **CPU Detection Errors**:
   - Verify kernel built for correct CPU (SH3 vs SH4)
   - Check CPU type in kernel config
   - Ensure correct arch flags during compilation

### Debug Output

Enable verbose boot messages:

```
options     DEBUG
options     DIAGNOSTIC
options     VERBOSE_INIT_SH
```

Add debug printfs in kernel:
```c
printf("Boot: CPU=%s at %dMHz\n", cpu_model, cpu_freq_mhz);
printf("Boot: Memory=%dMB at 0x%08x\n", mem_size_mb, mem_base);
```

### Hardware Debugging Tools

1. **JTAG Debugger**: Some boards support JTAG debugging
2. **Logic Analyzer**: For analyzing bus signals
3. **Oscilloscope**: For checking clock signals
4. **Multimeter**: For verifying power rails

## References

- NetBSD/evbsh3 Information: https://www.netbsd.org/ports/evbsh3/
- Hitachi SH7706 Hardware Manual
- Hitachi SH7708 Hardware Manual
- Renesas SH7750 Hardware Manual
- SH3/SH4 Programming Manual
- U-Boot Documentation: https://www.denx.de/wiki/U-Boot
- NetBSD Source: `/usr/src/sys/arch/evbsh3/`
- mesboot Source: `/usr/src/sys/arch/evbsh3/stand/mesboot/`

## Technical Contacts

For questions about NetBSD/evbsh3:
- NetBSD Port Maintainers: port-sh3@netbsd.org
- General NetBSD Questions: netbsd-help@netbsd.org

## Revision History

- Initial documentation based on NetBSD source analysis
- Boot process derived from mesboot implementation
- Configuration from `sys/arch/evbsh3/conf/` and `sys/arch/evbsh3/stand/`
