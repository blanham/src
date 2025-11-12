# NetBSD/cats Boot Process Documentation

## Platform Overview

NetBSD/cats supports the Chalice Technologies CATS (Chalice ARM Test System) motherboard, a development and workstation platform featuring:
- Intel StrongARM SA-110 processor (200-233 MHz)
- Digital/Intel 21285 (Footbridge) core logic chipset
- PCI bus support
- ISA bus via PCI-ISA bridge
- VGA graphics with PC-style console
- Standard PC peripherals (keyboard, mouse, serial, parallel)
- IDE and SCSI storage options

The CATS board was designed as a development platform and workstation, providing a PC-compatible environment for ARM processors.

## Boot Method

### Cyclone Firmware Bootloader

NetBSD/cats boots using the **Cyclone firmware**, a proprietary boot firmware built into the system ROM. Unlike many other ARM platforms, cats has **no standalone NetBSD bootloader** in `/sys/arch/cats/stand/`. Instead, it relies entirely on the Cyclone firmware to:

1. Initialize hardware
2. Load the kernel from disk
3. Set up initial page tables
4. Pass control to the NetBSD kernel

The Cyclone firmware provides a boot interface similar to PC BIOS but specifically designed for ARM-based systems.

### Firmware Capabilities

Cyclone firmware provides:
- Hardware initialization and POST
- Basic boot menu and configuration
- Disk boot support (IDE, SCSI)
- Network boot support (PXE-like)
- Initial MMU setup
- Debug console access

## Boot Process Stages

### Stage 1: Firmware Power-On Self Test

When powered on, Cyclone firmware:

1. **Hardware Initialization**
   - Initializes SA-110 processor
   - Configures DC21285 Footbridge chipset
   - Sets up SDRAM controller
   - Detects and sizes memory
   - Initializes PCI bus
   - Configures ISA bridge

2. **Device Enumeration**
   - Scans PCI bus
   - Identifies boot devices
   - Initializes VGA adapter
   - Sets up PC keyboard controller

3. **Console Setup**
   - Initializes VGA display (default console)
   - Sets up serial console (optional)
   - Displays firmware banner and version

### Stage 2: Boot Device Selection

Firmware boot sequence:

1. **Check Boot Configuration**
   - Read stored boot preferences
   - Determine boot device order
   - Check for network boot request

2. **Interactive Boot Menu** (if enabled)
   ```
   Cyclone Boot Menu:
   1. Boot from IDE
   2. Boot from SCSI
   3. Boot from Network
   4. Enter firmware setup
   5. Drop to firmware console
   ```

3. **Automatic Boot** (default)
   - Waits for interrupt (typically 5 seconds)
   - Proceeds with default boot device

### Stage 3: Kernel Loading

Cyclone firmware loads the NetBSD kernel:

1. **Locate Boot Partition**
   - Reads disk partition table
   - Locates NetBSD partition (typically type 0xA9)
   - Mounts filesystem (FFS or other supported)

2. **Load Kernel Image**
   - Reads `/netbsd` (or configured kernel path)
   - Loads to physical memory starting at 0x00000000
   - Parses ELF header
   - Loads program segments

3. **Parse Boot Arguments**
   - Reads firmware boot parameters
   - Constructs kernel argument string
   - Sets boot flags (single-user, verbose, etc.)

### Stage 4: Initial Page Table Setup

Firmware creates bootstrap page tables:

1. **L1 Page Table Creation**
   - Allocates 16KB for L1 table
   - Creates section mappings (1MB granularity)
   - Maps kernel to virtual address space

2. **Initial Memory Mappings**
   ```
   0x00000000 - 0x0FFFFFFF : Physical DRAM (256MB max)
   0xF0000000 - 0xF0FFFFFF : Kernel text/data (16MB)
   0xF1000000 - 0xFCFFFFFF : Kernel VM space (192MB)
   0xFD000000 - 0xFD0FFFFF : DC21285 CSR space
   0xFE000000 - 0xFE0FFFFF : DC21285 cache flush
   0xFF000000 - 0xFF0FFFFF : PCI I/O space
   ```

3. **Device Mappings**
   - Maps DC21285 control registers
   - Maps PCI configuration space
   - Maps ISA I/O space

### Stage 5: Boot Information Structure

Firmware prepares boot information:

**Structure: `struct ebsaboot`**
```c
struct ebsaboot {
    uint32_t    bt_magic;       // 0x43415453 ('CATS')
    uint32_t    bt_vargp;       // Virtual addr of arg page
    uint32_t    bt_pargp;       // Physical addr of arg page
    const char *bt_args;        // Kernel args string pointer
    pd_entry_t *bt_l1;          // Active L1 page table
    uint32_t    bt_memstart;    // Start of physical memory
    uint32_t    bt_memend;      // End of physical memory
    uint32_t    bt_memavail;    // Start of available memory
    uint32_t    bt_fclk;        // Footbridge clock frequency
    uint32_t    bt_pciclk;      // PCI bus frequency
    uint32_t    bt_vers;        // Structure version
    uint32_t    bt_features;    // Feature mask
};
```

Information includes:
- Memory layout (start, end, available)
- L1 page table location
- Boot arguments string
- Clock frequencies (DC21285 and PCI)
- Firmware version
- Feature flags

### Stage 6: Kernel Entry

Firmware transfers control to kernel:

1. **Processor State Setup**
   ```
   Mode:      SVC32 (Supervisor mode)
   MMU:       Enabled with firmware page tables
   Caches:    Enabled (I-cache and D-cache)
   Interrupts: Disabled
   ```

2. **Register State**
   ```
   r0:        Pointer to struct ebsaboot
   r1-r12:    Undefined
   r13 (SP):  Valid stack
   r14 (LR):  Return address (unused)
   PC:        Kernel entry point
   ```

3. **Control Transfer**
   - Jump to kernel entry point
   - Kernel's `initarm()` function called
   - Boot information structure passed in r0

### Stage 7: Kernel Initialization

The kernel's `initarm()` function processes boot information:

**File: `sys/arch/cats/cats/cats_machdep.c:initarm()`**

1. **Validate Boot Information**
   ```c
   if (ebsabootinfo.bt_magic != BT_MAGIC_NUMBER_CATS)
       panic("Incompatible magic number passed in boot args");
   ```

2. **Extract Memory Information**
   - Memory start: `bt_memstart`
   - Memory end: `bt_memend`
   - Available memory: `bt_memavail`
   - Creates bootconfig structure for kernel use

3. **Clock Configuration**
   - Validates Footbridge clock frequency (50-66 MHz)
   - Sets `dc21285_fclk` if in valid range
   - Used for timer calibration

4. **Bootstrap Existing Page Tables**
   - Uses firmware-provided L1 table initially
   - Maps device regions via `pmap_devmap_bootstrap()`
   - Eventually creates new kernel page tables

## ARM MMU Setup Requirements

### StrongARM SA-110 MMU

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
```

**Firmware MMU Configuration:**
- MMU enabled on entry
- I-cache and D-cache enabled
- Write buffer enabled
- Branch prediction enabled
- Little-endian mode
- Vectors at 0x00000000 (low vectors)

### L1 Page Table Format

**Section Descriptor (1MB pages):**
```
Bits 31-20: Section base address
Bits 19-12: Should Be Zero
Bits 11-10: Access Permission (AP)
Bit  9:     Reserved
Bit  8-5:   Domain
Bit  4:     Should Be One
Bit  3:     Cacheable (C)
Bit  2:     Bufferable (B)
Bits 1-0:   Section descriptor (10)
```

Firmware uses:
- Domain 0 for all sections
- AP = 01 (supervisor read/write, user no access)
- C/B bits set appropriately for memory types

### TLB Management

During boot:
- Firmware maintains TLB consistency
- Kernel inherits valid TLB entries
- Kernel flushes TLB when creating new mappings
- TLB flush: `MCR p15, 0, r0, c8, c7, 0`

## Memory Map

### Physical Memory Layout

```
0x00000000 - 0x000FFFFF : Low 1MB (kernel bootstrap)
0x00100000 - 0x0FFFFFFF : Available DRAM (varies, max 256MB)
0x42000000 - 0x420FFFFF : DC21285 CSR registers
0x50000000 - 0x500FFFFF : PCI I/O space
0x7C000000 - 0x7CFFFFFF : PCI Memory space (16MB)
0x80000000 - 0x8FFFFFFF : PCI Memory space (256MB)
```

### Virtual Memory Layout (Kernel)

```
0xF0000000 - 0xF0FFFFFF : Kernel text/data (16MB)
0xF1000000 - 0xFCFFFFFF : Kernel VM space (192MB)

Device Mappings:
0xFD000000 - 0xFD0FFFFF : DC21285 ARM CSR space (1MB)
0xFE000000 - 0xFE0FFFFF : DC21285 cache flush region (1MB)
0xFF000000 - 0xFF0FFFFF : DC21285 PCI I/O space (1MB)
0xFF100000 - 0xFF1FFFFF : DC21285 PCI IACK space (1MB)
0xFF200000 - 0xFF2FFFFF : PCI Type 1 config space (16MB)
0xFF300000 - 0xFF3FFFFF : PCI Type 0 config space (16MB)
```

### DC21285 Footbridge Registers

**Important Register Spaces:**
```
0x42000000: CSR Base
  + 0x00: SA Control Register
  + 0x04: SA Base Address Mask
  + 0x08: SA Base Address Offset
  + 0x10: SDRAM Configuration
  + 0x14: SDRAM Timing
  + 0x80: ROM Control Register
  + 0x84: ROM Timing Register
```

## Build and Installation

### Kernel Configuration

Configure kernel for cats:

```bash
cd /sys/arch/cats/conf
config GENERIC
cd ../compile/GENERIC
make depend && make
```

### Kernel Installation

1. **Copy Kernel to Boot Partition**
   ```bash
   # From NetBSD system
   cp /netbsd /boot/netbsd

   # Or from build machine
   scp netbsd root@cats:/boot/netbsd
   ```

2. **Update Boot Configuration**
   - Access Cyclone firmware menu
   - Set boot device and partition
   - Set kernel path (default: `/netbsd`)
   - Set boot parameters if needed

### Firmware Configuration

**Accessing Firmware:**
1. Power on or reset system
2. Press designated key during POST (typically ESC or DEL)
3. Navigate firmware menu

**Boot Configuration Options:**
```
Boot Device:        Primary IDE, Secondary IDE, SCSI, Network
Boot Partition:     0-3 (partition number)
Kernel Path:        /netbsd (or custom path)
Boot Delay:         0-99 seconds
Console:            VGA, Serial (COM1)
Serial Parameters:  Speed, parity, data bits
```

### Creating Bootable Disk

1. **Partition Disk**
   ```bash
   # Create NetBSD partition
   fdisk -u wd0
   # Create disklabel
   disklabel -e wd0
   ```

2. **Install Boot Blocks**
   ```bash
   # CATS uses standard ARM32 boot blocks
   installboot /dev/rwd0a /usr/mdec/bootxx_ffs
   ```

3. **Install Kernel**
   ```bash
   cp /netbsd /mnt/netbsd
   ```

## Debugging

### Firmware Debug Console

**Accessing Debug Console:**
- During POST, press Ctrl-C or designated key
- Provides low-level firmware shell

**Useful Commands:**
```
memory          - Display memory configuration
pci             - Show PCI devices
boot [device]   - Boot from specified device
setenv          - Set environment variables
printenv        - Display environment variables
reset           - Reset system
```

### Early Boot Debugging

1. **Serial Console**
   - Connect serial cable to COM1
   - Configure terminal: 38400 baud, 8N1
   - Set firmware to serial console mode
   - Captures all boot messages

2. **VGA Console**
   - Default console on CATS
   - Displays POST and boot messages
   - Scroll-back usually limited

### Memory Debugging

**Verify Boot Information:**
```c
printf("bt_magic:    0x%x\n", ebsabootinfo.bt_magic);
printf("bt_memstart: 0x%x\n", ebsabootinfo.bt_memstart);
printf("bt_memend:   0x%x\n", ebsabootinfo.bt_memend);
printf("bt_memavail: 0x%x\n", ebsabootinfo.bt_memavail);
printf("bt_fclk:     %u Hz\n", ebsabootinfo.bt_fclk);
```

**Expected Values:**
- bt_magic: 0x43415453 ('CATS')
- bt_memstart: 0x00000000
- bt_memend: Physical memory size
- bt_memavail: After kernel/firmware
- bt_fclk: 50000000-66000000 (50-66 MHz)

### Common Issues

**Issue: "Incompatible magic number"**
- Cause: Booted with wrong firmware or corrupted boot info
- Solution: Update firmware, verify kernel compatibility

**Issue: System hangs after "Booting NetBSD"**
- Cause: Invalid memory configuration or device mapping
- Solution: Check memory detection in firmware

**Issue: "panic: initarm: No root device"**
- Cause: Kernel can't find boot device
- Solution: Verify disk partitioning and kernel config

**Issue: VGA console not working**
- Cause: Incompatible VGA card
- Solution: Use serial console, replace VGA card with compatible model

### Kernel Debug Options

**Enable verbose boot:**
```
# In firmware, set boot flags
boot -v

# Or in kernel config
options VERBOSE_INIT_ARM
options DEBUG
options DDB
```

**Early debug output:**
```c
#define VERBOSE_INIT_ARM
// Enables VPRINTF macro in cats_machdep.c
// Displays initialization steps
```

## Technical Notes

### DC21285 Footbridge Integration

The DC21285 provides the interface between:
- SA-110 processor and system memory
- Processor and PCI bus
- Memory-mapped register access
- PCI configuration space
- Cache coherency for DMA

**Key Features:**
- 64-bit SDRAM interface
- 33 MHz PCI bus master
- Integrated PCI-to-PCI bridge
- DMA controllers
- Interrupt controller
- Timers and watchdog

### Firmware Boot Protocol

Cyclone firmware boot protocol:
1. Firmware initializes hardware
2. Firmware loads kernel to physical address 0
3. Firmware creates initial page tables
4. Firmware enables MMU
5. Firmware calls kernel with boot structure pointer
6. Kernel validates structure and continues

This is simpler than platforms requiring standalone bootloaders.

### Console Support

**VGA Console:**
- Standard PC VGA card required
- 80x25 text mode by default
- PC keyboard (PS/2 or AT) required
- Managed by pckbc(4) and vga(4) drivers

**Serial Console:**
- 16550-compatible UART (fcom driver)
- Standard PC serial port (COM1/COM2)
- Baud rates: 9600-115200 (default 38400)
- Useful for headless operation

### Network Boot

Cyclone firmware supports network boot:
1. Firmware contains PXE-like network stack
2. DHCP request for IP configuration
3. TFTP download of kernel image
4. Boot proceeds as with disk boot

Configuration in firmware setup menu.

### Firmware Updates

**Updating Cyclone Firmware:**
1. Obtain firmware update from vendor
2. Use firmware update utility
3. Follow vendor-specific procedure
4. Risk: Bad flash can brick system

Generally not needed for NetBSD operation.

## References

- `/sys/arch/cats/cats/cats_machdep.c` - Kernel initialization
- `/sys/arch/cats/include/cyclone_boot.h` - Boot structure definition
- `/sys/arch/arm/footbridge/` - DC21285 driver code
- Intel SA-110 Microprocessor Technical Reference Manual
- Digital Semiconductor 21285 Core Logic Technical Reference Manual
- CATS System Reference Manual (Chalice Technologies)
