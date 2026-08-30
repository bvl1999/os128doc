# Engineering notes

This document records the engineering-oriented observations about how OS128 is structured and why its design choices are notable.

## Related notes

- [Unusual aspects](unusual.md)
- [Interesting findings](interesting.md)
- [Engineering details](engineering-details.md)
- [Architecture overview](../architecture/00-overview.md)

## 1. Hardware-first architecture

The project is built around the expectation that the host machine is a C128 with a VDC display and REU expansion. This is not a software abstraction layer over a generic machine; it is a kernel designed to directly exploit a known retro platform.

Engineering consequence:

- low-level register use is pervasive
- memory mapping is explicit and fixed
- I/O controller interaction is deeply integrated into the OS

## 2. Split responsibilities between kernel, drivers, and libraries

The repo separates concerns into a few broad areas:

- `kernel/` for kernel logic
- `kernel/dev/` for device abstraction
- `kernel/lib/` for small helper routines
- `include/` for shared assembly definitions
- `os128/` and `cmd/` for userland and tooling

This is a disciplined structure for a tiny OS and makes the platform easier to reason about despite the low-level nature of the code.

## 3. A custom build pipeline generates interface artifacts

The top-level build does more than compile code. It also generates:

- exported kernel symbol definitions
- opcode documentation
- semantic index files
- device and API metadata

This means the system is designed to be not only buildable, but also inspectable and explainable from generated artifacts.

## 4. Memory management is a central design concern

The operating system cares heavily about:

- banked memory
- REU DMA transfer logic
- expanded memory detection
- process-owned memory regions
- virtual console storage in VDC RAM

This is one of the clearest indicators that the kernel is designed around constrained memory resources rather than a flat address model.

## 5. Scheduling and process state are implemented directly in assembly

The kernel uses explicit process, thread, and scheduler concepts, but these are implemented with low-level state tracking rather than high-level abstractions. This is an engineering strength for educational value, but it also means the code is tightly coupled to machine-level scheduling behavior.

## 6. The project intentionally uses hardware quirks and compatibility checks

The codebase includes compatibility checks and workarounds for various devices and generation-specific quirks, including:

- REU detection logic
- VDC timing assumptions
- performance-sensitive copy routines
- drive-type support and cautionary notes

This reflects an engineering culture of precise hardware compatibility, not generic portability.

## 7. The kernel is designed for a mixed boot-and-runtime environment

The build artifacts support not just native assembly but also CRT, floppy, and emulator targets. The project is set up to test itself in an emulated environment while still targeting actual C128 hardware assumptions.

## 8. The architecture is intentionally constrained and explicit

This codebase is unusual because it optimizes for clarity of low-level behavior over the convenience of abstraction layers. The tradeoff is a system that is harder to generalize, but much easier to understand in terms of actual hardware mechanics.

## 9. Observed design philosophy

The strongest overall engineering theme is this:

- keep the system close to the metal
- exploit known hardware features deliberately
- make memory and I/O layout visible and intentional
- document the platform constraints openly

That philosophy is unusual but coherent: OS128 is built as a minimal but deep exploration of OS design in a hardware-constrained environment.

## Summary

OS128 is less a portable software project and more a hardware-native operating-system experiment. The engineering notes revolve around tight platform coupling, explicit memory management, custom build generation, and a kernel architecture that prioritizes low-level understanding over portability.
