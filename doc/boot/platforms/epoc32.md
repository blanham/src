# NetBSD/epoc32 Boot Process Documentation

## Platform Overview

NetBSD/epoc32 supports 32-bit EPOC OS machines, specifically:
- Psion Series 5 (ARM710 @ 18.432 MHz)
- Psion Series 5mx (ARM710 @ 36 MHz)
- Psion Series 7 (ARM710 @ 36/74 MHz)
- Psion Revo/Revo Plus (ARM710 @ 36 MHz)
- Geofox One (Series 5 compatible)
- Osaris (Series 5 compatible)

These are pocket-sized handheld computers running Psion's EPOC OS (later Symbian OS). They feature ARM7 processors, built-in keyboards, touch screens, and CF/PCMCIA expansion.

## Boot Method

### EPOC Application Bootloader

NetBSD/epoc32 uses **e32boot**, a native EPOC application written in C++ that:
1. Runs as a standard EPOC application
2. Loads the NetBSD kernel from EPOC filesystem
3. Uses an EPOC Logical Device Driver (LDD) to access hardware
4. Transfers control to the NetBSD kernel

The bootloader consists of two components:
- **User-space application** (e32boot.exe) - File I/O and user interface
- **Kernel-space LDD** (e32boot.ldd) - Hardware access and kernel launch

Location: `/sys/arch/epoc32/stand/e32boot/`

### Components

**Application Layer (`exe/`):**
- `e32boot.cpp` - Main application, UI, and boot logic
- `netbsd.cpp` - Kernel file parsing and loading

**Driver Layer (`ldd/`):**
- `e32boot.cpp` - LDD implementation
- `cpu.cpp` - CPU-specific operations (MMU, cache)
- `epoc32.cpp` - EPOC kernel interfacing

**Include Files (`include/`):**
- `e32boot.h` - Application interface
- `netbsd.h` - NetBSD kernel structures
- `elf.h` - ELF parsing

## Boot Process Stages

### Stage 1: EPOC Application Start

**File: `stand/e32boot/exe/e32boot.cpp:E32Main()`**

The bootloader starts as a normal EPOC application:

1. **Heap Initialization**
   ```cpp
   __UHEAP_MARK;
   CTrapCleanup *cleanup = CTrapCleanup::New();
   ```

2. **Create Console**
   ```cpp
   console = Console::NewL(E32BootName,
                          TSize(KConsFullScreen, KConsFullScreen));
   ```

3. **Display Banner**
   ```
   >> e32boot, Revision X.XX
   ```

4. **Enter Main Loop**
   ```cpp
   TRAPD(error, E32BootL());
   ```

### Stage 2: System Information Gathering

**Function: `CreateBootInfo()`**

Queries EPOC for system configuration:

1. **Machine Information**
   ```cpp
   TMachineInfoV1Buf MachInfo;
   UserHal::MachineInfo(MachInfo);

   // Extract:
   // - iMachineName: "SERIES5 R1", "SERIES5mx", "SERIES7"
   // - iDisplaySizeInPixels: Screen resolution
   ```

2. **Memory Information**
   ```cpp
   TMemoryInfoV1Buf MemInfo;
   UserHal::MemoryInfo(MemInfo);

   // Extract:
   // - iTotalRamInBytes: Total system RAM
   ```

3. **Video Configuration**
   ```cpp
   TScreenInfoV01 screenInfo;
   UserSvr::ScreenInfo(sI);

   // Extract:
   // - iScreenAddress: Framebuffer physical address
   // - iScreenAddressValid: Whether address is valid
   ```

### Stage 3: Boot Information Structure Creation

Creates bootinfo for kernel:

**Structure Layout:**
```c
struct btinfo_model {
    int len;
    int type;              // BTINFO_MODEL
    char model[16];        // "SERIES5 R1", etc.
};

struct btinfo_video {
    int len;
    int type;              // BTINFO_VIDEO
    int width;             // Pixels
    int height;            // Pixels
};

struct btinfo_memory {
    int len;
    int type;              // BTINFO_MEMORY
    TUint address;         // Physical address
    TUint size;            // Size in bytes
};

struct btinfo_bootargs {
    int len;
    int type;              // BTINFO_BOOTARGS
    char bootargs[...];    // Boot argument string
};
```

**Memory Map Table:**

The bootloader contains hardcoded memory maps for each model:

```cpp
// Series 5 (4MB): Eight 512KB banks
{ 0xc0000000, 512 }, { 0xc0100000, 512 },
{ 0xc0400000, 512 }, { 0xc0500000, 512 },
{ 0xc1000000, 512 }, { 0xc1100000, 512 },
{ 0xc1400000, 512 }, { 0xc1500000, 512 }

// Series 5mx Pro (32MB): Four 8MB banks
{ 0xc0000000, 8192 }, { 0xc1000000, 8192 },
{ 0xd0000000, 8192 }, { 0xd1000000, 8192 }

// Series 7 (32MB): Two 16MB banks
{ 0xc0000000, 16384 }, { 0xc8000000, 16384 }
```

Memory detection algorithm:
1. Match model name from EPOC
2. Match screen resolution
3. Match total memory size
4. Select corresponding memory map

### Stage 4: Kernel File Selection

**Function: `LoadNetBSDL()`**

Interactive kernel selection:

1. **Display Prompt**
   ```
   Boot: [C:\netbsd]:
   ```

2. **User Input**
   - Default: `C:\netbsd`
   - Custom path supported: `D:\kernels\netbsd.test`
   - Boot arguments: `-s`, `-a`, `-d`, `-v`
   - Escape to cancel boot

3. **Parse Input**
   ```cpp
   // Format: [path] [args]
   // Example: "C:\netbsd -s"

   if (input[0] == '-')
       args = input;  // Just arguments
   else
       filename = input;  // Path and/or arguments
   ```

### Stage 5: Kernel Loading

**Class: `NetBSD`**

Loads kernel from EPOC filesystem:

1. **Open Kernel File**
   ```cpp
   NetBSD::New(filename, args)
   // Opens file using EPOC file APIs
   ```

2. **Parse ELF Header**
   ```cpp
   ParseHeader()
   // Reads ELF header
   // Validates magic number
   // Extracts program headers
   ```

3. **Load Kernel Segments**
   - Allocates buffer for kernel image
   - Reads ELF segments into memory
   - Maintains load addresses
   - Stores entry point

### Stage 6: LDD Loading

**Model-Specific LDD Selection:**

```cpp
if (model == "SERIES5 R1")
    ldd = "e32boot-s5.ldd";
else if (model == "SERIES5mx")
    ldd = "e32boot-s5mx.ldd";
// else if (model == "SERIES7")
//     ldd = "e32boot-s7.ldd";  // Not yet implemented
```

Load logical device driver:
```cpp
err = User::LoadLogicalDevice(ldd);
```

The LDD provides kernel-mode access to:
- MMU configuration
- Cache control
- Physical memory
- Direct hardware access

### Stage 7: Channel Creation

Create communication channel with LDD:

```cpp
E32BootLogicalChannel *channel = new E32BootLogicalChannel;
err = channel->DoCreate(NULL, KNullUnit, NULL, NULL);
```

Configure safe address (framebuffer):
```cpp
channel->DoControl(KE32BootSetSafeAddress, safeAddress);
```

### Stage 8: Kernel Transfer

**LDD Function: `BootNetBSD()`**

Final boot operation in kernel mode:

1. **Disable Interrupts**
   ```cpp
   // EPOC kernel call to disable interrupts
   ```

2. **Setup MMU**
   - Create new page tables for NetBSD
   - Map kernel at appropriate virtual address
   - Map physical memory
   - Identity map boot code region

3. **Cache Operations**
   ```cpp
   // Flush data cache
   // Invalidate instruction cache
   // Drain write buffer
   ```

4. **Transfer Control**
   ```cpp
   typedef void (*kernel_entry)(void *bootinfo);
   kernel_entry entry = (kernel_entry)kernel_entrypoint;
   entry(bootinfo);
   ```

Kernel entry:
- r0: Pointer to bootinfo structure
- PC: Kernel entry point
- Mode: SVC32 (Supervisor mode)
- MMU: Enabled with new page tables
- Interrupts: Disabled

## ARM MMU Setup Requirements

### ARM710 Processor

**Control Register (CP15 c1):**
```
Bit 0:  M - MMU enable
Bit 2:  C - Data cache enable
Bit 3:  W - Write buffer enable
Bit 4:  P - 32-bit exception handlers
Bit 5:  D - 32-bit addressing
Bit 7:  B - Big-endian (0 for little-endian)
Bit 8:  S - System protection
Bit 9:  R - ROM protection
Bit 12: I - Instruction cache enable
Bit 13: V - High vectors
```

**Boot Configuration:**
- MMU enabled
- Caches enabled
- Little-endian mode
- 32-bit addressing

### Page Table Setup

**L1 Table:**
- 16KB, 16KB aligned
- 4096 entries (1MB sections)
- Domain 0 for kernel

**Memory Mappings:**
```
0xC0000000+: Physical DRAM (identity mapped initially)
Kernel VA:   Varies by configuration
I/O Space:   Device registers
```

### Cache Management

ARM710 caches:
- 8KB instruction cache
- 8KB data cache (write-through)
- No write-back support

Operations:
```cpp
// Flush caches
MCR p15, 0, r0, c7, c7, 0     // Invalidate I&D cache
MCR p15, 0, r0, c7, c10, 4    // Drain write buffer (if present)
```

## Memory Map

### Physical Memory Layout

**Series 5 (4-8MB):**
```
0xC0000000 - 0xC07FFFFF : DRAM Bank 0 (512KB - 4MB)
0xC0800000 - 0xC0FFFFFF : DRAM Bank 1 (if present)
0xC1000000 - 0xC17FFFFF : DRAM Bank 2 (if present)
0xC1800000 - 0xC1FFFFFF : DRAM Bank 3 (if present)
0xD0000000 - 0xDFFFFFFF : Extended DRAM (Series 5 8MB)
```

**Series 5mx (8-32MB):**
```
0xC0000000 - 0xC07FFFFF : Revo/Revo+ (4MB each bank)
0xC0000000 - 0xC1FFFFFF : Series 5mx (16-32MB, multiple banks)
0xD0000000 - 0xD1FFFFFF : Extended DRAM (Series 5mx Pro 24-32MB)
```

**Series 7 (16-32MB):**
```
0xC0000000 - 0xC0FFFFFF : DRAM Bank 0 (16MB)
0xC8000000 - 0xC8FFFFFF : DRAM Bank 1 (16MB, if present)
```

### Device Space

```
0x00000000 - 0x0FFFFFFF : ROM (EPOC ROM)
0x40000000 - 0x4FFFFFFF : I/O Space
0x80000000 - 0x8FFFFFFF : LCD Controller
0x90000000 - 0x9FFFFFFF : System peripherals
```

### Memory Architecture

EPOC devices use non-contiguous memory banks due to:
- Historical memory controller design
- Chip-enable signal routing
- Expandability considerations

NetBSD kernel must handle discontiguous physical memory.

## Build and Installation

### Building e32boot

**Requirements:**
- Windows PC
- eMbedded Visual C++ 3.0 (or later)
- EPOC SDK for target device

**Build Process:**

1. **Generate Project Files**
   ```bash
   cd /sys/arch/epoc32/stand/e32boot
   make evc3    # For eMbedded Visual C++ 3.0
   ```

2. **Open in Visual Studio**
   ```
   Open hpc_stand.dsw
   Select target: ARM Release
   Build -> Build e32boot.exe
   Build -> Build e32boot-*.ldd
   ```

3. **Binaries Produced**
   ```
   binary/ARM/e32boot.exe
   binary/ARM/e32boot-s5.ldd
   binary/ARM/e32boot-s5mx.ldd
   ```

**Pre-built Binaries:**

Pre-compiled binaries available in:
```
/sys/arch/epoc32/stand/e32boot/binary/ARM/
```

Extract with:
```bash
cd binary/ARM
uudecode hpcboot.exe.uue
```

### Installation

1. **Copy to EPOC Device**
   ```
   Via CompactFlash card:
   - Copy e32boot.exe to C:\ or D:\
   - Copy e32boot-*.ldd to C:\System\Libs\

   Via Serial/USB connection:
   - Use EPOC file transfer software
   - Copy files to appropriate locations
   ```

2. **Copy NetBSD Kernel**
   ```
   - Copy compiled netbsd kernel to C:\netbsd
   - Or to CF card as D:\netbsd
   ```

3. **Create Kernel Shortcut** (optional)
   - Create EPOC shortcut to e32boot.exe
   - Set working directory
   - Add to startup folder for auto-boot

### Running e32boot

1. **From EPOC Shell:**
   - Navigate to e32boot.exe location
   - Tap to run
   - Follow boot prompts

2. **Command Line** (if available):
   ```
   e32boot C:\netbsd -s
   ```

3. **Auto-boot:**
   - Add e32boot to EPOC startup folder
   - Configure default kernel path
   - System boots automatically

## Debugging

### Application-Level Debugging

1. **Console Output**
   - All boot messages displayed on screen
   - Shows model detection
   - Displays memory configuration
   - Reports load progress

2. **Debug Messages**
   ```cpp
   console->Printf(_L("Model: %s\n"), model->model);
   console->Printf(_L("Memory: %d KB\n"), membytes / 1024);
   console->Printf(_L("Video: %d x %d\n"), width, height);
   ```

3. **Error Handling**
   ```cpp
   TRAP(err, netbsd = LoadNetBSDL());
   if (err != KErrNone) {
       console->Printf(_L("Load failed: %d\n"), err);
   }
   ```

### Common Issues

**Issue: "Load failed: -1"**
- Cause: Kernel file not found
- Solution: Verify path, check filesystem

**Issue: "Not Supported machine"**
- Cause: Unknown model or no LDD available
- Solution: Check model detection, verify LDD present

**Issue: LoadLogicalDevice failed**
- Cause: LDD file missing or wrong version
- Solution: Ensure correct LDD in C:\System\Libs\

**Issue: Screen goes blank after "Loaded"**
- Cause: Kernel panic or hardware mismatch
- Solution: Try serial console, verify kernel config

**Issue: "No BTINFO_MEMORY found"**
- Cause: Unknown memory configuration
- Solution: Add memory map to memmaps[] table

### Memory Detection Debugging

Verify memory map detection:
```cpp
printf("Model: %s\n", model->model);
printf("Resolution: %d x %d\n", video->width, video->height);
printf("Memory: %d KB\n", memsize / 1024);

// Check if configuration matched
for (i = 0; i < sizeof(memmaps) / sizeof(memmaps[0]); i++) {
    if (match found) {
        printf("Using memory map %d\n", i);
        // List memory blocks
    }
}
```

### LDD Debugging

Enable LDD debug output:
```cpp
// In e32boot.ldd
#define DEBUG_LDD
// Prints MMU setup, cache operations, etc.
```

### Serial Console

EPOC devices with serial port:
1. Connect serial cable
2. Configure for 9600 or 38400 baud
3. Capture kernel boot messages
4. Useful for kernel panic diagnosis

## Technical Notes

### EPOC OS Integration

e32boot must work within EPOC constraints:
- Runs in user mode initially
- Requires LDD for kernel-mode operations
- Uses EPOC file APIs for I/O
- Subject to EPOC memory management

**EPOC APIs Used:**
- `UserHal::MachineInfo()` - System information
- `UserHal::MemoryInfo()` - Memory info
- `UserSvr::ScreenInfo()` - Display info
- `User::LoadLogicalDevice()` - Load LDD
- `RLogicalChannel` - Driver communication

### LDD Architecture

The Logical Device Driver:
1. Loaded into EPOC kernel space
2. Provides DoControl() interface for commands
3. Executes privileged operations
4. Performs final boot transition

**LDD Commands:**
```cpp
KE32BootSetSafeAddress   // Protect framebuffer
KE32BootBootNetBSD       // Execute kernel
```

### Memory Bank Detection

Memory configuration detection is critical:
- EPOC doesn't provide complete physical memory map
- Bootloader uses lookup table based on:
  - Model name
  - Screen resolution
  - Total memory size
- If no match, uses conservative default (4MB at 0xC0000000)

**Adding New Configurations:**

To support new device:
1. Determine model name from EPOC
2. Measure screen resolution
3. Determine memory size
4. Find physical memory addresses (datasheet or experimentation)
5. Add entry to memmaps[] table

### Cache and MMU Transition

Critical sequence during boot:
1. Flush EPOC's page tables from cache
2. Disable MMU temporarily
3. Set new L1 page table base
4. Enable MMU with new mappings
5. Flush TLB
6. Enable caches
7. Jump to kernel

Timing is critical - code must execute from safe region.

### Discontiguous Memory Support

NetBSD kernel must support discontiguous memory:
```c
// Kernel config
options     MEMORY_DISK_IS_ROOT
options     MEMORY_DISK_SERVER=0
options     MEMORY_RBFLAGS=RB_SINGLE

# Memory configuration passed via bootinfo
# Kernel builds vm_physseg[] array
```

### Series 7 Status

Series 7 support is incomplete:
- LDD not yet implemented (e32boot-s7.ldd)
- Different processor (ARM9 vs ARM7)
- Different memory layout
- Requires additional development

## References

- `/sys/arch/epoc32/stand/e32boot/exe/e32boot.cpp` - Main bootloader
- `/sys/arch/epoc32/stand/e32boot/ldd/e32boot.cpp` - LDD implementation
- `/sys/arch/epoc32/stand/e32boot/ldd/cpu.cpp` - CPU operations
- `/sys/arch/epoc32/include/bootinfo.h` - Boot information structures
- Psion Series 5 Technical Reference
- ARM710T Technical Reference Manual
- Symbian OS Internals (for EPOC architecture)
