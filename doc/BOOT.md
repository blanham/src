# NetBSD Boot Process Documentation

$NetBSD$

## Overview

This document provides comprehensive documentation for bootstrapping the NetBSD kernel on all supported platforms. It includes detailed information about bootloader implementation, MMU setup, and platform-specific requirements needed to create or port a bootloader to any NetBSD platform.

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Boot Process Stages](#boot-process-stages)
3. [Platform Categories](#platform-categories)
4. [Platform-Specific Documentation](#platform-specific-documentation)
5. [Shared Architecture Code](#shared-architecture-code)
6. [Creating a New Bootloader](#creating-a-new-bootloader)

## Architecture Overview

NetBSD runs on 60 distinct platform ports spanning multiple CPU architectures. The boot process varies significantly between platforms based on:

- **CPU Architecture**: x86, ARM, MIPS, PowerPC, m68k, SPARC, VAX, Alpha, HP-PA, SuperH, RISC-V, Itanium
- **Firmware Interface**: BIOS, UEFI, OpenFirmware, ROM monitors, custom firmware
- **Boot Method**: Multi-stage bootloaders, firmware-direct, network boot
- **MMU Requirements**: Platform-specific memory management unit initialization

## Boot Process Stages

### Generic Boot Flow

Most NetBSD platforms follow a multi-stage boot process:

```
Power-On → Firmware → Primary Bootloader → Secondary Bootloader → Kernel
```

#### Stage 0: Firmware/ROM
- Platform firmware (BIOS, UEFI, OpenFirmware, ROM monitor)
- Hardware initialization
- Boot device selection
- Load first-stage bootloader

#### Stage 1: Primary Bootloader (bootxx)
- **Size-constrained** (typically 512 bytes to 8KB)
- **Minimal functionality**: locate and load secondary bootloader
- **Location**: Boot sector, ROM, firmware filesystem
- **No MMU**: Runs in real/physical mode on most platforms

#### Stage 2: Secondary Bootloader (boot)
- **Full-featured**: filesystem support, device drivers, user interface
- **Kernel loading**: Parse kernel format (ELF, a.out), decompress if needed
- **Environment setup**: Pass boot parameters, device information to kernel
- **MMU setup**: May initialize MMU for platforms requiring it

#### Stage 3: Kernel
- Takes control from bootloader
- Completes MMU/pmap initialization
- Device autoconfiguration
- Mounts root filesystem

## Platform Categories

### By Boot Method

#### Multi-Stage Bootloader Platforms (23)
Platforms using bootxx → boot → kernel chain:
- **Alpha**: SRM firmware → bootxx → boot
- **i386**: BIOS → biosboot/pxeboot/cdboot → boot
- **amd64**: BIOS/UEFI → prekern → boot
- **SPARC**: OpenBoot PROM → bootblk → ofwboot
- **MIPS** (pmax, sgimips, cobalt, etc.): ROM → bootxx → boot
- **PowerPC** (prep, bebox, mvmeppc): Firmware → bootxx → boot
- **SuperH** (landisk, mmeye): ROM → bootxx → boot
- **m68k** (luna68k, news68k, next68k, x68k): ROM → bootxx → boot
- **HP-PA**: PDC firmware → bootxx → boot
- **VAX**: Console subsystem → boot

#### Firmware-Direct Boot Platforms (14)
Platforms booting directly from firmware to kernel:
- **amigappc**, **cats**, **cesfic**, **dreamcast**, **ibmnws**, **iyonix**
- **mac68k**, **netwinder**, **playstation2**, **sun2**, **sun3**
- **usermode**, **virt68k**, **algor**

#### OpenFirmware Platforms (4)
Using OpenFirmware with ofwboot:
- **macppc**: OpenFirmware → ofwboot → kernel
- **ofppc**: OpenFirmware → ofwboot → kernel
- **shark**: OpenFirmware → ofwboot → kernel
- **sparc64**: OpenBoot PROM → ofwboot → kernel

#### Evaluation Board Platforms (18+)
Board-specific bootloaders with high variability:
- **evbarm** (84 board variants): ARM evaluation boards
- **evbmips** (54 board variants): MIPS evaluation boards
- **evbppc**: PowerPC evaluation boards
- **evbsh3**: SuperH evaluation boards
- **hpcarm**, **hpcmips**, **hpcsh**: Handheld PC platforms
- **acorn32**, **amiga**, **atari**, **epoc32**, **hp300**, **mvme68k**
- **riscv**, **sandpoint**, **zaurus**

## Platform-Specific Documentation

Detailed documentation for each platform is located in:
```
/doc/boot/platforms/<platform>.md
```

### Platform Documentation Index

#### ARM Platforms
- [aarch64](boot/platforms/aarch64.md) - ARMv8 64-bit (meta-architecture)
- [acorn32](boot/platforms/acorn32.md) - Acorn ARM 6/7/SA computers
- [cats](boot/platforms/cats.md) - Chalice Technologies CATS
- [epoc32](boot/platforms/epoc32.md) - 32-bit EPOC OS devices
- [evbarm](boot/platforms/evbarm.md) - ARM evaluation boards (84 variants)
- [hpcarm](boot/platforms/hpcarm.md) - ARM-based handheld PCs
- [iyonix](boot/platforms/iyonix.md) - Castle Technology XScale
- [netwinder](boot/platforms/netwinder.md) - StrongARM Netwinder
- [shark](boot/platforms/shark.md) - Digital Network Appliance Reference
- [zaurus](boot/platforms/zaurus.md) - Sharp Zaurus PDAs

#### x86 Platforms
- [i386](boot/platforms/i386.md) - Intel/AMD x86 32-bit
- [amd64](boot/platforms/amd64.md) - AMD/Intel x86-64

#### MIPS Platforms
- [algor](boot/platforms/algor.md) - Algorithmics MIPS evaluation boards
- [arc](boot/platforms/arc.md) - MIPS ARC specification machines
- [cobalt](boot/platforms/cobalt.md) - Cobalt Networks microservers
- [emips](boot/platforms/emips.md) - Extensible MIPS machines
- [evbmips](boot/platforms/evbmips.md) - MIPS evaluation boards (54 variants)
- [ews4800mips](boot/platforms/ews4800mips.md) - NEC EWS4800 workstations
- [hpcmips](boot/platforms/hpcmips.md) - MIPS-based handheld PCs
- [mipsco](boot/platforms/mipsco.md) - MIPS Corp Magnum 3000
- [newsmips](boot/platforms/newsmips.md) - Sony NEWS MIPS workstations
- [playstation2](boot/platforms/playstation2.md) - Sony PlayStation 2
- [pmax](boot/platforms/pmax.md) - Digital MIPS DECstation
- [sbmips](boot/platforms/sbmips.md) - Broadcom SiByte evaluation boards
- [sgimips](boot/platforms/sgimips.md) - Silicon Graphics MIPS

#### PowerPC Platforms
- [amigappc](boot/platforms/amigappc.md) - Phase 5 Amiga
- [bebox](boot/platforms/bebox.md) - Be Inc. BeBox
- [evbppc](boot/platforms/evbppc.md) - PowerPC evaluation boards
- [ibmnws](boot/platforms/ibmnws.md) - IBM Network Station thin clients
- [macppc](boot/platforms/macppc.md) - Apple Power Macintosh
- [mvmeppc](boot/platforms/mvmeppc.md) - Motorola VMEbus PowerPC
- [ofppc](boot/platforms/ofppc.md) - OpenFirmware PowerPC machines
- [prep](boot/platforms/prep.md) - PowerPC Reference Platform
- [rs6000](boot/platforms/rs6000.md) - IBM RS/6000 workstations
- [sandpoint](boot/platforms/sandpoint.md) - Motorola Sandpoint NAS

#### m68k Platforms
- [amiga](boot/platforms/amiga.md) - Commodore Amiga
- [atari](boot/platforms/atari.md) - Atari TT030/Falcon/Hades
- [cesfic](boot/platforms/cesfic.md) - FIC8234 VME processor board
- [hp300](boot/platforms/hp300.md) - HP 9000/300 and 400 series
- [luna68k](boot/platforms/luna68k.md) - OMRON LUNA workstations
- [mac68k](boot/platforms/mac68k.md) - Apple Macintosh m68k
- [mvme68k](boot/platforms/mvme68k.md) - Motorola VMEbus 68K SBCs
- [news68k](boot/platforms/news68k.md) - Sony NEWS m68k workstations
- [next68k](boot/platforms/next68k.md) - NeXT Computer cubes/slabs
- [sun2](boot/platforms/sun2.md) - Sun Microsystems Sun-2 (68010)
- [sun3](boot/platforms/sun3.md) - Sun Microsystems Sun-3 (68020/030)
- [virt68k](boot/platforms/virt68k.md) - QEMU m68k virtual machine
- [x68k](boot/platforms/x68k.md) - Sharp X68000/X68030

#### SPARC Platforms
- [sparc](boot/platforms/sparc.md) - Sun SPARC 32-bit (sun4/sun4c/sun4m)
- [sparc64](boot/platforms/sparc64.md) - Sun UltraSPARC 64-bit

#### SuperH Platforms
- [dreamcast](boot/platforms/dreamcast.md) - SEGA Dreamcast
- [evbsh3](boot/platforms/evbsh3.md) - SuperH evaluation boards
- [hpcsh](boot/platforms/hpcsh.md) - SuperH handheld PCs
- [landisk](boot/platforms/landisk.md) - I-O DATA SH4 NAS appliances
- [mmeye](boot/platforms/mmeye.md) - SuperH camera controller

#### Other Architectures
- [alpha](boot/platforms/alpha.md) - Digital/Compaq Alpha
- [hppa](boot/platforms/hppa.md) - HP PA-RISC 700-series
- [ia64](boot/platforms/ia64.md) - Intel Itanium/Itanium2
- [or1k](boot/platforms/or1k.md) - OpenRISC 1000
- [riscv](boot/platforms/riscv.md) - RISC-V
- [usermode](boot/platforms/usermode.md) - Usermode (process-based)
- [vax](boot/platforms/vax.md) - Digital VAX
- [xen](boot/platforms/xen.md) - Xen virtual machine monitor

## Shared Architecture Code

Several platforms share common CPU architecture code:

### Meta-Architectures (No Bootloaders)

- **aarch64**: ARMv8 64-bit architecture shared code
- **arm**: ARM 32-bit architecture shared code (used by 10+ platforms)
- **m68k**: Motorola 68000 family shared code (used by 13 platforms)
- **mips**: MIPS architecture shared code (used by 13 platforms)
- **powerpc**: PowerPC architecture shared code (used by 10 platforms)
- **sh3**: Hitachi SuperH sh3/sh4 shared code (used by 5 platforms)
- **x86**: Intel x86 architecture shared code (used by i386/amd64)

### Architectural Feature Bases

- **hpc**: Handheld PC reference platform (hpcarm, hpcmips, hpcsh)
- **sun68k**: Sun Microsystems m68k platform base (sun2, sun3)

## Creating a New Bootloader

### General Requirements

When implementing a bootloader for a new NetBSD platform:

1. **Understand the Platform**
   - CPU architecture and instruction set
   - Boot ROM/firmware interface
   - Memory map and address space
   - Available boot devices (disk, network, flash, etc.)

2. **Study Existing Implementations**
   - Review similar platform bootloaders in `/sys/arch/*/stand/`
   - Examine shared architecture code in meta-architecture directories
   - Check existing documentation in platform doc/ directories

3. **Bootloader Components**
   - **bootxx** (primary): Minimal, size-constrained first-stage loader
   - **boot** (secondary): Full-featured loader with filesystem support
   - **libsa**: Standalone library support (from `/sys/lib/libsa/`)
   - **libkern**: Kernel library support (from `/sys/lib/libkern/`)
   - **lib/libz**: Compression support for gzip kernels

4. **MMU Initialization**
   - Determine if MMU setup is required before kernel handoff
   - Document page table format and initialization sequence
   - Handle both physical and virtual addressing modes
   - Ensure proper cache and TLB management

5. **Kernel Handoff**
   - Pass boot parameters (command line, device info)
   - Preserve memory layout information
   - Transfer control to kernel entry point
   - Follow platform-specific calling conventions

### Platform-Specific Considerations

#### x86 (i386/amd64)
- BIOS or UEFI firmware interface
- Real mode → Protected mode → Long mode (amd64)
- A20 gate handling
- Memory map from firmware (E820 on BIOS)
- PXE network boot support

#### ARM
- MMU/cache setup before kernel (platform-dependent)
- Device tree (FDT) or ATAG parameter passing
- Secure/non-secure world considerations
- Per-board variations for evbarm

#### MIPS
- ROM monitor or firmware interface
- Cache initialization requirements
- TLB setup for platforms without firmware support
- Endianness handling (big/little endian variants)

#### PowerPC
- OpenFirmware interface (macppc, ofppc, shark)
- PPCBUG monitor (mvmeppc)
- BAT registers for memory mapping
- Endianness and MMU setup

#### m68k
- ROM monitor interfaces vary by platform
- MMU setup for 68030/68040/68060
- Board-specific device initialization
- Legacy bootloader formats

#### SPARC
- OpenBoot PROM interface
- bootblk format for sparc32
- PROM callback usage
- Device tree navigation

### Build System Integration

1. Add bootloader directories under `/sys/arch/<platform>/stand/`
2. Create appropriate Makefiles using `/sys/lib/libsa/` infrastructure
3. Define memory layout constraints in linker scripts
4. Integration with `/usr/mdec/` installation procedures

### Testing

1. **Boot verification**: Confirm bootloader loads and transfers to kernel
2. **Device support**: Test all supported boot devices
3. **Edge cases**: Large kernels, multiple configurations, error conditions
4. **Regression testing**: Ensure existing functionality continues working

## Additional Resources

- **Source Code**:
  - Platform-specific: `/sys/arch/<platform>/stand/boot/`
  - Shared libraries: `/sys/lib/libsa/`, `/sys/lib/libkern/`
  - Boot programs: `/usr/mdec/` (installed bootloaders)

- **Documentation**:
  - Platform-specific: `/sys/arch/<platform>/doc/` (where available)
  - This directory: `/doc/boot/platforms/*.md`
  - README files: `/sys/arch/<platform>/stand/*/README`

- **Man Pages**:
  - boot(8) - General boot procedures
  - installboot(8) - Installing bootloaders
  - Platform-specific: boot_<platform>(8)

## Contributing

When adding or updating boot documentation:

1. Update platform-specific file in `/doc/boot/platforms/<platform>.md`
2. Include memory maps, MMU requirements, and boot flow
3. Document any firmware interfaces or ROM monitor usage
4. Add code examples for critical initialization sequences
5. Update this index if adding new platforms

## Platform Status Summary

| Architecture | Platforms | Boot Method | Documentation Status |
|-------------|-----------|-------------|---------------------|
| ARM 32-bit  | 10 | Varied | See platform docs |
| x86         | 2  | BIOS/UEFI | Complete |
| MIPS        | 13 | ROM/Firmware | See platform docs |
| PowerPC     | 10 | OF/Firmware | See platform docs |
| m68k        | 13 | ROM/Direct | See platform docs |
| SPARC       | 2  | OpenBoot | Complete |
| SuperH      | 5  | ROM/Firmware | See platform docs |
| Alpha       | 1  | SRM | Complete |
| HP-PA       | 1  | PDC | Complete |
| VAX         | 1  | Console | Complete |
| RISC-V      | 1  | Firmware | In progress |
| Itanium     | 1  | EFI | Incomplete |
| OpenRISC    | 1  | Custom | Incomplete |

Total Platforms: 60 real ports + 14 meta-architectures = 74 architecture directories

---

*Last Updated: 2025-11-12*
*Maintainer: NetBSD Boot Documentation Project*
