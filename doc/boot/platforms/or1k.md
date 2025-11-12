# NetBSD/or1k Boot Documentation

$NetBSD$

## Platform Overview

**Architecture**: OpenRISC 1000 (OR1K)
**Port Date**: 2014-09-03
**Boot Method**: Not implemented (skeleton port)
**Firmware**: Varies by implementation (typically bare-metal or minimal firmware)
**MMU Requirements**: Software-managed TLB, virtual addressing
**Status**: Skeleton/stub port - no functional implementation

## Hardware Support

**Target Systems**: OpenRISC 1000 compatible processors
**Implementation Status**: Header files and architecture definitions only
**Actual Hardware Support**: None (port incomplete)

### OpenRISC 1000 Architecture

OpenRISC 1000 is an open-source RISC instruction set architecture:

- **CPU**: 32-bit or 64-bit RISC architecture
- **Register File**: 32 general-purpose registers
- **Instruction Set**: Load/store architecture
- **Cache**: Separate instruction and data caches
- **MMU**: Software-managed TLB

### Common OpenRISC Implementations

**Hardware**:
- OR1200: Original OpenRISC implementation
- mor1kx: Modern optimized implementation
- Various FPGA implementations

**Simulators/Emulators**:
- or1ksim: OpenRISC simulator
- QEMU: or1k target support

## Port Status

### Current State

The NetBSD/or1k port is a **skeleton port** that exists only as architectural definitions:

**What exists**:
- Header files in `/sys/arch/or1k/include/`
- Architecture type definitions
- Machine-dependent constants
- Register definitions (SPRs)
- MMU/TLB structures

**What does NOT exist**:
- Bootloader (`/sys/arch/or1k/stand/` does not exist)
- Kernel implementation
- Device drivers
- Board support
- Actual executable code

### Port History

**2014-09-03**: Initial skeleton port added by Matt Thomas
- Added architecture headers
- Defined basic types and structures
- Created minimal build infrastructure

**Since then**: No significant development
- Port remains incomplete
- No bootable kernel
- No active development visible

## Architecture Definitions

### Header Files Available

Located in `/sys/arch/or1k/include/`:

**Core definitions**:
- `types.h` - Architecture-specific types
- `param.h` - System parameters
- `cpu.h` - CPU definitions
- `reg.h` - Register definitions
- `spr.h` - Special Purpose Registers

**Memory management**:
- `pmap.h` - Physical memory map interface
- `pte.h` - Page table entry definitions
- `vmparam.h` - Virtual memory parameters

**Other**:
- `asm.h` - Assembly macros
- `elf_machdep.h` - ELF definitions
- `trap.h` - Exception/trap definitions
- `intr.h` - Interrupt handling

### Special Purpose Registers (SPRs)

From `/sys/arch/or1k/include/spr.h`, OpenRISC defines numerous SPRs:

**Supervision Register (SR)**:
- SM: Supervisor Mode
- TEE: Tick Timer Exception Enable
- IEE: Interrupt Exception Enable
- DCE: Data Cache Enable
- ICE: Instruction Cache Enable
- DME: Data MMU Enable
- IME: Instruction MMU Enable

**MMU Registers**:
- DTLBMR/ITLBMR: TLB Match Registers
- DTLBTR/ITLBTR: TLB Translate Registers

### Memory Layout (Theoretical)

Based on header definitions:

**Page Size**: 8KB (from `vmparam.h`)
**Virtual Address Space**: 32-bit or 64-bit (implementation-dependent)
**TLB**: Software-managed

## Boot Process (Not Implemented)

### Theoretical Boot Sequence

If the port were complete, a typical OpenRISC boot would involve:

1. **Reset/Power-On**:
   - CPU starts at reset vector (typically 0x100)
   - Initialize SPRs
   - Set up exception vectors

2. **Firmware/Bootloader**:
   - Initialize UART for console
   - Initialize memory controller
   - Load kernel from storage or network

3. **Kernel Entry**:
   - Enable MMU
   - Set up page tables
   - Initialize kernel data structures
   - Start scheduler

### Why No Bootloader Exists

NetBSD/or1k has no bootloader because:

1. **Port incomplete**: Core kernel functionality not implemented
2. **No target hardware**: No specific OpenRISC board targeted
3. **Ecosystem dependency**: OpenRISC systems typically use their own bootloaders (U-Boot, barebox)

## MMU Architecture

### TLB Structure (From Headers)

OpenRISC uses a software-managed TLB:

**TLB Size**: Typically 64-128 entries (implementation-dependent)
**TLB Type**: Separate ITLB (instruction) and DTLB (data)

**TLB Entry Format**:
```
Match Register (TLBMR):
- Valid bit
- Page Level (PL0, PL1)
- Context ID (CID)
- LRU bits
- Virtual Page Number (VPN)

Translate Register (TLBTR):
- Cache Coherency (CC)
- Cache Inhibit (CI)
- Write-Back Cache (WBC)
- Weakly-Ordered Memory (WOM)
- Access bits (URE, UWE, SRE, SWE, SXE)
- Dirty bit
- Physical Page Number (PPN)
```

**Page Protection**:
- URE/UWE: User Read/Write Enable
- SRE/SWE/SXE: Supervisor Read/Write/Execute Enable

## Development Status

### To Complete This Port

A complete NetBSD/or1k port would require:

1. **Kernel Implementation**:
   - `locore.S`: Low-level assembly startup
   - `machdep.c`: Machine-dependent initialization
   - `pmap.c`: Physical memory management
   - `trap.c`: Exception/trap handling
   - `cpu_switch.S`: Context switching

2. **Bootloader**:
   - First-stage bootloader for OpenRISC platforms
   - Or integration with existing OpenRISC bootloaders (U-Boot)

3. **Device Drivers**:
   - UART for console
   - Interrupt controller
   - Timer
   - Storage (SD, SATA, etc.)
   - Network (Ethernet)

4. **Board Support**:
   - Select target board(s)
   - Device tree or configuration
   - Platform initialization

5. **Build System**:
   - Toolchain integration
   - Kernel configuration files
   - Installation procedures

### Similar Completed Ports

For reference, similar RISC architectures in NetBSD:
- **NetBSD/riscv**: RISC-V port (completed, functional)
- **NetBSD/mips**: MIPS port (mature)
- **NetBSD/powerpc**: PowerPC port (mature)

## Potential Boot Methods

### Using U-Boot

If developed, NetBSD/or1k could use U-Boot:

**Theoretical process**:
```
U-Boot> dhcp
U-Boot> tftp 0x1000000 netbsd.or1k
U-Boot> bootelf 0x1000000
```

Or loading from storage:
```
U-Boot> load mmc 0:1 0x1000000 netbsd
U-Boot> bootelf 0x1000000
```

### Direct Loading

For simulation or embedded systems:
```
or1ksim -f config.or1k -m 64M kernel.elf
```

## References

### NetBSD Source

**Header definitions**:
- `/sys/arch/or1k/include/` - All or1k headers
- `/sys/arch/or1k/conf/majors.or1k` - Device major numbers

### OpenRISC Documentation

**Architecture**:
- OpenRISC 1000 Architecture Manual
- OpenCores website: https://opencores.org

**Implementations**:
- OR1200 documentation
- mor1kx documentation

**Tools**:
- or1ksim simulator
- OpenRISC GCC toolchain
- QEMU or1k target

### Similar Ports for Reference

**RISC-V** (similar open RISC architecture):
- `/sys/arch/riscv/` - NetBSD RISC-V port

**MIPS** (similar simple RISC):
- `/sys/arch/mips/` - NetBSD MIPS port

## Notes for Future Development

### Getting Started

To develop this port:

1. **Select target platform**:
   - Choose specific OpenRISC implementation
   - FPGA board or simulator

2. **Study working ports**:
   - Examine NetBSD/riscv for modern RISC port
   - Study NetBSD/mips for TLB management

3. **Implement minimal kernel**:
   - Start with simulator (or1ksim or QEMU)
   - Implement `locore.S` startup
   - Get console output working
   - Implement exception handling

4. **Add MMU support**:
   - Implement TLB miss handlers
   - Create `pmap.c` for memory management
   - Enable virtual memory

5. **Bootloader**:
   - Port U-Boot or create minimal loader
   - Implement kernel loading

### OpenRISC Ecosystem

**Bootloaders**:
- U-Boot: Most common for OpenRISC
- barebox: Alternative bootloader

**Operating Systems** with OpenRISC support:
- Linux: Mature OpenRISC support
- RTEMS: Real-time OS for embedded
- FreeRTOS: Lightweight RTOS

**Development Tools**:
- OpenRISC GCC: Compiler toolchain
- or1k-elf-gdb: Debugger
- FuseSoC: FPGA build system

## Conclusion

NetBSD/or1k is currently a **placeholder port** with no functional implementation. It exists only as a skeleton for potential future development. To use NetBSD on OpenRISC hardware, significant development work is required, including:

- Complete kernel implementation
- Bootloader development or integration
- Device drivers
- Board support packages

Developers interested in this port should consult the OpenRISC architecture manual and study similar NetBSD ports (particularly RISC-V) as reference implementations.

---

*Last Updated: 2025-11-12*
*Port Status: Skeleton/Incomplete - No Maintainer*
