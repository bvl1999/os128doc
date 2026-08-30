# Memory map and bank management

This document explains the fixed memory layout of OS128 and why the banked memory model is central to the operating system.

## Core idea

OS128 is designed for a machine with limited address space and banked RAM. Instead of a portable flat memory model, the kernel defines a fixed runtime layout in [include/memorymap.asm](../../include/memorymap.asm).

The key point is that the OS treats memory layout as part of the operating system architecture, not as a runtime convenience.

## Low RAM and runtime data

The low-memory region is reserved for:

- thread state
- process state
- runtime flags
- per-thread stack data
- scheduler bookkeeping
- device and I/O metadata

This region includes arrays such as:

- `thread_table`
- `process_table`
- `lock_table`
- `runqueue_*` positions
- `device_flags`, `device_busid`, `device_name_table`

The structure is intentionally dense and fixed. Many runtime subsystems directly index into these arrays by thread or process ID.

## MMU bank mapping

The kernel uses MMU registers to switch between kernel, process, and banked memory views. Several names in the code show this clearly:

- `memmap_kernel`
- `memmap_process`
- `memmap_runtimedata`
- `memmap_rom_and_bank_0`
- `memmap_rom_and_bank_1`

This design allows the system to keep the kernel and user processes in different memory views while still reusing the same CPU address space.

## Process and thread tables

The memory-map file lays out:

- per-process status
- process memory mapping
- process swap metadata
- per-thread stack pointer and state
- per-thread load counters and priority

This is how the scheduler knows which task owns which memory bank and which stack is active.

## REU-backed memory and swapping

The memory model is not just banked; it also assumes an external RAM expansion unit (REU) is available. The REU is used for:

- swap storage
- process memory extension
- memory transfer and DMA operations

This is one of the strongest signs that OS128 is designed around a very specific retro hardware model instead of a generic software abstraction.

## Why this matters

This memory map is one of the defining parts of the whole OS. It drives:

- process activation
- thread switching
- scheduler reloads
- REU swap logic
- IRQ context restore
- user vs kernel memory separation

Because the mapping is fixed and low-level, the project depends on careful manual coordination between the boot code, scheduler, and memory-management routines.

## Related files

- [../../include/memorymap.asm](../../include/memorymap.asm)
- [../../kernel/scheduler.asm](../../kernel/scheduler.asm)
- [../../kernel/swapper.asm](../../kernel/swapper.asm)
- [../../kernel/interrupt.asm](../../kernel/interrupt.asm)
- [../../kernel/process.asm](../../kernel/process.asm)
