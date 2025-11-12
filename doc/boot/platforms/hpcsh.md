# NetBSD/hpcsh Boot Documentation

## Platform Overview

NetBSD/hpcsh provides support for Windows CE-based handheld computers (H/PC, Handheld PC) powered by Hitachi SuperH SH3 and SH4 processors. These devices were popular in the late 1990s and early 2000s as personal digital assistants and mobile computing devices.

### Supported Devices

NetBSD/hpcsh supports various H/PC Pro devices including:

#### SH3-based Devices (7707, 7709, 7709A)
- HP Jornada 680/690 (SH7709A)
- Hitachi HPW-50PAD, HPW-650PA (SH7709)
- SHARP Telios HC-AJ1/AJ2/AJ3 (SH7707)
- SHARP Mobilon HC-4100/4500 (SH7707)
- Casio Cassiopeia A/E series (SH7707)

#### SH4-based Devices (7750)
- HP Jornada 820/728 (SH7750)
- SHARP Telios HC-VJ1C (SH7750)

### Hardware Specifications (Typical)

- **CPU**: SH7707 @ 60MHz, SH7709A @ 133MHz, SH7750 @ 166MHz
- **Architecture**: Little-endian SH3/SH4
- **Memory**: 8-32 MB RAM (device-dependent)
- **ROM**: Flash ROM with Windows CE
- **Display**: LCD with various resolutions (640x240 typical)
- **Storage**: CompactFlash, PC Card, internal Flash
- **Interfaces**: Serial, IrDA, USB (device-dependent)

## Boot Method

NetBSD/hpcsh uses a unique boot method that leverages the existing Windows CE operating system. Instead of replacing Windows CE, NetBSD is booted through a Windows CE application called **hpcboot**.

### hpcboot Bootloader

**Location**: `/sys/arch/hpc/stand/hpcboot/`

**Overview**: hpcboot is a Windows CE application (hpcboot.exe) that:
- Runs under Windows CE (H/PC Pro 2.11 or later)
- Provides a graphical boot menu
- Loads NetBSD kernel into memory
- Sets up initial boot parameters
- Transfers control to NetBSD kernel

**Supported Architectures**:
- ARM (separate binary)
- MIPS (uses pbsdboot instead)
- SH3 (SH7707, SH7709, SH7709A)
- SH4 (SH7750)

**Key Features**:
- Graphical user interface under Windows CE
- Boot configuration options
- Console selection (LCD or serial)
- Filesystem support (FAT, UFS via Windows CE)
- HTTP loading support for network boot
- Memory manager options (LockPages or HardMMU)

## Boot Process Stages

### Stage 1: Windows CE Operation

1. **Normal H/PC Operation**:
   - Device boots Windows CE from ROM
   - Windows CE initializes hardware
   - User operates device normally

2. **hpcboot Launch**:
   - User starts hpcboot.exe from Windows CE
   - Application displays graphical boot menu
   - User selects kernel and options

### Stage 2: hpcboot Initialization

1. **Platform Detection**:
   - Detects CPU type (SH3 vs SH4)
   - Identifies specific CPU model:
     - ARCHITECTURE_SH3_7707
     - ARCHITECTURE_SH3_7709
     - ARCHITECTURE_SH3_7709A
     - ARCHITECTURE_SH4_7750
   - Validates CPU matches selected architecture

2. **Console Setup**:
   - **LCD Console**: Framebuffer via Windows CE graphics
   - **Serial Console**: SCIF serial port
   - User selectable in hpcboot GUI

3. **Memory Manager Selection**:
   - **LockPages Method**: Uses Windows CE API to lock pages
     - Available on devices with LockPages API
     - More compatible with Windows CE
   - **HardMMU Method**: Direct MMU manipulation (SH3 only)
     - Bypasses Windows CE memory management
     - Required when LockPages unavailable
     - Not available for SH4 in current implementation

4. **File Manager Initialization**:
   - Supports FAT filesystem (via Windows CE)
   - Supports UFS filesystem (limited)
   - Supports HTTP loading (network boot)
   - Kernel typically stored on CompactFlash or PC Card

### Stage 3: Kernel Loading

1. **Kernel Location**:
   - Stored on removable media (CompactFlash, SD card)
   - Can be loaded from network via HTTP
   - Typical paths:
     - `\Storage Card\netbsd` (CompactFlash)
     - `\Hard Disk\netbsd` (internal Flash)
     - `http://server/netbsd` (network)

2. **Kernel Format**:
   - ELF or COFF format supported
   - Load address depends on device memory map
   - Default text address: **0x8c001000**

3. **Memory Preparation**:
   - hpcboot allocates memory for kernel
   - Uses LockPages to prevent Windows CE from using it
   - Or uses MMU manipulation to reserve memory
   - Copies kernel to load address

4. **Boot Information Structure**:
   - Creates bootinfo structure with:
     - Platform ID (platid)
     - Console configuration
     - Memory map
     - Boot arguments
     - Device tree information

### Stage 4: Transition to NetBSD

1. **Disable Windows CE**:
   - Disables interrupts
   - Stops Windows CE kernel
   - Takes control of hardware

2. **MMU Setup**:
   - For SH3: Uses HardMMU code to manipulate TLB
   - For SH4: Relies on existing setup (limited MMU support)
   - Sets up page tables for NetBSD

3. **Cache Configuration**:
   - Flushes instruction and data caches
   - Configures cache policies
   - Enables/disables as appropriate

4. **Transfer Control**:
   - Jumps to kernel entry point
   - Passes bootinfo structure pointer
   - Kernel starts in privileged mode

### Stage 5: Kernel Initialization

1. **Early Bootstrap** (`locore.s`):
   - Save bootinfo pointer
   - Set up stack
   - Initialize BSS
   - Set up exception vectors
   - Identify CPU type

2. **Platform Initialization**:
   - Parse bootinfo structure
   - Extract platform ID (platid)
   - Configure hardware based on device type
   - Initialize MMU for NetBSD

3. **Console Initialization**:
   - Set up console based on bootinfo
   - LCD framebuffer or SCIF serial
   - Display boot messages

4. **Device Probing**:
   - Probe platform-specific devices
   - Attach drivers based on platid
   - Initialize storage, network, etc.

## SH3 vs SH4 MMU Differences

The hpcsh port supports both SH3 and SH4 CPUs with different MMU handling:

### SH3 MMU (HardMMU Method)

**Supported CPUs**: SH7707, SH7709, SH7709A

1. **TLB Structure**:
   - 32-entry UTLB, 4-way set associative
   - Single unified TLB
   - 4-bit ASID (16 address spaces)

2. **Page Sizes**:
   - 1 KB and 4 KB pages supported
   - NetBSD uses 4 KB pages (PAGE_SIZE = 4096)
   - SH3_PAGE_SIZE defined in hpcboot

3. **MMU Access from hpcboot**:
   - Direct TLB manipulation via memory-mapped registers
   - Address Array: 0xf2000000 - 0xf2ffffff
   - Data Array: 0xf3000000 - 0xf3ffffff
   - Control registers: PTEH, PTEL, TTB, TEA, MMUCR

4. **HardMMU Implementation** (`sh_mmu.cpp`):
   - Reads/writes TLB entries directly
   - Sets up temporary mappings for kernel
   - Bypasses Windows CE page tables
   - Required for memory manager when LockPages unavailable

### SH4 MMU (Limited Support)

**Supported CPUs**: SH7750

1. **TLB Structure**:
   - 64-entry UTLB, 4-way set associative
   - 4-entry ITLB (instruction TLB)
   - 8-bit ASID (256 address spaces)

2. **Page Sizes**:
   - 1 KB, 4 KB, 64 KB, 1 MB pages supported
   - NetBSD uses 4 KB pages
   - SH4_PAGE_SIZE defined in hpcboot

3. **hpcboot Limitations**:
   - No SH4 HardMMU implementation in hpcboot
   - Must use LockPages memory manager
   - Relies on Windows CE for memory protection
   - Direct MMU manipulation more complex on SH4

4. **SH4-Specific Features**:
   - PTEA register (Page Table Entry Assistance)
   - Wired TLB entries (MMUCR.URB)
   - Store queues
   - Enhanced cache control

### Comparison for hpcboot

| Feature | SH3 (hpcsh) | SH4 (hpcsh) |
|---------|-------------|-------------|
| CPU Examples | SH7707, SH7709A | SH7750 |
| Typical Clock | 60-133 MHz | 166-200 MHz |
| Memory Manager | LockPages or HardMMU | LockPages only |
| MMU Manipulation | Full support | Limited |
| Page Table Access | Direct | Via LockPages |
| Cache Size | 4KB/8KB I/D | 8KB/16KB I/D |
| FPU | No | Yes |

## Memory Map

### Generic H/PC Memory Layout

Windows CE uses specific memory regions:

```
0x00000000 - 0x01ffffff : ROM (Windows CE, varies by device)
0x04000000 - 0x047fffff : Device registers (platform-specific)
0x0c000000 - 0x0dffffff : RAM (8-32 MB, device-dependent)
```

### SH3/SH4 Virtual Address Space

```
P0: 0x00000000 - 0x7fffffff : User space (2 GB, TLB-translated)
P1: 0x80000000 - 0x9fffffff : Kernel cached (512 MB, direct physical)
P2: 0xa0000000 - 0xbfffffff : Kernel uncached (512 MB, direct physical)
P3: 0xc0000000 - 0xdfffffff : Kernel virtual (512 MB, TLB-translated)
P4: 0xe0000000 - 0xffffffff : Control registers (512 MB)
```

### NetBSD Memory Layout

```
0x00000000 - 0x7ffff000 : User virtual memory (VM_MAXUSER_ADDRESS)
0x80000000 - 0x9fffffff : P1 cached kernel memory (direct-mapped)
0xa0000000 - 0xbfffffff : P2 uncached kernel memory
0xc0000000 - 0xdfffffff : P3 kernel virtual memory (TLB-mapped)
0xe0000000 - 0xffffffff : P4 memory-mapped I/O
```

### Typical H/PC Device Memory Map

```
Physical Memory:
0x00000000 - 0x01ffffff : ROM (32 MB, Windows CE Flash)
0x0c000000 - 0x0cffffff : RAM (16 MB typical)
0x0c000000 - 0x0c00ffff : Reserved for Windows CE
0x0c010000 - 0x0cffffff : Available for NetBSD

Virtual Memory (P1 mapping):
0x8c000000 : RAM base via P1
0x8c001000 : NetBSD kernel text (DEFTEXTADDR)
0x8c0xxxxx : Kernel data, BSS
0x8cxxxxxx : Kernel heap
```

### Device-Specific Registers

```
SH3/SH4 On-Chip Peripherals (P4):
0xfffe0000 - 0xfffeffff : Peripheral module space
0xffff0000 - 0xffff00ff : Cache control
0xffff0100 - 0xffff01ff : MMU control (TLB, PTEH, PTEL, etc.)
0xffff8000 - 0xffffbfff : Interrupt controller
0xffffc000 - 0xffffc0ff : Timer unit
0xffffc800 - 0xffffc8ff : SCIF (serial)
```

### CompactFlash / PC Card Memory Windows

```
0x10000000 - 0x13ffffff : PC Card memory space (varies by device)
0x14000000 - 0x17ffffff : PC Card attribute space
0x18000000 - 0x1bffffff : PC Card I/O space
```

## Build and Installation

### Building hpcboot

hpcboot is a Windows CE application that must be built with Microsoft development tools:

**Requirements**:
- eMbedded Visual C++ 3.0 (eVC3) or later
- Windows CE SDK for H/PC Pro 2.11
- Windows development host

**Build Process**:
```bash
# On NetBSD host (preparation)
cd /usr/src/sys/arch/hpc/stand/hpcboot
make evc3              # Generate project files for eVC3

# On Windows host (actual build)
# Open hpc_stand.vcw in eMbedded Visual C++
# Select target platform (SH3 or SH4)
# Build solution

# Result: binary/SH3/hpcboot.exe or binary/SH4/hpcboot.exe
```

**Pre-built Binaries**:

Pre-built hpcboot.exe binaries are available in the source tree:
```
/usr/src/sys/arch/hpc/stand/binary/SH3/hpcboot.exe
/usr/src/sys/arch/hpc/stand/binary/SH4/hpcboot.exe
```

These are uuencoded in the repository and can be extracted:
```bash
cd /usr/src/sys/arch/hpc/stand
make all               # Uudecode all binaries
```

### Building NetBSD Kernel

```bash
# Build toolchain
cd /usr/src
./build.sh -U -m hpcsh tools

# Build kernel
./build.sh -U -m hpcsh kernel=GENERIC

# Result: /usr/obj/sys/arch/hpcsh/compile/GENERIC/netbsd
```

### Kernel Configuration

Default configuration: `/usr/src/sys/arch/hpcsh/conf/GENERIC`

Key options:
```
machine hpcsh sh3
include "arch/sh3/conf/std.sh3el"

options     IOM_RAM_BEGIN=0x0c000000
makeoptions DEFTEXTADDR="0x8c001000"
makeoptions ENDIAN="-EL"              # Little-endian

# Typical SH3 options
options     SH3
options     SH7709A                   # Or SH7707, SH7709

# Typical SH4 options
#options    SH4
#options    SH7750

# Console options
options     SCIFCONSOLE               # Serial console
#options    PFCKBD                    # Keyboard support
```

### Installation Steps

#### 1. Prepare the Device

1. **Windows CE Requirements**:
   - H/PC Pro device with Windows CE 2.11 or later
   - CompactFlash or SD card for storage
   - Serial cable (optional, for serial console)

2. **Format Storage Card**:
   - Insert CompactFlash/SD card into device
   - Format as FAT16 or FAT32 from Windows CE
   - Windows CE will mount it as "Storage Card"

#### 2. Transfer Files to Device

**From Windows Host**:
```
1. Connect H/PC to Windows via ActiveSync
2. Copy hpcboot.exe to device
   - Suggested location: \My Documents\hpcboot.exe
3. Copy netbsd kernel to storage card
   - Copy to: \Storage Card\netbsd
```

**From NetBSD Host via Serial**:
```bash
# Use Kermit or similar
# Transfer files via serial port
# Slower but works without ActiveSync
```

#### 3. Running hpcboot

1. **On Device**:
   - Navigate to hpcboot.exe in File Explorer
   - Tap to launch hpcboot application

2. **hpcboot GUI**:
   - Select kernel file: `\Storage Card\netbsd`
   - Choose console: LCD or Serial
   - Set boot options (if needed)
   - Tap "Boot" button

3. **Boot Process**:
   - hpcboot loads kernel
   - Screen shows "Starting NetBSD..."
   - System transitions to NetBSD
   - Windows CE is suspended

#### 4. Installation from NetBSD

Once NetBSD boots:

```bash
# If booting from ramdisk kernel
sysinst                # Run installer

# Manual installation
# Mount storage device
mount /dev/wd0a /mnt

# Extract sets
cd /mnt
ftp ftp.netbsd.org
# ... download and extract distribution sets
```

### Creating Multi-Boot Setup

You can keep both Windows CE and NetBSD:

1. **Partition Storage**:
   - Partition 1: FAT (for Windows CE and kernel)
   - Partition 2: FFSv2 (for NetBSD root)

2. **Boot Configuration**:
   - Keep hpcboot.exe and kernel on FAT partition
   - Run hpcboot from Windows CE when you want NetBSD
   - Reset device to return to Windows CE

## Debugging

### Serial Console

Most H/PC devices have an accessible serial port:

**Hardware**:
- SCIF serial port (varies by device)
- Custom serial cable often required
- Some devices use non-standard connectors

**Configuration**:
```
Speed: 115200 bps (typical)
Data bits: 8
Parity: None
Stop bits: 1
Flow control: None
```

**Kernel Configuration**:
```
options     SCIFCONSOLE
options     CONSPEED=115200
options     "SCIFCN_SPEED=115200"
```

### hpcboot Debugging

Enable debug output in hpcboot:

**Debug Options**:
- Architecture Debug: Shows CPU detection, MMU operations
- Memory Manager Debug: Shows memory allocation, page locking
- Verbose output in hpcboot GUI

**Debug Build**:
- Build debug version in eVC
- Adds extensive logging
- Helps diagnose loading issues

### DDB (In-Kernel Debugger)

Enable DDB:
```
options     DDB
options     DDB_HISTORY_SIZE=512
makeoptions COPY_SYMTAB=1
```

**Accessing DDB**:
- On LCD console: Keyboard combination (device-dependent)
- On serial console: Send break or Ctrl+Alt+Esc
- Automatic on kernel panic

**Common DDB Commands**:
```
trace              # Stack backtrace
ps                 # Process list
show registers     # CPU registers
machine tlb        # Show TLB entries
machine platid     # Show platform ID
```

### Common Issues

1. **hpcboot Won't Start**:
   - Verify Windows CE version (need Pro 2.11+)
   - Check if correct architecture (SH3 vs SH4)
   - Ensure device has sufficient RAM
   - Try re-downloading hpcboot.exe

2. **Kernel Won't Load**:
   - Verify kernel file path
   - Check file is not corrupted
   - Ensure kernel built for correct CPU
   - Try different memory manager option

3. **Boot Hangs After "Starting NetBSD"**:
   - Check serial console for error messages
   - Verify memory configuration
   - Try serial console instead of LCD
   - Enable verbose boot in kernel

4. **Device-Specific Problems**:
   - Some devices have hardware quirks
   - Check NetBSD port page for device notes
   - Try different kernel configurations
   - Search mailing list archives

5. **Memory Manager Issues**:
   - LockPages API may not be available
   - HardMMU only works on SH3
   - Try different memory manager in hpcboot
   - Some devices need specific workarounds

### Debug Kernel Build

Build kernel with debug symbols:
```bash
./build.sh -U -m hpcsh \
    -V DBG="-g" \
    -V COPY_SYMTAB=1 \
    kernel=GENERIC
```

### Remote Debugging

KGDB not commonly used on H/PC devices, but possible:
```
options     KGDB
options     "KGDB_DEVNAME=\"scif\""
options     "KGDB_DEVRATE=115200"
```

## Device-Specific Notes

### HP Jornada 680/690

- CPU: SH7709A @ 133 MHz
- RAM: 16-32 MB
- Display: 640x240 LCD
- Serial: Custom cable required
- Notes: Well-supported, good performance

### HP Jornada 820/728

- CPU: SH7750 @ 166 MHz (SH4)
- RAM: 32 MB
- Display: 800x600 LCD
- Serial: Standard serial port
- Notes: Requires LockPages memory manager

### SHARP Telios HC-AJ Series

- CPU: SH7707 @ 60 MHz
- RAM: 8-16 MB
- Display: 640x240 LCD
- Notes: Slower but functional

### Casio Cassiopeia

- CPU: Various SH3
- RAM: 16-32 MB
- Notes: Multiple models, compatibility varies

## References

- NetBSD/hpcsh Homepage: https://www.netbsd.org/ports/hpcsh/
- hpcboot Documentation: `/usr/src/sys/arch/hpc/stand/hpcboot/`
- SH3 CPU Manual (Hitachi SH7707/SH7709)
- SH4 CPU Manual (Hitachi SH7750)
- Windows CE H/PC Pro Documentation (Microsoft)
- NetBSD Source: `/usr/src/sys/arch/hpcsh/`
- Common SH Code: `/usr/src/sys/arch/sh3/`

## Technical Contacts

For questions about NetBSD/hpcsh:
- NetBSD Port Maintainers: port-hpcsh@netbsd.org
- H/PC Ports Discussion: port-hpc@netbsd.org
- General NetBSD Questions: netbsd-help@netbsd.org

## Revision History

- Initial documentation based on NetBSD source analysis
- hpcboot architecture from `/sys/arch/hpc/stand/hpcboot/`
- Boot process derived from hpcboot C++ source code
- Memory maps from kernel configuration and source
