# NetBSD/ibmnws Boot Documentation

## Platform Overview

NetBSD/ibmnws is the port of NetBSD to IBM Network Station thin clients. These were diskless network computers designed in the late 1990s for corporate and educational environments.

Supported systems:
- IBM Network Station 1000 (model 8362-XXX)
- Other IBM Network Station PowerPC models

The Network Station was designed as a Java-based thin client, booting entirely from the network and running applications from a central server. NetBSD allows these systems to be repurposed as standalone workstations.

## Boot Method

**Primary Boot Method:** Network Boot (PXE-like)

IBM Network Station firmware uses:
- Proprietary boot ROM
- BOOTP/DHCP for network configuration
- TFTP for kernel download
- PReP-compatible boot process
- No local storage (diskless design)

### Boot Sequence

1. **Power-On** → Network Station firmware initialization
2. **Firmware** → Network configuration via BOOTP/DHCP
3. **Firmware** → Downloads kernel via TFTP
4. **Kernel** → Initializes from network-provided parameters

## Boot Loader Implementation

### No Standalone Bootloader

The ibmnws port **does not have** a bootloader in `/sys/arch/ibmnws/stand/`. Instead:

- Firmware loads kernel directly via TFTP
- Kernel must be in appropriate format for firmware
- No intermediate boot stage
- Simple, diskless-oriented design

The firmware expects:
- Kernel image at known TFTP location
- Proper network configuration in BOOTP/DHCP reply
- ELF or raw binary kernel format

## BAT Register Setup

**Source:** `/sys/arch/ibmnws/ibmnws/machdep.c:initppc()`

### Minimal BAT Configuration

The Network Station uses standard PReP initialization with no custom BAT setup:

```c
void initppc(u_long startkernel, u_long endkernel, u_int args, void *btinfo)
{
    uint32_t sa, ea, banks;
    u_long memsize = 0;
    pcitag_t tag;

    /*
     * Determine memory size by reading PCI host bridge
     * (IBM 82660 MPC-to-PCI bridge)
     */
    tag = genppc_pci_indirect_make_tag(NULL, 0, 0, 0);

    /* Read memory bank configuration from 82660 */
    out32rb(PCI_MODE1_ADDRESS_REG, tag | IBM_82660_MEM_BANK0_START);
    sa = in32rb(PCI_MODE1_DATA_REG);
    out32rb(PCI_MODE1_ADDRESS_REG, tag | IBM_82660_MEM_BANK0_END);
    ea = in32rb(PCI_MODE1_DATA_REG);
    out32rb(PCI_MODE1_ADDRESS_REG, tag | IBM_82660_MEM_BANK_ENABLE);
    banks = in32rb(PCI_MODE1_DATA_REG) & 0xFF;

    /* Calculate total memory from enabled banks */
    if (banks & IBM_82660_MEM_BANK0_ENABLED)
        memsize += IBM_82660_BANK0_ADDR(ea) - IBM_82660_BANK0_ADDR(sa) + 1;
    /* ... calculate for banks 1-3 ... */

    memsize <<= 20;  /* Convert to bytes */

    physmemr[0].start = 0;
    physmemr[0].size = memsize & ~PGOFSET;
    availmemr[0].start = (endkernel + PGOFSET) & ~PGOFSET;
    availmemr[0].size = memsize - availmemr[0].start;

    /* Hardcoded CPU clock (16.666 MHz decrementer) */
    ticks_per_sec = 16666666;
    ns_per_tick = 1000000000 / ticks_per_sec;

    /* Common PReP initialization (sets up standard BATs) */
    prep_initppc(startkernel, endkernel, args, 0);
}
```

### BAT Layout

Standard PReP BAT configuration via `prep_initppc()`:
```
DBAT0: PCI configuration/I/O space
DBAT1: PCI memory space
```

No special mappings required for Network Station hardware.

## Boot Process Stages

### Stage 1: Network Station Firmware

The firmware performs:
1. POST (Power-On Self Test)
2. Network interface initialization
3. BOOTP/DHCP request
4. Receives IP address and boot server information
5. TFTP download of kernel
6. Transfer control to kernel

### Stage 2: Kernel Entry Point

**Source:** `/sys/arch/ibmnws/ibmnws/locore.S`

The Network Station uses standard PowerPC PReP locore.S (no custom assembly entry point).

### Stage 3: Memory Detection

Unlike most platforms, ibmnws detects memory by reading the IBM 82660 PCI bridge registers:

```c
/* IBM 82660 MPC-to-PCI Bridge Memory Configuration */
#define IBM_82660_MEM_BANK0_START   0x80
#define IBM_82660_MEM_BANK0_END     0x84
#define IBM_82660_MEM_BANK_ENABLE   0x88

/* Each bank can be independently configured and enabled */
/* Memory size calculated from enabled bank ranges */
```

This is necessary because Network Stations don't provide memory size via standard methods.

### Stage 4: PCI and Interrupt Setup

```c
void cpu_startup(void)
{
    /* Map PReP interrupt vector register */
    prep_intr_reg = (vaddr_t) mapiodev(PREP_INTR_REG, PAGE_SIZE, false);
    if (!prep_intr_reg)
        panic("startup: no room for interrupt register");
    prep_intr_reg_off = INTR_VECTOR_REG;

    /* Common OEA startup */
    oea_startup("IBM NetworkStation 1000 (8362-XXX)");

    /* Initialize PIC (i8259) */
    pic_init();
    isa_pic = setup_prepivr(PIC_IVR_IBM);
    oea_install_extint(pic_ext_intr);

    /* Enable hardware interrupts */
    splraise(-1);
    __asm volatile ("mfmsr %0; ori %0,%0,%1; mtmsr %0"
                  : "=r"(msr) : "K"(PSL_EE));

    bus_space_mallocok();
}
```

## MMU Requirements

### Initial State

- **MMU:** Disabled during early boot
- **Caches:** Enabled by firmware or early kernel
- **Real Mode:** Early boot runs in real mode

### Processor

IBM Network Station 1000 uses:
- PowerPC 403GCX (embedded processor)
- Or PowerPC 603e (some models)

### Cache Configuration

- **403GCX:** 8KB I-cache, 8KB D-cache, write-through
- **603e:** 16KB I-cache, 16KB D-cache

### Memory Management

Standard PowerPC OEA after BAT setup:
- BATs for I/O regions
- Page tables for main memory
- Standard segment registers

## Memory Map

### Physical Memory Layout

```
0x00000000 - 0x00003fff    Exception vectors
0x00004000 - RAM_END       Main memory (typically 16-64 MB)
0x80000000 - 0x8fffffff    PCI memory space
0xc0000000 - 0xcfffffff    PCI I/O space
0xff000000 - 0xffffffff    Flash/ROM (firmware)
```

### IBM 82660 Bridge Registers

Memory configuration accessed via PCI config space:
- Device 0, Function 0 on PCI bus 0
- Configuration registers at standard PCI config offsets
- Memory bank enable and size registers

### Typical Configuration

Network Station 1000 typically has:
- 16 MB or 32 MB RAM
- No hard disk
- Integrated NIC (10/100 Ethernet)
- Integrated graphics
- PS/2 keyboard and mouse ports

## Build and Installation

### Building the Kernel

```bash
cd /sys/arch/ibmnws/conf
config GENERIC
cd ../compile/GENERIC
make depend
make
```

Produces:
- `netbsd` - ELF kernel for network boot

### Network Boot Server Setup

#### 1. DHCP/BOOTP Configuration

Create `/etc/dhcpd.conf` entry:

```
host nws1000 {
    hardware ethernet 00:00:00:00:00:00;  # Replace with actual MAC
    fixed-address 192.168.1.100;
    next-server 192.168.1.1;              # TFTP server
    filename "netbsd-ibmnws";
}
```

#### 2. TFTP Server Setup

```bash
# Install TFTP server
# Enable in /etc/inetd.conf:
tftp dgram udp wait root /usr/libexec/tftpd tftpd -s /tftpboot

# Copy kernel to TFTP directory
cp netbsd /tftpboot/netbsd-ibmnws
chmod 644 /tftpboot/netbsd-ibmnws
```

#### 3. NFS Root Filesystem

Network Stations boot diskless, so need NFS root:

```bash
# Export root filesystem
# /etc/exports:
/export/nws1000 -maproot=root:wheel -alldirs nws1000

# Create root filesystem
mkdir -p /export/nws1000
cd /export/nws1000
tar xzpf /path/to/base.tgz
tar xzpf /path/to/etc.tgz
# ... extract other sets ...

# Configure for NFS root in /export/nws1000/etc/fstab:
192.168.1.1:/export/nws1000 / nfs rw 0 0
```

#### 4. Kernel Configuration

Kernel must be configured for NFS root:

```
# In kernel config:
options NFS_BOOT_DHCP
options NFS_BOOT_BOOTPARAM
config netbsd root on ? type nfs
```

### Firmware Configuration

Network Station firmware typically requires:
- Network settings (can be obtained via BOOTP/DHCP)
- Boot server address
- Kernel filename

Access firmware setup (method varies by model):
- May have on-screen configuration menu
- May use serial port for configuration
- Refer to IBM Network Station documentation

## Debugging

### Serial Console

Network Stations typically have serial port for console:

```c
/* Serial port configuration */
/* Varies by model - check hardware documentation */
```

Connection:
- 9600 baud (typical), 8N1
- No flow control
- DB-9 or DB-25 connector

### Network Boot Debugging

Debug network boot process:

```bash
# On boot server, monitor DHCP:
tcpdump -i eth0 -n port 67 or port 68

# Monitor TFTP:
tcpdump -i eth0 -n port 69

# Check TFTP logs:
tail -f /var/log/daemon
```

### Kernel Debugging

Enable verbose network boot:
```c
options NFS_BOOT_DEBUG
options DEBUG
```

Enable DDB:
```c
options DDB
options DDB_HISTORY_SIZE=512
```

### Common Boot Issues

**Problem:** No DHCP response
- **Cause:** DHCP server not configured or MAC address wrong
- **Solution:** Verify DHCP configuration, check MAC address

**Problem:** TFTP timeout
- **Cause:** TFTP server not running or file not found
- **Solution:** Check TFTP service, verify filename

**Problem:** Kernel loads but panics
- **Cause:** Wrong kernel or NFS root not accessible
- **Solution:** Verify kernel built for ibmnws, check NFS export

**Problem:** Wrong memory size detected
- **Cause:** 82660 bridge detection failure
- **Solution:** May need manual memory size configuration

**Problem:** Network interface not working after boot
- **Cause:** Driver issue or wrong network configuration
- **Solution:** Verify network driver in kernel, check cabling

## Platform-Specific Notes

### IBM 82660 MPC-to-PCI Bridge

Key component of Network Station:
- Connects PowerPC processor to PCI bus
- Handles memory configuration
- Controls memory bank enable/disable
- Provides memory size information

### Diskless Operation

Network Stations designed for diskless operation:
- All storage via NFS
- Kernel loaded via TFTP
- No local boot media
- Depends on network availability

### Graphics

Network Station has integrated graphics:
- Limited graphics capability
- Designed for X terminal use
- May have VGA output
- Driver support varies

### Repurposing

Network Stations can be repurposed as:
- Thin clients (original purpose)
- Low-power servers (with NFS root)
- Embedded systems
- Educational systems

Advantages:
- Low power consumption
- Fanless (quiet operation)
- Network-centric design
- Small form factor

Limitations:
- No local storage
- Limited memory (typically ≤64MB)
- Older processor
- Requires network infrastructure

## References

### Source Files

- `/sys/arch/ibmnws/ibmnws/machdep.c` - Platform initialization
- `/sys/arch/ibmnws/conf/GENERIC` - Generic kernel config
- `/sys/arch/powerpc/prep/prep_machdep.c` - Common PReP code

### IBM Documentation

- IBM Network Station 1000 Setup Guide
- IBM Network Station Manager documentation
- IBM 82660 MPC-to-PCI Bridge documentation
- PowerPC 403GCX/603e User's Manuals

### Network Boot Standards

- BOOTP/DHCP protocol specifications
- TFTP protocol (RFC 1350)
- NFS protocol specifications
- PXE specification (related)

### Historical Notes

IBM Network Stations were part of the "network computer" trend of the late 1990s, designed to reduce the total cost of ownership by centralizing management and eliminating local storage. While the network computer concept didn't fully succeed commercially, NetBSD/ibmnws allows these well-built thin clients to be repurposed for modern use.

The platform is simple but robust, exemplifying the diskless workstation concept. It shares much code with the PReP port but has unique characteristics due to its thin client heritage.

### Configuration Examples

#### Complete Boot Server Setup

```bash
# 1. DHCP server (/etc/dhcpd.conf):
subnet 192.168.1.0 netmask 255.255.255.0 {
    range 192.168.1.100 192.168.1.200;
    option routers 192.168.1.1;
    option domain-name-servers 192.168.1.1;

    host nws1000 {
        hardware ethernet 08:00:5a:xx:xx:xx;
        fixed-address 192.168.1.100;
        next-server 192.168.1.1;
        filename "netbsd-ibmnws";
        option root-path "/export/nws1000";
    }
}

# 2. NFS server (/etc/exports):
/export/nws1000 -maproot=root -alldirs -network 192.168.1.0 -mask 255.255.255.0

# 3. TFTP server (via inetd):
tftp dgram udp wait root /usr/libexec/tftpd tftpd -s /tftpboot

# 4. Kernel in TFTP directory:
cp /path/to/netbsd /tftpboot/netbsd-ibmnws
```
