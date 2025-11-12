# NetBSD/sandpoint Boot Documentation

## Platform Overview

NetBSD/sandpoint is the port of NetBSD to Motorola Sandpoint reference designs and compatible embedded PowerPC systems. The Sandpoint platform was designed as a reference for embedded PowerPC NAS (Network Attached Storage) and network appliance devices.

Supported systems include:
- Motorola Sandpoint X3 (MPC8240/MPC8245)
- Kurobox (KuroBox HG, standard)
- Synology DS-101, DS-106, CS-406, RS-406
- QNAP TS-101, TS-201
- Allnet ALL6250, ALL6260
- NH230 NAS boards
- Other MPC824x-based appliances

These are typically fanless, low-power embedded systems designed for always-on NAS and server applications.

## Boot Method

**Primary Boot Method:** Firmware + altboot

The Sandpoint boot process is unique among PowerPC platforms:
- Proprietary firmware (varies by vendor)
- U-Boot on many systems
- Direct kernel loading from flash/disk
- altboot standalone bootloader
- No OpenFirmware

### Boot Sequence

1. **Power-On** → Firmware/U-Boot initialization
2. **Firmware** → May set up initial BATs
3. **altboot** → Standalone bootloader loaded
4. **altboot** → Detects hardware, loads kernel
5. **Kernel** → Initializes with bootinfo structure

## Boot Loader Implementation

### Primary Bootloader: altboot

**Location:** `/sys/arch/sandpoint/stand/altboot/`

The altboot bootloader is a sophisticated standalone program that:
- Runs without firmware services
- Performs complete hardware initialization
- Auto-detects board type and configuration
- Supports IDE/SATA, USB, and network boot
- Provides interactive boot menu
- Can update itself

**Key Source Files:**
- `main.c` - Main bootloader logic
- `brdsetup.c` - Board-specific setup
- `entry.S` - Assembly entry point
- `dev_disk.c` - Disk device support
- `dev_net.c` - Network device support

### Board Auto-Detection

altboot automatically detects the board type:

```c
/* Board types detected at runtime */
#define BOARD_UNKNOWN    0
#define BOARD_SANDPOINT  1
#define BOARD_KUROBOX    2
#define BOARD_SYNOLOGY   3
#define BOARD_QNAP       4
#define BOARD_ALLNET     5
#define BOARD_NH230      6

/* Detection in main.c */
brdtype = determine_board_type();
brdprop = brd_lookup(brdtype);
printf(">> %s, cpu %u MHz, bus %u MHz, %dMB SDRAM\n",
    brdprop->verbose, cpuclock / 1000000,
    busclock / 1000000, bi_mem.memsize >> 20);
```

## BAT Register Setup

**Source:** `/sys/arch/sandpoint/sandpoint/machdep.c:initppc()`

### BAT Configuration

The Sandpoint uses CHRP "Map B" layout:

```c
void initppc(u_int startkernel, u_int endkernel, u_int args, void *btinfo)
{
    /*
     * Setup fixed BAT registers for "Map B" layout:
     * - One BAT for PCI memory space
     * - One BAT for MPC107/MPC824x EUMB, ISA mem, PCI/ISA I/O,
     *   PCI config, PCI interrupt ack, and flash/ROM
     */
    oea_batinit(
        0x80000000, BAT_BL_256M,   /* SANDPOINT_BUS_SPACE_MEM */
        0xfc000000, BAT_BL_64M,    /* _EUMB|_IO */
        0x70000000, BAT_BL_8M,     /* NH230 board control only */
        0);
}
```

### Sandpoint BAT Layout

```
DBAT0: 0x80000000 - 256MB  (PCI memory space)
DBAT1: 0xfc000000 - 64MB   (EUMB + I/O regions)
DBAT2: 0x70000000 - 8MB    (NH230 board-specific, conditional)
```

### altboot BAT Setup

The bootloader sets up BATs before loading the kernel:

**Source:** `/sys/arch/sandpoint/stand/altboot/entry.S`

```assembly
/* Read existing BATs */
mfspr   3, SPR_DBAT0U
mfspr   4, SPR_DBAT0L
/* ... read all BATs ... */

/* Setup new BATs for boot environment */
BAT123:
    .long xBATL(0x80000000, BAT_I|BAT_G, BAT_PP_RW)
    .long xBATU(0x80000000, BAT_BL_256M, BAT_Vs)
    .long xBATL(0xfc000000, BAT_I|BAT_G, BAT_PP_RW)
    .long xBATU(0xfc000000, BAT_BL_64M, BAT_Vs)
    .long xBATL(0x70000000, BAT_I|BAT_G, BAT_PP_RW)
    .long xBATU(0x70000000, BAT_BL_128K, BAT_Vs)
```

## Boot Process Stages

### Stage 1: Firmware Initialization

Varies by device:
- **U-Boot:** Full bootloader, may load altboot or kernel directly
- **Proprietary:** Minimal init, loads altboot from flash
- **RedBoot:** Some systems use RedBoot

### Stage 2: altboot Entry Point

**Source:** `/sys/arch/sandpoint/stand/altboot/entry.S`

```assembly
_start:
    /* Disable MMU/FPU */
    li      0, 0
    mtmsr   0
    isync

    /* Enable caches */
    mfspr   8, 1008              # HID0
    ori     8, 8, (HID0_ICE | HID0_DCE)@l
    mtspr   1008, 8

    /* Clear BSS */
    li      0, 0
    lis     8, edata@ha
    addi    8, 8, edata@l
    lis     9, end@ha
    addi    9, 9, end@l
5:  cmpw    0, 8, 9
    bge     6f
    stw     0, 0(8)
    addi    8, 8, 4
    b       5b
6:

    /* Initialize CPU info and call main */
    INIT_CPUINFO(4, 1, 9, 0)
    lis     3, __start@ha
    addi    3, 3, __start@l
    bl      _C_LABEL(initppc)
    bl      _C_LABEL(main)
```

### Stage 3: Hardware Detection

altboot performs comprehensive hardware detection:

```c
void main(int argc, char *argv[], char *bootargs_start, char *bootargs_end)
{
    /* Detect board type */
    brdprop = brd_lookup(brdtype);
    printf(">> %s altboot, revision %s\n", bootprog_name, bootprog_rev);
    printf(">> %s, cpu %u MHz, bus %u MHz, %dMB SDRAM\n",
        brdprop->verbose, cpuclock / 1000000, busclock / 1000000,
        bi_mem.memsize >> 20);

    /* Detect storage devices */
    nata = pcilookup(PCI_CLASS_IDE, lata, 2);
    if (nata == 0)
        nata = pcilookup(PCI_CLASS_RAID, lata, 2);
    if (nata == 0)
        nata = pcilookup(PCI_CLASS_SCSI, lata, 2);

    /* Detect network interfaces */
    nnif = pcilookup(PCI_CLASS_ETH, lnif, 2);

    /* Detect USB controllers */
    nusb = pcilookup(PCI_CLASS_USB, lusb, 3);
}
```

### Stage 4: Kernel Loading

altboot supports multiple boot sources:

```c
/* Boot device precedence */
#define BNAME_DEFAULT "wd0:"

/* Supported boot devices */
- wd0:  /* IDE/SATA disk */
- sd0:  /* SCSI disk (if present) */
- net:  /* Network boot via NFS */
- usb0: /* USB mass storage */

/* Boot with arguments */
boot wd0:netbsd -s        /* Single user */
boot net:                 /* Network boot */
boot wd0:netbsd.old       /* Alternate kernel */
```

### Stage 5: Bootinfo Structure Creation

altboot creates comprehensive bootinfo:

```c
/* Bootinfo structures passed to kernel */
struct btinfo_memory bi_mem;        /* Memory size/layout */
struct btinfo_console bi_cons;      /* Console device */
struct btinfo_clock bi_clk;         /* CPU/bus clocks */
struct btinfo_prodfamily bi_fam;    /* Product family */
struct btinfo_model bi_model;       /* Specific model */
struct btinfo_bootpath bi_path;     /* Boot device path */
struct btinfo_rootdevice bi_rdev;   /* Root device */
struct btinfo_net bi_net;           /* Network config */
```

## MMU Requirements

### Initial State

- **MMU:** Disabled during altboot entry
- **Caches:** Enabled by altboot
- **Real Mode:** Boot code runs in real mode

### Cache Initialization

```assembly
/* Enable I-cache and D-cache */
mfspr   8, SPR_HID0
ori     8, 8, (HID0_ICE | HID0_DCE)@l
mtspr   SPR_HID0, 8
```

### Processor Support

- MPC8240 (603e core)
- MPC8241 (603e core)
- MPC8245 (603e core with enhanced features)

### Page Table Setup

After BAT initialization:
- Main memory via page tables
- I/O regions via BAT
- Standard PowerPC OEA paging

## Memory Map

### Physical Memory Layout

```
0x00000000 - 0x00003fff    Exception vectors
0x00004000 - RAM_END       Main memory (32MB-1GB, varies by board)
0x70000000 - 0x707fffff    NH230 board control (8MB, conditional)
0x80000000 - 0x8fffffff    PCI memory space (256MB)
0xfc000000 - 0xfdffffff    EUMB (Embedded Utilities Memory Block)
0xfe000000 - 0xfe7fffff    ISA memory
0xfe800000 - 0xfebfffff    PCI I/O space
0xfec00000 - 0xfedfffff    PCI configuration space
0xfee00000 - 0xfeefffff    PCI interrupt acknowledge
0xff000000 - 0xff7fffff    Flash/ROM (8MB)
0xff800000 - 0xffffffff    Boot flash (8MB)
```

### EUMB (Embedded Utilities Memory Block)

Located at 0xfc000000:
- EPIC (Embedded Programmable Interrupt Controller)
- I2C controller
- DMA controller
- Memory controller configuration
- Timers and other peripherals

### Board-Specific Regions

#### NH230:
- 0x70000000: Board control registers

#### Kurobox/Synology:
- Hardware monitoring via I2C
- LED controls
- Power management

## Build and Installation

### Building altboot

```bash
cd /sys/arch/sandpoint/stand/altboot
make cleandir
make depend
make
```

This produces:
- `altboot` - Raw binary bootloader
- `altboot.bin` - Binary for flash programming

### Installing altboot

#### Method 1: Flash Programming

```bash
# Copy to flash memory
# WARNING: Incorrect flash programming can brick the device!
flashcp -v altboot.bin /dev/mtd0

# Or using U-Boot:
tftp 0x1000000 altboot.bin
protect off 0xfff00000 0xffffffff
erase 0xfff00000 0xffffffff
cp.b 0x1000000 0xfff00000 ${filesize}
```

#### Method 2: U-Boot Loading

```bash
# Configure U-Boot to load altboot
setenv bootcmd 'ide reset; ext2load ide 0:1 0x1000000 /altboot; go 0x1000000'
setenv bootdelay 3
saveenv
```

#### Method 3: Network Boot

```bash
# U-Boot network boot of altboot
setenv ipaddr 192.168.1.100
setenv serverip 192.168.1.1
setenv bootfile altboot
tftpboot 0x1000000
go 0x1000000
```

### Installing NetBSD

1. **Prepare boot device** (typically IDE/SATA):
```bash
# Partition disk
fdisk -i wd0
# Create NetBSD partition (type 169)

# Create filesystem
newfs /dev/rwd0a

# Mount and install
mount /dev/wd0a /mnt
cd /mnt
tar xzpf sets.tgz
```

2. **Install altboot**:
```bash
cp altboot /mnt/
```

3. **Configure boot**:
```bash
# altboot reads boot.cfg from root filesystem
echo "banner=Welcome to NetBSD/sandpoint" > /mnt/boot.cfg
echo "timeout=5" >> /mnt/boot.cfg
echo "default=1" >> /mnt/boot.cfg
```

### Disk Layout

Typical layout:
```
/dev/wd0a    /       FFS     (root filesystem with altboot)
/dev/wd0b    swap            (swap partition)
/dev/wd0e    /usr    FFS     (user filesystem)
/dev/wd0f    /var    FFS     (variable data)
/dev/wd0g    /home   FFS     (user homes)
```

## Debugging

### Serial Console

Most Sandpoint boards have serial console:

```c
/* Serial port configuration */
#define CONSADDR 0x3f8       /* COM1 */
#define CONSPEED 115200      /* Common for embedded systems */
#define CONSFREQ 1843200     /* Clock frequency */
```

Connect serial cable:
- 115200 baud, 8N1
- No flow control

### LED Debugging

Many boards have status LEDs:

```c
/* Kurobox LED control */
#define LED_POWER_ON    0x01
#define LED_ACTIVITY    0x02

/* Accessed via I2C or GPIO */
```

### Boot Messages

altboot is very verbose:
```
>> NetBSD/sandpoint altboot, revision 1.12
>> Kurobox/LinkStation, cpu 266 MHz, bus 133 MHz, 128MB SDRAM
>> PATA device: wd0
>> Network device: re0
```

### Common Boot Issues

**Problem:** altboot not found
- **Cause:** Not in flash or wrong flash location
- **Solution:** Reflash or configure U-Boot correctly

**Problem:** "wd0: no disk"
- **Cause:** Disk not detected or bad cable
- **Solution:** Check cabling, try different port

**Problem:** Network boot fails
- **Cause:** No DHCP/NFS server
- **Solution:** Configure network infrastructure

**Problem:** Wrong board detected
- **Cause:** Board detection failure
- **Solution:** May need board-specific build

### U-Boot Integration

Check U-Boot environment:
```bash
printenv
# Look for bootcmd, bootargs, etc.

# Manual boot:
ide reset
ext2load ide 0:1 0x1000000 /altboot
go 0x1000000
```

### Low-Level Debugging

Enable debug in altboot:
```c
/* In main.c */
#ifdef DEBUG
int debug = 1;
#endif

/* Adds verbose output during hardware detection */
```

## Platform-Specific Notes

### KuroBox Variants

- **KuroBox HG:** Hard disk + Gigabit Ethernet
- **KuroBox Standard:** Hard disk + 100Mbit Ethernet
- **LinkStation:** Similar to KuroBox, different firmware

### Synology NAS

- DS-101, DS-106: Single-bay NAS
- CS-406, RS-406: 4-bay rack NAS
- Often use Marvell or Realtek Ethernet

### QNAP NAS

- TS-101, TS-201: Tower NAS
- Similar hardware to Synology
- May use different flash layout

### Clock Configuration

CPU and bus clocks vary by board:
```c
/* Common configurations */
200 MHz CPU / 100 MHz bus  (MPC8240)
266 MHz CPU / 133 MHz bus  (MPC8241)
400 MHz CPU / 133 MHz bus  (MPC8245)
```

### PCI Devices

Common onboard devices:
- Realtek RTL8110/8169 Gigabit Ethernet
- VIA VT6212 USB 2.0 controller
- IDE/SATA controllers (various)
- PCI SATA cards supported

### Power Management

Many boards support:
- Power button GPIO
- Automatic shutdown
- Wake-on-LAN
- Thermal monitoring via I2C

## References

### Source Files

- `/sys/arch/sandpoint/sandpoint/locore.S` - Assembly boot code
- `/sys/arch/sandpoint/sandpoint/machdep.c` - Platform initialization
- `/sys/arch/sandpoint/stand/altboot/` - altboot bootloader
- `/sys/arch/sandpoint/include/bootinfo.h` - Bootinfo structures

### Hardware Documentation

- Motorola Sandpoint X3 Reference Design
- MPC8240/8245 Integrated Processor User's Manual
- CHRP "Map B" Memory Layout Specification
- Individual NAS vendor documentation

### Community Resources

- NetBSD/sandpoint wiki pages
- KuroBox/LinkStation community forums
- NAS hardware modification guides

### altboot Features

- Auto-detection of ~20 different board variants
- Support for IDE, SATA, USB, and network boot
- Module loading capability
- Firmware update capability (with care)
- Interactive boot menu

### Boot Examples

```bash
# Default boot
altboot> boot

# Specify kernel
altboot> boot wd0:netbsd.old

# Boot single-user
altboot> boot -s

# Boot from network
altboot> boot net:

# Boot with root on different device
altboot> boot wd0:netbsd -a
```
