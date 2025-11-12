# NetBSD/ofppc Boot Documentation

## Platform Overview

NetBSD/ofppc is the generic OpenFirmware PowerPC port that supports various PowerPC systems with IEEE 1275 OpenFirmware. This is a "catch-all" port for PowerPC machines that:

- Have standard OpenFirmware firmware
- Don't fit into other specific ports (macppc, prep, etc.)
- Are not Apple Macintosh systems
- Use standard OEA (Operating Environment Architecture)

Supported systems include:
- Pegasos I and Pegasos II (Genesi/bPlan)
- IBM 7043-140 and 7044-170/270
- Motorola PowerStack II Pro 4000
- FirePower systems
- Various CHRP-compliant systems
- Other OpenFirmware PowerPC workstations

## Boot Method

**Primary Boot Method:** OpenFirmware (IEEE 1275)

The ofppc boot process uses standard OpenFirmware:
- IEEE 1275-1994 compliant firmware
- Device tree enumeration
- Client interface services
- Boot device selection via OF commands

### Boot Sequence

1. **Hardware Reset** → OpenFirmware initialization
2. **OpenFirmware** → Enumerates device tree
3. **OF** → Loads `ofwboot` bootloader
4. **ofwboot** → Loads NetBSD kernel
5. **Kernel** → Initializes using device tree data

## Boot Loader Implementation

### Primary Bootloader: ofwboot

**Location:** `/sys/arch/ofppc/stand/ofwboot/`

The ofppc bootloader:
- Runs under OpenFirmware control
- Uses OF client interface
- Auto-configures from device tree
- Supports FFS, NFS, CD-ROM boot
- Platform-agnostic design

**Key Source Files:**
- `ofwstart.S` - OpenFirmware entry point
- `ofwboot.c` - Bootloader main code
- Shares code with macppc's ofwboot

### Installation Tools

- `bootinfo/` - Boot information utility
- Standard ofwboot binary

## BAT Register Setup

**Source:** `/sys/arch/ofppc/ofppc/machdep.c:initppc()`

### Dynamic BAT Configuration

Unlike platform-specific ports, ofppc dynamically configures BATs based on the OpenFirmware device tree:

```c
void initppc(u_int startkernel, u_int endkernel, char *args)
{
    int node, i;
    uint16_t bitmap;

    /* Get model name from OpenFirmware */
    node = OF_finddevice("/");
    if (node != -1) {
        OF_getprop(node, "model", model_name, sizeof(model_name));
    }

    /* Initialize model-specific quirks */
    model_init();

    if ((oeacpufeat & OEACPU_NOBAT) == 0) {
        /* Scan device tree for memory ranges */
        bitmap = ranges_bitmap(node, 0);
        oea_batinit(0);

        /* Create BAT mappings for I/O regions */
        for (i = 1; i < 0x10; i++) {
            /* Skip kernel segments */
            if (i == USER_SR || i == KERNEL_SR || i == KERNEL2_SR)
                continue;

            if (bitmap & (1 << i)) {
                /* Map this 256MB region */
                oea_iobat_add(0x10000000 * i, BAT_BL_256M);
            }
        }
    }

    ofwoea_initppc(startkernel, endkernel, args);
}
```

### BAT Bitmap Algorithm

The `ranges_bitmap()` function:
1. Recursively scans OpenFirmware device tree
2. Examines "ranges" properties
3. Builds bitmap of 256MB regions needing I/O mapping
4. Creates BAT entries for detected regions

This allows ofppc to automatically configure itself for different hardware without hardcoded addresses.

### Platform-Specific BAT Adjustments

Some systems require special handling:

```c
/* Pegasos systems */
if (strncmp(model_name, "Pegasos", 7) == 0) {
    /* Pegasos-specific PCI I/O mapping */
    modeldata.pciiodata[0].start = 0x00001400;
    modeldata.pciiodata[0].limit = 0x0000ffff;
}

/* IBM 7044 systems */
if (strncmp(model_name, "IBM,7044", 8) == 0) {
    /* Different PCI I/O range */
    for (j = 0; j < MAX_PCI_BUSSES; j++) {
        modeldata.pciiodata[j].start = 0x00fff000;
        modeldata.pciiodata[j].limit = 0x00ffffff;
    }
}
```

## Boot Process Stages

### Stage 1: OpenFirmware Initialization

OpenFirmware performs:
1. POST (Power-On Self Test)
2. Device tree construction
3. PCI bus enumeration
4. Memory detection
5. Boot device selection

### Stage 2: ofwboot Loading

**Source:** `/sys/arch/ofppc/stand/ofwboot/ofwstart.S`

```assembly
_start:
    /* Save OpenFirmware arguments */
    mr      %r8, %r5           /* Save OF entry point */
    mr      %r9, %r6           /* Save arg address */
    mr      %r10, %r7          /* Save arg length */

    /* Setup initial BAT for bootloader */
    li      %r9, 0x12          /* BATL(0, BAT_M, BAT_PP_RW) */
    mtibatl 0, %r9
    mtdbatl 0, %r9
    li      %r9, 0x1ffe        /* BATU(0, BAT_BL_256M, BAT_Vs) */
    mtibatu 0, %r9
    mtdbatu 0, %r9
    isync

    /* Call main bootloader code */
    bl      main
```

### Stage 3: Device Tree Scanning

ofwboot uses OpenFirmware to:
- Detect boot device
- Find network interfaces
- Locate console device
- Read boot arguments

### Stage 4: Kernel Loading

ofwboot:
1. Opens boot device via OF
2. Reads kernel image
3. Validates ELF header
4. Loads kernel segments
5. Transfers control to kernel entry point

### Stage 5: Kernel Initialization

```c
/* Kernel entry from locore.S */
__start:
    /* OpenFirmware passes:
     * %r5 = OF entry point
     * %r6 = args
     * %r7 = args length
     */

    /* Initialize with OF still available */
    bl      ofwinit

    /* Disable MMU */
    li      %r0, 0
    mtmsr   %r0
    isync

    /* Call initppc */
    bl      initppc
```

## MMU Requirements

### Initial State

- **MMU:** Enabled by OpenFirmware
- **Translation:** OF provides initial page tables
- **BATs:** May be set up by OF

### OpenFirmware Memory Model

- OpenFirmware sets up initial translation
- Bootloader runs with OF services available
- Kernel disables OF translation and sets up own

### Cache Requirements

- Varies by processor (750, 74xx, 970)
- Generally 32KB I-cache, 32KB D-cache
- Cache line size: 32 bytes (older) or 128 bytes (970)

### Page Table Transition

The kernel:
1. Saves OF translation tables
2. Disables MMU
3. Sets up BATs
4. Creates new page tables
5. Re-enables MMU with new translations

## Memory Map

### Physical Memory Layout (Generic)

```
0x00000000 - 0x00003fff    Exception vectors
0x00004000 - RAM_END       Main memory (varies)
0x80000000 - 0x8fffffff    PCI memory (typical, varies)
0xf0000000 - 0xffffffff    I/O regions (typical, varies)
```

Actual layout determined from OpenFirmware device tree.

### Pegasos-Specific Layout

```
0x00000000 - RAM_END       Main memory (up to 2GB)
0x80000000 - 0x8fffffff    PCI memory
0xf0000000 - 0xf00fffff    MV64361 system controller
0xf1000000 - 0xf1ffffff    PCI I/O
0xff000000 - 0xffffffff    Boot ROM
```

### IBM 7044-Specific Layout

```
0x00000000 - RAM_END       Main memory
0x80000000 - 0xbfffffff    PCI memory space
0xc0000000 - 0xcfffffff    PCI I/O space
0xf0000000 - 0xffffffff    System I/O
```

### Virtual Memory Layout

Standard OEA layout:
```
0x00000000 - 0x0fffffff    User space (segment 0)
0x10000000 - 0xefffffff    User space (segments 1-14)
0xf0000000 - 0xffffffff    Kernel space (segment 15)
```

## Build and Installation

### Building the Bootloader

```bash
cd /sys/arch/ofppc/stand
make cleandir
make depend
make
```

Produces:
- `ofwboot` - OpenFirmware bootloader
- `ofwboot.elf` - ELF format (preferred)

### Creating Boot Media

#### Method 1: Boot from Hard Disk

```bash
# Install bootloader
cp ofwboot /boot/ofwboot.elf

# Boot via OpenFirmware
boot hd:,ofwboot.elf netbsd
```

#### Method 2: Boot from Network

```bash
# Setup TFTP server with ofwboot and kernel
# In OpenFirmware:
boot net:,ofwboot.elf netbsd
```

#### Method 3: Boot from CD-ROM

```bash
# Create ISO with bootloader
# In OpenFirmware:
boot cd:,ofwboot.elf netbsd
```

### OpenFirmware Boot Configuration

Configure OF to auto-boot:

```
setenv boot-device hd:,ofwboot.elf
setenv boot-file netbsd
setenv auto-boot? true
setenv boot-command boot
reset-all
```

### Installing NetBSD

1. **Partition disk:**
```bash
# May need special partitioning for some systems
fdisk /dev/rwd0c
# Or use OF partition tools
```

2. **Create filesystems:**
```bash
newfs /dev/rwd0a
newfs /dev/rwd0e
```

3. **Install system:**
```bash
mount /dev/wd0a /mnt
cd /mnt
tar xzpf /path/to/sets.tgz
cp /path/to/ofwboot.elf /mnt/boot/
```

## Debugging

### OpenFirmware Console

Access OF console:
```
# At boot, press appropriate key (varies by system)
# Pegasos: ESC during boot
# IBM: depends on model

ok printenv
ok devalias
ok dev / ls
ok .properties
```

### Device Tree Exploration

```
ok dev /
ok ls
ok dev /pci
ok ls
ok dev /chosen
ok .properties
```

### Boot Device Detection

```
ok dev /chosen
ok .properties
# Look for bootpath, bootargs
```

### Verbose Boot

```
ok boot -v
# Or set in environment:
ok setenv boot-args -v
ok reset-all
```

### Serial Console

Configure for serial console:
```
ok setenv output-device scca
ok setenv input-device scca
ok reset-all
```

### Common Boot Issues

**Problem:** "Can't open boot device"
- **Cause:** Wrong boot path
- **Solution:** Use devalias to find correct device name

**Problem:** "Invalid memory access"
- **Cause:** BAT mapping incorrect for this system
- **Solution:** May need model-specific code

**Problem:** Pegasos graphics issues
- **Cause:** Display device_type not set
- **Solution:** ofppc sets it automatically via OF_setprop

**Problem:** Wrong PCI I/O range
- **Cause:** Model not recognized
- **Solution:** Add model_init() entry

### Model-Specific Debugging

Enable model detection debug:
```c
/* In machdep.c */
printf("Model: %s\n", model_name);
printf("Ranges offset: %d\n", modeldata.ranges_offset);
printf("PCI I/O: %x-%x\n",
       modeldata.pciiodata[0].start,
       modeldata.pciiodata[0].limit);
```

### BAT Mapping Debug

Add debug output to initppc():
```c
printf("BAT bitmap: 0x%x\n", bitmap);
for (i = 1; i < 0x10; i++) {
    if (bitmap & (1 << i)) {
        printf("Mapping region 0x%x (256MB)\n", 0x10000000 * i);
    }
}
```

## Platform-Specific Notes

### Pegasos I/II

- Genesi/bPlan PowerPC workstations
- MV64361 system controller
- ATI or Radeon graphics
- Ethernet (rtk or via)
- USB 2.0
- SATA support

Pegasos quirks:
- L2 cache disabled by default (enabled by NetBSD)
- Device_type needs fixing for graphics
- Special PCI I/O range

### IBM 7043-140 and 7044-170/270

- CHRP-compliant systems
- Similar to RS/6000 but with OpenFirmware
- May have SCSI and/or IDE
- Often use serial console

### Motorola PowerStack II Pro 4000

- PReP/CHRP hybrid
- OpenFirmware firmware
- PCI expansion
- VGA graphics support

### FirePower Systems

- Clean OpenFirmware implementation
- ranges_offset = 0 (different from most)
- Good reference platform

### RTAS Support

Some systems support RTAS (Run-Time Abstraction Services):

```c
#if NRTAS > 0
extern int machine_has_rtas;

/* RTAS provides runtime services */
/* - Power management */
/* - Hardware configuration */
/* - Error logging */
#endif
```

## References

### Source Files

- `/sys/arch/ofppc/ofppc/locore.S` - Kernel entry point
- `/sys/arch/ofppc/ofppc/machdep.c` - Platform initialization
- `/sys/arch/ofppc/stand/ofwboot/` - Bootloader
- `/sys/arch/powerpc/oea/ofwoea_machdep.c` - OEA+OF common code

### Standards

- IEEE 1275-1994: Standard for Boot (Initialization Configuration) Firmware
- PowerPC Processor Binding to IEEE 1275
- CHRP (Common Hardware Reference Platform) specification
- PAPR (Power Architecture Platform Reference)

### Platform Documentation

- Pegasos II Hardware Manual
- IBM 7044-270 Service Guide
- Motorola PowerStack documentation
- OpenFirmware command reference

### OpenFirmware Resources

- Open Firmware Working Group documents
- Device tree specification
- Client interface specification
- Forth language reference

### Boot Examples

```bash
# Interactive boot
ok boot hd:,ofwboot.elf netbsd

# Auto-boot setup
ok setenv boot-device hd:,ofwboot.elf
ok setenv boot-file netbsd
ok setenv auto-boot? true
ok saveenv
ok reset-all

# Network boot
ok setenv boot-device enet:,ofwboot.elf
ok setenv boot-file netbsd
ok boot

# Single user
ok boot hd:,ofwboot.elf netbsd -s

# Verbose
ok boot hd:,ofwboot.elf netbsd -v
```

### Model Detection

ofppc automatically detects and configures for:
- Pegasos I and II
- IBM 7043 and 7044 series
- Motorola PowerStack
- FirePower systems
- Generic CHRP systems
- And others via device tree examination

The port is designed to be extensible - new systems can be added by:
1. Adding model detection in model_init()
2. Setting appropriate modeldata parameters
3. No hardcoded addresses in core code
