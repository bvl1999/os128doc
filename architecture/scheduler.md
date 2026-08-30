# Scheduler — detailed design

This document describes the OS128 scheduler implementation, including how the scheduler selects threads, how priorities are represented, and how process/thread memory context is activated before a thread begins executing.

## 1. Scheduler architecture

The OS128 scheduler is implemented across several kernel modules:

- `kernel/scheduler.asm` — main scheduler initialization and dispatch logic
- `kernel/runqueue.asm` — runqueue management and priority-based selection
- `kernel/thread.asm` — thread lifecycle, status flags, yield, and blocking
- `kernel/process.asm` — process management, process memory mapping, and swap handling
- `kernel/interrupt.asm` — timer IRQ entry and scheduler invocation
- `include/macros.asm` — MMU mapping macros used for thread/process context switching

The scheduler uses per-thread and per-process tables in low RAM, combined with a multi-level runqueue design.

## 2. Thread, process, and runqueue state

### 2.1 Thread state

Thread state is stored in arrays located in the low-memory runtime region, with offsets defined in `include/memorymap.asm`:

- `thread_status` — thread flags
- `thread_prio` — thread priority value
- `thread_bdata` — blocking countdown / wakeup data
- `thread_sp` — saved stack pointer
- `thread_proc` — associated process ID
- `thread_signals` — pending signals
- `thread_parent` — creator thread ID
- `thread_countl`, `thread_counth` — per-thread load accounting

`thread_status` bits include:

- `.status_active = $80` — thread is active
- `.status_blocked = $40` — thread is blocked
- `.status_suspended = $02` — thread is suspended

### 2.2 Process state

Process state is stored in the process table and includes:

- `process_status` — process flags including active/swapped status
- `process_ithread` — primary thread for the process
- `process_memmap` — memory map configuration for the process
- `process_swapmap` — swap metadata
- `process_prioema` — exponential moving average of process priority load
- `process_swapcnt` — swap count history

Process memory mappings are encoded as MMU configuration bytes and are used to set the CPU view of the process’s memory.

### 2.3 Runqueue design

The scheduler uses four separate runqueues defined in `kernel/runqueue.asm`:

- `runqueue_crit` — critical queue
- `runqueue_high` — high priority queue
- `runqueue_low` — low priority queue
- `runqueue_idle` — idle queue

Each queue is a circular buffer with separate read and write positions. The scheduler selects the next runnable thread by scanning these queues in priority order.

## 3. Priority assignments

Priority values are assigned through `thread_prio` and mapped to queues in `runqueue_write`:

- `thread_prio == 1` → critical queue
- `thread_prio == 2` → high queue
- `thread_prio == 3` → low queue (default)
- `thread_prio == 4` → idle queue

The default thread priority is defined in `include/config.asm`:

- `default_priority = 3`

The `thread_create` routine in `kernel/thread.asm` initializes new threads with this default priority.

### 3.1 Critical queue override

`runqueue_write` also sends any thread with pending signals (`thread_signals != 0`) into the critical queue, regardless of its normal priority. This ensures signal-aware threads are handled before normal queue order.

## 4. Selecting the next thread

### 4.1 `runqueue_read`

`kernel/runqueue.asm` implements the queue selection algorithm in `runqueue_read`.

The selection order is:

1. `runqueue_crit`
2. `runqueue_high`
3. `runqueue_low`
4. `runqueue_idle`

If a queue entry is found, it is returned as `A` and the queue read pointer is advanced.

### 4.2 Runnable/thread checks

After reading a thread ID from a queue, `runqueue_read` confirms the thread is still runnable:

- checks `thread_is_runnable`
- if the thread is no longer runnable, it is skipped
- if the thread is active and runnable, it is returned

If a thread is not runnable, `runqueue_read` may requeue it or continue scanning.

### 4.3 Starvation promotion

`runqueue_read` maintains starvation counters and promotes threads to higher queues on long waits:

- idle → low
- low → high
- high → critical

This prevents a lower-priority task from being starved indefinitely.

## 5. Dispatch and context switching

### 5.1 `scheduler_run_next`

This is the dispatcher in `kernel/scheduler.asm`.

Key actions:

- checks `next_thread` and verifies `thread_status`
- if the current thread remains runnable, requeues it via `runqueue_write`
- if a thread is dead, the scheduler removes it from scheduling
- reads the next thread from the runqueues using `runqueue_read`
- if no runnable thread is found, selects idle thread `0`

Before allowing the new thread to execute, it performs process context activation and thread context setup.

### 5.2 Process memory activation

For the selected `next_thread`:

- reads `thread_proc[next_thread]`
- copies that process ID into `next_process`
- calls `process_context_load` in `kernel/process.asm`
- if the process is swapped out, `process_context_load` swaps it in
- `current_run_bank` is updated to the process’s assigned RAM bank

After process context load, the scheduler writes the process memory map into MMU control registers:

- `lda process_memmap,x`
- `ora current_run_bank`
- `sta mmu_pcr_d`
- `sta mmu_pcr_c`

This sets the current CPU memory banking for the process.

### 5.3 Thread context activation

Thread context setup uses macros in `include/macros.asm`:

- `select_thread_context` maps the thread’s stack bank and zero page
- `select_io_context` maps thread-local I/O zero-page offsets

The scheduler also restores the thread’s saved stack pointer:

- `lda thread_sp,x`
- `tax`
- `txs`

Finally, `current_thread` and `current_process` are updated.

### 5.4 `irq_skip_scheduler`

In `kernel/interrupt.asm`, the IRQ return path restores the new thread’s runtime state after `scheduler_run_next` completes. It sets:

- `current_thread = next_thread`
- CPU stack pointer from `thread_sp[current_thread]`
- thread context mapping via `select_thread_context`
- IO mapping via `select_io_context`

Then the interrupt return path exits with `rti`, allowing the selected thread to resume on its restored stack.

## 6. Yielding and preemption

### 6.1 Voluntary yield

`thread_yield_cpu` in `kernel/thread.asm` causes the current thread to give up the CPU voluntarily.

It does this by:

- saving CPU state and MMU register values onto the current stack
- storing the current thread SP into `thread_sp[current_thread]`
- mapping the scheduler stack context
- jumping to `irq_short_slice`

This effectively re-enters the scheduler path without waiting for a hardware IRQ.

### 6.2 Timer-driven preemption

The hardware timer and IRQ path are initialized in `kernel/interrupt.asm`.

On each timer interrupt, the IRQ handler eventually jumps to `irq_scheduler`, which calls `scheduler_run_next`. That causes preemption and dispatch of the next thread.

## 7. Process swapping and memory banks

### 7.1 Process memory mapping

`process_context_load` handles the memory mapping for the selected process.

For user processes beyond `0` and `1`, it may:

- choose a swap victim if the process is not already present in RAM
- swap out the victim process
- swap in the target process
- update `bank0_process` and `current_run_bank`

This ensures the memory bank selected for the process is actually available.

### 7.2 Runtime memory registers

Process context activation uses the MMU registers:

- `mmu_pcr_c` — process read/write memory config
- `mmu_pcr_d` — process data RAM bank config
- `memmap_process` — process-specific mapping pattern stored per process

The scheduler combines `process_memmap[next_process]` with `current_run_bank` to configure the CPU’s view before the thread runs.

## 8. Additional details

### 8.1 Idle thread

Thread `0` is the idle thread. It is always considered the fallback when no other runnable thread exists.

### 8.2 Blocking and sleep

Blocking operations set `thread_status` flags and `thread_bdata` countdown values via routines such as:

- `thread_block_count`
- `thread_block_read`
- `thread_block_write`
- `thread_suspend`

Blocked threads are not returned as runnable until their status clears.

### 8.3 Thread destruction

`thread_finish` and `thread_kill` mark threads dead by clearing status bits and removing them from their process. Dead threads are skipped by the scheduler.

## 9. Execution path example

A simplified creation and dispatch sequence:

1. `thread_create` allocates the next free thread ID.
2. The new thread is marked active and its priority set to `default_priority`.
3. The thread receives a fresh per-thread stack and zero page mapping.
4. The thread is written into the appropriate runqueue via `runqueue_write`.
5. On the next timer IRQ or `thread_yield_cpu`, `scheduler_run_next` requeues the old thread and reads the next thread.
6. `process_context_load` activates the process memory map. If the process is swapped out, it is swapped back in.
7. `select_thread_context` / `select_io_context` restore the thread’s stack and zero page mapping.
8. The IRQ return path restores the thread SP and returns to the selected thread.

## 10. Key code references

- `kernel/scheduler.asm` — `scheduler_init`, `scheduler_run_next`, `.update_load`, `scheduler_decrease_bdata`
- `kernel/runqueue.asm` — `runqueue_init`, `runqueue_read`, `runqueue_write`, starvation promotion
- `kernel/thread.asm` — `thread_create`, `thread_set_priority`, `thread_yield_cpu`, `thread_finish`
- `kernel/process.asm` — `process_context_load`, `process_create`, memory map handling
- `kernel/interrupt.asm` — `irq_scheduler`, `irq_skip_scheduler`, `irq_short_slice`
- `include/macros.asm` — `select_thread_context`, `select_io_context`

## 11. Summary

The OS128 scheduler is a priority-based multi-queue dispatcher with process-aware memory activation. It selects threads by queue priority, handles pending-signal critical work, protects against starvation with promotion, and ensures the process and per-thread memory context are mapped before execution.

This design keeps kernel, user, and idle threads separated while allowing the OS to swap process memory banks dynamically and preserve independent thread stacks and zero pages.
