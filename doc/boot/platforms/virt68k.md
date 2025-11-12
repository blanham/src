# NetBSD/virt68k Boot Documentation

## Platform Overview

NetBSD/virt68k is a port to QEMU's m68k virtual machine platform. Unlike physical m68k platforms, virt68k runs entirely in the QEMU emulator, providing a virtual 68k system for development, testing, and education.

**Virtual Hardware:**
- CPU: Emulated 68040 or 68060 (QEMU setting)
- RAM: Configurable (typically 128MB-1GB)
- Graphics: Goldfish framebuffer (virtual)
- Storage: virtio-blk (virtual block device)
- Network: virtio-net (virtual network device)
- Serial: virtio-serial or goldfish-tty
- RTC: Goldfish RTC

**Advantages:**
- No physical hardware required
- Easy deployment and testing
- Snapshots and save states
- Network and storage via host
- Modern development on vintage architecture

## Boot Method

QEMU's virt-m68k machine boots from a kernel image loaded directly by QEMU, with bootinfo passed via a special memory structure.

### Boot Chain

1. **QEMU Initialization** - QEMU sets up virtual hardware
2. **Direct Kernel Load** - Kernel loaded to RAM by QEMU
3. **Bootinfo Structure** - QEMU provides boot parameters
4. **NetBSD Kernel** - Kernel starts at entry point

**No Traditional Bootloader:**
- QEMU acts as the "firmware"
- Kernel loaded directly by QEMU
- No BIOS/PROM/ROM monitor stage

## Starting NetBSD/virt68k with QEMU

### Basic QEMU Command

```bash
qemu-system-m68k \
    -M virt \
    -cpu m68040 \
    -m 128M \
    -kernel netbsd-GENERIC \
    -drive file=netbsd-disk.img,format=raw \
    -serial stdio
```

### QEMU Options Explained

**Machine and CPU:**
```bash
-M virt          # Use virt-m68k machine type
-cpu m68040      # Emulate 68040 (or m68060)
```

**Memory:**
```bash
-m 128M          # 128MB RAM (adjust as needed)
```

**Kernel:**
```bash
-kernel netbsd-GENERIC     # NetBSD kernel ELF image
-append "root=wd0a"        # Kernel command line (optional)
```

**Storage:**
```bash
-drive file=disk.img,format=raw,if=none,id=hd0
-device virtio-blk-device,drive=hd0
```

**Network:**
```bash
-netdev user,id=net0
-device virtio-net-device,netdev=net0
```

**Serial Console:**
```bash
-serial stdio    # Serial to stdout/stdin
-nographic       # No graphical window (terminal only)
```

### Complete Example

```bash
qemu-system-m68k \
    -M virt \
    -cpu m68040 \
    -m 256M \
    -kernel /path/to/netbsd \
    -drive file=netbsd-root.img,format=raw,if=none,id=hd0 \
    -device virtio-blk-device,drive=hd0 \
    -netdev user,id=net0,hostfwd=tcp::2222-:22 \
    -device virtio-net-device,netdev=net0 \
    -serial stdio \
    -nographic
```

**This provides:**
- 256MB RAM
- virtio disk
- virtio network with SSH forwarding (host:2222 → guest:22)
- Serial console on terminal

## 68040/68060 MMU (Emulated)

QEMU emulates either 68040 or 68060 CPU, selected with `-cpu` flag.

### 68040 Emulation

**Features:**
- Full 68040 MMU emulation
- Integrated FPU
- Separate I/D TLBs
- Cache simulation

**QEMU CPU:**
```bash
-cpu m68040      # Default for virt-m68k
```

### 68060 Emulation

**Features:**
- 68060 MMU emulation
- Superscalar simulation
- Branch prediction
- Enhanced caches

**QEMU CPU:**
```bash
-cpu m68060      # Advanced users
```

**MMU Setup (Same as Physical 68040):**
```assembly
| 68040/060 MMU initialization
.long   0x4e7b1807              | movc d1,srp
.word   0xf4f8                  | cpusha bc
.word   0xf518                  | pflusha
movl    #MMU40_TCR_BITS,%d0
.long   0x4e7b0003              | movc d0,tc
```

**Note:** QEMU's emulation is functionally accurate but not cycle-accurate.

## Boot Process Stages

### Stage 1: QEMU Initialization

**QEMU Actions:**
1. Create virtual m68k machine
2. Initialize virtual devices
3. Allocate RAM (specified by -m)
4. Load kernel ELF image to RAM
5. Parse bootinfo structure
6. Setup initial CPU state
7. Set PC to kernel entry point

**No firmware/BIOS involved** - QEMU directly loads and starts kernel.

### Stage 2: Bootinfo Passing

**Bootinfo Structure:**

QEMU creates a bootinfo structure in memory containing:
```c
struct bootinfo {
    uint32_t bi_magic;      // Magic number
    uint32_t bi_version;    // Bootinfo version
    uint32_t bi_machtype;   // Machine type (virt68k)
    uint32_t bi_cputype;    // CPU type (040 or 060)
    uint32_t bi_fputype;    // FPU type
    uint32_t bi_mmutype;    // MMU type
    uint32_t bi_memsize;    // RAM size
    uint32_t bi_vram_addr;  // Framebuffer address
    uint32_t bi_vram_size;  // Framebuffer size
    // Additional device info...
};
```

**Location:** Passed to kernel via register or fixed address

### Stage 3: Kernel Entry (locore.s)

**Location:** `/sys/arch/virt68k/virt68k/locore.s`

**Entry Point:** `start` at 0x2000 (offset 0x400 reserved for vectors)

**Initial Code:**
```assembly
    .text
GLOBAL(kernel_text)

/*
 * Memory starts at 0x0, kernel linked at 0x2000
 * VA==PA initially
 */
    .data
    .space  PAGE_SIZE
ASLOCAL(tmpstk)

    .globl  _C_LABEL(edata)
    .globl  _C_LABEL(etext),_C_LABEL(end)

    .text
GLOBAL(kernel_text)

ASENTRY_NOPROFILE(start)
    movw    #PSL_HIGHIPL,%sr    | no interrupts

    /*
     * Determine relocation offset
     * We're loaded where QEMU put us,
     * need to calculate offset from link address
     */
    lea     %pc@(_ASM_LABEL(start)), %a5
    movl    %a5,%d0             | physical address
    subl    #_ASM_LABEL(start), %d0 | subtract virtual
    movl    %d0, %a5            | %a5 = relocation offset

    /*
     * NOTE: Can't use globals until MMU enabled
     * Use RELOC() macro to translate addresses
     */

    ASRELOC(tmpstk, %a0)
    movl    %a0,%sp             | temp stack

    /* Clear BSS */
    RELOC(edata,%a0)
    movl    #_C_LABEL(end) - 4, %d0
    subl    #_C_LABEL(edata), %d0
    lsrl    #2,%d0
1:  clrl    %a0@+
    dbra    %d0,1b

    /* Initialize cache */
    movl    #CACHE_OFF,%d0
    movc    %d0,%cacr

    /*
     * Parse bootinfo from QEMU
     * Location passed in %a4 or at fixed address
     */
    pea     %a5@                | reloff
    pea     %a4@                | bootinfo address
    RELOC(bootinfo_startup1,%a0)
    jbsr    %a0@                | bootinfo_startup1()
    addql   #8,%sp

    /*
     * Initialize MMU
     */
    pea     %a5@                | reloff
    movl    %d0,%sp@-           | nextpa (from bootinfo)
    RELOC(pmap_bootstrap1,%a0)
    jbsr    %a0@                | pmap_bootstrap1()
    addql   #8,%sp

    /* Enable MMU and caches */
    /* CPU-specific (040 or 060) */

    /* Setup source/destination control */
    moveq   #FC_USERD,%d0
    movc    %d0,%sfc
    movc    %d0,%dfc

    /* Jump to high kernel code */
    jmp     _C_LABEL(main):l
```

### Stage 4: Machine Init

**Location:** `/sys/arch/virt68k/virt68k/machdep.c`

**Initialization:**
1. Parse bootinfo completely
2. Initialize virtual devices:
   - virtio-blk disk driver
   - virtio-net network driver
   - goldfish-tty serial
   - goldfish-rtc clock
   - goldfish-fb framebuffer (if graphics)
3. Setup interrupt system
4. Mount root filesystem (virtio disk)
5. Start init

**Device Discovery:**
- QEMU provides device tree or fixed addresses
- No device probing like physical hardware
- All devices at known locations

## Memory Map

```
Virtual Machine Memory Layout:

0x00000000 - 0x00001FFF    Reserved (vectors, etc.)
0x00002000 - ...           Kernel text/data/bss
                           (kernel linked at 0x2000)
...                        Kernel heap
...                        User space

RAM size: As specified by QEMU -m option

Device MMIO regions (examples):
0x10000000 - ...           virtio-mmio devices
0xFF000000 - ...           goldfish devices
                           (addresses assigned by QEMU)
```

**Memory is contiguous** - No gaps like physical machines

## Build and Installation

### Building Kernel

```bash
cd /usr/src/sys/arch/virt68k/conf
config GENERIC
cd ../compile/GENERIC
make depend && make
```

**Result:** ELF kernel `netbsd`

### Creating Disk Image

**Create empty disk:**
```bash
# Create 2GB disk image
dd if=/dev/zero of=netbsd-virt68k.img bs=1M count=2048
```

**Partition and format:**
```bash
# From NetBSD or cross-tools
vnconfig vnd0 netbsd-virt68k.img
disklabel -e vnd0
newfs /dev/rvnd0a
mount /dev/vnd0a /mnt
# Install NetBSD sets
cd /mnt
tar xzpf /path/to/base.tgz
tar xzpf /path/to/etc.tgz
# ... more sets
umount /mnt
vnconfig -u vnd0
```

### Running

```bash
qemu-system-m68k \
    -M virt \
    -cpu m68040 \
    -m 256M \
    -kernel netbsd \
    -drive file=netbsd-virt68k.img,format=raw \
    -serial stdio \
    -nographic
```

## Debugging

### GDB Debugging

**Start QEMU with GDB stub:**
```bash
qemu-system-m68k \
    -M virt \
    -kernel netbsd \
    -s \         # GDB stub on port 1234
    -S \         # Pause at start
    -nographic
```

**Connect GDB:**
```bash
m68k-elf-gdb netbsd
(gdb) target remote localhost:1234
(gdb) continue
```

**GDB Commands:**
```
(gdb) break main
(gdb) info registers
(gdb) x/10i $pc
(gdb) disassemble
```

### QEMU Monitor

**Access monitor:**
```
Ctrl-A c         # Switch to QEMU monitor
```

**Useful commands:**
```
(qemu) info registers    # Show CPU state
(qemu) info mem          # Memory mappings
(qemu) info tlb          # TLB entries
(qemu) x /10x 0x2000    # Examine memory
(qemu) system_reset      # Reset VM
(qemu) quit              # Exit QEMU
```

### DDB (Kernel Debugger)

**Enable:**
```
options DDB
```

**Break:**
- Send BREAK on serial: Ctrl-A c, then sendkey ctrl-break
- Programmed breakpoint
- Panic

## Performance and Optimization

### CPU Selection

**68040:**
- Good compatibility
- Stable emulation
- Default choice

**68060:**
- Faster (if emulated faster)
- More modern
- May have quirks

### Memory Size

**Recommendations:**
```
Minimal: 64MB
Standard: 128MB
Comfortable: 256MB
Development: 512MB-1GB
```

### Storage Performance

**virtio-blk is fastest:**
```bash
-drive file=disk.img,format=raw,if=none,id=hd0,cache=writeback
-device virtio-blk-device,drive=hd0
```

### Network Performance

**virtio-net is fastest:**
```bash
-netdev user,id=net0
-device virtio-net-device,netdev=net0
```

## Use Cases

### Development

**Advantages:**
- Fast iteration
- Easy snapshots
- Host filesystem sharing
- GDB integration

**Setup for development:**
```bash
qemu-system-m68k \
    -M virt \
    -kernel netbsd \
    -m 512M \
    -drive file=devdisk.img,format=qcow2 \
    -netdev user,id=net0,hostfwd=tcp::2222-:22 \
    -device virtio-net-device,netdev=net0 \
    -serial mon:stdio \
    -nographic
```

### Testing

**Automated testing:**
```bash
#!/bin/bash
qemu-system-m68k \
    -M virt \
    -kernel testkernel \
    -m 128M \
    -nographic \
    -serial file:test-output.log \
    -no-reboot \
    -append "test_mode=1"
```

### Education

**Learning m68k architecture:**
- No hardware required
- Safe experimentation
- GDB for instruction tracing
- Can break and restart easily

### CI/CD Integration

**Example GitLab CI:**
```yaml
test:
  script:
    - qemu-system-m68k -M virt -kernel netbsd -nographic -no-reboot
```

## References

**Source Files:**
- `/sys/arch/virt68k/` - Port source tree
- `/sys/arch/virt68k/virt68k/locore.s` - Kernel entry
- `/sys/arch/virt68k/virt68k/machdep.c` - Machine init

**QEMU Documentation:**
- QEMU m68k System Emulation
- QEMU virt-m68k machine documentation
- QEMU Monitor Protocol reference

**Tools:**
- QEMU m68k system emulator
- m68k cross-compiler toolchain
- GDB for m68k

## Tips and Tricks

**Save/Restore State:**
```
(qemu) savevm snapshot1
(qemu) loadvm snapshot1
```

**Screenshot:**
```
(qemu) screendump screenshot.ppm
```

**Record/Replay:**
```bash
# Record
qemu-system-m68k ... -icount shift=7,rr=record,rrfile=replay.bin

# Replay
qemu-system-m68k ... -icount shift=7,rr=replay,rrfile=replay.bin
```

**Host filesystem sharing:**
```bash
# Use virtio-9p (if supported)
-virtfs local,path=/host/path,mount_tag=host,security_model=none
```
