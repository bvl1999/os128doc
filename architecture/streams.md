# Streams, ringbuffers, and buffered I/O

This document describes the sequential-data path in OS128: the stream interface, the ringbuffer implementation beneath it, and the way stream slots map to buffer storage. The runtime design is implemented in [../../kernel/stream.asm](../../kernel/stream.asm), [../../kernel/ringbuffer.asm](../../kernel/ringbuffer.asm), and [../../kernel/buffer.asm](../../kernel/buffer.asm), with slot and memory layout defined in [../../include/memorymap.asm](../../include/memorymap.asm).

## Overview

For smaller or more sequential data, OS128 does not use a DMA-like bulk transfer model. Instead, it exposes a byte-stream abstraction backed by a circular buffer. This is the general-purpose path for ordered data movement between threads, devices, and process I/O.

The stack is intentionally small and cheap:

- stream: user-facing byte-oriented abstraction
- ringbuffer: circular buffer with read/write positions, error state, and blocking behavior
- buffer slots: per-stream metadata and backing storage assigned from a fixed pool

This is a classic “lightweight stream over ringbuffer” design, and it fits the overall OS128 philosophy: keep the hot path compact, keep metadata small, and avoid turning every I/O event into a large object model.

## Stream model

The stream layer is the sequential-data equivalent of the signal layer’s event model. A stream is a queue-like byte channel, but it is intentionally simpler than a general-purpose message system.

Key properties:

- byte-oriented access, not object-oriented message passing
- blocking or non-blocking read/write variants
- explicit end-of-file and error states
- per-stream read and write positions
- lock support for producer/consumer synchronization
- inline escape handling for framed control sequences

The main difference from signals is that a thread must actively read from a stream to receive its data. A stream does not forcibly interrupt the thread the way a signal does; it is a passive buffer that must be consumed by the target thread when it chooses to read from it. A signal is sent regardless of thread state, while a stream is only meaningful once the consumer takes action.

The actual read/write wrappers in [../../kernel/stream.asm](../../kernel/stream.asm) call into the ringbuffer implementation. This keeps the stream interface stable while letting the ringbuffer maintain the actual queue semantics.

## Ringbuffer design

The backing circular buffer keeps the read and write positions in per-stream state. The memory layout in [../../include/memorymap.asm](../../include/memorymap.asm) stores these values in a small fixed table:

- stream read positions
- stream write positions
- stream status flags
- buffer backing memory

The ringbuffer logic in [../../kernel/ringbuffer.asm](../../kernel/ringbuffer.asm) does the work:

- read and write positions advance around the circular buffer
- empty and full conditions are detected cheaply by comparing positions
- blocking reads or writes suspend the current thread until space or data becomes available
- EOF is marked by setting a status flag and then blocking further writes until the peer clears the condition

This is intentionally a very compact design. The buffer is not a sophisticated queue with descriptors or heap allocations; it is just a circular byte store plus a few state variables.

## Buffer allocation and mapping

The stream system keeps a pool of stream slots and associated backing storage. The allocation path is in [../../kernel/buffer.asm](../../kernel/buffer.asm).

A stream is allocated by selecting a free slot and marking it active. In the code:

- `buffer_alloc` scans for the first free `stream_status` entry
- it marks the slot active by setting the allocated bit
- `buffer_free` clears the status and makes the slot reusable
- `buffer_ptr` can return the base pointer for a given buffer slot

This means the system effectively maintains a small set of reusable stream buffers rather than treating every stream as a heap-allocated object. That is a good match for a ROM- and RAM-constrained OS: a fixed pool is simpler, cheaper, and easier to reason about under interrupts and scheduler preemption. More importantly, the buffers remain resident in kernel-owned memory, so ordinary stream traffic does not require swapping a process back in just to service queued data. The communication path stays ready even when the owning process is sleeping or blocked elsewhere.

## Blocking and wakeup semantics

The ringbuffer implementation is tightly connected to thread scheduling.

When a thread attempts to read and no data is available, the code calls `thread_block_read` and then yields. Likewise, a write that blocks on full space calls `thread_block_write` and yields. This means the stream path is not a passive queue; it is integrated with the scheduler and thread wait mechanism.

This is a critical design point:

- stream waits are thread-blocking waits, not polling loops
- the stream may wake a thread when data or space becomes available
- the scheduler can continue running other tasks while the blocked thread sleeps

This gives the stream layer the same basic “async event + queue state” style that the signal layer uses, but for ordinary byte data rather than control events.

## EOF, errors, and inline framing

The ringbuffer also supports a compact message/record framing pattern using inline conditions rather than a separate messaging layer.

The code implements this in a few ways:

- `ringbuffer_set_error` marks the stream as having an error or EOF condition
- `ringbuffer_eof` sets the EOF bit on the current output buffer
- reads detect EOF and return a special error code, while clearing the EOF state on consumption
- writes are blocked once EOF is set until the condition is cleared

This makes EOF behave like an inline control condition in the stream, which is exactly the sort of thing that supports higher-level message framing and record boundaries without inventing a full message queue abstraction. EOF is a stream-level terminal condition: once set, writes are blocked until the condition is cleared, so it acts as a hard boundary rather than a byte-level protocol marker.

The escape mechanism is part of the same framing idea, but with a different purpose. In [../../kernel/stream.asm](../../kernel/stream.asm), an escape byte (`0x1b`) is treated specially when reading from a stream. If an escape is encountered, the implementation reads the next byte and, if it is not another escape byte, dispatches to an escape handler pointer. This lets the stream layer carry control sequences, record delimiters, and lightweight protocol events in-band without requiring a separate message channel. In other words, the stream can combine ordinary data with inline control metadata, and the receiver decides how to interpret the escape sequence. This is especially natural for terminal-style protocols, but the mechanism is generic enough to support other in-band control uses as well.

## Relationship to signals and DMA

The stream path complements the signal and DMA mechanisms:

- signals are the asynchronous event/notification primitive
- streams carry the byte sequence or payload
- DMA is for large bulk transfer when performance matters
- ringbuffers are the low-overhead sequential data path for ordinary I/O and inter-thread communication

The model is therefore layered:

- a signal wakes or informs a thread
- a stream carries a byte sequence or record
- the receiving thread may inspect the stream and decide what to do next
- if the transfer is large or cross-process, a DMA path may be used instead

This gives OS128 a useful spectrum of data movement primitives rather than a single “one size fits all” mechanism.

## Typical usage patterns

The stream/ringbuffer model is well suited for:

- terminal and console input/output
- device driver I/O
- FIFOs between threads or processes
- buffered command/data exchange
- stream framing where an `EOF` or status bit marks the end of a record

This is especially useful in a tiny system where a full IP-like message queue would be too expensive in both code and memory.

## Related files

- [../../kernel/stream.asm](../../kernel/stream.asm)
- [../../kernel/ringbuffer.asm](../../kernel/ringbuffer.asm)
- [../../kernel/buffer.asm](../../kernel/buffer.asm)
- [../../include/memorymap.asm](../../include/memorymap.asm)
- [../../kernel/signal.asm](../../kernel/signal.asm)
- [../../kernel/hal/dma.asm](../../kernel/hal/dma.asm)
- [../../kernel/extjumptable.asm](../../kernel/extjumptable.asm)
