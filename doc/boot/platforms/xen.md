# NetBSD/xen Boot Documentation

$NetBSD$

## Platform Overview

**Architecture**: Xen virtual machine monitor (hypervisor)
**Port Date**: 2004-03-11 (Christian Limpach)
**Boot Method**: Xen hypervisor → Domain configuration → NetBSD kernel
**Firmware**: Xen hypervisor (not traditional firmware)
**MMU Requirements**: Paravirtualized or hardware-assisted (HVM/PVH)
**Unique Aspect**: NetBSD runs as a guest OS under Xen hypervisor

## What is NetBSD/xen?

NetBSD/xen is **not a hardware architecture** but a **virtualization platform**:

- **Xen**: Type-1 hypervisor that runs directly on hardware
- **NetBSD/xen**: NetBSD kernel adapted to run as Xen guest (domain)
- **Virtualization modes**: PV (paravirtualized), HVM (hardware virtual machine), PVH (paravirtualized hardware)

### Xen Terminology

**Hypervisor**: Low-level software that controls hardware and manages guests

**Domain**: Virtual machine instance
- **Dom0**: Privileged domain (management domain, runs first)
- **DomU**: Unprivileged domain (guest domain)

**PV (Paravirtualization)**: Guest OS modified to run on Xen (what NetBSD/xen uses)
- Direct hypercalls to Xen
- No hardware emulation needed
- Better performance than full virtualization

**HVM (Hardware Virtual Machine)**: Unmodified OS using hardware virtualization
- Requires Intel VT-x or AMD-V
- Uses QEMU for device emulation

**PVH (Paravirtualized Hardware)**: Hybrid mode
- Paravirtualized kernel
- Hardware virtualization for memory management
- No device emulation

## Boot Process

### Stage 0: Physical Hardware Boot

On physical machine:

1. **BIOS/UEFI** initializes hardware
2. **Bootloader** (GRUB, etc.) loads Xen hypervisor
3. **Xen hypervisor** starts

**GRUB Configuration** (example):
```
title Xen 4.17 / NetBSD
    kernel /boot/xen.gz dom0_mem=1024M
    module /boot/netbsd-XEN3_DOM0.gz root=/dev/xbd0a
```

### Stage 1: Xen Hypervisor

**Xen hypervisor** (`xen.gz`):

1. **Initialize hardware**: CPU, memory, interrupts
2. **Set up virtualization**: Page tables, event channels
3. **Load Dom0 kernel**: From GRUB module
4. **Create Dom0**: First privileged domain
5. **Transfer control**: Start Dom0 kernel

**Xen provides**:
- Memory management hypercalls
- Virtual CPU scheduling
- Event channels (virtual interrupts)
- Grant tables (shared memory)
- Virtual device backends

### Stage 2: Dom0 Boot

**Dom0** is the privileged management domain:

**From GRUB module** (typically `netbsd-XEN3_DOM0.gz`):
1. **Xen loads kernel**: Into Dom0 memory
2. **Set up initial environment**: Pass start_info structure
3. **Start kernel**: Jump to entry point

**Dom0 responsibilities**:
- Manage other domains (create, destroy, configure)
- Provide device backends (disk, network)
- Interface with physical hardware via Xen
- Run management tools (`xl`, `xm`)

**No traditional bootloader needed**: Xen acts as bootloader

### Stage 3: DomU Boot

**DomU** (unprivileged guest) boot:

**Via domain configuration**:

```python
# Example DomU config file: netbsd.cfg
name = "netbsd-guest"
memory = 512
vcpus = 2
kernel = "/path/to/netbsd-XEN3_DOMU"
disk = [ 'file:/path/to/netbsd.img,xvda,w' ]
vif = [ 'bridge=xenbr0' ]
```

**Boot sequence**:
1. **xl create netbsd.cfg**: Management command
2. **Xen allocates resources**: Memory, vCPUs
3. **Load kernel**: From config file
4. **Create domain**: Set up virtual environment
5. **Start kernel**: Transfer control

**No bootloader**: Kernel loaded directly by Xen

## Xen Kernel Variants

NetBSD provides multiple Xen kernel configurations:

### Dom0 Kernels

**XEN3_DOM0** (amd64/i386):
- Privileged domain kernel
- Can manage other domains
- Has backend drivers (blkback, netback)
- Can access physical hardware

**XEN3PAE_DOM0** (i386 only):
- PAE (Physical Address Extension) for >4GB RAM
- Otherwise same as XEN3_DOM0

### DomU Kernels

**XEN3_DOMU** (amd64/i386):
- Unprivileged guest kernel
- Frontend drivers only (xbd, xennet)
- No hardware access (except via Xen)

**XEN3PAE_DOMU** (i386 only):
- PAE support for DomU

### PVH Kernels

**XEN3_DOMU_PVH** (amd64):
- PVH mode (paravirtualized hardware)
- Uses hardware virtualization extensions
- Better performance than pure PV

## Paravirtualization Details

### Hypercalls

NetBSD/xen uses **hypercalls** instead of privileged instructions:

**Example hypercalls**:
```c
// Memory management
HYPERVISOR_update_va_mapping(va, pte, flags);
HYPERVISOR_mmu_update(updates, count, success_count, domid);

// Grant tables (shared memory)
HYPERVISOR_grant_table_op(cmd, args, count);

// Event channels (interrupts)
HYPERVISOR_event_channel_op(cmd, args);

// Virtual CPU operations
HYPERVISOR_vcpu_op(cmd, vcpuid, args);

// Domain control (Dom0 only)
HYPERVISOR_domctl(op);
HYPERVISOR_sysctl(op);
```

**From `/sys/arch/xen/include/hypervisor.h`**

### Shared Info Page

Xen shares a page with each domain:

**shared_info**:
- Event channel bitmasks (virtual interrupts)
- vCPU info (one per virtual CPU)
- Wall clock time
- System time

**Location**: Mapped into kernel VA space
**Access**: Direct memory read/write (no hypercall needed)

### Event Channels

**Event channels** are Xen's interrupt mechanism:

**Types**:
- **VIRQ**: Virtual IRQ (timer, debug, console)
- **IPI**: Inter-processor interrupt
- **Physical IRQ**: From real hardware (Dom0 only)
- **Interdomain**: Between domains

**Handling** (from `/sys/arch/xen/xen/evtchn.c`):
1. Xen sets bit in shared_info
2. Xen delivers upcall to domain
3. Domain checks shared_info
4. Domain handles event
5. Domain clears bit and re-enables events

### Grant Tables

**Grant tables** allow safe memory sharing:

**Usage**:
1. **Dom0 grants** pages to DomU (or vice versa)
2. **DomU maps** granted pages
3. **I/O operations** on shared memory
4. **DomU unmaps** when done
5. **Dom0 revokes** grant

**Example**: Disk I/O
- DomU grants page to Dom0
- Dom0 (blkback) reads/writes granted page
- Dom0 signals completion via event channel

## Virtual Devices

NetBSD/xen uses **split device model**:

### Frontend (DomU)

**xbd** (block device):
- Virtual disk
- Communicates with backend via shared ring
- Location: `/sys/arch/xen/xen/xbd_xenbus.c`

**xennet** (network device):
- Virtual network interface
- TX/RX rings shared with backend
- Location: `/sys/arch/xen/xen/if_xennet_xenbus.c`

**xencons** (console):
- Virtual console
- Shared ring with Xen/Dom0
- Location: `/sys/arch/xen/xen/xencons.c` (implied from device list)

### Backend (Dom0)

**xbdback** (block backend):
- Serves virtual disks to DomU
- Accesses physical disk or file
- Location: `/sys/arch/xen/xen/xbdback_xenbus.c`

**xennetback** (network backend):
- Bridges DomU network to physical network
- Location: implied from code

### XenBus

**XenBus**: Device discovery and configuration mechanism

**From `/sys/arch/xen/xenbus/`**:
- Provides device tree in XenStore
- Dynamic device attach/detach
- Configuration via XenStore paths

**XenStore**:
- Hierarchical key-value store
- Shared between all domains
- Used for configuration and coordination

## Memory Management

### Paravirtualized Memory

NetBSD/xen does not directly control page tables:

**Address translation**:
```
Guest Virtual → Guest Physical (pseudo-physical) → Machine Physical
```

- **Guest Physical**: What kernel thinks is physical
- **Machine Physical**: Real hardware physical addresses
- **Xen manages** the GPA → MPA mapping

### Page Table Operations

**Normal OS**: Directly writes to page tables
**NetBSD/xen**: Hypercalls to update page tables

```c
// Update single PTE
HYPERVISOR_update_va_mapping(va, new_pte, flags);

// Batch updates (more efficient)
mmu_update_t updates[BATCH_SIZE];
updates[0].ptr = pte_machine_address;
updates[0].val = new_pte_value;
HYPERVISOR_mmu_update(updates, count, &success, DOMID_SELF);
```

**From `/sys/arch/xen/x86/xen_pmap.c`**

### Balloon Driver

**Balloon driver** adjusts domain memory:

**From `/sys/arch/xen/xen/balloon.c`**:

- **Inflate**: Give memory back to Xen (reduce domain memory)
- **Deflate**: Request memory from Xen (increase domain memory)

**Usage**:
```sh
# Inside guest
sysctl -w machdep.xen.balloon.current=524288  # Set to 512MB pages
```

## No Traditional Bootloader

NetBSD/xen **does not have** a bootloader in `/sys/arch/xen/stand/`:

**Why?**
1. **Xen is the bootloader**: Xen loads kernel directly
2. **Configuration-based boot**: Domain config specifies kernel
3. **No BIOS/firmware**: Xen provides virtualized environment
4. **Simplified boot**: No multi-stage boot needed

**Kernel is loaded**:
- Dom0: By Xen from GRUB module
- DomU: By Xen from domain config

## Building

### Building Xen Kernels

```sh
# Build Dom0 kernel (amd64)
cd /usr/src
./build.sh -m amd64 kernel=XEN3_DOM0

# Build DomU kernel (amd64)
./build.sh -m amd64 kernel=XEN3_DOMU

# Build i386 PAE Dom0
./build.sh -m i386 kernel=XEN3PAE_DOM0
```

**Kernel locations**:
- `/usr/src/sys/arch/amd64/compile/XEN3_DOM0/netbsd`
- `/usr/src/sys/arch/amd64/compile/XEN3_DOMU/netbsd`

### Kernel Configurations

**Configuration files**:
- `/sys/arch/amd64/conf/XEN3_DOM0`
- `/sys/arch/amd64/conf/XEN3_DOMU`
- `/sys/arch/i386/conf/XEN3PAE_DOM0`
- `/sys/arch/i386/conf/XEN3PAE_DOMU`

## Installation

### Installing Dom0

**On physical machine with Xen**:

1. **Install Xen hypervisor**:
   ```sh
   pkgin install xenkernel417  # Or appropriate version
   ```

2. **Install NetBSD with XEN3_DOM0 kernel**:
   ```sh
   # Copy kernel
   cp /usr/src/sys/arch/amd64/compile/XEN3_DOM0/netbsd /netbsd-XEN3_DOM0
   ```

3. **Configure GRUB**:
   ```
   menuentry 'NetBSD-XEN3_DOM0' {
       multiboot /boot/xen.gz dom0_mem=1024M
       module /netbsd-XEN3_DOM0.gz root=/dev/xbd0a
   }
   ```

4. **Reboot**: Boot into Xen with Dom0

### Creating DomU

**Disk image**:
```sh
# Create disk image
dd if=/dev/zero of=netbsd-guest.img bs=1M count=4096

# Install NetBSD into image (various methods)
# - Loop mount and extract sets
# - Use another system to partition and format
```

**Domain configuration** (`netbsd-guest.cfg`):
```python
name = "netbsd-guest"
kernel = "/netbsd-XEN3_DOMU"
memory = 512
vcpus = 2
disk = [ 'file:/path/to/netbsd-guest.img,xvda,w' ]
vif = [ 'bridge=xenbr0,mac=00:16:3e:xx:xx:xx' ]
```

**Start guest**:
```sh
xl create netbsd-guest.cfg
xl console netbsd-guest  # Attach to console
```

## Debugging

### Dom0 Console

Dom0 console is physical machine console (via Xen):
- Serial console if configured
- VGA console
- Xen console output

### DomU Console

**Attach to DomU console**:
```sh
xl console <domain-name>
```

**Detach**: Ctrl-]

### Xen Debug Output

**Enable Xen debug**:
```sh
xl dmesg  # Show Xen hypervisor messages
```

### DDB (Kernel Debugger)

**Build kernel with**:
```
options DDB
```

**Enter debugger**:
- From console: Send break
- From code: `Debugger()` call

### Common Issues

**Problem**: Domain won't start
**Solution**: Check `xl dmesg` and `/var/log/xen/` logs

**Problem**: Network not working
**Solution**: Check bridge configuration, vif settings

**Problem**: Disk I/O errors
**Solution**: Verify disk image path, permissions

**Problem**: "Cannot allocate memory"
**Solution**: Reduce memory in domain config, check Dom0 memory

## Platform-Specific Considerations

### CPU Architecture Support

**amd64** (x86-64):
- Most common
- Best performance
- PVH support

**i386** (32-bit x86):
- Legacy support
- PAE for >4GB memory
- PV only

**Other architectures**:
- ARM Xen support exists (not in NetBSD/xen yet)

### Dom0 vs. DomU Differences

**Dom0 capabilities**:
- Access to physical hardware
- Backend drivers
- Domain management (xl commands)
- PCI passthrough support

**DomU limitations**:
- No hardware access
- Frontend drivers only
- Managed by Dom0
- Isolated from other domains

### Performance Considerations

**Advantages**:
- Near-native performance (especially with PVH)
- Efficient I/O via shared memory
- CPU scheduling by Xen

**Disadvantages**:
- Hypercall overhead (vs. native instructions)
- Memory management indirection
- No direct hardware access (DomU)

### Security

**Isolation**:
- Domains isolated by Xen
- Hypervisor enforces boundaries
- Grant tables for safe sharing

**Dom0 privilege**:
- Dom0 can control other domains
- Compromise of Dom0 = compromise of all guests
- Keep Dom0 minimal and secure

### Migration

**Live migration** supported:
```sh
xl migrate <domain> <dest-host>
```

**Requirements**:
- Shared storage (or storage migration)
- Network connectivity
- Compatible Xen versions

## Xen Versions

NetBSD/xen supports multiple Xen versions:

**Xen 3.x**: Older, still supported
**Xen 4.x**: Current, recommended (4.11+)

**Compatibility**: NetBSD kernels work with range of Xen versions

## Comparison with Other Platforms

### vs. Native NetBSD

**NetBSD/xen**:
- Virtualized
- Multiple instances on one machine
- Managed by hypervisor

**Native NetBSD**:
- Direct hardware access
- Best performance
- Full control

### vs. Other Hypervisors

**Xen**:
- Type-1 (bare-metal)
- Paravirtualization
- Open source

**KVM**:
- Type-2 (runs on Linux)
- Hardware virtualization
- Integrated with Linux

**VMware ESXi**:
- Type-1
- Commercial
- Hardware virtualization

## References

### Source Files

**Core**:
- `/sys/arch/xen/xen/hypervisor.c` - Hypervisor interface
- `/sys/arch/xen/xen/evtchn.c` - Event channels
- `/sys/arch/xen/x86/xen_pmap.c` - Memory management
- `/sys/arch/xen/xen/xbd_xenbus.c` - Block device frontend
- `/sys/arch/xen/xen/if_xennet_xenbus.c` - Network frontend

**Headers**:
- `/sys/arch/xen/include/hypervisor.h` - Hypercall definitions
- `/sys/arch/xen/include/xen.h` - Xen types and constants

### Xen Documentation

**Xen Project**:
- https://xenproject.org/
- Xen architecture documentation
- Hypervisor interfaces

**Xen Wiki**:
- Domain configuration
- Management tools
- Troubleshooting guides

### NetBSD Documentation

**NetBSD Guide**:
- NetBSD/xen setup guide
- Domain configuration examples

**Man Pages**:
- `xen(4)` - Xen driver
- `xbd(4)` - Xen block device
- `xennet(4)` - Xen network interface

---

*Last Updated: 2025-11-12*
*Architecture Maintainer: NetBSD/xen Port*
*Original Xen Port: Christian Limpach, Manuel Bouyer*
