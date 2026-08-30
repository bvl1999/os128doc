# Engineering details

This document collects the subsystem-by-subsystem observations about OS128, including the unusual constraints, implementation patterns, and engineering trade-offs that stand out in this codebase.

## Related notes

- [Unusual aspects](unusual.md)
- [Interesting findings](interesting.md)
- [Engineering notes](engineering-notes.md)
- [Architecture overview](../architecture/00-overview.md)

## 1. Memory map and system layout

The memory model is one of the most unusual and defining aspects of the project.

### Findings

- The kernel uses a custom memory map instead of a conventional flat model.
- A large portion of the layout is defined in `include/memorymap.asm`, with low RAM, runtime data, process tables, thread tables, device tables, and runqueue structures all laid out explicitly.
- The design uses banked RAM and process-specific memory contexts, which is essential on a machine with limited address space.
- The runtime data area is not generic metadata; it stores scheduler state, REU state, VDC memory size, current process, current thread, swap counters, device state, and IRQ state.
- The memory map is tightly coupled to the machine's MMU model and to process switching.

### Engineering significance

This is not just a label file; it is the operating system’s memory contract. Every subsystem depends on these fixed addresses and assumptions. In a modern OS, this would normally be abstracted away by the runtime or compiler; here it is an explicit part of the kernel design.

## 2. Boot and ROM initialization

The startup path is unusual because it behaves like a hardware bootstrap sequence more than a conventional program entry point.

### Findings

- `kernel/boot.asm` and the ROM header area in `kernel/kernel.asm` establish a custom ROM boot sequence.
- The code clears and initializes the machine state before the OS is fully active.
- It explicitly disables interrupts and clears MMU and VIC/CIAs state before moving into kernel initialization.
- There are power-on checks for keys like Esc and Run/Stop that influence whether the ROM should continue booting.
- The code configures MMU registers and then jumps into `main` or the early init flow.

### Engineering significance

The boot process is a classic example of a hardware-specific “bring-up” routine. It is not portable, and it is not designed for a general-purpose environment. It is purpose-built for a specific retro platform and makes assumptions about device state from the moment the machine starts.

## 3. Process model and thread model

The OS is built around the idea that work is done by threads assigned to processes, with explicit state tables and scheduling metadata.

### Findings

- Process and thread tables are explicitly allocated in low RAM.
- There are process status flags, thread priorities, thread parent references, load counters, and swap metadata.
- The runqueue structure is intentionally maintained as separate idle/low/high/critical queues.
- Scheduler logic uses counters like `thread_counth`, `scheduler_run`, and `load_tableh` to estimate runtime load.
- Threads are not abstracted away; they are effectively hardware-visible state machines.

### Engineering significance

This is a very “bare metal” process model. The OS treats process scheduling as a low-level data-structure problem rather than a higher-level concurrency framework.

## 4. Scheduler and runqueue subsystem

The scheduler is one of the core subsystems of the kernel and shows a strong low-level design philosophy.

### Findings

- `kernel/scheduler.asm` initializes threads and processes and sets the initial active task.
- Runqueue state is tracked in separate queues rather than a single ready list.
- The code updates run-load metrics and uses starvation and count-based heuristics.
- Scheduler decisions are affected by process memory map, thread state, and load tables.
- Thread activation is tied to process context loading and MMU setup.

### Engineering significance

The scheduler is not generic or abstract; it is highly tuned to the constraints of a tiny 8-bit machine. The design favors deterministic state management and compact tables over modern scheduling abstractions.

## 5. Interrupt and exception handling

The interrupt path is a major subsystem and is unusually elaborate for a project of this size.

### Findings

- `kernel/interrupt.asm` sets up CIA timer IRQs and acknowledges pending interrupts.
- The IRQ handler checks multiple sources, including VIC-II, ACIA, and CIA interrupts.
- A dedicated break-handler path exists for `BRK` and exception handling.
- The kernel preserves register state and swaps memory context before dispatching tasks.
- `exception.asm` is used for specific error cases such as REU or memory failures.

### Engineering significance

The interrupt system is handling both timing and device events while also coordinating the scheduler. This makes the IRQ path a core coordinator rather than just a system-level callback.

## 6. VDC display subsystem

The display driver is one of the most specialized subsystems in the project.

### Findings

- The VDC driver is defined in `kernel/hal/display/vdc.asm`.
- It handles VDC register writes, reads, address set operations, and screen memory setup.
- There are explicit routines for setting character base, display base, and attribute base.
- VDC memory size detection is performed by manipulating registers and probing memory behavior.
- The display code includes waits for vertical blanking and memory synchronization, which are direct hardware timing controls.

### Engineering significance

The OS is designed around the VDC not simply as a display, but as a core memory and console resource. It is a good example of a kernel that assumes display memory is part of the system architecture rather than a separate peripheral.

## 7. Virtual console and terminal layer

The console subsystem is directly tied to the VDC and to virtualized terminal behavior.

### Findings

- `kernel/dev/console.asm` and `kernel/vtty.asm` indicate a virtual terminal model.
- Multiple consoles are tracked, and each console has its own screen and stream behavior.
- The kernel can switch between consoles and maintain output/input streams.
- The initialization path sets up console classes and associated device registration.

### Engineering significance

This is unusual because the OS supports multiple virtual terminals on hardware that originally had a much more limited display system. The implementation is a very direct mapping of a modern console model onto 1980s hardware.

## 8. Device abstraction and driver model

The project has an explicit device model with registration, class handling, bus IDs, and I/O abstractions.

### Findings

- `kernel/devd.asm` and related device setup code manage device registration classes.
- Devices are assigned classes like console, disk, RAM, ROM, and handler-backed devices.
- There is a bus abstraction for IEC and UCI/Ultimate device interfaces.
- Device registration is central to how storage and console hardware are discovered and exposed.

### Engineering significance

This is a true OS-like abstraction layer, with device discovery and registration as first-class functionality. It is far richer than a simple hardware direct-call model.

## 9. IEC bus and storage subsystem

The storage and bus subsystem is a defining part of the project’s hardware support.

### Findings

- The system supports IEC devices, floppy devices, CMD devices, and bootable mass storage.
- `kernel/hal/bus/iec/` contains the protocol transport, device layer, and probing logic.
- The system depends on burst-mode and device capability checks, and it documents non-recommended storage choices such as 1541 or slower devices.
- The storage model is not just file I/O; it is an embedded hardware bus system with device negotiation and command handling.

### Engineering significance

This is a real bus-driven storage architecture. It makes the OS feel like a retro hardware platform with an operating-system shell wrapped around it.

## 10. REU and expanded memory subsystem

This is one of the most unusual and important engineering choices in the entire repo.

### Findings

- The kernel explicitly checks for REU capability and RAM size.
- `kernel/hal/ram/reu.asm` contains REU detection and DMA setup logic.
- `kernel/swapper.asm` uses the REU as swap storage for processes.
- The system memory model uses REU banks to support process swapping and memory expansion.
- The project documents that GEORAM-like alternatives are not supported.

### Engineering significance

The amount of memory is not just a convenience; it is a high-level subsystem. The REU is used as a memory extension and for process swapping, making it a core part of the kernel’s memory management model.

## 11. DMA and memory copy subsystem

The DMA and copying logic is optimized for speed and for the specific machine hardware.

### Findings

- `kernel/hal/dma.asm` exposes DMA functions for moving blocks between RAM and REU-backed memory.
- `kernel/lib/memcopy.asm` contains a special-case copy implementation.
- The copy logic decides between a simple loop and a DMA-based transfer depending on payload size and same-process/vs-cross-process behavior.
- There are explicit comments about hardware quirks and “arbitrary” thresholds.

### Engineering significance

This is a low-level optimization layer tuned for a machine with constrained CPU power and strong DMA support. It shows a clear pattern: when possible, offload to the DMA engine rather than do general-purpose copy work in CPU code.

## 12. Hardware abstraction layer (HAL)

The HAL is broad and highly device-specific rather than device-agnostic.

### Findings

- `kernel/hal/hal.asm` wires together keyboard, RTC, ACIA, IEC, and hardware init paths.
- The HAL also launches init threads for RTC and ACIA-related tasks.
- The design is layered around hardware devices, not an abstract platform API.

### Engineering significance

The HAL is intentionally opinionated. It exists to hide the ugly details of the machine while still exposing exactly the set of hardware features this OS requires.

## 13. Keyboard and human input subsystem

The keyboard layer is more complex than a simple polling loop.

### Findings

- `kernel/hal/hid/keyboard.asm` and the interrupt flow interact with keyboard scan timing.
- The code temporarily adjusts CPU speed for keyboard scanning to ensure stable timing.
- Keyboard state is tracked through runtime flags and device-specific scan logic.

### Engineering significance

Input handling is treated as a hardware timing problem and not as a simple event callback. This is characteristic of the OS as a whole: it is engineered around exact machine timing and controller behavior.

## 14. RTC and clock logic

The clock subsystem is present, but it is integrated into the interrupt and hardware abstraction layers rather than treated as a standalone modern subsystem.

### Findings

- RTC device synchronization is tied to interrupt handling.
- Clock state is tracked in runtime data and includes configuration values like `rtc_device` and `rtc_mode`.
- There are checks for carrying the date/time into the system clock and device state.

### Engineering significance

This subsystem demonstrates another recurring pattern: small but important real-time devices are integrated at the kernel level rather than isolated behind a general abstraction library.

## 15. ACIA and serial communication subsystem

The project includes serial support using ACIA hardware, and it is treated as a meaningful subsystem.

### Findings

- ACIA support is optional but clearly integrated into interrupt paths.
- Serial handler logic is present for status, command, and carrier checks.
- A dedicated interrupt path exists that can invoke an ACIA handler.

### Engineering significance

Serial support is not an afterthought. It is part of the system’s communication model and fits within the project’s broader device- and bus-centric kernel design.

## 16. Debugging and inspection support

The kernel has a substantial debugging culture built into it.

### Findings

- There are debug streams, error print routines, and stream outputs.
- The code includes debug printing helpers, breakpoints, and process/thread diagnostics.
- `kernel/debug.asm` and the generated label outputs support low-level inspection.
- The project also includes remote monitor support via VICE configuration.

### Engineering significance

This is a toolchain-oriented OS: it expects you to inspect and step through execution rather than relying only on high-level logs. That is common among educational and hardware-oriented kernels.

## 17. Built-in command and userland layer

The project contains a user-visible command system and internal shell functionality.

### Findings

- `kernel/minishell.asm` and `kernel/icmd/` provide a command model.
- `cmd/` houses utilities and demonstration programs.
- OS128’s interface includes device switching, shell commands, and command execution semantics.

### Engineering significance

The design is not just the core kernel; it also includes a small userspace environment. That gives the project a broader OS feel, even though it remains tightly bound to the machine architecture.

## 18. Bytecode interpreter and internal execution model

The repository includes a bytecode interpreter subsystem, which is surprisingly advanced for a retro OS.

### Findings

- `kernel/bci.asm` and the generated opcode table documentation are clearly designed around an internal bytecode interpreter.
- The docs describe the bytecode layer as a key subsystem and associated learning target.
- This is not just a command interpreter; it is a VM-like execution engine.

### Engineering significance

This is one of the more unusual “modern OS” concepts embedded in a small 8-bit system. The project merges classic microkernel design with a lightweight virtual machine model.

## 19. File descriptor and stream abstraction

The OS contains a stream and descriptor layer that is more advanced than it initially appears.

### Findings

- `kernel/stream.asm`, `kernel/fdesc.asm`, `kernel/channel.asm`, and related files define stream-oriented behavior.
- File descriptors, buffers, and channels are tracked as operating-system entities.
- The kernel supports input/output channels and terminal-style communication flows.

### Engineering significance

This is a classic OS-building block and helps explain why the project feels like an educational operating system rather than just a kernel skeleton.

## 20. Error and exception model

The error handling system is more than a few messages; it is part of the runtime model.

### Findings

- `kernel/errors.asm` and `kernel/exception.asm` define OS-level error values.
- The OS traps memory, REU, VDC, and process-related failures.
- Error handling is wired into the interrupt and startup flow.

### Engineering significance

This makes the OS more coherent as a system: it is not merely code that runs; it is a structured runtime with explicit failure states and recovery logic.

## Summary

Across the subsystems, the recurring pattern is that OS128 is designed around a very specific Commodore-128 hardware environment, with each subsystem tightly aligned to the machine’s architecture. The project’s unusual character comes from the fact that memory, scheduling, display, I/O, storage, and debugging are all implemented as explicit hardware-aware operating system features rather than as modern, portable abstractions.

That combination makes the codebase both fascinating and difficult to generalize: it is a real kernel for a constrained platform, a teaching artifact, and a hardware-driven engineering experiment all at once.
