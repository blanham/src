# NetBSD/hpcarm Boot Process Documentation

## Platform Overview

NetBSD/hpcarm supports ARM-based Windows CE (H/PC) handheld devices including:
- HP Jornada 720/728 (StrongARM SA-1110 @ 206 MHz)
- Compaq iPAQ H36xx/H38xx series (StrongARM SA-1110)
- Sharp Telios HC-AJ1/AJ2/AJ3 (ARM710T)
- HP Jornada 820 (StrongARM SA-1100 @ 190 MHz)
- Various other Windows CE ARM devices

These are handheld personal computers designed to run Microsoft Windows CE (also called H/PC Pro). They feature color touchscreens, keyboards, PCMCIA expansion, and various ARM processors.

## Boot Method

### Windows CE Application Bootloader

NetBSD/hpcarm uses **hpcboot.exe**, a Windows CE application that runs under the native Windows CE environment. The bootloader is shared across multiple H/PC platforms (ARM, MIPS, SH3, SH4) with architecture-specific code.

**Key Characteristics:**
- Runs as native Windows CE application
- Written in C++ using Windows CE APIs
- Provides graphical user interface
- Loads kernel from Windows CE filesystem or storage card
- Uses Windows CE kernel mode for hardware access

Location: `/sys/arch/hpc/stand/hpcboot/` (shared)
Reference from: `/sys/arch/hpcarm/stand/README`

### Bootloader Architecture

**Components:**
```
hpcboot/
├── arm/           - ARM-specific code (CPU, MMU)
├── menu/          - User interface
├── mips/          - MIPS code (not used on hpcarm)
├── sh3/           - SH3 code (not used on hpcarm)
├── sh4/           - SH4 code (not used on hpcarm)
└── binary/        - Pre-compiled binaries
    └── ARM/
        └── hpcboot.exe
```

The bootloader provides:
- Graphical menu for kernel selection
- File browser for navigating storage
- Boot parameter configuration
- Platform and CPU detection
- Kernel loading and launching

## Boot Process Stages

### Stage 1: Windows CE Application Launch

**User starts hpcboot.exe:**

1. **Application Initialization**
   ```cpp
   WinMain(HINSTANCE hInstance, ...)
   // Standard Windows CE entry point
   ```

2. **Create User Interface**
   - Creates main window
   - Displays device information
   - Shows menu of boot options
   - Registers touch screen events

3. **Platform Detection**
   ```cpp
   HpcArm::detect()
   // Identifies specific device model
   // Detects processor type (SA-1100/1110, ARM710, etc.)
   // Reads Windows CE registry for device info
   ```

4. **Display Configuration**
   ```
   Device: HP Jornada 720
   CPU: StrongARM SA-1110
   Memory: 32 MB
   Display: 640x240 8bpp
   ```

### Stage 2: Kernel File Selection

**User Interface:**

1. **Kernel Selection Screen**
   ```
   [NetBSD Kernel]
   Path: \Storage Card\netbsd

   [Boot Options]
   [ ] Single User (-s)
   [ ] Verbose (-v)
   [ ] Ask for Root (-a)
   [ ] Debug (-d)

   [Boot]  [Configure]  [Exit]
   ```

2. **File Browser**
   - Navigate Windows CE filesystem
   - Access storage cards
   - Select kernel file
   - Display file information

3. **Common Locations**
   ```
   \Storage Card\netbsd         - CF/SD card
   \My Documents\netbsd         - Main storage
   \NFS Share\netbsd           - Network share
   \Windows\netbsd             - System storage
   ```

### Stage 3: Boot Preparation

**Function: `Boot::boot()`**

1. **Parse Boot Arguments**
   ```cpp
   int howto = 0;
   if (single_user_mode)
       howto |= RB_SINGLE;
   if (verbose)
       howto |= RB_VERBOSE;
   // etc.
   ```

2. **Allocate Boot Memory**
   ```cpp
   // Allocate memory for:
   // - Kernel image
   // - Page tables
   // - Boot information structure
   // - Kernel arguments
   ```

3. **Query System Information**
   ```cpp
   // From Windows CE:
   // - Physical memory size and layout
   // - Display framebuffer address
   // - Processor information
   // - Device-specific parameters
   ```

### Stage 4: Kernel Loading

**Kernel Load Process:**

1. **Open Kernel File**
   ```cpp
   HANDLE hFile = CreateFile(kernel_path, ...);
   // Uses Windows CE file APIs
   ```

2. **Parse ELF Header**
   ```cpp
   ReadFile(hFile, &ehdr, sizeof(ehdr), ...);
   // Validate ELF magic
   // Extract program headers
   // Determine load addresses
   ```

3. **Load Kernel Segments**
   ```cpp
   for each program header:
       ReadFile(hFile, load_address, size, ...);
       // Load text, data, etc.
   ```

4. **Symbol Table Loading**
   - Optionally load kernel symbols
   - Required for DDB (kernel debugger)
   - Stored in separate memory region

### Stage 5: Boot Information Structure

**Structure: `bootinfo`**

hpcarm uses platform ID (platid) system:

```cpp
struct hpcarm_bootinfo {
    // Platform identification
    platid_t platid;           // Device identification

    // Memory configuration
    struct {
        paddr_t start;         // Physical start address
        psize_t size;          // Size in bytes
    } memory[MEMORY_BANKS];

    // Display information
    paddr_t fb_addr;           // Framebuffer physical address
    int fb_width;              // Width in pixels
    int fb_height;             // Height in pixels
    int fb_bpp;                // Bits per pixel
    int fb_stride;             // Bytes per line

    // Boot arguments
    char bootargs[256];
};
```

**Platform ID (platid):**
```
Hierarchical device identification:
CPU_TYPE.VENDOR.SERIES.MODEL

Example: ARM.HP.JORNADA7XX.720
- ARM processor architecture
- HP vendor
- Jornada 7xx series
- Model 720
```

### Stage 6: Hardware Preparation

**ARM-Specific Operations:**

1. **Disable Windows CE Services**
   ```cpp
   // Stop Windows CE threads
   // Suspend system services
   // Prepare for transition
   ```

2. **Allocate Physical Memory**
   ```cpp
   VirtualAlloc(..., MEM_RESERVE | MEM_PHYSICAL, ...)
   // Reserve physical memory for kernel
   // Must be contiguous if possible
   ```

3. **Create Page Tables**
   ```cpp
   CreateInitialPageTables()
   // L1 table (16KB, 16KB aligned)
   // Map kernel at virtual address
   // Identity map boot transition code
   // Map devices and I/O space
   ```

### Stage 7: MMU and Cache Setup

**ARM MMU Configuration:**

1. **Build L1 Page Table**
   ```
   Virtual Address Space:
   0x00000000 - 0x1FFFFFFF : Identity mapped (for transition)
   0xC0000000 - 0xCFFFFFFF : Kernel space
   0xD0000000 - 0xFFFFFFFF : Device mappings
   ```

2. **Section Descriptors**
   - 1MB sections for large mappings
   - AP (Access Permission) = supervisor
   - Domain 0
   - Cacheable/Bufferable as appropriate

3. **Cache Configuration**
   ```cpp
   // Flush data cache
   // Invalidate instruction cache
   // Drain write buffer
   // Prepare for MMU switch
   ```

### Stage 8: Kernel Transfer

**Final Boot Sequence:**

1. **Switch to Privileged Mode**
   ```cpp
   // Windows CE kernel-mode transition
   // Call into kernel-mode component
   // Or use documented API
   ```

2. **Disable Interrupts**
   ```
   MRS r0, CPSR
   ORR r0, r0, #(I32_bit | F32_bit)
   MSR CPSR_c, r0
   ```

3. **Switch to SVC Mode**
   ```
   MRS r0, CPSR
   BIC r0, r0, #PSR_MODE
   ORR r0, r0, #PSR_SVC32_MODE
   MSR CPSR_c, r0
   ```

4. **Setup New MMU**
   ```
   // Load new L1 page table base
   MCR p15, 0, r0, c2, c0, 0

   // Flush TLB
   MCR p15, 0, r0, c8, c7, 0

   // Enable MMU
   MRC p15, 0, r0, c1, c0, 0
   ORR r0, r0, #CPU_CONTROL_MMU_ENABLE
   MCR p15, 0, r0, c1, c0, 0
   ```

5. **Jump to Kernel**
   ```cpp
   typedef void (*kernel_entry_t)(void *bootinfo);
   kernel_entry_t entry = (kernel_entry_t)kernel_entry_point;
   entry(bootinfo);
   ```

Kernel entry state:
- r0: Pointer to bootinfo structure
- Mode: SVC32
- MMU: Enabled
- Caches: Enabled
- Interrupts: Disabled

## ARM MMU Setup Requirements

### StrongARM SA-1100/SA-1110

**Control Register (CP15 c1):**
```
Bit 0:  M - MMU enable
Bit 2:  C - Data cache enable
Bit 3:  W - Write buffer enable
Bit 7:  B - Big-endian
Bit 9:  R - ROM protection
Bit 11: Z - Branch prediction
Bit 12: I - Instruction cache enable
Bit 13: V - High vectors (0xFFFF0000)
Bit 16: RR - Round-robin cache replacement
```

**Cache Specifications:**
- 16KB instruction cache (32-way)
- 16KB data cache (32-way, write-back)
- 8-entry write buffer
- No L2 cache

**Cache Operations:**
```
Clean D-cache:       MCR p15, 0, rd, c7, c10, 4
Invalidate I-cache:  MCR p15, 0, rd, c7, c5, 0
Invalidate D-cache:  MCR p15, 0, rd, c7, c6, 0
Flush I&D cache:     MCR p15, 0, rd, c7, c7, 0
Drain WB:            MCR p15, 0, rd, c7, c10, 4
```

### ARM710T (Sharp Telios)

**Specifications:**
- 8KB instruction cache
- 8KB data cache (write-through)
- Simpler cache management
- ARMv4T architecture
- Thumb support

## Memory Map

### Physical Memory Layout

**HP Jornada 720/728:**
```
0x00000000 - 0x00FFFFFF : Boot ROM (16MB)
0xC0000000 - 0xC1FFFFFF : DRAM (32MB standard)
0x40000000 - 0x4FFFFFFF : SA-1110 peripherals
0x48000000 - 0x48FFFFFF : LCD controller
0x49000000 - 0x49FFFFFF : System controller
```

**Compaq iPAQ H3600:**
```
0x00000000 - 0x00FFFFFF : Boot ROM
0xC0000000 - 0xC1FFFFFF : DRAM (32MB standard)
0xC2000000 - 0xC3FFFFFF : Extended DRAM (if 64MB)
0x10000000 - 0x1FFFFFFF : SA-1110 peripherals
0x49000000 - 0x49FFFFFF : EGPIO registers
```

**Sharp Telios:**
```
0x00000000 - 0x00FFFFFF : ROM
0x0C000000 - 0x0DFFFFFF : DRAM
0x10000000 - 0x1FFFFFFF : Peripherals
```

### Virtual Memory Layout

**Kernel Address Space:**
```
0xC0000000 - 0xC0FFFFFF : Kernel text/data
0xC1000000 - 0xCFFFFFFF : Kernel VM
0xD0000000 - 0xDFFFFFFF : Device mappings
0xE0000000 - 0xEFFFFFFF : Additional device space
0xF0000000 - 0xFFFFFFFF : High mappings
```

## Build and Installation

### Building hpcboot.exe

**Requirements:**
- Windows PC or cross-compilation environment
- Microsoft eMbedded Visual C++ 3.0 or 4.0
- Windows CE SDK for target platform

**Build Steps:**

1. **Generate Project Files**
   ```bash
   cd /sys/arch/hpc/stand/hpcboot
   make evc3    # For eVC++ 3.0
   # or
   make evc4    # For eVC++ 4.0
   ```

2. **Open in Visual C++**
   ```
   File -> Open Workspace
   Select: hpc_stand.dsw or hpc_stand.vcw
   ```

3. **Select Configuration**
   ```
   Build -> Set Active Configuration
   Select: ARM Release
   ```

4. **Build**
   ```
   Build -> Build hpcboot.exe
   Output: compile/ARM/Release/hpcboot.exe
   ```

**Pre-built Binaries:**

Available in `/sys/arch/hpc/stand/hpcboot/binary/ARM/`:
```bash
uudecode hpcboot.exe.uue
```

### Installation on Device

1. **Copy to Device**
   ```
   Via ActiveSync/RAPI:
   - Connect device to PC
   - Copy hpcboot.exe to device storage

   Via Storage Card:
   - Copy hpcboot.exe to CF/SD card
   - Insert card into device
   ```

2. **Recommended Location**
   ```
   \My Documents\hpcboot.exe
   or
   \Storage Card\hpcboot.exe
   ```

3. **Create Shortcut** (optional)
   - Create shortcut in Windows CE Start Menu
   - Add to Startup folder for auto-boot

### Kernel Installation

1. **Build Kernel**
   ```bash
   cd /sys/arch/hpcarm/conf
   config GENERIC
   cd ../compile/GENERIC
   make depend && make
   ```

2. **Copy to Device**
   ```
   # Via card:
   cp netbsd /mnt/card/

   # Via ActiveSync:
   pcp netbsd ":\Storage Card\netbsd"
   ```

3. **Verify File**
   - Check file size (should be several MB)
   - Verify it's ELF format
   - Ensure storage has sufficient space

## Debugging

### Boot Loader Debugging

1. **Debug Output**
   - hpcboot.exe displays messages in GUI
   - Shows kernel load progress
   - Reports any errors

2. **Common Messages**
   ```
   "Loading kernel..."
   "Creating page tables..."
   "Setting up MMU..."
   "Jumping to kernel..."
   ```

3. **Enable Verbose Mode**
   - Check "Verbose" option in GUI
   - Shows detailed boot process
   - Useful for troubleshooting

### Common Issues

**Issue: "Cannot open kernel file"**
- Cause: Wrong path or file not found
- Solution: Verify file path, check storage card

**Issue: "Not a valid ELF file"**
- Cause: Corrupted or wrong kernel file
- Solution: Re-copy kernel, verify file integrity

**Issue: "Insufficient memory"**
- Cause: Not enough RAM for kernel
- Solution: Close Windows CE apps, use smaller kernel

**Issue: Device resets after "Jumping to kernel"**
- Cause: MMU setup error or kernel mismatch
- Solution: Verify kernel is for hpcarm, check MMU code

**Issue: Screen goes blank immediately**
- Cause: Kernel panic or framebuffer issue
- Solution: Try serial console if available

### Serial Console Debugging

**Devices with Serial Port:**

1. **Connect Serial Cable**
   - Use null modem cable
   - Connect to PC COM port
   - Standard serial parameters: 115200 8N1

2. **Configure hpcboot**
   - Select "Serial Console" option
   - Set baud rate (default 115200)

3. **Capture Output**
   ```bash
   # On PC:
   screen /dev/ttyS0 115200
   # or
   minicom -D /dev/ttyS0 -b 115200
   ```

4. **View Boot Messages**
   - Kernel boot output sent to serial
   - Panic messages visible
   - Useful for debugging kernel issues

### Platform Detection Issues

**Verify Platform ID:**

hpcboot should display:
```
Platform: ARM.HP.JORNADA7XX.720
CPU: StrongARM SA-1110 @ 206 MHz
```

If detection fails:
- Check Windows CE registry
- Verify device model
- May need platform-specific kernel

### Memory Configuration Debug

**Check Memory Layout:**
```
DRAM: 0xC0000000 - 0xC1FFFFFF (32 MB)
Framebuffer: 0xC8000000 (640x240x8)
```

Issues:
- Discontiguous memory not handled
- Insufficient kernel memory
- Framebuffer overlap with kernel

## Technical Notes

### Windows CE Integration

hpcboot must operate within Windows CE:
- Uses Win32 API subset
- Limited kernel-mode access
- Subject to Windows CE memory management
- Must coordinate with CE services

**Windows CE APIs Used:**
- `CreateFile()` - File access
- `ReadFile()` - Read kernel
- `VirtualAlloc()` - Memory allocation
- `SystemParametersInfo()` - Device info
- Registry APIs - Configuration

### Platform ID System

**platid Structure:**
```cpp
struct platid {
    uint32_t dw;  // Encoded as hierarchy
};

// Decoding:
CPU_TYPE   = (platid.dw >> 24) & 0xFF
VENDOR     = (platid.dw >> 16) & 0xFF
SERIES     = (platid.dw >> 8) & 0xFF
MODEL      = (platid.dw) & 0xFF
```

Used for:
- Device-specific initialization
- Driver selection
- Hardware quirk handling

### Cache Coherency

Critical during boot:
1. Kernel loaded via normal memory accesses
2. Data cache may contain stale instructions
3. Must clean D-cache before executing kernel
4. Must invalidate I-cache after loading
5. TLB must be flushed on MMU switch

### Framebuffer Handling

Important considerations:
- Framebuffer location varies by device
- May overlap with kernel memory
- Must be preserved or remapped
- Kernel console may reuse framebuffer

### Multi-Architecture Support

hpcboot supports ARM, MIPS, SH3, SH4:
- Shared UI and file handling code
- Architecture-specific boot code
- CPU-specific MMU and cache handling
- Platform detection at runtime

## Supported Devices

### HP Jornada Series
- **Jornada 720**: SA-1110, 32MB, 640x240
- **Jornada 728**: SA-1110, 32MB, 640x240 (UK version)
- **Jornada 820**: SA-1100, 32MB, 800x600

### Compaq/HP iPAQ Series
- **iPAQ H3600**: SA-1110, 32MB, 240x320
- **iPAQ H3700**: SA-1110, 64MB, 240x320
- **iPAQ H3800**: SA-1110, 64MB, 240x320

### Sharp Telios
- **HC-AJ1**: ARM710T, 8MB, 640x240
- **HC-AJ2**: ARM710T, 16MB, 640x240
- **HC-AJ3**: ARM710T, 32MB, 640x240

### Others
Various other Windows CE H/PC devices with ARM processors.

## References

- `/sys/arch/hpc/stand/hpcboot/` - Shared bootloader source
- `/sys/arch/hpcarm/stand/README` - Build instructions
- `/sys/arch/hpcarm/hpcarm/` - Platform-specific code
- Intel StrongARM SA-1110 Developer's Manual
- ARM710T Technical Reference Manual
- Microsoft Windows CE documentation
