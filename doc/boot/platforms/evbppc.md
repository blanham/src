# NetBSD/evbppc Boot Documentation

## Platform Overview

NetBSD/evbppc is the port of NetBSD to PowerPC evaluation boards and embedded systems. Unlike other PowerPC ports that target specific commercial systems, evbppc is a "meta-port" supporting a wide variety of development boards, evaluation boards, and embedded PowerPC systems.

Supported board families:
- **IBM 4xx series:** Walnut, Ebony (PowerPC 40x/44x)
- **Freescale MPC85xx:** MPC8536DS, MPC8548CDS, P2020DS, TWR-P1025
- **Marvell Discovery:** EV64260
- **AMCC 405GP:** Explora451
- **Nintendo Wii:** Broadway (750CL derivative)
- **Virtex PowerPC:** Xilinx Virtex FPGA boards
- **PlanetCore:** PMPPC
- **MikroTik:** RouterBOARD 850Gx2
- **DHT Walnut+:** DHT evaluation board

Each board family has its own subdirectory and may have unique boot requirements.

## Boot Method

**Primary Boot Method:** Varies by Board

evbppc boards use different boot methods depending on the hardware:

- **U-Boot:** Most common (MPC85xx, Wii, many 4xx boards)
- **PPCBoot:** Older boards
- **PPCBUG:** Some evaluation boards
- **Custom firmware:** Board-specific (e.g., Wii, RouterBOARD)
- **Direct flash boot:** Some embedded systems

### Boot Sequence (Generic)

1. **Power-On** → Firmware initialization
2. **Firmware** → Loads bootloader or kernel
3. **Bootloader (optional)** → May load NetBSD kernel
4. **Kernel** → Initializes based on board type

## Boot Loader Implementation

### No Universal Bootloader

Unlike most ports, evbppc **does not have** a common bootloader in `/sys/arch/evbppc/stand/`. Each board type handles booting differently:

- **U-Boot systems:** U-Boot loads kernel directly
- **Wii:** Custom HBC/bootmii loader
- **Others:** Board-specific methods

Kernels are typically loaded as:
- ELF images
- U-Boot uImage format
- Raw binary
- Board-specific format

## BAT Register Setup

**Source:** Varies by board subdirectory

BAT configuration is highly board-specific. Examples:

### MPC85xx Boards (e5500 core, Book E, no BATs)

MPC85xx uses Book E architecture which uses TLB entries instead of BATs:

```c
/* Book E uses TLB, not BATs */
/* TLB entries set up in platform-specific code */
/* Memory management very different from OEA */
```

### IBM 4xx Boards (also Book E, no BATs)

PowerPC 40x and 44x also use Book E:

```c
/* 4xx series uses different MMU model */
/* TLB-based, not BAT-based */
```

### Wii (750CL, has BATs)

**Source:** `/sys/arch/evbppc/wii/machdep.c`

The Wii uses a 750CL processor (G3-based) with BATs:

```c
void initppc(u_int startkernel, u_int endkernel, char *args)
{
    /* Wii-specific BAT setup */
    oea_batinit(
        0x0c000000, BAT_BL_256M,  /* Hollywood registers */
        0x10000000, BAT_BL_256M,  /* MEM2 (additional RAM) */
        0);
}
```

### EV64260 (750, has BATs)

Discovery II-based board with BATs similar to other 750-based systems.

## Boot Process Stages

### Highly Board-Specific

Each board family has its own boot process. Key examples:

### Example 1: MPC85xx (e500 core)

```c
/* Source: sys/arch/evbppc/mpc85xx/machdep.c */

void initppc(u_int startkernel, u_int endkernel, u_int args, void *btinfo)
{
    /* Book E initialization */
    /* No BATs - uses TLB entries */

    /* U-Boot passes device tree blob */
    /* Parse FDT (Flattened Device Tree) */

    /* Initialize e500 core specific features */
    /* - L1 cache
     * - MMU (TLB-based)
     * - CoreNet fabric (on some models)
     */
}
```

### Example 2: Wii (750CL)

```c
/* Source: sys/arch/evbppc/wii/machdep.c */

void initppc(u_int startkernel, u_int endkernel, char *args)
{
    /* Wii has two memory regions:
     * MEM1: 24MB main memory
     * MEM2: 64MB additional memory
     */

    /* Setup BATs for Hollywood chip (Wii GPU/I/O) */
    oea_batinit(
        0x0c000000, BAT_BL_256M,  /* Hollywood */
        0x10000000, BAT_BL_256M,  /* MEM2 */
        0);

    /* Wii uses AES engine for encryption */
    /* May need special handling */
}
```

### Example 3: Walnut (405GP)

```c
/* Source: sys/arch/evbppc/walnut/machdep.c */

void initppc(u_int startkernel, u_int endkernel)
{
    /* PowerPC 405GP is Book E architecture */
    /* Uses software TLB loading */

    /* Initialize IBM 405GP specifics:
     * - UIC (Universal Interrupt Controller)
     * - PLB (Processor Local Bus)
     * - OPB (On-chip Peripheral Bus)
     */

    ibm4xx_init(/* ... */);
}
```

## MMU Requirements

### Varies by Processor Architecture

evbppc supports multiple PowerPC architectures:

### OEA (Operating Environment Architecture)

Used by: 750CL (Wii), EV64260

- Standard PowerPC BATs
- Hash page table
- Segment registers
- Traditional PowerPC MMU

### Book E

Used by: 40x, 44x, MPC85xx (e500/e500mc/e5500)

- TLB-based (no BATs)
- Software TLB miss handling (40x/44x)
- Hardware TLB miss (e500+)
- Different exception model
- Variable page sizes

### Cache Configuration

Varies widely:
- **405GP:** 16KB I-cache, 16KB D-cache
- **440GP:** 32KB I-cache, 32KB D-cache
- **750CL:** 32KB I-cache, 32KB D-cache
- **e500:** 32KB I-cache, 32KB D-cache
- **e500mc:** 32KB I-cache, 32KB D-cache + L2
- **e5500:** 32KB I-cache, 32KB D-cache + L2 + L3

## Memory Map

### Board-Specific Memory Maps

Each board has unique memory layout. Examples:

### Wii Memory Map

```
0x00000000 - 0x017fffff    MEM1 (24MB main RAM)
0x0c000000 - 0x0cffffff    Hollywood registers
0x0d000000 - 0x0d00ffff    Hollywood EXI/SI/AI/VI
0x10000000 - 0x13ffffff    MEM2 (64MB additional RAM)
```

### MPC8548CDS Memory Map

```
0x00000000 - RAM_END       DDR SDRAM (varies: 256MB-8GB)
0x80000000 - 0x9fffffff    PCI memory
0xa0000000 - 0xbfffffff    PCI I/O
0xe0000000 - 0xe00fffff    CCSR (Configuration, Control, Status Registers)
0xf0000000 - 0xf7ffffff    Local bus (flash, NVRAM, etc.)
```

### Walnut Memory Map

```
0x00000000 - RAM_END       SDRAM (varies: 32MB-256MB)
0x40000000 - 0x4fffffff    PCI memory
0x80000000 - 0x8fffffff    PCI I/O
0xef400000 - 0xef5fffff    DCRs (Device Control Registers) via PLB
0xfffe0000 - 0xffffffff    Boot flash (128KB)
```

## Build and Installation

### Building Kernels

Each board has specific kernel configuration:

```bash
# For Wii
cd /sys/arch/evbppc/conf
config WII
cd ../compile/WII
make depend
make

# For MPC8548CDS
config MPC8548CDS
cd ../compile/MPC8548CDS
make depend
make

# For Walnut
config WALNUT
cd ../compile/WALNUT
make depend
make
```

### Installation Methods

#### U-Boot Systems (MPC85xx, etc.)

```bash
# Convert kernel to U-Boot format
mkimage -A ppc -O netbsd -T kernel -C none \
    -a 0x100000 -e 0x100000 -n "NetBSD/evbppc" \
    -d netbsd netbsd.ub

# Load via TFTP
setenv ipaddr 192.168.1.100
setenv serverip 192.168.1.1
tftpboot 0x1000000 netbsd.ub
bootm 0x1000000

# Or boot from flash
cp.b 0x1000000 0xff800000 ${filesize}
bootm 0xff800000
```

#### Wii (Homebrew Channel)

```bash
# Wii uses special boot.elf format
# Copy to SD card
cp netbsd /media/sd/apps/netbsd/boot.elf

# Boot via Homebrew Channel or BootMii
```

#### PPCBoot Systems

```bash
# Similar to U-Boot but older
ppcboot> tftp 0x400000 netbsd
ppcboot> go 0x400000
```

## Debugging

### Serial Console (Most Boards)

Most evbppc boards use serial console:

```c
/* Common serial settings */
#define CONSPEED 115200    /* or 9600 */
#define CONADDR  0x4500    /* varies by board */
```

Connection:
- 115200 or 9600 baud
- 8N1
- No flow control

### U-Boot Debugging

```bash
# Print environment
printenv

# Memory commands
md 0x0 0x100          ; Memory display
mm 0x1000             ; Memory modify

# Boot debugging
setenv bootdelay 10
setenv bootargs -v    ; Verbose boot
```

### Wii-Specific Debugging

Wii has unique debugging via:
- USB Gecko (serial over USB)
- BootMii for low-level access
- EXI (External Interface) for debugging

### DDB Kernel Debugger

```c
options DDB
options DDB_HISTORY_SIZE=512
```

### Common Boot Issues

**Problem:** Kernel doesn't boot
- **Cause:** Wrong kernel for board type
- **Solution:** Verify kernel config matches board

**Problem:** U-Boot can't load kernel
- **Cause:** Wrong load address or format
- **Solution:** Check mkimage parameters

**Problem:** No serial output
- **Cause:** Wrong serial port or baud rate
- **Solution:** Check board documentation

**Problem:** Memory size wrong
- **Cause:** Memory detection failure
- **Solution:** May need manual configuration

## Platform-Specific Notes

### Nintendo Wii

Unique aspects:
- Broadway CPU (750CL derivative)
- Hollywood GPU/I/O chip
- Two memory regions (MEM1, MEM2)
- SD card boot
- USB support
- AES encryption engine (recently added support)

### MPC85xx Boards

Features:
- e500/e500mc/e5500 cores
- Book E architecture
- Multi-core support (some models)
- High-speed networking
- Advanced interrupt controller (MPIC)
- CoreNet fabric (newer models)

### IBM 40x/44x Boards

Characteristics:
- Embedded-focused
- Lower power consumption
- Real-time capabilities
- Configurable I/O
- Industrial temperature range

### EV64260 (Discovery II)

Features:
- GT-64260 system controller
- Dual Gigabit Ethernet
- PCI and PCI-X
- Development board for embedded systems

## References

### Source Files

Board-specific directories:
- `/sys/arch/evbppc/wii/` - Nintendo Wii
- `/sys/arch/evbppc/mpc85xx/` - Freescale MPC85xx
- `/sys/arch/evbppc/walnut/` - IBM 40x boards
- `/sys/arch/evbppc/ev64260/` - Marvell Discovery
- `/sys/arch/evbppc/explora/` - AMCC 405GP
- `/sys/arch/evbppc/virtex/` - Xilinx Virtex
- `/sys/arch/evbppc/pmppc/` - PlanetCore
- `/sys/arch/evbppc/dht/` - DHT boards

### Configuration Files

- `/sys/arch/evbppc/conf/WII` - Wii kernel config
- `/sys/arch/evbppc/conf/MPC8548CDS` - MPC8548 config
- `/sys/arch/evbppc/conf/WALNUT` - Walnut config
- `/sys/arch/evbppc/conf/EV64260` - Discovery config
- Many others in `/sys/arch/evbppc/conf/`

### Documentation

- U-Boot User Manual
- Freescale MPC85xx Reference Manuals
- IBM PowerPC 40x/44x Embedded Processor User's Manuals
- Wii development documentation
- Board-specific manuals

### Architecture References

- **Book E:** PowerPC Book E specification
- **OEA:** PowerPC Architecture Book III
- **e500:** Freescale e500 Core Reference Manual
- **405/440:** IBM 40x/44x documentation

### Special Notes

evbppc is the most diverse PowerPC port because:
- Multiple processor architectures (OEA, Book E)
- Wide range of hardware (consumer to industrial)
- Different boot methods (U-Boot, custom, etc.)
- Various memory sizes and layouts
- From low-power embedded to high-performance network

Each board subdirectory is almost like its own mini-port, sharing common PowerPC code but with unique initialization and hardware support.
