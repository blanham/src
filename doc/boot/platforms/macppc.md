# NetBSD/macppc Boot Documentation

## Platform Overview

NetBSD/macppc is the port of NetBSD to Apple Power Macintosh systems and compatible machines. This includes:
- Power Macintosh G3, G4, and G5 systems
- PowerBook laptops
- iMac, eMac, Mac mini
- Xserve G4 and G5
- Compatible clones

The platform supports machines with OpenFirmware firmware and uses the OEA (Operating Environment Architecture) PowerPC model.

## Boot Method

**Primary Boot Method:** OpenFirmware

The macppc boot process relies on Apple's OpenFirmware implementation, which provides:
- Device tree enumeration
- Boot device selection
- Initial program loading
- Runtime services during boot

### Boot Sequence

1. **Hardware Reset** → OpenFirmware initialization
2. **OpenFirmware** → Loads `ofwboot` bootloader
3. **ofwboot** → Loads NetBSD kernel
4. **Kernel** → System initialization via `initppc()`

## Boot Loader Implementation

### Primary Bootloader: ofwboot

**Location:** `/sys/arch/macppc/stand/ofwboot/`

The `ofwboot` bootloader is a standalone OpenFirmware application that:
- Runs in real mode with OpenFirmware services available
- Supports multiple file systems (FFS, HFS, UFS, NFS)
- Provides interactive boot menu
- Loads ELF and XCOFF format kernels

**Key Source Files:**
- `boot.c` - Main bootloader logic
- `Locore.c` - Low-level initialization
- `ofdev.c` - OpenFirmware device interface
- `net.c` - Network boot support

**Supported Formats:**
- `ofwboot` - Raw binary for direct loading
- `ofwboot.elf` - ELF format
- `ofwboot.xcf` - XCOFF format for older OF implementations

### Installation Methods

The bootloader can be installed via:

1. **bootxx/installboot** - Installs boot blocks on FFS
2. **boothfs** - Creates HFS-blessed boot files
3. **mkboothfs** - Generates bootable HFS volumes

## BAT Register Setup

The macppc platform uses Block Address Translation (BAT) registers for efficient I/O mapping and kernel memory access.

### BAT Configuration

**Source:** `/sys/arch/macppc/macppc/machdep.c:initppc()`

#### For MPC601 Processors:
```
DBAT0: 0x80000000 - 256MB (I/O region)
DBAT1: 0x90000000 - 256MB (I/O region)
DBAT2: 0xa0000000 - 256MB (I/O region)
DBAT3: 0xb0000000 - 256MB (I/O region)
IBAT2: 0xf0000000 - 256MB (I/O region)
```

#### For Later Processors (603/604/750/74xx/970):
```
DBAT0: 0x80000000 - 1GB   (PCI memory space)
DBAT1: 0xf0000000 - 128MB (I/O region)
DBAT2: 0xf8000000 - 64MB  (I/O region)
DBAT3: 0xfe000000 - 8MB   (Grackle I/O space)
```

### BAT Setup Code

```c
void initppc(u_int startkernel, u_int endkernel, char *args)
{
    /* Platform-specific initialization */

#ifdef PPC_OEA601
    if ((mfpvr() >> 16) == MPC601) {
        oea_batinit(
            0x80000000, BAT_BL_256M,
            0x90000000, BAT_BL_256M,
            0xa0000000, BAT_BL_256M,
            0xb0000000, BAT_BL_256M,
            0xf0000000, BAT_BL_256M,
            0);
    } else
#endif
    {
        oea_batinit(
            0x80000000, BAT_BL_1G,    /* PCI memory */
            0xf0000000, BAT_BL_128M,  /* I/O */
            0xf8000000, BAT_BL_64M,   /* I/O */
            0xfe000000, BAT_BL_8M,    /* Grackle IO */
            0);
    }

    ofwoea_initppc(startkernel, endkernel, args);
}
```

## Boot Process Stages

### Stage 1: OpenFirmware Execution

**Assembly Entry:** `/sys/arch/macppc/macppc/locore.S:__start`

```assembly
__start:
    /* Save OpenFirmware arguments */
    mr      %r13, %r6              # Save args pointer
    mr      %r14, %r7              # Save args length

    /* Zero SPRG0 for curcpu() detection */
    li      %r0, 0
    mtsprg0 %r0

    /* Initialize OpenFirmware */
    bl      ofwinit

    /* Disable MMU/FPU/exceptions */
    li      %r0, 0
    mtmsr   %r0
    isync

    /* Initialize CPU model detection */
    bl      cpu_model_init
```

### Stage 2: CPU-Specific Initialization

For G5 (970) processors:
```assembly
    /* Clear HID5 DCBZ bits */
    mfspr   %r11, SPR_HID5
    rldimi  %r11, %r0, 6, 56
    sync
    mtspr   SPR_HID5, %r11
    isync

    /* Setup HID1 features */
    mfspr   %r0, SPR_HID1
    li      %r11, 0x1200
    sldi    %r11, %r11, 44
    or      %r0, %r0, %r11
    mtspr   SPR_HID1, %r0
    isync
```

### Stage 3: CPU Info Initialization

```assembly
    /* Compute end of kernel memory */
    lis     %r4, _C_LABEL(end)@ha
    addi    %r4, %r4, _C_LABEL(end)@l

    /* Initialize cpu_info[0] */
    INIT_CPUINFO(%r4, %r1, %r9, %r0)

    /* Call C initialization */
    lis     %r3, __start@ha
    addi    %r3, %r3, __start@l
    mr      %r5, %r6               # Restore args
    bl      initppc
    bl      main
```

### Stage 4: C Initialization (initppc)

**Source:** `/sys/arch/macppc/macppc/machdep.c`

1. Detect model name from OpenFirmware device tree
2. Enable CPU full-speed mode (G5 systems)
3. Set up BAT mappings for I/O regions
4. Call `ofwoea_initppc()` for common OEA initialization
5. Proceed to `main()`

## MMU Requirements

### Initial State

- **MMU:** Disabled during early boot
- **Address Translation:** OpenFirmware provides identity mapping
- **Real Mode:** Boot code runs in real mode initially

### BAT Requirements

- Minimum 4 DBAT registers
- Minimum 4 IBAT registers
- Support for various block sizes (8MB - 1GB)
- Write-through and cache-inhibited attributes

### Page Table Setup

After BAT initialization, the kernel sets up:
- 256MB segment registers for kernel space
- Hash page table for virtual memory
- Exception vectors at physical address 0

### Special Considerations

**G5 (970) Processors:**
- Require special HID register setup
- Different cache line size (128 bytes)
- Enhanced performance features
- 64-bit addressing capability (with PPC_OEA64_BRIDGE)

## Memory Map

### Physical Memory Layout

```
0x00000000 - 0x00003fff    Exception vectors
0x00004000 - RAM_SIZE      Main memory
0x80000000 - 0xbfffffff    PCI memory space (1GB)
0xf0000000 - 0xf7ffffff    I/O region 1 (128MB)
0xf8000000 - 0xfbffffff    I/O region 2 (64MB)
0xfe000000 - 0xfe7fffff    Grackle I/O (8MB)
0xffc00000 - 0xffffffff    ROM/firmware (4MB)
```

### Virtual Memory Layout (Kernel)

```
0x00000000 - 0x0fffffff    User space (segment 0)
0x10000000 - 0xefffffff    User space (segments 1-14)
0xf0000000 - 0xffffffff    Kernel space (segment 15)
```

### OpenFirmware Memory Allocation

The bootloader reserves memory for:
- Boot arguments structure
- Symbol table (if DDB enabled)
- Initial page tables
- Message buffer

## Build and Installation

### Building the Bootloader

```bash
cd /sys/arch/macppc/stand
make cleandir
make depend
make
make install
```

This creates:
- `ofwboot` - Primary bootloader
- `ofwboot.elf` - ELF format
- `ofwboot.xcf` - XCOFF format

### Installing Boot Files

#### Method 1: Using installboot

```bash
# Install on FFS root partition
installboot -v -o timeout=5 /dev/rsd0a /usr/mdec/ofwboot /usr/mdec/bootxx
```

#### Method 2: HFS Blessing

```bash
# Create HFS boot partition
newfs_hfs /dev/rsd0a
mount -t hfs /dev/sd0a /mnt
cp /usr/mdec/ofwboot /mnt/ofwboot.xcf
# Bless the file (requires HFS tools)
hfsattrib -b /mnt/ofwboot.xcf
```

#### Method 3: Network Boot

Configure OpenFirmware:
```
setenv boot-device enet:10.0.0.1,ofwboot,10.0.0.2
setenv boot-file netbsd
```

### Creating Installation Media

#### Bootable CD-ROM

```bash
# Create HFS blessed boot image
mkboothfs -o cd.hfs ofwboot netbsd
makefs -t cd9660 -o rockridge,bootimage=macppc:cd.hfs \
    netbsd.iso cd_content/
```

#### Bootable USB Drive

```bash
# Partition and format
pdisk /dev/rsd0c
# Create Apple_HFS partition
# Format and install
newfs_hfs /dev/rsd0a
mount -t hfs /dev/sd0a /mnt
cp ofwboot.xcf /mnt/
hfsattrib -b /mnt/ofwboot.xcf
cp netbsd /mnt/
```

## Debugging

### OpenFirmware Debugging

Access OpenFirmware console:
```
# At boot, press: Cmd-Opt-O-F

ok printenv
ok dev /
ok ls
ok dev enet
ok .properties
```

### Boot Verbosity

Enable verbose boot:
```
ok boot -v
```

### DDB Kernel Debugger

Enable in kernel config:
```
options DDB
options DDB_HISTORY_SIZE=512
```

Break into debugger:
- Press `Cmd-Power` on keyboard
- Or build with `options DDB_ONPANIC`

### Serial Console Debugging

Configure OpenFirmware:
```
setenv output-device scca
setenv input-device scca
reset-all
```

Kernel boot arguments:
```
boot -c   # Enter configuration mode
boot -d   # Enter DDB at boot
```

### Common Boot Issues

**Problem:** "DEFAULT CATCH!, code=FFF00300"
- **Cause:** BAT mapping conflict or invalid memory access
- **Solution:** Check BAT setup, verify physical addresses

**Problem:** Hangs after "initppc"
- **Cause:** Interrupt controller not initialized
- **Solution:** Verify PIC initialization in cpu_startup()

**Problem:** "panic: no vm"
- **Cause:** Insufficient memory or bad memory size detection
- **Solution:** Check memory regions, verify bootinfo structure

### Low-Level Debugging

Add debug output in locore.S:
```assembly
/* Debug: write character to serial port */
lis     %r10, 0xf300        # Serial port base
ori     %r10, %r10, 0x0200
li      %r11, 'A'           # Debug marker
stb     %r11, 0(%r10)
```

### Tracing Boot Execution

Enable BOOT_DEBUG in bootloader:
```c
#define DEBUG
// In boot.c, DPRINTF() statements will output
```

### Memory Dump Tools

Use OpenFirmware to examine memory:
```
ok 0 20 dump     # Dump exception vectors
ok f0000000 100 dump   # Dump I/O region
```

### Hardware Diagnostic Tools

```
ok test /memory
ok test /cpu
ok test /screen
ok test-all
```

## Platform-Specific Notes

### G5 Considerations

- Requires 64-bit bridge code (PPC_OEA64_BRIDGE)
- Different HID register initialization
- May need OpenFirmware workarounds (ofw_quiesce)
- Memory above 2GB requires special handling

### PowerBook Considerations

- PMU/SMU initialization for power management
- Special display initialization
- Battery and thermal management
- Sleep/wake support

### Multi-Processor Support

Enabled with `options MULTIPROCESSOR`:
- Secondary CPUs started via OpenFirmware
- IPI (Inter-Processor Interrupt) support
- Separate CPU initialization paths

## References

### Source Files

- `/sys/arch/macppc/macppc/locore.S` - Assembly boot code
- `/sys/arch/macppc/macppc/machdep.c` - Machine-dependent initialization
- `/sys/arch/macppc/stand/ofwboot/` - Bootloader implementation
- `/sys/arch/powerpc/oea/oea_machdep.c` - OEA common code
- `/sys/arch/powerpc/oea/ofwoea_machdep.c` - OpenFirmware OEA code

### Documentation

- OpenFirmware IEEE 1275 specification
- PowerPC 32-bit Architecture Book III (OEA)
- Apple Technote 1061: Fundamentals of Open Firmware
- NetBSD macppc port documentation

### Boot Command Examples

```
# Boot from first hard disk
boot hd:,ofwboot netbsd

# Boot from CD-ROM
boot cd:,ofwboot netbsd

# Boot from network
boot enet:0,ofwboot netbsd

# Boot with options
boot hd:,ofwboot netbsd -s    # Single user
boot hd:,ofwboot netbsd -a    # Ask for root device
boot hd:,ofwboot netbsd -v    # Verbose
```
