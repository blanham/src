# NetBSD/zaurus Boot Process Documentation

## Platform Overview

NetBSD/zaurus supports Sharp Zaurus clamshell PDAs:
- Sharp Zaurus SL-C700 (Corgi)
- Sharp Zaurus SL-C750 (Shepherd)
- Sharp Zaurus SL-C760 (Husky)
- Sharp Zaurus SL-C860 (Boxer)
- Sharp Zaurus SL-C1000 (Akita)
- Sharp Zaurus SL-C3000/C3100/C3200 (Spitz/Borzoi)

**Hardware Features:**
- Intel XScale PXA255/PXA270 processors (400 MHz)
- 64MB RAM (standard models) to 128MB (C3x00 models)
- Built-in keyboard and touchscreen
- CompactFlash and SD card slots
- 640x480 VGA LCD (rotatable)
- USB host and client support
- Audio, infrared, Bluetooth (some models)

The Zaurus line originally ran Sharp's embedded Linux (OpenZaurus/Cacko ROM). NetBSD provides an alternative open-source operating system.

## Boot Method

### Linux-based Bootloader (zboot)

NetBSD/zaurus uses **zboot**, a Linux application bootloader that runs under the native Sharp Linux environment:

Location: `/sys/arch/zaurus/stand/zboot/`

**Boot Architecture:**
1. Device boots Sharp Linux from internal flash
2. User runs zboot application
3. zboot loads NetBSD kernel from storage
4. zboot transitions from Linux to NetBSD
5. NetBSD kernel takes over

This approach leverages the existing Linux environment for hardware initialization and file access.

### Zbsdmod Kernel Module

**Optional kernel module:**
Location: `/sys/arch/zaurus/stand/zbsdmod/`

Provides kernel-mode support for:
- Direct hardware access
- Memory management
- Final boot transition

## Boot Process Stages

### Stage 1: Sharp Linux Boot

The Zaurus boots Sharp's embedded Linux:

1. **Bootloader in Flash**
   - Built-in bootloader in ROM/Flash
   - Initializes PXA255/270 hardware
   - Loads Linux kernel from flash

2. **Linux Kernel Start**
   - Linux kernel initializes
   - Mounts root filesystem
   - Starts system services

3. **User Environment**
   - Sharp's Qtopia-based GUI
   - Shell access available
   - Can run Linux applications

### Stage 2: zboot Application Launch

User starts zboot from Linux:

**File: `stand/zboot/boot.c:main()`**

1. **Application Initialization**
   ```c
   main(int argc, char *argv[])
   {
       consinit(CONSDEV_GLASS, -1);
       print_banner();
   ```

2. **Display Banner**
   ```
   >> NetBSD/zaurus Boot, Revision X.XX
   >>
   ```

3. **Initialize Environment**
   - Setup console (LCD or serial)
   - Initialize bootinfo structure
   - Configure device access

### Stage 3: Disk Probing

**Function: `diskprobe()`**

Probe for available storage:

1. **Detect Disks**
   ```c
   diskprobe(probed_disks, sizeof(probed_disks));
   // Probes: hd (CF card), sd (SD card)
   ```

2. **Display Disks**
   ```
   disks: hd0 sd0
   ```

3. **Set Defaults**
   ```c
   default_devname = "hd";    // CF card
   default_unit = 0;
   default_partition = 0;     // 'a' partition
   default_filename = "netbsd";
   ```

### Stage 4: Boot Configuration

**Parse boot.cfg:**

Location: `/boot.cfg` on boot device

```c
snprintf(bootconfpath, sizeof(bootconfpath),
         "%s%d%c:%s",
         default_devname, default_unit,
         'a' + default_partition,
         BOOTCFG_FILENAME);
parsebootconf(bootconfpath);
```

**boot.cfg format:**
```
banner=Welcome to NetBSD/zaurus
timeout=5
load=/netbsd
load=/netbsd.old

menu=Boot NetBSD:load /netbsd;boot
menu=Boot NetBSD (single user):load /netbsd;boot -s
menu=Boot NetBSD (verbose):load /netbsd;boot -v
```

### Stage 5: User Interaction

**Boot Menu:**

```c
if (bootcfg_info.nummenu > 0) {
    doboottypemenu();  // Display menu
} else {
    // Auto-boot
    printf("Press return to boot now, any other key for boot menu\n");
    printf("booting hd0a:netbsd - starting in 5...\n");

    c = awaitkey(bootcfg_info.timeout, 1);
    if (c != '\r' && c != '\n' && c != '\0') {
        bootmenu();  // Interactive menu
    }
}
```

**Interactive Commands:**
```
Boot Commands:
  boot [device:][filename] [-flags]
  ls [path]
  disk
  consdev {glass|com [speed]}
  help
  quit
```

### Stage 6: Kernel Loading

**Function: `exec_netbsd()`**

Load NetBSD kernel:

1. **Parse Boot Specification**
   ```c
   parsebootfile(filename, &fsname, &devname,
                &unit, &partition, &file);
   ```

   Examples:
   - `hd0a:netbsd` - CF card, partition a, file netbsd
   - `sd0a:netbsd.old` - SD card, partition a, netbsd.old

2. **Open Device**
   ```c
   // Uses Linux system calls
   open("/dev/hda1", O_RDONLY);  // CF card partition 1
   ```

3. **Load Kernel**
   ```c
   marks[MARK_START] = 0;
   loadfile_zboot(file, marks, LOAD_KERNEL);
   ```

   **Custom loader:**
   - `loadfile_zboot()` is specialized for zaurus
   - Handles Linux environment constraints
   - Uses Linux file I/O
   - Allocates memory via Linux

4. **Parse ELF**
   - Validate ELF header
   - Load program segments
   - Load symbol table (if present)
   - Record entry point

### Stage 7: Boot Information Structure

**Create bootinfo:**

```c
BI_ALLOC(BTINFO_MAX);

struct btinfo_howto bi_howto;
bi_howto.howto = howto;  // Boot flags
BI_ADD(&bi_howto, BTINFO_HOWTO, sizeof(bi_howto));
```

**Boot information includes:**
- Boot flags (RB_SINGLE, RB_ASKNAME, etc.)
- Console configuration
- Memory layout
- Device information

### Stage 8: Transition to NetBSD

**Critical boot transition:**

This is the most complex part - transitioning from Linux to NetBSD:

1. **Prepare Processor State**
   ```c
   // Disable Linux services
   // Flush caches
   // Prepare for MMU switch
   ```

2. **Execute Kernel**
   ```c
   // Jump to kernel entry point
   // Pass bootinfo structure
   // Never returns
   ```

   **Assembly transition:**
   ```assembly
   // Disable interrupts
   MRS r0, CPSR
   ORR r0, r0, #(I32_bit | F32_bit)
   MSR CPSR_c, r0

   // Flush caches
   // Clean D-cache
   // Invalidate I-cache
   // Drain write buffer

   // Setup registers
   MOV r0, bootinfo_ptr
   MOV r1, #0
   MOV r2, #0

   // Jump to kernel
   MOV pc, kernel_entry
   ```

3. **Kernel Entry State**
   ```
   r0:        Pointer to bootinfo
   r1-r2:     Zero
   Mode:      Supervisor (SVC32)
   MMU:       May be enabled (Linux state)
   Interrupts: Disabled
   ```

### Stage 9: Kernel Initialization

NetBSD kernel takes over:

1. **Early Setup**
   - Validate bootinfo
   - Parse boot flags
   - Setup basic console

2. **Hardware Reinitialization**
   - Reprogram PXA registers
   - Setup interrupt controller
   - Configure GPIO pins
   - Initialize LCD controller

3. **Memory Management**
   - Create new page tables
   - Map kernel memory
   - Map devices
   - Enable NetBSD MMU configuration

4. **Device Initialization**
   - Probe for devices
   - Initialize drivers
   - Setup keyboard and touchscreen
   - Configure storage

5. **Continue Boot**
   - Mount root filesystem
   - Start init
   - Launch userland

## ARM MMU Setup Requirements

### Intel XScale PXA255/PXA270

**Control Register (CP15 c1):**
```
Bit 0:  M - MMU enable
Bit 2:  C - Data cache enable
Bit 3:  W - Write buffer enable
Bit 7:  B - Big-endian (0 for little-endian)
Bit 9:  R - ROM protection
Bit 11: Z - Branch prediction
Bit 12: I - Instruction cache enable
Bit 13: V - High vectors (0xFFFF0000)
Bit 14: RR - Cache replacement strategy
```

**PXA255 Specifications:**
- ARMv5TE architecture
- 32KB instruction cache
- 32KB data cache (write-back)
- 2KB mini data cache
- Branch Target Buffer

**PXA270 Enhancements:**
- 32KB I-cache
- 32KB D-cache
- Faster processor (up to 624 MHz)
- Wireless MMX instructions

**Cache Operations:**
```
Clean D-cache line:
    MCR p15, 0, rd, c7, c10, 1

Invalidate I-cache:
    MCR p15, 0, rd, c7, c5, 0

Drain write buffer:
    MCR p15, 0, rd, c7, c10, 4

Invalidate TLB:
    MCR p15, 0, rd, c8, c7, 0
```

### Page Table Configuration

**Kernel Page Tables:**
- L1 table: 16KB, 16KB aligned
- L2 tables: 1KB each, 1KB aligned
- Map kernel at 0xC0000000
- Map devices at 0xF0000000+

## Memory Map

### Physical Memory Layout

**PXA255/PXA270:**
```
0x00000000 - 0x00FFFFFF : Internal Boot ROM (1MB)
0x04000000 - 0x04FFFFFF : LCD Controller
0x0C000000 - 0x0CFFFFFF : USB Controller
0x10000000 - 0x100FFFFF : GPIO, Interrupt Controller
0x14000000 - 0x140FFFFF : OS Timer
0x40000000 - 0x400FFFFF : Memory Controller
0x44000000 - 0x440FFFFF : LCD registers
0xA0000000 - 0xAFFFFFFF : DRAM (64MB-128MB)
```

**Zaurus-Specific:**
```
0x0C000000: CF card controller
0x0E000000: SD card controller
0x10000000: GPIO (keyboard, touchscreen)
0x14000000: Timers
```

### Virtual Memory Layout

**Linux (initial state):**
```
0x00000000 - 0xBFFFFFFF : User space
0xC0000000 - 0xFFFFFFFF : Kernel space
```

**NetBSD (after transition):**
```
0xC0000000 - 0xCFFFFFFF : Kernel text/data
0xD0000000 - 0xEFFFFFFF : Kernel VM
0xF0000000 - 0xFFFFFFFF : Device mappings
```

## Build and Installation

### Building zboot

```bash
cd /sys/arch/zaurus/stand/zboot
make
```

Output: `zboot` (Linux ELF binary)

**Cross-compilation:**
```bash
# If cross-compiling from another system
MACHINE=zaurus MACHINE_ARCH=arm make
```

### Building Kernel

```bash
cd /sys/arch/zaurus/conf
config GENERIC
cd ../compile/GENERIC
make depend && make
```

Output: `netbsd` (kernel for zaurus)

### Installation to Device

#### Method 1: CF Card Installation

**Most Common Method:**

1. **Prepare CF Card**
   ```bash
   # From NetBSD or Linux system
   fdisk -u /dev/rsd0
   disklabel -e sd0  # or appropriate device
   newfs /dev/rsd0a
   ```

2. **Copy Files**
   ```bash
   mount /dev/sd0a /mnt
   cp zboot /mnt/
   cp netbsd /mnt/
   umount /mnt
   ```

3. **On Zaurus (Sharp Linux)**
   ```bash
   # Insert CF card
   mount /dev/hda1 /mnt/cf
   cd /mnt/cf
   ./zboot
   ```

#### Method 2: Internal Flash Installation

**For advanced users:**

1. **Create NetBSD Partition**
   - Repartition internal flash
   - Preserve Sharp Linux for fallback
   - Install NetBSD on separate partition

2. **Install Bootloader**
   ```bash
   cp zboot /hdd2/  # Internal flash
   cp netbsd /hdd2/
   ```

3. **Boot Script**
   ```bash
   # Create boot script in Sharp Linux
   #!/bin/sh
   /hdd2/zboot hdd2:netbsd
   ```

#### Method 3: SD Card Installation

**Similar to CF card:**

1. **Prepare SD Card**
2. **Copy files** (zboot, netbsd)
3. **Boot from SD:**
   ```bash
   # On Zaurus
   mount /dev/mmcda1 /mnt/card
   cd /mnt/card
   ./zboot sd0a:netbsd
   ```

### Creating boot.cfg

**Example boot.cfg:**

```
banner=NetBSD/zaurus Boot Menu
timeout=10
clear=1

menu=Boot NetBSD:load hd0a:netbsd;boot
menu=Boot NetBSD (single user):load hd0a:netbsd;boot -s
menu=Boot NetBSD (verbose):load hd0a:netbsd;boot -v
menu=Boot old kernel:load hd0a:netbsd.old;boot
menu=Drop to boot prompt:prompt

default=1
```

Place in root of boot partition as `boot.cfg`.

## Debugging

### Console Options

**Glass Console (LCD):**
- Default console
- Uses built-in LCD
- Keyboard input
- Suitable for normal operation

**Serial Console:**
- Requires serial adapter
- Connection to PXA UART
- 115200 baud, 8N1
- Useful for debugging

**Switching Consoles:**
```bash
# From zboot menu:
> consdev com 115200
> boot
```

### Common Issues

**Issue: zboot won't start**
- Cause: Permission or format problem
- Solution: `chmod +x zboot`, verify ELF format
- Check: File not corrupted

**Issue: "Cannot open hd0a"**
- Cause: Device or partition not found
- Solution: Verify CF card inserted, check partition
- Try: `disk` command to see detected disks

**Issue: "load failed"**
- Cause: Kernel file not found or corrupted
- Solution: Verify netbsd exists, check filename
- Try: `ls hd0a:/` to list files

**Issue: Kernel loads but system hangs**
- Cause: Hardware detection failure
- Solution: Try verbose boot (-v), check hardware
- Debug: Use serial console for messages

**Issue: Display problems after boot**
- Cause: LCD controller not initialized
- Solution: Verify kernel configuration
- Check: Display driver (wsdisplay, rasops)

### Debugging zboot

**Enable Debug Output:**

In `boot.c`:
```c
#define DEBUG
// Enables debug printfs
```

**Verbose Mode:**
```bash
> boot -v
```

Shows:
- Device probing
- File operations
- Memory allocation
- Kernel loading progress

### Kernel Debugging

**Build Debug Kernel:**

In kernel config:
```
options     DEBUG
options     DDB
options     DIAGNOSTIC
options     KGDB
```

**Serial Debugging:**
```
options     KGDB
options     KGDB_DEVNAME="\"com\""
options     KGDB_DEVRATE=115200
```

**Boot Messages:**
```
NetBSD/zaurus booting...
initarm: Configuring PXA270...
cpu0 at mainbus0: Intel XScale PXA270 rev 7 (XScale core)
cpu0: DC enabled IC enabled WB enabled LABT
cpu0: 32KB/32B 32-way L1 VIPT Instruction cache
cpu0: 32KB/32B 32-way L1 VIPT Data cache
```

### Testing Hardware

**From zboot:**
```bash
> ls hd0a:/          # Test CF card
> ls sd0a:/          # Test SD card
> disk               # Show detected disks
```

**From NetBSD:**
```bash
dmesg | grep wd      # CF card detection
dmesg | grep ld      # SD card detection
wsconscfg -d         # Display info
```

## Technical Notes

### Linux-NetBSD Transition

**Key Challenge:**
Transitioning from running Linux to NetBSD:

1. **Linux Services**
   - Must cleanly exit Linux
   - Shutdown services gracefully
   - Release resources

2. **Hardware State**
   - Hardware initialized by Linux
   - NetBSD must reinitialize
   - Some state preserved

3. **Memory Management**
   - Linux MMU active
   - Must switch to NetBSD page tables
   - Coordinate memory usage

### Unix System Calls

zboot uses Linux system calls:

```c
// File operations
open(), read(), close(), lseek()

// Console
write() to stdout/stderr

// Memory
mmap() for allocations

// Process
exit(), _exit()
```

**Unixdev Layer:**
- `stand/zboot/unixdev.c` - Device layer
- `stand/zboot/unixcons.c` - Console
- `stand/zboot/unixsys.S` - System call interface

### File System Access

**Pathfs:**
- `stand/zboot/pathfs.c`
- Provides filesystem abstraction
- Uses Linux filesystems through syscalls
- Supports FFS, ext2, FAT, etc.

**Disk Access:**
- `stand/zboot/diskprobe.c` - Disk detection
- `stand/zboot/disk.h` - Disk structures
- Uses Linux block devices

### Hardware Specifics

**Keyboard:**
- Matrix keyboard via GPIO
- Driver: `zkbd(4)`
- Special key mappings

**Touchscreen:**
- ADC-based touchscreen
- Driver: `zts(4)`
- Calibration required

**LCD:**
- Sharp 640x480 VGA LCD
- Rotatable (portrait/landscape)
- Driver: `w100(4)` for ATI Imageon

**Storage:**
- CF card: IDE interface, `wd(4)`
- SD card: MMCSD interface, `ld(4)` via `pxamci(4)`
- Internal flash: MTD, not fully supported

**Power Management:**
- APM support via `zapm(4)`
- Suspend/resume
- Battery monitoring

### Model Differences

**SL-Cxx00 series (PXA255):**
- 400 MHz PXA255
- 64MB RAM
- 640x480 LCD
- CF + SD slots

**SL-C3x00 series (PXA270):**
- 416 MHz PXA270
- 128MB RAM (C3x00)
- Faster performance
- Built-in HDD (C3x00)

**Spitz/Akita/Borzoi:**
- Various PXA270 speeds
- Different flash sizes
- Some hardware variations

## Device Support

**Well-Supported:**
- PXA255/270 CPU
- LCD display
- Keyboard
- Touchscreen
- CF card
- SD card
- Serial port
- GPIO
- Timers

**Partially Supported:**
- USB (host/client)
- Audio
- Power management
- LEDs

**Not Yet Supported:**
- Built-in WiFi (on some models)
- Bluetooth
- Internal HDD (C3000)
- Some sensors

## References

- `/sys/arch/zaurus/stand/zboot/boot.c` - Main bootloader
- `/sys/arch/zaurus/stand/zboot/bootinfo.h` - Boot information
- `/sys/arch/zaurus/zaurus/` - Machine-dependent code
- `/sys/arch/arm/xscale/pxa2x0*` - PXA support
- Intel PXA255/PXA270 Developer's Manual
- Sharp Zaurus Hardware Documentation
- OpenZaurus/Cacko ROM documentation
