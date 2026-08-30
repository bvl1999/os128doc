# Interrupt handler flow (kernel/interrupt.asm)

This document explains how the hardware interrupt entry and handler in
`kernel/interrupt.asm` work. It describes the sequence the kernel uses
to save CPU state, inspect IRQ sources, protect the scheduler state,
invoke kernel helpers (including `hal_interrupt`), and call the
scheduler. The scheduler details are summarized and a reference to the
full scheduler design is provided at the end.

**Files referenced:**

- `kernel/interrupt.asm` — interrupt entry and IRQ/NMI handling (source of
  this document)
- `docs/architecture/scheduler.md` — detailed scheduler design (see
  "Scheduler handoff" below)

## High-level flow

1. Hardware vector entry: CPU jumps to the IRQ or NMI vector which
   routes to `irq_handler` / `nmi_handler` labels.
2. Early state save: the handler pushes registers and system state onto
   the current CPU stack (PHA/TXA/PHAs) and stores key bytes such as
   `$ff00`/`$ff03` and the current MMU pointers into `memmap_runtimedata`.
3. Fast-path checks: the handler tests `memmap_runtimedata` and a few
   VIC/CIA registers to determine whether to take a quick exit or to
   run the full IRQ handling path.
4. Device-specific handling: ACIA (serial) IRQ paths are checked and
   dispatched first when enabled; the code reads device status and may
   call `acia_handler`/`acia_interrupt`.
5. Protect scheduler state: the interrupt attempts to acquire an IRQ
   semaphore/lock (`lock_acquire_or_fail`). If it fails, the handler
   jumps to a lock-failure path that cleans up and returns.
6. Stack and break checks: the handler verifies the current stack is
   sane and checks for BRK/break events that might escalate to
   exceptions.
7. Save current thread state and switch to scheduler stack: the IRQ
   saves `thread_sp[current_thread]`, switches the CPU stack to the
   scheduler's stack, clears `current_thread`, and sets up context
   mapping macros (`select_thread_context`, `select_io_context`) for
   scheduler execution.
8. Service device IRQs and bookkeeping: the handler increments
   `system_irq_act` for pending devices, calls `hal_interrupt`, and
   updates per-thread blocking counters via `scheduler_decrease_bdata`.
9. Determine scheduling action: based on which devices signalled and
   time-slice accounting, the handler may take a fast return path or
   jump to `irq_scheduler` to run `scheduler_run_next`.
10. Scheduler handoff: `irq_scheduler` calls `scheduler_run_next`, which
    decides the `next_thread`, performs process activation and thread
    context setup, and returns to the IRQ return path at
    `irq_scheduler_run_return_address`.
11. Restore context and return: the IRQ exit path (`irq_skip_scheduler`)
    restores MMU pointers, thread stack pointer, thread mapping, and
    finally performs `rti` to resume execution in the selected thread.

## Detailed steps and key code points

- Vector prolog (labels `nmi_handler` / `irq_handler`): pushes
  registers and reads MMU/zero-page bytes into `memmap_runtimedata` so
  the handler knows whether the CPU is already in a user process
  context. The prolog also saves `$ff00`/`$ff03` which hold vector
  context on this platform.

- Quick-exit test: if `memmap_runtimedata` is non-zero the code jumps
  to the main IRQ path; otherwise it performs a small set of fixes and
  does `irq_quick_exit` to restore the original context and `rti`.

- ACIA / serial handling: when ACIA IRQ support is enabled the code
  checks `acia_handler_t` and `acia_irq_status`. If a serial device
  requires service, the handler calls `acia_handler` or
  `acia_interrupt` and then jumps to the IO exit path to avoid running
  the scheduler.

- Device hooks are IRQ-context only: `hal_interrupt`, `acia_handler`,
  and similar device hooks are invoked while the kernel is already in
  an interrupt context. They must not perform blocking operations, do
  scheduler work, or call any routine that can block waiting on a lock
  or a semaphore. The design assumes these hooks perform only bounded,
  nonblocking bookkeeping and device acknowledgement work. This is an
  important contract: a device interrupt may signal an event, but it
  must not turn the IRQ path into a scheduler or process-management
  operation.

- Lock acquisition: the IRQ code attempts `lock_acquire_or_fail` on a
  kernel `irq_lock`. On success it continues; on failure it executes a
  cleanup path (`.lock_failed`) that releases any partially saved
  state and returns without touching scheduler structures.

- Stack integrity checks: the handler examines the current stack to
  detect BRK conditions and to ensure the stack pointer has not
  exceeded configured high-water limits. If signatures or limits are
  invalid the code prints diagnostics and calls `thread_finish` to
  terminate bad threads.

- Thread state save & scheduler stack switch: the handler saves the
  current thread's stack pointer in `thread_sp[current_thread]`, then
  sets `next_thread = 0` and switches to the scheduler stack using `txs`
  to allow scheduler functions to run without corrupting the user
  thread's stack. It also zeroes `current_thread` to indicate the CPU
  is on the scheduler stack.

- Device bookkeeping and HAL call: the handler reads CIA/VIC IRQ
  registers and increments `system_irq_act` for each active device.
  It then calls `hal_interrupt` (platform-specific interrupt hooks)
  and `scheduler_decrease_bdata` to age blocked thread timers.

- ACIA / CIA secondary checks: after the HAL call the code inspects
  other serial/CIA devices (if enabled). Handling those devices may
  again increment `system_irq_act` or call device handlers.

- Timeslice and scheduling decision: the handler examines
  `system_irq_act` and other indicators. If no scheduling is required
  it skips the scheduler (`irq_skip_scheduler`) and proceeds to the
  restore path. If a timeslice or a preemption is needed it jumps to
  the `irq_scheduler` label which calls `scheduler_run_next`.

## Scheduler handoff (short explanation)

When the IRQ path determines that a scheduling decision is required it
invokes `scheduler_run_next` (via the `irq_scheduler` label). The
scheduler performs these high-level actions (see
`docs/architecture/scheduler.md` for full details):

- Select the next runnable thread by scanning runqueues in priority
  order (`runqueue_read`).
- If the chosen thread belongs to a different process, activate its
  memory map and bank via `process_context_load` and MMU register
  writes.
- Restore the chosen thread's stack pointer and mappings (`select_thread_context`, `select_io_context`).
- Update `current_thread` and related runtime variables so that the
  IRQ return path restores the correct context.

The scheduler returns control to the IRQ path (to
`irq_scheduler_run_return_address`) with `next_thread` and process
context set, and the IRQ exit path completes the MMU/thread stack
restores and performs `rti` to resume execution in the newly selected
thread.

For full scheduler details, see `docs/architecture/scheduler.md`.

## Error paths and diagnostics

- Stack signature failures print short diagnostic strings and jump to
  `thread_finish` to remove corrupted threads from scheduling.
- Invalid or unexpected IRQ sources go to `unknown_irq`/`irq_other`
  branches which log a small status code and then perform a fast
  scheduler skip/return.
- Lock acquisition failure (`.lock_failed`) cleans up the saved state
  and returns quickly to avoid deadlocking the system.

## Return path and context restore

- `irq_skip_scheduler` and the common exit path restore saved MMU
  pointers from the stack, restore zero-page/MMU registers, and
  re-load the thread stack pointer into `tsc`/`txs` so the CPU stack
  points at `thread_sp[current_thread]` for the resumed thread.
- A final `rti` returns from the IRQ and resumes execution in the
  selected thread (or in the interrupted context if no scheduling
  happened).

## References

- Kernel interrupt source: `kernel/interrupt.asm`
- Scheduler design: `docs/architecture/scheduler.md`

## Code cross-links

Key labels and locations in the interrupt source (open the file and search the labels):

- `nmi_handler`, `irq_handler` — IRQ/NMI entry and prolog. See [kernel/interrupt.asm](kernel/interrupt.asm).
- `irq_quick_exit`, `.irq_io_exit` — fast IO/exit paths and MMU restore. See [kernel/interrupt.asm](kernel/interrupt.asm).
- `irq_scheduler`, `irq_scheduler_run_return_address`, `irq_skip_scheduler` — scheduler handoff and IRQ return paths. See [kernel/interrupt.asm](kernel/interrupt.asm).
- `.lock_failed`, `.stack_error_continue`, `.stacksig_error` — lock failure and stack-diagnostic paths. See [kernel/interrupt.asm](kernel/interrupt.asm).
- `acia_handler`, `acia_interrupt`, `hal_interrupt` — device/host hooks invoked by the IRQ path. See [kernel/interrupt.asm](kernel/interrupt.asm) and device sources.

Open [kernel/interrupt.asm](kernel/interrupt.asm) and search for these labels to inspect the exact implementation locations referenced by this document.
