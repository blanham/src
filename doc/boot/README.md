# NetBSD Boot Documentation

This directory contains comprehensive boot documentation for all NetBSD platforms.

## Documentation Structure

- **BOOT.md** - Main boot documentation with platform overview and shared concepts
- **platforms/** - Platform-specific boot documentation (62 platforms)

## Quick Navigation

### By Architecture Family

#### x86 Platforms
- [i386](platforms/i386.md) - Intel/AMD x86 32-bit
- [amd64](platforms/amd64.md) - AMD/Intel x86-64

#### ARM Platforms (32-bit)
- [acorn32](platforms/acorn32.md) - Acorn ARM 6/7/SA computers
- [cats](platforms/cats.md) - Chalice Technologies CATS
- [epoc32](platforms/epoc32.md) - 32-bit EPOC OS devices
- [evbarm](platforms/evbarm.md) - ARM evaluation boards (84+ variants)
- [hpcarm](platforms/hpcarm.md) - ARM handheld PCs
- [iyonix](platforms/iyonix.md) - Castle Technology XScale
- [netwinder](platforms/netwinder.md) - StrongARM Netwinder
- [shark](platforms/shark.md) - Digital Network Appliance Reference
- [zaurus](platforms/zaurus.md) - Sharp Zaurus PDAs

#### MIPS Platforms
- [algor](platforms/algor.md) - Algorithmics evaluation boards
- [arc](platforms/arc.md) - MIPS ARC machines
- [cobalt](platforms/cobalt.md) - Cobalt Networks microservers
- [emips](platforms/emips.md) - Extensible MIPS
- [evbmips](platforms/evbmips.md) - MIPS evaluation boards (54+ variants)
- [ews4800mips](platforms/ews4800mips.md) - NEC EWS4800 workstations
- [hpcmips](platforms/hpcmips.md) - MIPS handheld PCs
- [mipsco](platforms/mipsco.md) - MIPS Corp Magnum 3000
- [newsmips](platforms/newsmips.md) - Sony NEWS MIPS
- [playstation2](platforms/playstation2.md) - Sony PlayStation 2
- [pmax](platforms/pmax.md) - Digital MIPS DECstation
- [sbmips](platforms/sbmips.md) - Broadcom SiByte evaluation boards
- [sgimips](platforms/sgimips.md) - Silicon Graphics MIPS

#### PowerPC Platforms
- [amigappc](platforms/amigappc.md) - Phase 5 Amiga
- [bebox](platforms/bebox.md) - Be Inc. BeBox
- [evbppc](platforms/evbppc.md) - PowerPC evaluation boards
- [ibmnws](platforms/ibmnws.md) - IBM Network Station
- [macppc](platforms/macppc.md) - Apple Power Macintosh
- [mvmeppc](platforms/mvmeppc.md) - Motorola VMEbus PowerPC
- [ofppc](platforms/ofppc.md) - OpenFirmware PowerPC
- [prep](platforms/prep.md) - PowerPC Reference Platform
- [rs6000](platforms/rs6000.md) - IBM RS/6000
- [sandpoint](platforms/sandpoint.md) - Motorola Sandpoint NAS

#### m68k Platforms
- [amiga](platforms/amiga.md) - Commodore Amiga
- [atari](platforms/atari.md) - Atari TT030/Falcon/Hades
- [cesfic](platforms/cesfic.md) - FIC8234 VME board
- [hp300](platforms/hp300.md) - HP 9000/300 and 400
- [luna68k](platforms/luna68k.md) - OMRON LUNA
- [mac68k](platforms/mac68k.md) - Apple Macintosh m68k
- [mvme68k](platforms/mvme68k.md) - Motorola VMEbus 68K
- [news68k](platforms/news68k.md) - Sony NEWS m68k
- [next68k](platforms/next68k.md) - NeXT Computer
- [sun2](platforms/sun2.md) - Sun Microsystems Sun-2
- [sun3](platforms/sun3.md) - Sun Microsystems Sun-3
- [virt68k](platforms/virt68k.md) - QEMU m68k virtual
- [x68k](platforms/x68k.md) - Sharp X68000/X68030

#### SPARC Platforms
- [sparc](platforms/sparc.md) - Sun SPARC 32-bit
- [sparc64](platforms/sparc64.md) - Sun UltraSPARC 64-bit

#### SuperH Platforms
- [dreamcast](platforms/dreamcast.md) - SEGA Dreamcast
- [evbsh3](platforms/evbsh3.md) - SuperH evaluation boards
- [hpcsh](platforms/hpcsh.md) - SuperH handheld PCs
- [landisk](platforms/landisk.md) - I-O DATA SH4 NAS
- [mmeye](platforms/mmeye.md) - SuperH camera controller

#### Other Architectures
- [alpha](platforms/alpha.md) - Digital/Compaq Alpha
- [hppa](platforms/hppa.md) - HP PA-RISC 700-series
- [ia64](platforms/ia64.md) - Intel Itanium/Itanium2
- [or1k](platforms/or1k.md) - OpenRISC 1000
- [riscv](platforms/riscv.md) - RISC-V
- [usermode](platforms/usermode.md) - Usermode (process-based)
- [vax](platforms/vax.md) - Digital VAX
- [xen](platforms/xen.md) - Xen virtual machine monitor

## Documentation Coverage

Each platform document includes:

1. **Platform Overview** - Hardware, port date, boot method
2. **Boot Process** - Detailed stage-by-stage boot sequence
3. **MMU Requirements** - Architecture-specific memory management initialization
4. **Memory Map** - Physical and virtual address layouts
5. **Bootloader Implementation** - Source code locations and details
6. **Build Instructions** - Compiling bootloaders and kernels
7. **Installation** - Step-by-step installation procedures
8. **Debugging** - Console setup, troubleshooting, common issues
9. **References** - Source files, man pages, external documentation

## Common Boot Patterns

### Multi-Stage Bootloader
Traditional boot chain with separate stages:
- **Firmware/ROM** → **Primary bootloader (bootxx)** → **Secondary bootloader (boot)** → **Kernel**
- Examples: i386, alpha, most MIPS platforms

### Firmware-Direct Boot
Firmware loads kernel directly without intermediate bootloader:
- **Firmware** → **Kernel**
- Examples: cats, iyonix, netwinder, many embedded boards

### OpenFirmware Boot
IEEE 1275-compliant OpenFirmware:
- **OpenFirmware** → **ofwboot** → **Kernel**
- Examples: macppc, ofppc, shark, sparc, sparc64

### Foreign OS Loaders
Bootloader runs under another operating system:
- **Host OS** → **Bootloader application** → **Kernel**
- Examples: hpcarm/hpcmips/hpcsh (Windows CE), zaurus (Linux), epoc32 (EPOC)

### Virtual Machine Boot
Virtualization platforms:
- **Hypervisor** → **Kernel** (with paravirtualization)
- Examples: xen, usermode

## Quick Reference

### BIOS/UEFI Platforms
- i386, amd64, ia64

### OpenFirmware Platforms
- macppc, ofppc, shark, sparc, sparc64

### U-Boot Platforms
- Many evbarm, evbmips, evbppc boards

### ROM Monitor Platforms
- alpha (SRM), hppa (PDC), vax (Console ROM)

### Device Tree (FDT) Platforms
- Modern evbarm, riscv, some evbppc

## Contributing

When updating boot documentation:

1. Update the relevant platform file in `platforms/<platform>.md`
2. Include source code references with file paths and line numbers
3. Document MMU setup sequences with actual register values
4. Provide complete memory maps
5. Include working command examples
6. Test installation procedures
7. Update this README if adding new platforms

## Man Pages

See also:
- boot(8) - General boot procedures
- installboot(8) - Installing boot blocks
- boot_<platform>(8) - Platform-specific boot documentation

## Version Information

This documentation was created for NetBSD-current as of 2025-11-12.

For the most up-to-date information, consult:
- Platform source code in `/sys/arch/<platform>/`
- Platform README files
- NetBSD mailing lists and wiki

---

*NetBSD Boot Documentation Project*
*Last Updated: 2025-11-12*
