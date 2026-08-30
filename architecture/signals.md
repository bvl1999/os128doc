# Signals and thread notification

This document describes the signal model in OS128 at a high level. The signal machinery is implemented in [../../kernel/signal.asm](../../kernel/signal.asm), with thread state in [../../include/memorymap.asm](../../include/memorymap.asm) and user-visible entry points in [../../kernel/extjumptable.asm](../../kernel/extjumptable.asm).

## Overview

OS128 uses a lightweight signal system to allow threads and processes to notify each other without building a full message-passing framework. The model is intentionally small and low-overhead.

The key ideas are:

- a signal is a compact numeric event code
- each thread can have a signal handler
- signals are delivered to a target thread or process
- delivery is not a general-purpose IPC system; it is an inter-thread control mechanism
- the kernel can trap and interpret critical signals such as `SIGINT`, `SIGHUP`, `SIGKILL`, `SIGSTOP`, `SIGCONT`, and `SIGCHLD`

This is a direct, hardware-aware design: the kernel does not model a modern POSIX signal subsystem in full detail, but it does provide the practical control flow needed for process cleanup, terminal interruption, and thread lifecycle management.

The key architectural point is that signals are primarily asynchronous events delivered to a target thread. They are not defined solely as message-passing tokens. A signal is an asynchronous software interrupt-like event with a signal number, a source, and a one-byte payload. The payload is optional in the sense that not every signal needs meaningful data, but the event model allows it when useful.

Unlike a stream, a signal is not something a thread has to “poll for” or actively read out of. The signal is delivered regardless of what the thread is doing, and the thread only responds when it reaches the handler or the scheduler path that processes the pending event. This makes signals fundamentally different from buffered sequential I/O: they are asynchronous notifications, not passive data channels that require deliberate consumption.

That makes signals a general-purpose thread notification primitive rather than a one-way message queue. They are ideal for things like “wake this thread”, “this device has changed state”, “this target is being interrupted”, or “this thread has completed an action and needs attention”. In many cases, the receiver can process the signal directly; in other cases, it can use the signal as a cue to inspect a stream, a buffer, or another shared state region.

This is why signals and streams are complementary rather than exclusive. A stream can carry payload data, but the signal itself remains the asynchronous event. The stream can be used to hold the message body, and `EOF` can mark the end of a message or record, but that is a usage pattern on top of the primitive—not the definition of what a signal is.

## Signal model

A signal is conceptually a small event that is posted to a target thread. The target thread keeps a signal count in its thread state and the kernel may dispatch a handler when the signal is processed.

The most important design characteristics are:

- signal delivery is compact and fast
- signal handlers are per-thread, with a default handler for process-level behavior
- signal processing is integrated with thread stack manipulation and scheduler wakeup logic
- signals are used to represent both user-visible events and kernel-managed lifecycle events

The kernel uses signal delivery for things like:

- terminal interrupt handling (`SIGINT`)
- process termination (`SIGKILL`)
- suspend/resume (`SIGSTOP`, `SIGCONT`)
- hangup handling (`SIGHUP`)
- child process notification (`SIGCHLD`)
- device-specific cues such as DMA completion (`SIGDMA`)

`SIGDMA` is especially important because it serves a second, more specialized IPC-like role: it coordinates inter-process DMA transfers. When one process sends data to another, the destination process is signalled, receives the transfer metadata, and can inspect the target address, length, and transfer properties. It may accept the transfer, refuse it, or modify the destination/length and allow the operation to continue with adjusted parameters. In this case the signal is an asynchronous control event for a DMA handoff, not merely a trivial status bit.

## Execution model

Signals are delivered by building a small synthetic stack frame and switching execution into a signal handler. This is done in a way that preserves the current thread context and allows the thread to return cleanly after signal processing.

At a high level this looks like:

1. the kernel identifies the target thread or process
2. it validates that the target is still valid
3. it increments the per-thread signal count
4. it creates the minimal stack frame for the handler
5. it switches to the handler context
6. the handler decides whether to process, ignore, or terminate the thread
7. control returns through the signal-finish path back to the scheduler or thread resume flow

This design is intentionally very lightweight and engineered for a system with strict memory and CPU constraints.

## Why the model is small

The signal system is intentionally not a full general-purpose IPC model. It focuses on the cases the OS actually needs in a small multitasking environment:

- interrupting a running thread
- terminating a thread or process
- waking or suspending a thread
- signalling lifecycle and I/O events

This keeps the runtime cost low and avoids turning signal delivery into a large object model or a broad messaging framework.

## Event-driven use, not only message passing

The signal mechanism can be used to build a lightweight event-driven message pattern, but that is a usage model layered on top of the underlying primitive. The primitive itself is an asynchronous software interrupt delivered to a thread.

Typical usage patterns include:

- a signal announces that a device or stream has become ready
- a handler or receiver inspects a stream for data after the event
- a thread reacts to a system event such as termination, resume, or child-state change
- a user-defined signal carries a tiny status or opcode byte as part of the event
- `SIGDMA` coordinates an inter-process transfer, allowing the receiving process to validate or adjust the DMA target and length before completion
- `EOF` can indicate message completion when the stream is being used as a payload carrier

This is not the same as saying signals are defined as message-passing objects. The event remains primary, and the stream acts as an optional secondary channel when data needs to be carried alongside the asynchronous wakeup. `SIGDMA` is a good example of a signal used for a specialized control path: it is an asynchronous event that carries the transfer contract, not just a wake-up flag.

## Default handlers and process policy

The default signal handler in [../../kernel/signal.asm](../../kernel/signal.asm) interprets a number of system signals and chooses a policy. For example:

- `SIGKILL` and similar termination signals terminate the thread or process
- `SIGINT` is used for interrupt-like flow control
- `SIGCHLD` is used for child-process state notification
- device or DMA events may route to custom handlers

This means the signal subsystem is not only a notifier; it is also a simple process-control mechanism.

## Relationship to the scheduler and thread model

Signals are tightly connected to the scheduler and thread lifecycle:

- a signal can wake or suspend a thread
- a signal can trigger cleanup paths during thread termination
- a signal may be processed by the signal handler as part of the scheduler-driven return path
- the thread state tracks whether a signal is pending and how many are queued

This is why the signal model belongs alongside the thread model and scheduler design, even though it is conceptually a separate mechanism.

## Related files

- [../../kernel/signal.asm](../../kernel/signal.asm)
- [../../kernel/thread.asm](../../kernel/thread.asm)
- [../../kernel/scheduler.asm](../../kernel/scheduler.asm)
- [../../kernel/interrupt.asm](../../kernel/interrupt.asm)
- [../../include/signals.asm](../../include/signals.asm)
- [../../kernel/extjumptable.asm](../../kernel/extjumptable.asm)
- [../../include/memorymap.asm](../../include/memorymap.asm)
