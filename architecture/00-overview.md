# OS128 architecture overview

This page provides a project-level architecture map for OS128 and points to the more detailed subsystem documents in [docs/architecture](docs/architecture).

## High-level architecture

OS128 is a compact, hardware-bound operating system for the Commodore 128. The design mixes several classic OS ideas into a very small 6502 environment:

- process and thread scheduling
- block/queue based execution
- device registration and bus abstractions
- hardware interrupt handling
- memory banking and REU-backed swapping
- virtual consoles and terminal-style I/O
- a bytecode interpreter and shell command layer

At the source level, the architecture is arranged around several large areas:

- kernel logic: [kernel](../kernel)
- shared assembly definitions: [include](../include)
- userspace and shell utilities: [os128](../os128), [cmd](../cmd)
- hardware abstraction: [kernel/hal](../kernel/hal)
- device and runtime data: [kernel/dev](../kernel/dev), [kernel/lib](../kernel/lib)

## Core subsystems

### 1. Boot and system bring-up

The ROM and boot sequence initialize machine state, MMU banking, VIC-II/CIA/VDC state, and the kernel environment before control transfers into the OS runtime.

Key files:

- [kernel/boot.asm](../kernel/boot.asm)
- [kernel/kernel.asm](../kernel/kernel.asm)
- [include/memorymap.asm](../include/memorymap.asm)

### 2. Scheduler and execution model

The scheduler is one of the central subsystems. It tracks threads and processes, prioritizes runnable work using runqueues, activates process memory contexts, and periodically preempts threads through the interrupt system.

Detailed design:

- [docs/architecture/scheduler.md](docs/architecture/scheduler.md)

Source files:

- [kernel/scheduler.asm](../kernel/scheduler.asm)
- [kernel/thread.asm](../kernel/thread.asm)
- [kernel/process.asm](../kernel/process.asm)
- [kernel/runqueue.asm](../kernel/runqueue.asm)
- [kernel/interrupt.asm](../kernel/interrupt.asm)

### 3. Interrupt and exception flow

Interrupts coordinate timer ticks, device activity, serial events, and scheduler handoff. Breaks and exceptions route into a separate error path.

Detailed design:

- [docs/architecture/interrupt.md](docs/architecture/interrupt.md)

Source files:

- [kernel/interrupt.asm](../kernel/interrupt.asm)
- [kernel/exception.asm](../kernel/exception.asm)
- [kernel/errors.asm](../kernel/errors.asm)

### 4. Memory, REU, and swapping

The OS uses a highly specific banked memory model and relies on REU-backed expanded RAM for swapping and larger process footprints.

Important files:

- [include/memorymap.asm](../include/memorymap.asm)
- [kernel/hal/ram/reu.asm](../kernel/hal/ram/reu.asm)
- [kernel/hal/extram.asm](../kernel/hal/extram.asm)
- [kernel/swapper.asm](../kernel/swapper.asm)

### 5. VDC display and virtual consoles

The display subsystem is not a generic framebuffer; it is a C128 VDC-based console and screen model, including virtual console support and low-level display register control.

Relevant files:

- [kernel/hal/display.asm](../kernel/hal/display.asm)
- [kernel/hal/display/vdc.asm](../kernel/hal/display/vdc.asm)
- [kernel/dev/console.asm](../kernel/dev/console.asm)
- [kernel/vtty.asm](../kernel/vtty.asm)

### 6. Device layer and storage bus

OS128 has a device registry and bus-oriented device model. Storage devices and external hardware are discovered and exposed through a set of classes and handlers.

Relevant files:

- [kernel/devd.asm](../kernel/devd.asm)
- [kernel/hal/hal.asm](../kernel/hal/hal.asm)
- [kernel/hal/storage.asm](../kernel/hal/storage.asm)
- [kernel/hal/bus/iec](../kernel/hal/bus/iec)
- [kernel/hal/iodrv](../kernel/hal/iodrv)

### 7. Shell and command system

The shell layer converts input into tokens and dispatches commands, while the command implementations live in the internal command modules.

Detailed design:

- [docs/architecture/parser.md](docs/architecture/parser.md)

Relevant files:

- [kernel/minishell.asm](../kernel/minishell.asm)
- [kernel/lib/tokenize.asm](../kernel/lib/tokenize.asm)
- [kernel/lib/matchword.asm](../kernel/lib/matchword.asm)
- [kernel/icmd](../kernel/icmd)

### 8. Signals and lifecycle control

The kernel has a lightweight signal mechanism for interrupt delivery, process control, and thread lifecycle events. It is intentionally compact and designed to work within the process/thread model instead of becoming a full general-purpose IPC subsystem.

Detailed design:

- [docs/architecture/signals.md](docs/architecture/signals.md)

Relevant files:

- [kernel/signal.asm](../kernel/signal.asm)
- [kernel/thread.asm](../kernel/thread.asm)
- [include/signals.asm](../include/signals.asm)
- [kernel/extjumptable.asm](../kernel/extjumptable.asm)

### 9. DMA and memcopy

The DMA path wraps the hardware transfer engine and coordinates cross-process bulk transfers with a signal-based control flow. The memcopy interface chooses between a small in-process copy loop and a hardware-assisted DMA transfer depending on size and ownership.

Detailed design:

- [docs/architecture/dma.md](docs/architecture/dma.md)

Relevant files:

- [kernel/hal/dma.asm](../kernel/hal/dma.asm)
- [kernel/lib/memcopy.asm](../kernel/lib/memcopy.asm)
- [kernel/signal.asm](../kernel/signal.asm)
- [kernel/hal/ram/reu.asm](../kernel/hal/ram/reu.asm)

### 10. Streams and ringbuffers

For more sequential, byte-oriented data movement, OS128 uses a stream layer backed by circular buffers and fixed stream slots. This is the ordinary path for device I/O, command channels, and lightweight inter-thread messaging without the overhead of a full queue abstraction.

Detailed design:

- [docs/architecture/streams.md](docs/architecture/streams.md)

Relevant files:

- [kernel/stream.asm](../kernel/stream.asm)
- [kernel/ringbuffer.asm](../kernel/ringbuffer.asm)
- [kernel/buffer.asm](../kernel/buffer.asm)
- [include/memorymap.asm](../include/memorymap.asm)

### 11. Bytecode interpreter

The bytecode interpreter provides a compact internal virtual machine with a custom instruction set, register model, immediate operands, and dispatch tables.

Detailed design:

- [docs/architecture/bytecode-interpreter.md](docs/architecture/bytecode-interpreter.md)

Relevant files:

- [kernel/bci.asm](../kernel/bci.asm)
- [docs/dev/BCI-OPCODES-TABLE.md](../docs/dev/BCI-OPCODES-TABLE.md)
- [include/bcibytecode.asm](../include/bcibytecode.asm)

## Cross-cutting themes

### Hardware specificity

The codebase is tightly bound to the Commodore 128 platform. Much of the architecture is shaped by the machine’s MMU, VDC, CIA, IRQ timing, and REU features rather than by portable abstractions.

### Low-level engineering style

The system relies heavily on fixed memory addresses, register manipulation, custom macros, and explicit bank switching. This is not a high-level operating system framework; it is a highly intentional low-level machine OS.

### Educational design

Many of the architecture docs exist to make the system understandable as a learning artifact. The repo is organized not only to build an OS, but also to explain how classic OS concepts map onto 8-bit hardware.

## Architecture document map

- [docs/architecture/scheduler.md](docs/architecture/scheduler.md) — scheduler design and thread/process dispatch
- [docs/architecture/interrupt.md](docs/architecture/interrupt.md) — IRQ/NMI flow and scheduler handoff
- [docs/architecture/signals.md](docs/architecture/signals.md) — lightweight signal model and thread/process lifecycle notifications
- [docs/architecture/streams.md](docs/architecture/streams.md) — stream and ringbuffer model for sequential byte-oriented I/O and buffered control flow
- [docs/architecture/dma.md](docs/architecture/dma.md) — DMA transfer path, memcopy policy, and cross-process transfer coordination
- [docs/architecture/error-handling.md](docs/architecture/error-handling.md) — kernel faults, hard-stop paths, and debugging exceptions
- [docs/architecture/parser.md](docs/architecture/parser.md) — shell parsing and command matching
- [docs/architecture/bytecode-interpreter.md](docs/architecture/bytecode-interpreter.md) — VM interpreter design

## Missing architecture areas that are worth adding

While the repo already has several architecture notes, a few areas would benefit from dedicated reference pages:

- memory map and bank management
- device registration and device classes
- VDC console architecture
- stream/file descriptor model
- REU swapping and process memory handoff
- HAL wiring between hardware controllers and OS services

These are represented in the source tree and can be expanded into additional focused docs under [docs/architecture](docs/architecture).
