# Unusual aspects of OS128

This project is unusual in several ways for a modern codebase, even though it is a compact, educational operating system for a classic 8-bit machine.

## Related notes

- [Interesting findings](interesting.md)
- [Engineering notes](engineering-notes.md)
- [Engineering details](engineering-details.md)
- [Architecture overview](../architecture/00-overview.md)

## 1. It is built around a very specific 1980s hardware profile

The project is not a generic OS. It assumes a Commodore 128 with a very particular configuration:

- C128 host machine
- 80-column VDC display
- at least 64 KB of VDC memory
- RAM expansion unit (REU) with at least 512 KB
- 1571-style floppy or CMD-compatible mass storage

That is unusual in a software project because the platform requirements are extremely tight and hardware-specific. The repository even documents a requirement for REU/DMA support and notes that GEORAM-style alternatives are not supported.

This means the operating system is more like a hardware-native runtime than a portable OS.

## 2. It uses VDC RAM and REU as first-class OS resources

A notable design choice is that the OS does not treat the display memory and expansion RAM as optional extras. They are part of the operating model.

Examples from the codebase:

- VDC memory is used to hold multiple virtual consoles
- REU DMA is used to move memory blocks efficiently
- the swapper depends on REU-backed memory management
- memory layout and startup code explicitly configure MMU and RAM banking

This is unusual because modern systems abstract memory and graphics behind layers, but OS128 is tightly bound to specific hardware memory regions and controllers.

## 3. It is a real-time, multitasking kernel for a 6502 system

This is not just a toy demo. The source includes:

- threads
- schedulers
- process IDs
- semaphores
- locks
- signals
- context switching
- blocking and wakeup logic

The code is more like an actual microkernel teaching artifact than a one-off program. It intentionally implements classic OS concepts on extremely constrained hardware.

## 4. The assembly is highly custom and macro-driven

The assembly source uses unusual conventions such as:

- zone blocks like `!zone init { ... }`
- custom macros for critical sections, bus ops, and register access
- direct register manipulation for memory, DMA, and device state
- generated label tables and exported symbol tables

The structure is not plain assembly in the conventional sense; it reads like a mini DSL for managing low-level OS construction.

## 5. It generates its own interfaces and metadata during the build

The `Makefile` is unusual because it does not just assemble code. It also generates derived files such as:

- `include/kernel.asm`
- `docs/dev/BCI-OPCODES-TABLE.md`
- `docs/dev/SEMANTIC-INDEX.md`
- label files and symbol tables

The build pipeline runs Python scripts to generate opcode tables and semantic indexes from assembly output. This is uncommon in older bare-metal projects, and it creates a very documentation-aware build process.

## 6. The project openly documents itself as a work in progress

The docs are honest about scope and limitations:

- `docs/README.txt` states that OS128 is “a work in progress” and not yet a usable operating system
- the system-requirements document describes a very specialized hardware stack
- the README frames the project as an educational OS and a learning platform, not a polished commercial product

That makes the repository feel more like a research/teaching project with a strong engineering narrative than a conventional software product.

## 7. It mixes emulator support, hardware support, and OS development in one project

The project is set up to work with:

- VICE emulator
- remote monitor commands
- Ultimate II/II+ cartridge flow
- REU emulation and hardware-level testing
- build artifacts for `.d64`, `.d71`, and CRT images

This is unusual because there is no clean separation between “OS code,” “emulator harness,” and “platform validation.” The repository is all of those at once.

## 8. It assumes a device-driver ecosystem, not a monolithic kernel

The kernel includes a broad hardware abstraction layer:

- console drivers
- IEC bus interfaces
- storage devices
- VDC display
- keyboard handling
- ACIA / RTC support
- device registration and class management

That makes the project feel closer to a microsystem or embedded OS architecture than a simple demo kernel.

## 9. There is a strong emphasis on low-level performance and hardware quirks

The code includes explicit workarounds, timing assumptions, and register-level tricks such as:

- forcing CPU speed during keyboard scan
- checking REU bank behavior to detect hardware
- waiting for VDC blanking periods
- memory copy optimization based on payload size
- device-specific compatibility notes for 1541/1581/SD2IEC/pi1541

This indicates a project shaped by hardware constraints and performance tuning rather than a simplified software-first design.

## 10. The repository is intentionally educational and exploratory

The docs repeatedly frame the project as a way to study:

- interrupt handling
- process scheduling
- buffer and stream abstractions
- bytecode interpreter design
- OS construction on 8-bit machines

This is unusual in a modern repo because it is not only building software; it is also deliberately documenting how operating-system ideas map onto old hardware.

## Overall impression

OS128 is unusual because it is a full, low-level operating-system experiment built for a very specific retro platform, with direct hardware dependencies, custom generated metadata, and educational documentation woven into the build itself. It feels less like a conventional codebase and more like a curated engineering artifact: a working OS prototype, a hardware reference platform, and a teaching machine all at once.
