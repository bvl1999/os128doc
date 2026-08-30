# Error and exception handling

This document describes how OS128 reports failures, traps machine-level faults, and enters debugging or shutdown paths in a hardware-constrained environment.

## Overview

OS128 does not try to hide the machine behind a high-level abstraction layer. Instead, the kernel keeps failure handling explicit and low-level. The key pieces are:

- the OS error table in [../../kernel/errors.asm](../../kernel/errors.asm)
- the exception handlers in [../../kernel/exception.asm](../../kernel/exception.asm)
- the interrupt path in [../../kernel/interrupt.asm](../../kernel/interrupt.asm)
- the boot-time hardware preflight in [../../kernel/boot.asm](../../kernel/boot.asm)

The result is a system that fails loudly, reports exactly what hardware requirement was violated, and often drops directly into a debug or halt state rather than attempting a generic recovery path.

## Error codes and messages

The file [../../kernel/errors.asm](../../kernel/errors.asm) defines a compact table of error identifiers and their corresponding printable strings.

The runtime path is intentionally simple:

1. an error ID is selected or returned
2. `error_print` looks it up in the string table
3. the kernel prints the matching message
4. a generic `unknown error` string is used as a fallback if the ID is not recognized

This is a classic low-memory design choice: compact indices into a static string table are cheaper than building a dynamic object-oriented exception model.

Examples in the table include:

- too many open streams
- not open / not found
- bad device
- null pointer
- out of memory
- read error / write error
- timeout
- stack overflow

This is not just a user-facing diagnostic layer; it is part of the kernel’s runtime contract. Error IDs are the bridge between low-level failure detection and a human-readable message.

## Exception handling model

The real runtime fault handling is in [../../kernel/exception.asm](../../kernel/exception.asm).

There are four main categories of exception flow:

### 1. Breakpoints and debug traps

`exception_break` is the most elaborate handler. It is reached when the system encounters a `BRK` instruction, which is treated as a trap rather than a silent no-op.

The handler does several things:

- saves the current runtime state
- switches output to the debug stream if needed
- prints a break message with the current thread identifier
- reconstructs the saved CPU state from the stack
- reads the program counter and registers from the saved stack frame
- prints the memory/register dump
- optionally switches to the debug console environment
- kills the offending thread
- enters a loop waiting for monitor/debug interaction

This is a deliberate debugging model: the OS treats a break as a controlled interruption of the current thread, then exposes enough register state to understand what failed.

The code is especially careful about the 6502 stack layout and stack signature configuration, because BRK handling relies on decoding the saved CPU state from the machine stack. That is a very hardware-specific implementation detail and an example of how the kernel treats debugging as part of the runtime rather than an external tool concern.

### 2. REU configuration failures

`exception_reu` is triggered when the system detects that the installed REU memory is insufficient.

The handler prints:

- `OSS128 requires 512k+ reu memory`

and then jumps to `halt`.

This is a hard stop because the system’s process and swap model assumes enough external RAM to support memory-resident operating assumptions. On this platform, the runtime cannot meaningfully continue with an undersized REU.

### 3. VRAM configuration failures

`exception_vram` is used when the VDC video RAM does not meet the minimum requirement.

The message is:

- `OSS128 requires 64k video ram`

This is another hard fail, reflecting the kernel’s assumption that its display and screen model require a sufficiently large VDC memory area for the console, buffers, and display state.

### 4. Console misconfiguration

`exception_log_console` prints:

- `log console not configured???`

and halts. This is an explicit sanity check for the logging/console environment; if the system cannot configure the log console, the kernel will not silently continue in a half-initialized state.

## Direct output path

The exception handlers intentionally avoid the normal stream abstraction when reporting startup and fatal failures.

The routine `direct_printstr` in [../../kernel/exception.asm](../../kernel/exception.asm) writes directly to the display hardware, clears the display state, sets the screen attributes, and prints the message without relying on a regular stream or file descriptor.

This is crucial in failure modes where the usual runtime environment may be broken or unavailable. It ensures the kernel can still produce a visible diagnostic message even when the active stream configuration is not healthy.

## Halt behavior

The common halt path is the `halt` routine:

- disables interrupts with `sei`
- executes `JAM` to stop execution
- then loops forever

This makes the system fail closed. In a bare-metal project like OS128, that is often preferable to attempting a vague “recovery” that could corrupt memory or exit into unpredictable state.

## Relationship to the interrupt and startup flow

Error handling is not isolated from the rest of the system.

- startup checks in [../../kernel/boot.asm](../../kernel/boot.asm) verify required machine resources before the system becomes operational
- the interrupt path in [../../kernel/interrupt.asm](../../kernel/interrupt.asm) can trigger a `BRK` or other fault condition
- the scheduler and process logic assume the runtime environment is consistent

As a result, OS128 treats faults as “system contract violations,” not merely as ordinary application-level exceptions.

## Why this design matters

The unusual aspect is that the kernel is very explicit about the machine assumptions it depends on.

The direct result of treating certain errors as system contract violations and using a fail-closed pattern is that the kernel itself becomes very robust. It does not allow real faults to be ignored, hidden, or silently papered over. If a resource assumption is broken or the machine cannot satisfy a required runtime condition, the system stops in a well-defined way instead of continuing in an inconsistent state.

At the same time, this is not a blanket policy of stopping on every minor issue. In most cases, when an error does not threaten the runtime stability of the kernel itself, the code simply returns an error code to the caller and lets the caller decide how to handle it. That is the practical boundary in OS128: kernel-critical faults fail closed, while recoverable application or subsystem errors are surfaced as ordinary return values and handled at the appropriate level.

A particularly important distinction is the difference between a failed thread and a failed kernel. If a thread reaches a point where it can no longer execute correctly, the system may terminate that specific thread rather than taking down the whole OS. The thread can be killed, its state cleaned up, and the scheduler can continue running other threads. This is a thread-local failure, not a kernel-wide contract breach.

Typical examples include stack underflow or overflow conditions, where the thread’s execution context is corrupted or its stack no longer matches the expected lifetime model, and signature-check failures, where a thread has reached an invalid or tampered state and is no longer trustworthy as a running execution context. In those cases, the thread is considered dead and removed, but the system itself continues running.

By contrast, if the failure represents a violation of an invariant the kernel depends on — for example, missing required hardware assumptions, invalid memory state, or a broken low-level runtime contract — then the kernel fails closed rather than trying to continue in a corrupted state.

This is a very important design distinction. It keeps the kernel strict where strictness matters, while avoiding the trap of turning every small failure into a global halt.

Instead of generic exceptions and portable error recovery, OS128 uses:

- hardware-specific fault checks
- direct display output for fatal messages
- per-resource hard stops
- a debugger-style breakpath for thread inspection

This mirrors the project’s broader philosophy: the system is designed to behave like a serious, usable multitasking OS on a specific machine, not as a portable abstraction that can hide hardware realities.

## Related files

- [../../kernel/errors.asm](../../kernel/errors.asm)
- [../../kernel/exception.asm](../../kernel/exception.asm)
- [../../kernel/interrupt.asm](../../kernel/interrupt.asm)
- [../../kernel/boot.asm](../../kernel/boot.asm)
- [../../include/memorymap.asm](../../include/memorymap.asm)
- [../../docs/architecture/interrupt.md](interrupt.md)
