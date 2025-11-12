# NetBSD/sparc64 Boot Documentation

## Platform Overview

NetBSD/sparc64 is the port of NetBSD to Sun Microsystems' 64-bit UltraSPARC workstations and servers. This includes:

- **sun4u** - UltraSPARC I/II/III/IV based systems
  - Ultra 1, 2, 5, 10, 30, 45, 60, 80
  - Enterprise 250, 450, 3000-6500
  - Fire V100, V120, V210, V240, V440, V880, V890
  - Blade 100, 150, 1000, 1500, 2500
  - Netra T1, X1, T2000, T4, T5
  - SunBlade 1000, 2000, 2500
- **sun4v** - Hypervisor-based SPARC T-series systems
  - T1000, T2000 (UltraSPARC T1 - Niagara)
  - T5120, T5220 (UltraSPARC T2 - Niagara 2)
  - T3, T4, T5 systems

The platform supports both traditional sun4u architecture with hardware MMU and sun4v architecture with hypervisor-managed memory.

## Boot Method

**Primary Boot Method:** OpenBoot PROM / OpenBoot Firmware

NetBSD/sparc64 relies exclusively on OpenBoot PROM, which provides:

### OpenBoot PROM (Version 3.x and 4.x)
- Full IEEE 1275-1994 (Open Firmware) compliance
- 64-bit address space support
- Device tree with detailed hardware description
- Forth interpreter with enhanced functionality
- Network booting (RARP/DHCP/TFTP/NFS)
- PCI device enumeration
- USB support (on capable systems)
- Graphical console support

### Boot Sequence

1. **Hardware Reset** → POST and PROM initialization
2. **PROM** → Loads first-stage bootloader (bootblk)
3. **bootblk** → Parses filesystem and loads ofwboot
4. **ofwboot** → Initializes MMU and loads kernel
5. **Kernel** → 64-bit kernel initialization

## Boot Loader Implementation

### Architecture Overview

NetBSD/sparc64 shares bootloader code with NetBSD/sparc but includes sparc64-specific MMU and memory management:

**Shared Code Location:** `/sys/arch/sparc/stand/`

The sparc64 Makefile references the sparc bootloaders:
```makefile
# From /sys/arch/sparc64/stand/Makefile
.if ${MACHINE} == sparc64
SUBDIR= ../../sparc/stand/ofwboot
SUBDIR+=../../sparc/stand/bootblk
SUBDIR+=../../sparc/stand/binstall
.endif
```

### First-Stage Bootloader: bootblk

**Location:** `/sys/arch/sparc/stand/bootblk/`

**Implementation:** Forth language (OpenBoot native)

The bootblk runs entirely within the OpenBoot PROM environment and provides:

- **Multi-filesystem support:**
  - FFS v1 (UFS1)
  - FFS v2 (UFS2) with extended attributes
  - LFS v1 and v2
  - RAIDframe partition detection

- **Disklabel parsing:** Reads NetBSD disk label to find root filesystem

- **Inode resolution:** Locates `/ofwboot` file through directory traversal

**Key Features:**

```forth
\ From bootblk.fth
\ NetBSD IEEE 1275 Multi-FS Bootblock
\ Version $NetBSD: bootblk.fth,v 1.18 2025/02/28 09:07:12 andvar Exp $

: do-boot ( bootfile -- )
   ." NetBSD IEEE 1275 Multi-FS Bootblock" cr
   boot-path load-file ( -- load-base )
   dup 0<>  if  " init-program " evaluate  then
;
```

**Filesystem Implementation:**

The bootblock includes full filesystem parsers:
- Superblock reading with multiple location support (8KB, 64KB, 128KB offsets)
- Indirect block following (single and double indirect)
- Symbolic link traversal
- Large file support (64-bit file sizes on UFS2)
- Directory entry parsing

**RAIDframe Support:**

Automatically detects RAIDframe partitions by trying superblock at offset 64 sectors:

```forth
: check-supers ( -- found? )
   -1
   0                    \ Standard offset
   d# 128 KB           \ FFS2 common location
   d# 64 KB            \ FFS2 alternate
   8 KB                \ FFS1 standard

   begin  dup -1 <>  while
      raid-offset dev_bsize * + read-super
      sb-buf fs-magic?  if
         begin  -1 =  until  \ Clean stack
         true exit
      then
   repeat
   drop false
;
```

### Second-Stage Bootloader: ofwboot

**Location:** `/sys/arch/sparc/stand/ofwboot/`

**Key Source Files:**
- `boot.c` - Main bootloader logic
- `Locore.c` - Low-level OpenFirmware interface
- `ofdev.c` - Device abstraction layer
- `net.c` - Network stack
- `loadfile_machdep.c` - **sparc64-specific MMU and memory management**

**Capabilities:**

1. **Kernel Format Support:**
   - ELF64 kernels (standard)
   - Legacy compatibility kernels
   - Compressed kernels (gzip)

2. **Boot Configuration:**
   - `boot.cfg` file support
   - Interactive menu system
   - Boot argument parsing
   - Device override

3. **Memory Management:**
   - 4MB permanent TLB mappings
   - Both OpenFirmware and direct MMU management
   - Support for sun4u and sun4v MMU architectures

4. **Network Booting:**
   - TFTP protocol
   - NFS root support
   - DHCP configuration

**Entry Point:**

```c
// From boot.c
void main(void *ofw)
{
    // Save OpenFirmware interface pointer
    romp = ofw;
    prom_init();

    printf("\r>> %s, Revision %s\n", bootprog_name, bootprog_rev);

    // Parse boot device and arguments
    strncpy(bootdev, prom_getbootpath(), sizeof(bootdev) - 1);
    kboothowto = boothowto =
        bootoptions(prom_getbootargs(), bootdev, kernel, bootline);

    isfloppy = bootdev_isfloppy(bootdev);

    // Main boot loop
    for (;; *kernel = '\0') {
        if (boothowto & RB_ASKNAME) {
            // Interactive mode
            printf("Boot: ");
            kgets(cmdline, sizeof(cmdline));
            // ... handle commands
        }

        check_boot_config();
        start_kernel(kernel, bootline, ofw, isfloppy, kboothowto);
    }
}
```

## MMU Requirements

The UltraSPARC architecture uses a fundamentally different MMU design from 32-bit SPARC:

### UltraSPARC MMU (sun4u)

**Type:** Software-managed TLB with hardware page table walk support

**Characteristics:**
- 64-bit virtual and physical addresses
- Software-managed TLBs (Instruction and Data)
- Multiple page sizes: 8KB, 64KB, 512KB, 4MB
- Context-based address spaces (13-bit context IDs - 8192 contexts)
- Split I-TLB and D-TLB
- No hardware page table walk (pure software)

**TLB Structure:**

```
Instruction TLB (I-TLB):
- 16-64 entries (processor dependent)
- Locked entries for kernel text
- Fully associative

Data TLB (D-TLB):
- 16-64 entries (processor dependent)
- Locked entries for kernel data
- Fully associative
```

**Virtual Address Format (VA):**

```
63        43  42  40  39      28  27       0
+----------+------+------------+------------+
| Reserved | Ctxt | Virtual Page | Offset   |
+----------+------+------------+------------+
```

**TTE (Translation Table Entry) Format:**

```c
// From loadfile_machdep.c
#define SUN4U_TSB_DATA(g, sz, pa, priv, w, c, cv, e, no_fault) \
    ((g) << 63 | (sz) << 61 | (pa) | (priv) << 8 | \
     (w) << 6 | (c) << 4 | (cv) << 3 | (e) << 2 | (no_fault) << 1)

data = SUN4U_TSB_DATA(
    0,          // global=0 (context-specific)
    PGSZ_4M,    // 4MB page size
    pa,         // physical address
    1,          // privileged
    1,          // writable
    1,          // cacheable
    1,          // virtually-cacheable
    1,          // side-effects
    0           // no-fault
);
data |= SUN4U_TLB_L | SUN4U_TLB_CV; // locked, virt cache
```

**Page Sizes:**

| Size Code | Page Size | Use Case |
|-----------|-----------|----------|
| 0 | 8 KB | Standard pages |
| 1 | 64 KB | Large pages |
| 2 | 512 KB | Very large pages |
| 3 | 4 MB | Kernel text/data |

### UltraSPARC T-Series MMU (sun4v)

**Type:** Hypervisor-managed TLB

**Characteristics:**
- Hypervisor manages all MMU operations
- TLB entries requested via hypercalls
- Multiple page sizes: 8KB, 64KB, 4MB, 256MB
- Simplified software interface
- TSB (Translation Storage Buffer) managed by hypervisor

**Hypervisor Calls:**

```c
// From loadfile_machdep.c
#include <machine/hypervisor.h>

hv_rc = hv_mmu_map_perm_addr(va, data, MAP_DTLB);
if (hv_rc != H_EOK) {
    panic("hv_mmu_map_perm_addr() failed - rc = %ld", hv_rc);
}
```

**TTE Format (sun4v):**

```c
#define SUN4V_TSB_DATA(g, sz, pa, priv, w, c, cv, e, no_fault) \
    ((g) << 63 | (sz) << 61 | (pa) | (priv) << 8 | \
     (w) << 6 | (c) << 4 | (cv) << 3 | (e) << 2 | (no_fault) << 1)

data = SUN4V_TSB_DATA(
    0,          // global
    PGSZ_4M,    // 4MB page
    pa,         // physical address
    1,          // privileged
    1,          // writable
    1,          // cacheable
    1,          // virtually-cacheable
    1,          // valid
    0,          // endianness
    0           // write-combining
);
data |= SUN4V_TLB_CV | SUN4V_TLB_X; // virtual cache, executable
```

### MMU Initialization in Bootloader

**Location:** `/sys/arch/sparc/stand/ofwboot/loadfile_machdep.c`

The bootloader provides three memory allocation strategies:

#### 1. NOP Allocator (Header Loading)

Used for initial kernel header inspection:
```c
static ssize_t nop_read(int f, void *addr, size_t size)
{
    return read(f, addr, size);  // No mapping needed
}
```

#### 2. OpenFirmware Allocator

Uses PROM services for memory management:
```c
static int ofw_mapin(vaddr_t rva, vsize_t len)
{
    len  = roundup2(len + (rva & PAGE_MASK_4M), PAGE_SIZE_4M);
    rva &= ~PAGE_MASK_4M;

    if ((len = kvamap_extract(rva, len, &va)) != 0) {
        if (OF_claim((void *)(long)va, len, PAGE_SIZE_4M) == (void*)-1) {
            panic("ofw_mapin: Cannot claim memory.");
        }
        kvamap_enter(va, len);
    }
    return 0;
}
```

#### 3. MMU Allocator (Direct TLB Management)

Creates permanent 4MB locked TLB entries:

**sun4u Implementation:**

```c
static int mmu_mapin_sun4u(vaddr_t rva, vsize_t len)
{
    uint64_t data;
    paddr_t pa;
    vaddr_t va, mva;

    for (pa = (paddr_t)-1; len > 0; rva = va) {
        if ((len = kvamap_extract(rva, len, &va)) == 0) {
            break;  // Already mapped
        }

        if (dtlb_va_to_pa(va) == (u_long)-1 ||
            itlb_va_to_pa(va) == (u_long)-1) {

            // Allocate physical page
            if (pa == (paddr_t)-1) {
                pa = OF_alloc_phys(PAGE_SIZE_4M, PAGE_SIZE_4M);
                if (pa == (paddr_t)-1)
                    panic("out of memory");

                mva = OF_claim_virt(va, PAGE_SIZE_4M);
                if (mva != va) {
                    panic("can't claim virtual page");
                }
                continue;
            }

            // Create TLB entry
            data = SUN4U_TSB_DATA(0, PGSZ_4M, pa, 1, 1, 1, 1, 1, 0, 0);
            data |= SUN4U_TLB_L | SUN4U_TLB_CV;

            dtlb_store[dtlb_slot].te_pa = pa;
            dtlb_store[dtlb_slot].te_va = va;
            dtlb_slot++;
            dtlb_enter(va, hi(data), lo(data));
            pa = (paddr_t)-1;
        }

        kvamap_enter(va, PAGE_SIZE_4M);
        len -= len > PAGE_SIZE_4M ? PAGE_SIZE_4M : len;
        va += PAGE_SIZE_4M;
    }
    return 0;
}
```

**sun4v Implementation:**

```c
static int mmu_mapin_sun4v(vaddr_t rva, vsize_t len)
{
    uint64_t data;
    paddr_t pa;
    vaddr_t va, mva;
    int64_t hv_rc;

    for (pa = (paddr_t)-1; len > 0; rva = va) {
        if ((len = kvamap_extract(rva, len, &va)) == 0) {
            break;
        }

        if (pa == (paddr_t)-1) {
            pa = OF_alloc_phys(PAGE_SIZE_4M, PAGE_SIZE_4M);
            if (pa == (paddr_t)-1)
                panic("out of memory");

            mva = OF_claim_virt(va, PAGE_SIZE_4M);
            if (mva != va) {
                panic("can't claim virtual page");
            }
        }

        data = SUN4V_TSB_DATA(0, PGSZ_4M, pa, 1, 1, 1, 1, 1, 0, 0);
        data |= SUN4V_TLB_CV;

        dtlb_store[dtlb_slot].te_pa = pa;
        dtlb_store[dtlb_slot].te_va = va;
        dtlb_slot++;

        hv_rc = hv_mmu_map_perm_addr(va, data, MAP_DTLB);
        if (hv_rc != H_EOK) {
            panic("hv_mmu_map_perm_addr() failed");
        }

        kvamap_enter(va, PAGE_SIZE_4M);
        pa = (paddr_t)-1;
        len -= len > PAGE_SIZE_4M ? PAGE_SIZE_4M : len;
        va += PAGE_SIZE_4M;
    }
    return 0;
}
```

### TLB Finalization

Before jumping to the kernel, the bootloader finalizes TLB mappings:

```c
void sparc64_finalize_tlb_sun4u(u_long data_va)
{
    int i;
    int64_t data;

    // Remove write permissions from text pages
    // Add I-TLB entries for executable pages
    for (i = 0; i < dtlb_slot; i++) {
        if (dtlb_store[i].te_va >= data_va) {
            continue;  // Skip data pages
        }

        // Create read-only entry
        data = SUN4U_TSB_DATA(0, PGSZ_4M,
            dtlb_store[i].te_pa,
            1,      // privileged
            0,      // write=0 (read-only)
            1, 1, 1, 0, 0);
        data |= SUN4U_TLB_L | SUN4U_TLB_CV;

        // Update D-TLB (read-only)
        dtlb_replace(dtlb_store[i].te_va, hi(data), lo(data));

        // Add I-TLB entry
        itlb_store[itlb_slot] = dtlb_store[i];
        itlb_slot++;
        itlb_enter(dtlb_store[i].te_va, hi(data), lo(data));
    }
}
```

## Memory Map

### Physical Memory Layout

**sun4u Systems:**

```
0x0000000000000000 - 0x0000000000003FFF   Trap table (16 KB)
0x0000000000004000 - 0x00000000000FFFFF   Boot loader
0x0000000100000000 - END_OF_RAM           Main memory
0x00000001FC000000 - 0x00000001FFFFFFFF   I/O space (sun4u)
0xFFFFFFFFC0000000 - 0xFFFFFFFFFFFFFFFF   I/O space (high)
```

**sun4v Systems:**

```
0x0000000000000000 - Physical RAM          Guest physical memory
(Hypervisor manages actual physical addresses)
```

### Virtual Memory Layout

**Kernel Virtual Address Space:**

```
0x0000000000000000 - 0x000007FFFFFFFFFF   User space (32TB)
0x0000080000000000 - 0xFFFFFFFFFFFFFFFF   Kernel space

Kernel Layout:
0xC0000000 - 0xDFFFFFFF   Kernel text and data (typical)
0xE0000000 - 0xEFFFFFFF   Kernel heap and dynamic allocations
0xF0000000 - 0xFFBFFFFF   Device mappings
0xFFC00000 - 0xFFFFFFFF   OpenBoot PROM
```

### Boot Loader Memory Usage

The bootloader uses:

- **Text/Data:** Loaded by OpenBoot at PROM-determined address
- **Stack:** Provided by OpenBoot
- **Heap:** Dynamic allocation via OpenBoot services
- **Kernel:** Mapped at virtual addresses with 4MB pages
- **TLB Entries:**
  - sun4u: Typically 8-16 locked entries for kernel
  - sun4v: Similar, managed through hypervisor

### Memory Allocation Strategy

```c
// From loadfile_machdep.c
static struct kvamap {
    uint64_t start;
    uint64_t end;
} kvamap[MAXSEGNUM];  // Track allocated regions

// Check if region is already mapped
static uint64_t kvamap_extract(vaddr_t va, vsize_t len, vaddr_t *new_va)
{
    int i;
    *new_va = va;

    for (i = 0; (len > 0) && (i < MAXSEGNUM); i++) {
        if (kvamap[i].start == 0)
            break;
        if ((kvamap[i].start <= va) && (va < kvamap[i].end)) {
            uint64_t va_len = kvamap[i].end - va;
            len = (va_len < len) ? len - va_len : 0;
            *new_va = kvamap[i].end;
        }
    }
    return len;
}
```

## OpenBoot Commands and Device Aliases

### Common OpenBoot PROM Commands

#### Basic Boot Commands

```forth
ok boot                          # Boot default device/kernel
ok boot disk0                    # Boot from primary disk
ok boot net                      # Network boot
ok boot disk0:a netbsd          # Boot specific kernel
ok boot cdrom                    # Boot from CD-ROM
ok boot disk0 -s                # Single-user mode
ok boot disk0 -a                # Ask for root device
ok boot disk0 -d                # Boot with kernel debugger
ok boot disk0 -v                # Verbose boot
ok boot disk0 -c                # User kernel configuration
```

#### Advanced Boot Syntax

```forth
# Full device path boot
ok boot /pci@1f,0/scsi@1/disk@0,0:a netbsd

# Network boot with explicit protocol
ok boot net:dhcp

# Boot with multiple options
ok boot disk0 netbsd -asv       # Ask, single-user, verbose

# Boot alternate kernel
ok boot disk0 netbsd.old

# Boot from USB (if supported)
ok boot /pci@1e,600000/usb@a/disk@1
```

#### Device Management

```forth
ok show-devs                     # Show all devices in tree
ok devalias                      # Show/set device aliases
ok devalias disk0 /pci@1f,0/scsi@1/disk@0,0
ok printenv                      # Show NVRAM variables
ok setenv auto-boot? false      # Disable auto-boot
ok setenv boot-device disk0     # Set default boot device
ok setenv boot-file netbsd      # Set default kernel
ok setenv use-nvramrc? true     # Enable NVRAMRC scripts
```

#### System Information

```forth
ok .version                      # Show PROM version
ok .idprom                       # Show ID PROM contents
ok .traps                        # Show trap table
ok .speed                        # Show CPU speed
ok .memory                       # Show memory configuration
ok .maps                         # Show virtual memory mappings
ok .registers                    # Show CPU registers
```

### Device Tree Exploration

```forth
ok dev /                         # Go to root node
ok ls                           # List children
ok pwd                          # Show current path
ok .properties                  # Show current node properties
ok words                        # List Forth words

# Navigate device tree
ok dev /pci@1f,0                # Go to PCI bus
ok ls                           # List PCI devices
ok dev scsi@1                   # Go to SCSI controller
ok ls                           # List SCSI devices
ok dev disk@0,0                 # Go to disk
ok .properties                  # Show disk properties

# Query specific properties
ok dev /memory
ok reg .                        # Show memory regions

ok dev /cpus
ok ls                           # Show all CPUs
```

### Diagnostic Commands

```forth
ok test /memory                  # Test memory
ok test net                      # Test network
ok test /pci@1f,0/scsi@1        # Test SCSI controller
ok test-all                     # Test all testable devices
ok probe-scsi                   # Probe SCSI bus
ok probe-scsi-all              # Probe all SCSI buses
ok probe-ide                    # Probe IDE (if present)
ok watch-net                    # Monitor network traffic
ok watch-clock                  # Display clock ticks
```

### Memory and Register Inspection

```forth
# Memory operations
ok 0 20 dump                    # Dump memory from address 0
ok 1000000 100 dump            # Dump 256 bytes from 0x1000000
ok 1000 l@                     # Read long (64-bit)
ok 1000 x@                     # Read extended (64-bit)
ok 1000 w@                     # Read word (32-bit)
ok 1000 c@                     # Read byte

# Modify memory
ok deadbeef 1000 l!            # Write long to address
ok 42 1000 c!                  # Write byte

# Register access
ok %g0 .                       # Show global register 0
ok %g7 .                       # Show global register 7
ok %o0 .                       # Show output register 0
ok %pc .                       # Show program counter
ok %pstate .                   # Show processor state
```

### Stack and Execution

```forth
ok ctrace                       # C stack trace
ok .locals                      # Show local variables
ok ftrace                       # Forth execution trace
ok see [word]                  # Decompile Forth word
ok ' [word] dis                # Disassemble word
```

### Common Device Aliases

#### Disk Devices

```
disk0, disk1, ...   SCSI/SAS disks (typically)
disk                Primary boot disk
cdrom               CD-ROM/DVD drive
tape                Tape drive
```

#### Network Devices

```
net                 Primary network interface
net0, net1, ...     Additional interfaces
ge0, ge1, ...       Gigabit Ethernet
qfe0-qfe3          Quad FastEthernet
hme0               Happy Meal Ethernet
eri0               ERI Fast Ethernet
```

#### Serial and Console

```
ttya                Serial port A
ttyb                Serial port B
keyboard            Keyboard
screen              Framebuffer
virtual-console     Virtual console (Logical Domains)
```

#### USB Devices (if supported)

```
usb0, usb1, ...     USB controllers
disk@0              USB mass storage
```

### Boot Device Specification

Full syntax: `[device][:partition][/file] [options]`

Examples:

```forth
# Partition specification
ok boot disk0:a                      # Partition 'a'
ok boot disk0:b netbsd.old          # Partition 'b'
ok boot disk0:d netbsd-GENERIC      # Partition 'd'

# Full device paths
ok boot /pci@1f,0/scsi@1/disk@0,0:a
ok boot /pci@1e,600000/network@0

# Network boot
ok boot net                          # Default network boot
ok boot net:dhcp                     # DHCP
ok boot net:192.168.1.100           # Explicit server
ok boot net:,192.168.1.100,,netbsd  # RARP with arguments

# CD-ROM boot
ok boot cdrom
ok boot cdrom:f netbsd              # BSD partition on CD

# USB boot (if supported)
ok boot /pci@1e,600000/usb@a/disk@1:a
```

### Setting NVRAM Variables

#### Boot Configuration

```forth
# Auto-boot settings
ok setenv auto-boot? true
ok setenv boot-device disk0
ok setenv boot-file netbsd
ok setenv boot-command boot

# Multiple boot devices (failover)
ok setenv boot-device disk0 net

# Diagnostic mode
ok setenv diag-switch? true
ok setenv diag-device net
ok setenv diag-file netbsd-diag
```

#### Console Configuration

```forth
# Graphics console
ok setenv input-device keyboard
ok setenv output-device screen

# Serial console
ok setenv input-device ttya
ok setenv output-device ttya
ok setenv ttya-mode 9600,8,n,1,-

# Serial console on ttyb
ok setenv input-device ttyb
ok setenv output-device ttyb
ok setenv ttyb-mode 115200,8,n,1,-
```

#### Network Configuration

```forth
# Manual network setup
ok setenv network-boot-arguments host-ip=192.168.1.10,router-ip=192.168.1.1,subnet-mask=255.255.255.0,hostname=mysparc,file=netbsd

# Or simpler
ok setenv network-boot-arguments ip=192.168.1.10
```

#### Security

```forth
# Set security mode
ok setenv security-mode command    # Require password for commands
ok setenv security-mode full       # Require password for everything
ok setenv security-mode none       # No security (default)

# Set password (prompts for input)
ok password

# Clear password
ok setenv security-password
```

#### Power Management

```forth
ok setenv auto-power-on? true      # Power on when AC applied
ok setenv power-on-delay 30        # Delay before auto power-on
```

### NVRAMRC Scripts

Create custom boot scripts:

```forth
# Edit NVRAMRC
ok nvedit
  0: boot disk0 netbsd -v
  1: <Ctrl-C>
ok nvstore                         # Save changes
ok setenv use-nvramrc? true       # Enable execution

# View NVRAMRC
ok nvquit                         # Exit edit mode
ok printenv nvramrc               # Display contents
```

### Reset and Recovery

```forth
ok reset                          # Soft reset
ok reset-all                      # Hard reset (reinit PROM)
ok power-off                      # Power off system
ok set-defaults                   # Reset NVRAM to defaults
ok sync                           # Sync disks before reset
```

## Build and Installation Instructions

### Building the Bootloaders

#### Full System Build

```bash
# Build complete NetBSD/sparc64 system
cd /usr/src
./build.sh -m sparc64 -U tools
./build.sh -m sparc64 -U distribution

# Bootloaders are in:
# obj/usr/src/sys/arch/sparc/stand/
```

#### Building Only Bootloaders

```bash
# Build bootloader components
cd /usr/src/sys/arch/sparc/stand
make cleandir
make depend
make
make install DESTDIR=/

# Individual components:
cd /usr/src/sys/arch/sparc/stand/bootblk
make

cd /usr/src/sys/arch/sparc/stand/ofwboot
make

cd /usr/src/sys/arch/sparc/stand/binstall
make
```

### Installation Methods

#### Method 1: Using installboot (Recommended)

```bash
# Standard installation
installboot -v /dev/rsd0a /usr/mdec/bootblk /ofwboot

# Force installation even if disk label looks wrong
installboot -f /dev/rsd0a /usr/mdec/bootblk /ofwboot

# Clear existing boot blocks first
installboot -c /dev/rsd0a

# Specify filesystem type explicitly
installboot -t ffs /dev/rsd0a /usr/mdec/bootblk /ofwboot
```

**How installboot works:**

1. Reads the partition's filesystem superblock
2. Locates `/ofwboot` file within the filesystem
3. Reads the file's block list
4. Writes `bootblk` to the boot block area (sectors 1-15)
5. Embeds the block list into bootblk
6. bootblk will use this list to load ofwboot at boot time

#### Method 2: Using binstall Script

The `binstall.sh` script provides an alternative method:

```bash
# Location: /usr/mdec/binstall.sh
cd /usr/mdec
./binstall.sh net /dev/rsd0a

# Or for disk boot
./binstall.sh ffs /dev/rsd0a
```

#### Method 3: Manual Installation (Advanced)

```bash
# Write boot blocks manually
dd if=/usr/mdec/bootblk of=/dev/rsd0c bs=512 count=16 seek=1

# Ensure ofwboot is in root
cp /usr/mdec/ofwboot /
chmod 444 /ofwboot
```

**Warning:** Manual installation requires understanding disk layout and may not work correctly with all filesystem configurations.

### Installing on Different Filesystems

#### FFS/UFS Installation

```bash
# Create FFS filesystem
newfs /dev/rsd0a

# Mount and populate
mount /dev/sd0a /mnt
cd /mnt
# ... extract sets ...

# Install bootloader
installboot /dev/rsd0a /usr/mdec/bootblk /ofwboot
```

#### FFS2/UFS2 Installation

```bash
# Create FFS2 filesystem with 64-bit support
newfs -O 2 /dev/rsd0a

# Rest is same as FFS
mount /dev/sd0a /mnt
# ... populate ...
installboot /dev/rsd0a /usr/mdec/bootblk /ofwboot
```

#### LFS Installation

```bash
# Create LFS filesystem
newfs_lfs /dev/rsd0a

# Mount and populate
mount -t lfs /dev/sd0a /mnt
# ... populate ...

# Install bootloader (same command)
installboot /dev/rsd0a /usr/mdec/bootblk /ofwboot
```

### Disklabel Configuration

NetBSD/sparc64 requires a proper disklabel:

```bash
# Interactive disklabel editing
disklabel -e sd0

# Example layout:
#   a: 2097152    64  4.2BSD  2048  16384  # / (1GB)
#   b: 4194304  2097216  swap                # swap (2GB)
#   c: 143374738 0  unused 0 0              # NetBSD portion
#   d: 143374738 0  unused 0 0              # whole disk
```

**Important considerations:**

1. Partition 'a' should start at cylinder 0 or close to it
2. Partition 'c' represents the NetBSD portion of the disk
3. Partition 'd' represents the entire physical disk
4. Leave space at the beginning for Sun disklabel (8KB)

### Network Boot Setup

#### Server Configuration

**DHCP Configuration** (`/etc/dhcpd.conf`):

```
host sparc64-client {
    hardware ethernet 00:03:ba:12:34:56;
    fixed-address 192.168.1.100;
    filename "ofwboot";
    option root-path "/export/sparc64-root";
    server-name "boot-server.domain.com";
}
```

**RARP Configuration** (`/etc/ethers`):

```
00:03:ba:12:34:56  sparc64-client
```

**TFTP Setup**:

```bash
# Copy bootloader to TFTP directory
cp /usr/mdec/ofwboot /tftpboot/
chmod 644 /tftpboot/ofwboot

# For specific client (use hex MAC address)
cp /usr/mdec/ofwboot /tftpboot/C0A80164.SUN4U

# Start TFTP daemon
tftpd -l -s /tftpboot
```

**NFS Root Export** (`/etc/exports`):

```
/export/sparc64-root -alldirs -maproot=root sparc64-client
```

**bootparams** (`/etc/bootparams`):

```
sparc64-client root=boot-server:/export/sparc64-root
```

#### Client Boot Commands

```forth
# Simple network boot
ok boot net

# DHCP boot
ok boot net:dhcp

# RARP boot with explicit settings
ok setenv network-boot-arguments ip=192.168.1.100,file=netbsd
ok boot net

# Manual server specification
ok boot net:,192.168.1.1,,netbsd
```

### RAIDframe Boot Installation

NetBSD/sparc64 supports booting from RAIDframe level 1 mirrors:

```bash
# Create RAID1
raidctl -C raid0.conf raid0
raidctl -I 12345 raid0
raidctl -A root raid0

# Partition the RAID device
disklabel -e raid0

# Install bootloader on RAID components
installboot /dev/rwd0a /usr/mdec/bootblk /ofwboot
installboot /dev/rwd1a /usr/mdec/bootblk /ofwboot

# Newfs and populate
newfs /dev/rraid0a
mount /dev/raid0a /mnt
# ... extract sets ...
```

**Note:** The bootblk automatically detects RAIDframe partitions by checking for superblock at offset 64 sectors.

### USB Boot (Newer Systems)

Some sparc64 systems support USB booting:

```forth
# Find USB device
ok show-devs | grep usb
ok devalias usbdisk /pci@1e,600000/usb@a/disk@1

# Boot from USB
ok boot usbdisk:a

# Make persistent
ok setenv boot-device usbdisk
ok setenv auto-boot? true
```

### Troubleshooting Installation

#### Boot Block Verification

```bash
# Verify boot blocks are installed
dd if=/dev/rsd0a bs=512 count=1 | od -x | head

# Should see bootblk magic number
```

#### ofwboot Not Found

```bash
# Check if ofwboot exists in root
ls -l /ofwboot

# Verify it's accessible
file /ofwboot

# Reinstall if needed
cp /usr/mdec/ofwboot /
installboot /dev/rsd0a /usr/mdec/bootblk /ofwboot
```

#### Permission Issues

```bash
# Ensure proper permissions
chmod 444 /ofwboot
chown root:wheel /ofwboot
```

#### Filesystem Corruption

```bash
# Check filesystem
fsck -f /dev/rsd0a

# If severe corruption, reinstall
newfs /dev/rsd0a
mount /dev/sd0a /mnt
# restore from backup
installboot /dev/rsd0a /usr/mdec/bootblk /ofwboot
```

## Debugging with OpenBoot PROM

### Entering OpenBoot

#### From Running System

**Hardware Methods:**
- **SPARCstation:** Press Stop-A (L1-A on Sun keyboards)
- **Server with serial console:** Send BREAK signal
- **Remote serial:** Depends on terminal server (usually Ctrl-Shift-6 or similar)

**Software Method:**
```bash
# Requires OpenBoot support in kernel
shutdown -r now
# Press Stop-A during shutdown
```

#### Boot Interruption

```
# Hold Stop key during power-on
# Or press Stop-A immediately after POST
```

### Debug Output Control

```forth
# Enable verbose PROM output
ok setenv diag-switch? true

# Verbose boot
ok boot -V

# Extra debugging in bootloader
ok boot -D

# Both verbose and debug
ok boot -VD
```

### Memory Debugging

#### Memory Inspection

```forth
# Dump memory regions
ok .memory                        # Show memory config
ok dev /memory
ok reg                           # Show memory regions
ok available                     # Show available memory

# Dump physical memory
ok 0 1000 dump                   # Dump from physical 0
ok 1000000 100 dump             # Dump from 16MB

# Virtual memory translation
ok f0000000 mmu-translate       # Translate VA to PA
ok .maps                        # Show all mappings
```

#### TLB Inspection (sun4u)

```forth
# Dump TLB entries
ok .dtlb                        # Data TLB
ok .itlb                        # Instruction TLB

# More detailed TLB dump
ok dump-dtlb
ok dump-itlb

# Context information
ok .mmu                         # MMU state
ok .context                     # Current context
```

### CPU and Register Debugging

```forth
# CPU state
ok .registers                   # All registers
ok .locals                      # Local registers
ok .ins                         # Input registers
ok .outs                        # Output registers
ok .globals                     # Global registers

# Special registers
ok %pstate .                    # Processor state
ok %tstate .                    # Trap state
ok %pc .                        # Program counter
ok %npc .                       # Next PC
ok %pil .                       # Processor interrupt level
ok %cwp .                       # Current window pointer
ok %ver .                       # Version register

# Trap state
ok .traps                       # Show trap table
ok .trap-registers              # Trap register state
```

### Stack Traces

```forth
# Stack dumps
ok ctrace                       # C stack trace
ok .locals                      # Show stack frame
ok ftrace                       # Forth execution trace

# Window state
ok .windows                     # Show register windows
ok .window 0                    # Show specific window
```

### Breakpoint and Stepping

```forth
# Set breakpoints
ok f0004000 bp                  # Set breakpoint at address
ok .bp                          # List breakpoints
ok unbp                         # Remove all breakpoints

# Single stepping (limited support)
ok f0004000 dis                 # Disassemble
```

### Device Debugging

```forth
# Test devices
ok test /memory                 # Memory test
ok test net                     # Network test
ok test /pci@1f,0/scsi@1       # SCSI test
ok test-all                     # Test all devices

# Device properties
ok dev /pci@1f,0/scsi@1/disk@0,0
ok .properties                  # Show all properties
ok reg .                        # Register property
ok interrupts .                 # Interrupt property

# SCSI debugging
ok probe-scsi                   # Probe SCSI bus
ok probe-scsi-all              # Probe all buses
```

### Network Debugging

```forth
# Watch network traffic
ok watch-net                    # Monitor packets

# Network device info
ok dev net
ok .properties
ok local-mac-address .         # Show MAC address
ok address .                    # Alternative

# Test network
ok test net
ok test /pci@1f,0/network@1

# Network boot debugging
ok boot net -V                  # Very verbose network boot
```

### Boot Process Debugging

```forth
# Trace boot process
ok boot -V disk0                # Verbose PROM messages
ok boot -D disk0                # Debug bootloader
ok boot -d disk0                # Enable kernel DDB

# Combined
ok boot -VDd disk0              # Maximum debug output

# Stop before kernel
ok boot disk0                   # Let it fail or Ctrl-C
ok .registers                   # Check state
ok ctrace                       # See where we are
```

### Custom Debugging Scripts

```forth
# Define debugging function
ok : debug-boot
     .version cr
     .memory cr
     .registers cr
     boot disk0 -d
;
ok debug-boot                   # Execute

# Save to NVRAMRC
ok nvedit
  0: : my-debug
  1:   ." Starting debug boot" cr
  2:   .memory
  3:   boot disk0 netbsd -d
  4: ;
  5: <Ctrl-C>
ok nvstore
ok setenv use-nvramrc? true
```

### Analyzing Boot Failures

#### Cannot Find Bootloader

```forth
# Check boot device
ok printenv boot-device
ok devalias disk0

# Try explicit path
ok dev /pci@1f,0/scsi@1/disk@0,0
ok pwd                          # Verify path
ok boot /pci@1f,0/scsi@1/disk@0,0:a

# Directory listing (limited)
ok dev disk0
ok dir                          # May work on some systems
```

#### Bootloader Fails to Load Kernel

```forth
# Boot with debug output
ok boot -D disk0

# This shows:
# - File system type detected
# - Kernel file search
# - Memory allocation
# - Loading progress
```

#### Kernel Panics Early

```forth
# Boot to debugger
ok boot -d disk0

# If it reaches DDB:
db> show registers
db> trace
db> ps

# If it doesn't reach DDB:
# The problem is in early kernel init
# Check console for panic messages
```

### System-Specific Debug Features

#### sun4u Debug Features

```forth
# CPU-specific registers
ok .cpu-state                   # CPU state information
ok .error-state                 # Error registers
```

#### sun4v Debug Features

```forth
# Hypervisor debugging
ok .guest-state                 # Guest state
ok .hv-version                  # Hypervisor version

# Logical domains info (if applicable)
ok .ldom-variables
```

### Serial Console Debugging

For remote debugging:

```forth
# Configure serial console
ok setenv input-device ttya
ok setenv output-device ttya
ok setenv ttya-mode 115200,8,n,1,-
ok reset-all

# On remote system, connect:
$ cu -l /dev/ttyU0 -s 115200

# Or use tip:
$ tip -115200 /dev/ttyU0
```

### Firmware Update Debugging

```forth
# Check firmware version
ok .version
ok banner

# Flash ROM information
ok dev /flashprom
ok .properties

# Update firmware (carefully!)
# See Sun documentation for specific procedures
```

### Recovery Procedures

#### Cannot Access PROM

1. **Remove power**
2. **Locate NVRAM battery or jumper**
3. **Clear NVRAM:**
   - Remove battery for 10+ minutes, or
   - Short NVRAM clear jumper
4. **Restore power**
5. **PROM will boot with defaults**
6. **Reconfigure:**
   ```forth
   ok set-defaults
   ok setenv auto-boot? false
   ok reset-all
   ```

#### Password Recovery

If security-mode is enabled:

1. **Clear NVRAM** (as above)
2. Or use **hardware password bypass** (if available)
3. Or contact Sun support for master password

#### Corrupt NVRAM

```forth
ok set-defaults                 # Reset to factory defaults
ok setenv auto-boot? true
ok setenv boot-device disk0
ok setenv input-device keyboard
ok setenv output-device screen
ok reset-all
```

### Advanced Debugging

#### Custom Forth Code

```forth
# Define temporary debugging functions
ok : hexdump ( addr len -- )
     0 do
       dup i + c@ .
     loop drop
   ;
ok 100000 100 hexdump          # Dump 256 bytes

# More complex example
ok : find-magic ( addr len magic -- addr | 0 )
     rot rot 0 do
       2dup i + l@ = if
         drop i + unloop exit
       then
     loop
     2drop 0
   ;
```

#### Disassembly

```forth
# Disassemble code
ok f0004000 100 dis            # Disassemble 256 bytes
ok ' my-function dis           # Disassemble Forth word
```

## Advanced Topics

### Multi-Boot Configuration

Create `/boot.cfg` in root filesystem:

```
banner=NetBSD/sparc64 Boot Menu
timeout=10
default=1

menu=Boot NetBSD:boot netbsd
menu=Boot NetBSD (single user):boot netbsd -s
menu=Boot NetBSD (verbose):boot netbsd -v
menu=Boot NetBSD with DDB:boot netbsd -d
menu=Boot old kernel:boot netbsd.old
menu=Boot to OFW prompt:prompt
```

### Custom Bootloader Builds

Modify build configuration:

```bash
cd /usr/src/sys/arch/sparc/stand/ofwboot

# Edit Makefile to add options
# For example, enable debugging:
CPPFLAGS += -DDEBUG -DBOOT_DEBUG -DNETIF_DEBUG

# Rebuild
make clean
make
make install
```

### Cross-Building

Build sparc64 bootloaders on another architecture:

```bash
# From amd64, for example:
cd /usr/src
./build.sh -m sparc64 -U tools
./build.sh -m sparc64 -U distribution

# Bootloaders will be in:
# obj/sys/arch/sparc/stand/*/
```

### Logical Domains (LDoms)

For sun4v systems with Logical Domains:

```forth
# From primary domain, create guest:
ok> ldm create guest1
ok> ldm set-vcpu 4 guest1
ok> ldm set-memory 2G guest1
ok> ldm add-vnet vnet1 primary-vsw guest1
ok> ldm add-vdisk vdisk1 /dev/dsk/c1t0d0s2 guest1
ok> ldm bind guest1
ok> ldm start guest1

# Connect to guest console:
ok> telnet localhost 5000
```

### OpenBoot Forth Programming

Create custom boot scripts:

```forth
# NVRAMRC example
ok nvedit
  0: : netboot-fallback
  1:   boot net
  2:   if  exit  then
  3:   ." Network boot failed, trying disk" cr
  4:   boot disk0
  5: ;
  6: ' netboot-fallback  to  boot-command
  7: <Ctrl-C>
ok nvstore
ok setenv use-nvramrc? true
```

## References

### Source Code Locations

- `/sys/arch/sparc/stand/` - Boot loader source (shared)
- `/sys/arch/sparc/stand/bootblk/` - First-stage (Forth)
- `/sys/arch/sparc/stand/ofwboot/` - Second-stage
- `/sys/arch/sparc/stand/ofwboot/loadfile_machdep.c` - **sparc64-specific MMU code**
- `/sys/arch/sparc64/stand/` - Build configuration
- `/sys/arch/sparc64/include/` - Architecture headers
- `/sys/arch/sparc64/sparc64/locore.s` - Kernel entry point

### Documentation

- `installboot(8)` - Boot block installation
- `boot(8)` - System bootstrapping procedures
- `openprom(4)` - OpenBoot PROM interface
- `eeprom(8)` - NVRAM variable modification
- IEEE 1275-1994 - Open Firmware Standard
- IEEE 1275.1 - Open Firmware SPARC Supplement

### Hardware References

- UltraSPARC Architecture Manual
- UltraSPARC IIi User's Manual
- UltraSPARC III Cu User's Manual
- UltraSPARC IV User's Manual
- UltraSPARC T1 Supplement
- UltraSPARC T2 Supplement
- OpenBoot PROM Command Reference
- Sun System Handbook (all models)

### Online Resources

- NetBSD/sparc64 FAQ: https://www.NetBSD.org/ports/sparc64/faq.html
- NetBSD Installation Guide
- Sun OpenBoot documentation
- SPARC International documentation
