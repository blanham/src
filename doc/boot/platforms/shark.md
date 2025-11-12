# NetBSD/shark Boot Process Documentation

## Platform Overview

NetBSD/shark supports the Digital Network Appliance Reference Design (DNARD), also known as the "Shark":
- Digital StrongARM SA-110 processor (233 MHz)
- Open Firmware (IEEE 1275-1994 compliant)
- PCI expansion bus
- ISA bus support
- VGA graphics
- Standard PC peripherals
- IDE and SCSI storage

The Shark was designed as a reference platform for network appliances and thin clients, featuring Open Firmware similar to SPARC systems but on ARM hardware.

## Boot Method

### OpenFirmware Bootloader

NetBSD/shark uses **ofwboot**, a standalone bootloader that runs under Open Firmware. This is similar to bootloaders on SPARC and PowerPC platforms.

Location: `/sys/arch/shark/stand/ofwboot/`

**Boot Flow:**
1. Open Firmware initializes hardware
2. Open Firmware loads ofwboot
3. ofwboot loads NetBSD kernel
4. ofwboot transfers control to kernel

### OpenFirmware Capabilities

Open Firmware provides:
- Device tree abstraction
- Boot device selection
- Network protocols (BOOTP, TFTP)
- Console management
- Device drivers
- Scriptable boot process

## Boot Process Stages

### Stage 1: Open Firmware Initialization

On power-on or reset:

1. **Hardware POST**
   - Initialize StrongARM SA-110
   - Configure memory controller
   - Detect and size SDRAM
   - Initialize PCI bus
   - Enumerate devices
   - Setup ISA bridge

2. **Build Device Tree**
   - Create hierarchical device representation
   - Populate with hardware properties
   - Store device information
   - Available via OF methods

3. **Display Banner**
   ```
   OpenFirmware 2.x
   Copyright (C) Digital Equipment Corporation

   Shark Network Computer
   StrongARM SA-110 @ 233 MHz
   Memory: 32 MB
   ```

4. **Initialize Console**
   - Setup VGA display (default)
   - Or serial console if configured
   - Initialize keyboard

### Stage 2: Boot Device Selection

**Open Firmware Boot Process:**

1. **Check Boot Configuration**
   ```
   ok boot [device][:partition][,filename] [arguments]
   ```

2. **Default Boot**
   ```
   ok boot
   # Uses "boot-device" and "boot-file" OF variables
   ```

3. **Interactive Mode**
   ```
   Press Stop-A or appropriate key during boot

   ok help
   ok printenv
   ok setenv boot-device disk:a
   ok setenv boot-file netbsd
   ok boot
   ```

**Common Boot Devices:**
```
disk        - IDE disk (wd0)
cd          - CD-ROM
net         - Network (TFTP)
floppy      - Floppy drive (if present)
```

### Stage 3: ofwboot Loading

Open Firmware loads ofwboot:

1. **Locate Bootloader**
   - Typically at beginning of boot partition
   - Or specified location in OF device tree
   - Path: `disk:a,ofwboot` or similar

2. **Load ofwboot**
   ```
   Loading: ofwboot
   Size: XXXXX bytes
   ```

3. **Transfer Control**
   - OF claims memory for ofwboot
   - Maps ofwboot to virtual address
   - Calls ofwboot entry point

4. **ofwboot Entry State**
   ```
   r0:        Open Firmware callback pointer
   PC:        ofwboot entry point
   Mode:      Depends on OF (typically virtual)
   ```

### Stage 4: ofwboot Execution

**File: `sys/arch/shark/stand/ofwboot/boot.c:main()`**

ofwboot bootloader:

1. **Initialize**
   ```c
   main(void)
   {
       printf(">> NetBSD/shark OpenFirmware Boot\n");
       printf(">> Version %s\n", bootprog_rev);
   ```

2. **Get Boot Parameters**
   ```c
   // Query Open Firmware
   chosen = OF_finddevice("/chosen");
   OF_getprop(chosen, "bootpath", bootdev, ...);
   OF_getprop(chosen, "bootargs", bootline, ...);
   ```

3. **Parse Arguments**
   ```c
   parseargs(bootline, &boothowto);
   // Extract boot flags: -a, -s, -d, -v, etc.
   ```

4. **Display Configuration**
   ```
   Boot device: disk:a
   Boot file: /netbsd
   Boot flags: -s
   ```

### Stage 5: Kernel Loading

**ofwboot loads kernel:**

1. **Allocate Memory**
   ```c
   // Claim memory from OF for kernel
   startbuf = OF_claim((void *)0xF0100000, 5*1024*1024, 0);
   ```

   Per ARM OF bindings:
   - OF allocates 6MB at 0xF0000000
   - ofwboot uses 0xF0000000
   - Kernel loaded at 0xF0100000

2. **Open Kernel File**
   ```c
   // Try each kernel in sequence
   for (i = 0; kernels[i]; i++) {
       marks[MARK_START] = 0xF0100000;
       if (loadfile(kernels[i], marks, LOAD_KERNEL) >= 0)
           break;
   }
   ```

   Default kernels tried:
   - `/netbsd`
   - `/netbsd.gz`
   - `/netbsd.shark`

3. **Load ELF Segments**
   ```c
   loadfile() from libsa:
   - Parse ELF header
   - Load program segments
   - Load symbol table
   - Record entry point and bounds
   ```

4. **Release Excess Memory**
   ```c
   // Free unused memory back to OF
   cp = (marks[MARK_END] + 0xFFF) & ~0xFFF;
   size = endbuf - cp;
   if (size)
       OF_release(cp, size);
   ```

### Stage 6: Prepare Boot Arguments

**Build Boot Argument String:**

```c
// Construct boot arguments for kernel
strcpy(bootline, opened_name);
cp = bootline + strlen(bootline);
*cp++ = ' ';
*cp = '-';

// Add boot flags
if (boothowto & RB_ASKNAME)
    *++cp = 'a';
if (boothowto & RB_SINGLE)
    *++cp = 's';
if (boothowto & RB_KDB)
    *++cp = 'd';
// etc.
```

### Stage 7: Kernel Entry

**Transfer control to kernel:**

```c
void chain(entry_point, args, ssym, esym)
{
    // Call OF to chain to new program
    OF_chain((void *)RELOC,      // bootloader location
             end - (char *)RELOC, // bootloader size
             entry,               // kernel entry point
             args,                // argument string
             arg_length);         // argument length
}
```

**Arguments passed to kernel:**
- Boot arguments string
- Symbol table pointers (ssym, esym)
- Magic number (0x19730224) for identification

**Kernel Entry State:**
```
r0:        Argument structure pointer
PC:        Kernel entry point (from ELF)
OF:        Still active (can make OF calls initially)
MMU:       Enabled (OF mappings)
```

### Stage 8: Kernel Initialization

Kernel takes over from ofwboot:

1. **Early Initialization**
   - Parse boot arguments
   - Extract symbol table pointers
   - Validate boot parameters

2. **Interact with Open Firmware**
   - Query device tree
   - Get memory information
   - Identify devices
   - Configure hardware

3. **Create Native Page Tables**
   - Build kernel page tables
   - Map kernel memory
   - Map devices
   - Prepare to disable OF

4. **Switch to Native Operation**
   - Exit Open Firmware
   - Enable native MMU mappings
   - Take over interrupt handling
   - Full kernel operation

## ARM MMU Setup Requirements

### StrongARM SA-110

**Control Register (CP15 c1):**
```
Bit 0:  M - MMU enable
Bit 2:  C - Data cache enable
Bit 3:  W - Write buffer enable
Bit 7:  B - Big-endian
Bit 9:  R - ROM protection
Bit 11: Z - Branch prediction
Bit 12: I - Instruction cache enable
Bit 13: V - High vectors
```

**SA-110 Cache:**
- 16KB instruction cache (32-way)
- 16KB data cache (32-way, write-back)
- 8-entry write buffer

**Cache Operations:**
```
Sync I-cache:        MCR p15, 0, r0, c7, c5, 0
Clean D-cache line:  MCR p15, 0, r0, c7, c10, 1
Drain write buffer:  MCR p15, 0, r0, c7, c10, 4
```

### Open Firmware MMU Management

**OF Memory Management:**
- OF sets up initial page tables
- Provides memory allocation (claim/release)
- Maps I/O devices
- Kernel inherits OF mappings initially

**Transition:**
1. Kernel operates under OF MMU initially
2. Creates its own page tables
3. Copies necessary mappings
4. Exits OF and switches to native MMU

## Memory Map

### Physical Memory Layout

```
0x00000000 - 0x00FFFFFF : Boot ROM/Flash (16MB)
0x08000000 - 0x0FFFFFFF : PCI Memory space
0x10000000 - 0x1FFFFFFF : DRAM (up to 256MB)
0x40000000 - 0x4FFFFFFF : PCI I/O space
0x50000000 - 0x5FFFFFFF : ISA I/O space
0x60000000 - 0x6FFFFFFF : PCI Config space
```

### Virtual Memory Layout (under OF)

```
0xF0000000 - 0xF00FFFFF : ofwboot (1MB)
0xF0100000 - 0xF5FFFFFF : Kernel space (5MB)
```

### Virtual Memory Layout (Native)

```
0xF0000000 - 0xF0FFFFFF : Kernel text/data
0xF1000000 - 0xFCFFFFFF : Kernel VM
0xFD000000 - 0xFFFFFFFF : Device mappings
```

## Build and Installation

### Building ofwboot

```bash
cd /sys/arch/shark/stand/ofwboot
make depend && make
```

Output: `ofwboot` (ELF binary for OF loading)

### Building Kernel

```bash
cd /sys/arch/shark/conf
config GENERIC
cd ../compile/GENERIC
make depend && make
```

Output: `netbsd` (ELF kernel)

### Installation

#### Method 1: IDE Disk Installation

1. **Partition Disk**
   ```bash
   fdisk -u wd0
   disklabel -e wd0
   ```

2. **Install Boot Blocks**
   ```bash
   # Install boot blocks
   installboot /dev/rwd0a /usr/mdec/ofwboot /usr/mdec/bootxx_ffs
   ```

3. **Copy Kernel**
   ```bash
   mount /dev/wd0a /mnt
   cp netbsd /mnt/
   umount /mnt
   ```

4. **Configure Open Firmware**
   ```
   ok setenv boot-device disk:a
   ok setenv boot-file netbsd
   ok setenv auto-boot? true
   ok reset-all
   ```

#### Method 2: Network Boot

1. **Setup TFTP Server**
   ```bash
   # Copy ofwboot and kernel to TFTP directory
   cp ofwboot /tftpboot/
   cp netbsd /tftpboot/

   # Start TFTP server
   in.tftpd -l -s /tftpboot
   ```

2. **Configure BOOTP/DHCP**
   ```bash
   # Add Shark MAC address to dhcpd.conf
   host shark {
       hardware ethernet XX:XX:XX:XX:XX:XX;
       fixed-address 192.168.1.100;
       filename "ofwboot";
   }
   ```

3. **Boot from Network**
   ```
   ok boot net
   ```

#### Method 3: CD-ROM Boot

1. **Create Bootable CD**
   ```bash
   # Create ISO with ofwboot and kernel
   mkisofs -R -J -o netbsd.iso \
           -b ofwboot \
           -c boot.catalog \
           cdroot/
   ```

2. **Boot from CD**
   ```
   ok boot cd:,ofwboot cd:,netbsd
   ```

## Debugging

### Open Firmware Console

**Accessing OF Console:**

Press Stop-A (or configured key) during boot:

```
ok
```

**Useful Commands:**

```
ok help                    - List commands
ok printenv                - Show OF variables
ok setenv var value        - Set variable
ok devalias                - Show device aliases
ok dev /                   - Navigate device tree
ok ls                      - List devices
ok .properties             - Show device properties
ok boot net -v             - Verbose boot from network
ok words                   - List OF words
```

**Device Tree Navigation:**

```
ok dev /                   - Go to root
ok ls                      - List children
ok dev pci                 - Enter pci node
ok .properties             - Show properties
ok ..                      - Go to parent
```

### Serial Console

**Configure Serial Console:**

```
ok setenv output-device ttya
ok setenv input-device ttya
ok reset-all
```

**Serial Parameters:**
- Speed: 9600 baud (default)
- Format: 8N1
- No flow control

### Common Issues

**Issue: "Can't open boot device"**
- Cause: Wrong device specification
- Solution: Check device aliases
- Command: `ok devalias` to see available devices

**Issue: "ofwboot not found"**
- Cause: Boot blocks not installed
- Solution: Run installboot again
- Verify: Check partition is active

**Issue: "Can't load kernel"**
- Cause: Kernel file missing or corrupted
- Solution: Verify /netbsd exists
- Try: Alternative kernels (/netbsd.gz, /netbsd.shark)

**Issue: Kernel loads but hangs**
- Cause: Hardware detection failure
- Solution: Use verbose boot (-v flag)
- Check: Device tree for hardware info

**Issue: Network boot fails**
- Cause: BOOTP/TFTP configuration
- Solution: Verify network settings
- Test: `ok ping 192.168.1.1`

### Debugging ofwboot

**Enable Debug Output:**

In `boot.c`:
```c
#define DEBUG
// Enables DPRINTF macro
```

Recompile:
```bash
make clean && make DEBUG_FLAGS=-DDEBUG
```

**Debug Output:**
```
DPRINTF("bootline=%s\n", bootline);
DPRINTF("Trying %s\n", kernels[i]);
DPRINTF("Calling OF_chain(...)\n");
```

### Kernel Debug Options

**Build Debug Kernel:**

In kernel config:
```
options     DEBUG
options     DDB
options     DIAGNOSTIC
makeoptions DEBUG="-g"
```

**Verbose Boot:**
```
ok boot -v
```

**Kernel Messages:**
```
NetBSD/shark booting...
initarm: Configuring system...
OF device tree:
/pci@...
mainbus0 (root)
...
```

## Technical Notes

### Open Firmware Standard

Shark uses IEEE 1275-1994 Open Firmware:
- Device tree representation
- Forth-based command language
- Client interface for loaded programs
- Standard boot protocols

**OF Client Interface:**

```c
// ofwboot uses OF client interface:
int OF_finddevice(const char *name);
int OF_getprop(int node, const char *name, void *buf, int buflen);
void *OF_claim(void *virt, u_int size, u_int align);
void OF_release(void *virt, u_int size);
int OF_open(const char *device);
int OF_read(int handle, void *buf, int len);
```

### ARM OpenFirmware Bindings

**ARM OF Boot Protocol:**
- OF must allocate and map 6MB at 0xF0000000
- Bootloader loaded at 0xF0000000
- Kernel loaded at 0xF0100000
- Memory after kernel can be unmapped

### OF Chain Operation

**OF_chain() Function:**

Allows bootloader to chain to kernel:
1. Preserves OF callbacks
2. Transfers control
3. Passes arguments
4. Kernel can still call OF initially

Eventually kernel exits OF:
```c
OF_exit();  // Never returns
```

### Cache Synchronization

**Critical during boot:**

1. **After loading kernel:**
   ```c
   if (cache_syncI != NULL)
       (*cache_syncI)();
   ```

2. **Before jumping to kernel:**
   - Ensure all loads are complete
   - Clean data cache
   - Invalidate instruction cache
   - Drain write buffer

### Device Tree Usage

**Kernel uses OF device tree:**
- Enumerate hardware
- Get device properties
- Configure drivers
- Match hardware to drivers

**Example:**
```c
// Find device in OF tree
node = OF_finddevice("/pci/ethernet");

// Get MAC address
OF_getprop(node, "local-mac-address", mac, 6);
```

### Comparison to Other Platforms

**Similar to:**
- SPARC (also uses OF)
- PowerPC (also uses OF)

**Different from:**
- x86 (BIOS-based)
- Most ARM (firmware or no loader)

Shark is unique ARM platform with OF.

## Supported Hardware

**Well-Supported:**
- StrongARM SA-110 CPU
- PCI bus and devices
- IDE disk controllers
- Common Ethernet cards
- VGA graphics
- Serial ports

**Partially Supported:**
- Some PCI cards
- ISA devices

**Not Supported:**
- Some proprietary devices

## References

- `/sys/arch/shark/stand/ofwboot/boot.c` - ofwboot main code
- `/sys/arch/shark/stand/ofwboot/Locore.c` - Low-level OF interface
- `/sys/arch/shark/stand/ofwboot/ofdev.c` - OF device access
- `/sys/arch/shark/shark/` - Kernel initialization
- IEEE 1275-1994: Standard for Boot Firmware (Open Firmware)
- Intel StrongARM SA-110 Microprocessor Technical Reference
- Digital DNARD Hardware Reference Manual
