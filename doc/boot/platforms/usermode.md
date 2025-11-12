# NetBSD/usermode Boot Documentation

$NetBSD$

## Platform Overview

**Architecture**: User-mode (process-based virtualization)
**Port Date**: 2007-11-20 (Jared McNeill), significantly enhanced 2011 (Reinoud Zandijk)
**Boot Method**: Direct process execution on host OS
**Firmware**: None (uses host OS services)
**MMU Requirements**: Host OS virtual memory management
**Unique Aspect**: NetBSD kernel runs as a regular user-space process

## Hardware Support

NetBSD/usermode is **not a traditional hardware port**. Instead, it runs the NetBSD kernel as a user-space process on a host operating system.

**Host Operating Systems**:
- NetBSD (native)
- Linux
- FreeBSD
- OpenBSD
- macOS (theoretically)
- Other POSIX-compliant systems

**Host Architectures** (examples):
- x86 (i386)
- x86-64 (amd64)
- ARM
- Others (host-dependent)

## Boot Process

### Stage 0: Host OS

The host operating system is already running. No firmware or bootloader involved.

### Stage 1: Kernel Executable Launch

NetBSD/usermode kernel is executed directly as a user program:

```sh
# Run NetBSD/usermode kernel
$ ./netbsd [options]
```

**From `/sys/arch/usermode/usermode/machdep.c`**:

```c
void main(int argc, char *argv[])
{
    extern void ttycons_consinit(void);
    extern void pmap_bootstrap(void);
    extern void kernmain(void);

    saved_argv = argv;

    // Get host machine information
    thunk_getmachine(machine, sizeof(machine),
                     machine_arch, sizeof(machine_arch));

    // Initialize console
    ttycons_consinit();

    // Parse command-line options
    // ... (parse arguments)

    // Bootstrap virtual memory
    pmap_bootstrap();

    // Enter kernel main
    kernmain();
}
```

### Entry Point

**Unlike traditional ports**, there is no assembly startup code. The entry point is a standard C `main()` function.

### Command-Line Options

From `machdep.c`, supported options:

**Usage**:
```
netbsd [-acdqsvxz]
       [net=<tapdev>,<eaddr>]
       [audio=<audiodev>]
       [disk=<diskimg> ...]
       [root=<device>]
       [vnc=<width>x<height>,<port>]
       [vdev=atapi,device]
```

**Options**:
- `-a`: Ask for root device
- `-c`: User kernel configuration
- `-d`: Drop to debugger
- `-q`: Quiet boot
- `-s`: Single-user mode
- `-v`: Verbose boot
- `-x`: Debug mode
- `-z`: ???

**Device Configuration**:
- `net=tap0,00:00:be:ef:ca:fe`: Network via TAP device
- `audio=audio0`: Audio device (e.g., `/dev/audio0`)
- `disk=root.fs`: Disk image file (up to 4 images)
- `root=ld0`: Root device
- `vnc=640x480,5900`: VNC framebuffer
- `vdev=atapi,/dev/rcd0d`: Virtual ATAPI device

**Example**:
```sh
$ ./netbsd -v disk=root.fs disk=swap.fs \
  net=tap0,00:00:be:ef:ca:fe \
  audio=/dev/audio0 \
  root=ld0
```

### Kernel Initialization

After argument parsing:

1. **Console initialization**: `ttycons_consinit()`
2. **Memory bootstrap**: `pmap_bootstrap()` - set up virtual memory
3. **Kernel main**: `kernmain()` - standard NetBSD kernel init
4. **Device probing**: Attach virtual devices
5. **Root mount**: Mount root filesystem from disk image
6. **Init**: Start `/sbin/init`

## Memory Management

### Virtual Memory via Host OS

NetBSD/usermode **does not manage physical memory directly**. Instead:

1. **Allocate from host**: Use host OS `mmap()`, `malloc()`, etc.
2. **Virtual page tables**: Emulate page tables in usermode memory
3. **No hardware MMU**: All memory management is software

### Thunking Layer

**From `/sys/arch/usermode/usermode/thunk.c`**:

The "thunk" layer bridges NetBSD kernel code to host OS system calls:

```c
// Examples from thunk layer
int thunk_open(const char *path, int flags, mode_t mode);
int thunk_close(int fd);
ssize_t thunk_read(int fd, void *buf, size_t len);
ssize_t thunk_write(int fd, const void *buf, size_t len);
void *thunk_mmap(void *addr, size_t len, int prot, int flags, int fd, off_t offset);
int thunk_munmap(void *addr, size_t len);
```

**Purpose**: Allow kernel to perform operations via host OS without knowing specific host OS.

### Memory Layout

```
Host Process Address Space:
0x00000000...        : Host program segments
  NetBSD kernel .text
  NetBSD kernel .data
  NetBSD kernel .bss
  Heap (via host malloc/mmap)
  Stack (host-managed)
...                  : Host OS mappings
```

**No physical memory**: All memory is virtual from host OS perspective.

### Page Table Emulation

From `pmap.c`:

```c
// NetBSD/usermode pmap operations
// Emulate page tables using host memory
void pmap_bootstrap(void);
pmap_t pmap_create(void);
void pmap_enter(pmap_t pmap, vaddr_t va, paddr_t pa, vm_prot_t prot, u_int flags);
```

**Implementation**: Software page tables in kernel memory, no hardware MMU interaction.

## Device Support

### Virtual Devices

NetBSD/usermode provides virtual devices that interface with host OS:

**Block Devices** (`ld`):
- Disk images as files
- Maximum: 4 disk images (configurable via `MAX_DISK_IMAGES`)
- Accessed via `thunk_open()`, `thunk_read()`, `thunk_write()`

**Network** (`veth`):
- TAP devices on host
- Requires host TAP interface configured
- Example: `net=tap0,00:00:be:ef:ca:fe`

**Audio** (`audio`):
- Host audio device
- Example: `audio=/dev/audio0`
- Passes through to host

**Console** (`ttycons`):
- stdin/stdout of host process
- Simple terminal emulation

**Framebuffer** (`vncfb`):
- VNC server for graphical output
- Example: `vnc=640x480,5900`
- Accessible via VNC client

**Virtual ATAPI** (`vatapi`):
- Access host CD/DVD drive
- Example: `vdev=atapi,/dev/rcd0d`

### Device Bus

**From `mainbus.c`**:

```c
// Usermode mainbus
// Attaches virtual devices
cpu         at mainbus
thunkbus    at mainbus    // Virtual device bus
ttycons     at thunkbus   // Console
ld*         at thunkbus   // Disk images
veth*       at thunkbus   // Network
audio*      at thunkbus   // Audio
vncfb*      at thunkbus   // VNC framebuffer
```

## No Traditional Bootloader

NetBSD/usermode has **no bootloader** because:

1. **Host OS loads it**: The host OS loader (`exec()`) loads the kernel
2. **ELF executable**: Kernel is a standard ELF executable
3. **No firmware**: No BIOS, EFI, or similar
4. **No boot sequence**: Just `./netbsd` to run

**`/sys/arch/usermode/stand/`**: Does not exist (not needed)

## Building

### Building NetBSD/usermode

```sh
# Build for native host architecture
cd /usr/src
./build.sh -m usermode tools
./build.sh -m usermode kernel=GENERIC

# Kernel output: sys/arch/usermode/compile/GENERIC/netbsd
```

### Cross-Building

```sh
# Build on amd64 for usermode
./build.sh -m usermode -U kernel=GENERIC

# Build userland sets
./build.sh -m usermode -U distribution
```

### Kernel Configurations

- **GENERIC**: Standard kernel
- **INSTALL**: Installation kernel (minimal)

## Installation

### Creating Disk Images

**Create root filesystem image**:

```sh
# Create empty disk image (1GB)
dd if=/dev/zero of=root.fs bs=1m count=1024

# Create filesystem
vnconfig vnd0 root.fs
disklabel -I vnd0
newfs /dev/rvnd0a

# Mount and populate
mount /dev/vnd0a /mnt
cd /mnt
tar xzpf /path/to/base.tgz
tar xzpf /path/to/etc.tgz
# ... other sets
cd /
umount /mnt
vnconfig -u vnd0
```

**Create swap image**:
```sh
dd if=/dev/zero of=swap.fs bs=1m count=256
```

### Running NetBSD/usermode

**Simple boot**:
```sh
$ ./netbsd disk=root.fs root=ld0
```

**With networking**:
```sh
# On host, create TAP interface
$ sudo ifconfig tap0 create
$ sudo ifconfig tap0 192.168.1.1 netmask 255.255.255.0 up

# Run NetBSD/usermode
$ ./netbsd disk=root.fs root=ld0 \
  net=tap0,00:00:be:ef:ca:fe
```

**Multi-disk**:
```sh
$ ./netbsd disk=root.fs disk=home.fs disk=var.fs root=ld0
```

### First Boot Configuration

**After first boot into single-user mode**:

```sh
# Configure network (inside NetBSD/usermode)
# ifconfig veth0 192.168.1.2 netmask 255.255.255.0
# route add default 192.168.1.1

# Configure /etc/rc.conf, /etc/fstab, etc.
# exit to multi-user
```

## Debugging

### GDB Debugging

Since the kernel is a user process, standard debugging tools work:

```sh
# Run under GDB
$ gdb ./netbsd
(gdb) run disk=root.fs root=ld0
```

**Set breakpoints**:
```
(gdb) break main
(gdb) break syscall
(gdb) continue
```

### Core Dumps

If the kernel crashes:
```sh
$ gdb ./netbsd netbsd.core
(gdb) bt
```

### Verbose Boot

```sh
$ ./netbsd -v disk=root.fs root=ld0
```

### Single-User Mode

```sh
$ ./netbsd -s disk=root.fs root=ld0
```

### DDB (Kernel Debugger)

```sh
$ ./netbsd -d disk=root.fs root=ld0
```

**Inside kernel**:
```
db> show all procs
db> trace
db> continue
```

## Platform-Specific Considerations

### Host Dependencies

**Required on host**:
- POSIX-compliant OS
- Standard C library
- mmap(), malloc() support
- File I/O
- (Optional) TAP device support for networking

### Performance

**Slow**: NetBSD/usermode is significantly slower than native:
- No hardware acceleration
- Software memory management
- Context switches via host OS
- I/O through host system calls

**Use cases**:
- Development and testing
- Debugging kernel code
- Teaching/learning
- Running NetBSD where native port unavailable

### Limitations

**No hardware access**:
- Cannot access real hardware devices directly
- All I/O via host OS
- No DMA, interrupts are simulated

**Host OS restrictions**:
- Limited by host user permissions
- Cannot perform privileged operations unless host allows
- Network requires TAP device (may need root on host)

**Multiprocessing**:
- SMP support is limited
- Scheduling depends on host OS

### Security

**Isolation**: Kernel runs as unprivileged user process
- Cannot harm host OS
- Cannot access hardware directly
- Good for experimentation

**Not secure as VM**: Not suitable for security-sensitive applications
- Shares host memory space
- No hardware isolation

### Signals

Host OS signals are caught and translated:
- SIGALRM → timer interrupt
- SIGIO → I/O event
- SIGUSR1, SIGUSR2 → IPI (inter-processor interrupt) simulation

### Context Switching

From `pmap.c` and `trap.c`:

Uses host OS `setcontext()`, `getcontext()`:
```c
// Save/restore context via host OS ucontext
getcontext(&lwp->l_context);
setcontext(&newlwp->l_context);
```

**mcontext_t**: Uses host OS machine context (architecture-specific)

### Byte Order and Alignment

**Inherited from host**:
- Endianness: Host architecture
- Alignment: Host requirements
- Word size: Host (32-bit or 64-bit)

### File Systems

**Supported**:
- FFS (Fast File System)
- LFS (Log-structured File System)
- Any in-kernel filesystem

**Not supported**:
- Raw disk access
- Low-level disk tools

## Use Cases

### Development

**Rapid iteration**:
```sh
# Edit kernel source
$ vi sys/kern/vfs_syscalls.c

# Rebuild
$ ./build.sh kernel=GENERIC

# Test immediately
$ ./netbsd disk=test.fs root=ld0
```

**No reboot** of host machine needed.

### Testing

**Safe testing**:
- Test kernel changes without risking hardware
- Crash kernel without affecting host
- Easy to restart

### Education

**Learning kernel internals**:
- Debug kernel code with standard tools
- Step through kernel functions
- Understand kernel behavior

### Continuous Integration

**Automated testing**:
```sh
#!/bin/sh
./netbsd disk=test.fs root=ld0 < test_commands > test_output
# Analyze test_output
```

## Comparison with Other Virtualization

### vs. QEMU/VMware

**NetBSD/usermode**:
- Pros: Simpler, lighter, easier debugging
- Cons: Slower, no hardware emulation, host-dependent

**QEMU/VMware**:
- Pros: Full hardware emulation, faster (with KVM), portable
- Cons: Complex setup, harder debugging

### vs. Xen/NetBSD

**NetBSD/usermode**:
- User-space process
- No hypervisor

**NetBSD/xen**:
- Runs on Xen hypervisor
- Hardware virtualization
- Better performance

### vs. Rump Kernel

**NetBSD/usermode**:
- Full kernel as single process
- All kernel subsystems

**Rump kernel**:
- Selected kernel subsystems
- Embeddable in applications
- More flexible, modular

## References

### Source Files

**Core**:
- `/sys/arch/usermode/usermode/machdep.c` - Main entry point
- `/sys/arch/usermode/usermode/pmap.c` - Virtual memory
- `/sys/arch/usermode/usermode/thunk.c` - Host OS interface
- `/sys/arch/usermode/usermode/trap.c` - Exception handling

**Devices**:
- `/sys/arch/usermode/dev/ld_thunkbus.c` - Disk device
- `/sys/arch/usermode/dev/if_veth.c` - Network device
- `/sys/arch/usermode/dev/ttycons.c` - Console
- `/sys/arch/usermode/dev/vncfb.c` - VNC framebuffer

**Headers**:
- `/sys/arch/usermode/include/thunk.h` - Thunk layer definitions
- `/sys/arch/usermode/include/machdep.h` - Machine-dependent defs

### Documentation

**NetBSD**:
- NetBSD/usermode wiki pages
- Mailing list archives (port-usermode@)

**Papers**:
- "User-Mode NetBSD" - Jared McNeill
- Various blog posts and presentations

### Similar Projects

**User Mode Linux (UML)**:
- Linux kernel running as user process
- Similar concept to NetBSD/usermode

**Other BSD user-mode ports**:
- FreeBSD user-mode (experimental)

---

*Last Updated: 2025-11-12*
*Architecture Maintainer: NetBSD/usermode Port*
*Original Author: Jared McNeill, Reinoud Zandijk*
