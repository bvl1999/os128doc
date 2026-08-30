# DMA and memcopy

This document describes the DMA path and the related memcopy interface in OS128 at a high level. The implementation lives in [../../kernel/hal/dma.asm](../../kernel/hal/dma.asm) and the cross-process copy entry points are in [../../kernel/lib/memcopy.asm](../../kernel/lib/memcopy.asm). The signal coordination for DMA handoff is in [../../kernel/signal.asm](../../kernel/signal.asm).

## Overview

The actual design history matters here: the DMA engine was created first as a reusable transfer primitive and channel multiplexer, and only later was a memcopy acceleration layer built on top of it for same-process and cross-process copies. In other words, DMA is not a thin wrapper around a memcpy convenience routine; it is the underlying transfer infrastructure, and the memcopy interface is one of the consumers of that machinery.

OS128 does not treat memory copying as a generic library feature only. It is a cross-process operation with hardware-aware behavior and process ownership boundaries. The system has two related layers:

- the DMA engine and channelized transfer path as the primary primitive
- the memcopy acceleration layer that routes same-process or short transfers, and larger cross-process transfers, into the DMA-backed machinery when appropriate

The design is very conscious of hardware constraints: DMA is fast, but it must be carefully coordinated with process memory maps, bank switching, and signal handling. The result is a mechanism that can move large blocks efficiently while still respecting process isolation.

## Memcopy interface

The high-level interface is exposed through `memcopy` and `memcopy_xp` in [../../kernel/lib/memcopy.asm](../../kernel/lib/memcopy.asm).

This layer was added after the DMA engine already existed. The practical effect is that the memcopy interface is a policy layer on top of a richer primitive: it chooses how to route a requested transfer based on size, ownership, and hardware capability, rather than inventing a separate transfer mechanism from scratch.

At a high level:

- if the source and destination belong to the same process, the kernel can use a compact inline copy loop for small transfers
- if the transfer is large, or if it crosses process boundaries, it routes into the DMA memcopy path
- zero-length transfers are rejected as an `ERROR_EOF`-style no-data condition

This lets the system avoid using DMA for tiny copies while still using it where the cost/performance tradeoff is beneficial.

The key decision is intentionally simple:

- same process + short size -> plain memory copy
- cross process or large transfer -> DMA-assisted path

This keeps the common case cheap and preserves the hardware-accelerated path for the larger, more expensive transfers.

## DMA transfer model

The DMA implementation in [../../kernel/hal/dma.asm](../../kernel/hal/dma.asm) treats transfers as a chunked operation with channel state. The sender prepares a source address, the receiver prepares a destination address, and the DMA hardware moves data between the system RAM and an intermediate buffer bank.

The important design details are:

- transfer size is broken into chunks controlled by `dmacounter` and `dmaochunklen`
- the sender updates its source pointer as each chunk completes
- the receiver updates its destination pointer after each chunk is moved
- each DMA channel owns a little bit of metadata so multiple transfers can be tracked safely
- the code keeps memory-map switching and hardware state updates in critical sections to avoid corruption during context switches

This is a classic “hardware-aware bulk transfer” design: the OS intentionally keeps the DMA path around the real memory-map and bank-switching rules so no process sees a stale or incorrect address mapping.

The chunking is not accidental. It serves two important purposes. First, a very large transfer cannot monopolize a time slice or a scheduler window; by splitting it into bounded chunks, the kernel can yield control back to the scheduler and keep responsiveness high. Second, the same channel and transfer machinery can be multiplexed across multiple logical transfers, so several concurrent flows can share the same underlying DMA infrastructure without permanently consuming a whole transfer path. In effect, the design behaves like a small channel-switching, packetized transport layer: transfer metadata is negotiated per chunk, and each chunk is treated as a dispatchable unit on a logical channel.

## Cross-process DMA coordination

The part that makes this more than a simple memory copy is the inter-process handoff. When a process copies memory to another process, OS128 does not blindly allow that DMA to proceed without coordination.

The flow is:

1. the sender prepares the source chunk and requests the transfer
2. the destination process is identified and checked
3. the kernel signals the destination process with `SIGDMA`
4. the target thread receives the signal and handles the receive side
5. the destination can inspect the transfer target address and length
6. the destination may accept the transfer, reject it, or modify the transfer properties before the receive continues

The actual implementation is in `dma_memcopy` and `dma_signal_handler` in [../../kernel/hal/dma.asm](../../kernel/hal/dma.asm). The signal is therefore not just a wakeup; it is a transfer-control event used to negotiate and coordinate cross-process DMA. This is one of the clearest examples of the signal model being used as an asynchronous control channel rather than a pure “status bit”.

## DMA as a memory transfer primitive, not just a copy helper

The code is intentionally written as a transfer primitive above the simple `memcopy` helper. The same service is used for large copy operations, buffer movement, and process-to-process transfers.

The design pattern is:

- keep the transfer metadata in per-channel state
- move data through a dedicated DMA bank
- update source and destination pointers as chunks complete
- preserve process isolation
- invoke signal-based coordination for cross-process transfer control

This is very different from a “just memcpy” abstraction. It is a low-level memory channel protocol with explicit synchronization and ownership boundaries.

## Locking and synchronization

DMA is not free of synchronization concerns. The implementation includes semaphore-style locking around channel usage (`dma_semaphore_acquire`, `dma_semaphore_release`, and the wait-clear/wait-set helpers in [../../kernel/hal/dma.asm](../../kernel/hal/dma.asm)).

This matters because the transfer moves data through shared hardware state. Without coordination, one process could race another process or overrun a channel while the DMA controller is mid-transfer. The locking layer is therefore part of the transfer contract, not an optional add-on.

## Relationship to signals

The DMA engine and the signal layer are intentionally coupled:

- the DMA path moves bytes efficiently
- `SIGDMA` coordinates the receiving side and allows transfer negotiation
- the receiving thread may inspect or modify the target address and length before the next chunk is accepted

This yields a useful hybrid: hardware-accelerated bulk movement plus an asynchronous control signal that keeps the transfer policy inside the destination process.

## Design inspiration: IBM channel I/O

The architecture here is not a generic modern IPC abstraction. It is intentionally patterned after the channel-oriented transfer model familiar from IBM mainframe I/O.

The conceptual parallels are clear:

- the sender presents a transfer contract: source, target, and length
- the channel or engine performs the actual data movement asynchronously
- the receiving side can inspect the requested transfer
- the transfer may be accepted, rejected, or adjusted before completion
- the control event is separate from the raw data movement itself

That model is directly mirrored in the OS128 DMA path: the sender sets up the transfer, the signal notifies the target process, and the receiving side evaluates or modifies the transfer before the next chunk is accepted. In other words, the design is best understood as a compact, low-level channel model adapted to a 6502-era multitasking kernel, not as a conventional message-passing API.

This also explains why the pattern feels so natural in hardware-oriented systems: channel I/O separates the mechanics of data movement from the policy of whether the transfer is valid and acceptable. OS128 keeps that distinction while fitting it into a very small, tight runtime environment.

## Related files

- [../../kernel/hal/dma.asm](../../kernel/hal/dma.asm)
- [../../kernel/lib/memcopy.asm](../../kernel/lib/memcopy.asm)
- [../../kernel/signal.asm](../../kernel/signal.asm)
- [../../kernel/hal/ram/reu.asm](../../kernel/hal/ram/reu.asm)
- [../../include/memorymap.asm](../../include/memorymap.asm)
