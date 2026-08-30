# Bytecode interpreter (BCI) — design and operation

This document describes the bytecode interpreter implemented in
[kernel/bci.asm](kernel/bci.asm). It explains how the interpreter decodes
and executes instructions, the operand encoding used by the instruction
stream, and how handlers and status flags interact. The opcode and
register enumerations are generated into
[docs/dev/BCI-OPCODES-TABLE.md](docs/dev/BCI-OPCODES-TABLE.md) (see Appendix).

## Overview

The BCI is a compact, stack-oriented virtual machine inspired by 6502
semantics. It exposes a custom instruction set with multiple operand
widths (8/16/24/32-bit) and seven general-purpose registers (`A`..`G`).
The interpreter loop implements a classic fetch–decode–execute cycle and
dispatches to handler routines via an `opcode_handlers` table.

Key implementation files:

- Interpreter: [kernel/bci.asm](kernel/bci.asm)
- Opcode / register reference: [docs/dev/BCI-OPCODES-TABLE.md](docs/dev/BCI-OPCODES-TABLE.md)

## Runtime layout and registers

- General-purpose registers are stored in memory starting at `$80` (per-VM register block). There are seven general-purpose registers (`reg_a`..`reg_g`) with low/high bytes accessible for multi-byte widths.
- `reg_pc` is the interpreter program counter (24-bit) and points at the next bytecode instruction byte in the program memory.
- `reg_stack` and `reg_stack_page` implement the interpreter stack pointer and the current stack page pointer; stack pages are bounded by `cpu_stack_low` / `cpu_stack_high` limits.
- `reg_status` stores interpreter condition flags (carry, zero, negative, overflow) used by conditional jump opcodes.
- Several VM-temporary locations (`cpu_current_opcode`, `cpu_current_operand`, `cpu_current_mode`, `cpu_work_value`, etc.) hold the current decode/execution state while a handler runs.

Refer to `kernel/bci.asm` for the exact memory layout and symbol addresses.

## Fetch–Decode–Execute cycle

The interpreter loop (label `interpreter_loop`) repeatedly performs a
fetch–decode–execute cycle:

1. Fetch opcode: `.fetch_opcode` reads the next byte from `(reg_pc)` into
 `cpu_current_opcode` and increments the PC (`.inc_pc`).
2. Decode & dispatch: `.execute_instruction` masks the opcode
 (`and #$3f`) to normalize the opcode range, indexes into
 `opcode_handlers` (a table of handler addresses) and performs an
 indirect `jmp` to the selected handler.
3. Execute handler: the handler implements the operation and uses
 helper routines to fetch operands and manipulate registers/stack/
 status.

Handlers are small subroutines (labels named `op_*` or implementation
functions). The indirect-jump dispatch keeps the hot path compact and
lets handlers be regular ASM routines.

## Operand encoding and modes

Operands are encoded as bytes immediately following an opcode. The
interpreter uses `.fetch_operand` to read the operand byte(s) and to
populate two key fields:

- `cpu_current_operand` — raw operand descriptor byte (from the stream).
- `cpu_current_mode` — derived mode/size flags extracted from the
 operand descriptor.

Broadly, the operand descriptor encodes:

- A register index in the low 3 bits (0..7). A register index of 0
  denotes an immediate value: the interpreter then reads one or more
  following bytes as the immediate value.
- A FLAG_LAST bit (high bit) which, when set, marks this operand as
  the final operand for the current instruction. The interpreter
  continues decoding operand descriptors until it encounters an
  operand byte with FLAG_LAST set; handlers therefore expect a list of
  one-or-more operands terminated by the FLAG_LAST-marked operand.
- Size / addressing / sign flags in higher bits (the handler masks and
 shifts these bits to decide between 8/16/24/32-bit values and to
 select addressing modes). The interpreter sets `cpu_current_mode`
 accordingly so handlers can perform size-aware operations.

The generated register and operand constants in
[docs/dev/BCI-OPCODES-TABLE.md](docs/dev/BCI-OPCODES-TABLE.md) list exact
encodings (for example `REG_IMM8`, `REG_A8`, `REG_A16`, ...). Handlers
use `.get_register_index` to convert the low-bit register index into the
offset within the VM register block.

Immediate operands: when the register index is zero, `.fetch_operand`
reads subsequent bytes into `reg_imm` (and `reg_imm+1` /
`reg_imm+2` / `reg_imm+3` for multi-byte immediates) according to the
mode flags. Remember that an immediate operand descriptor may also
carry the `FLAG_LAST` bit to signal the end of the operand list.

Register operands: when a non-zero register index is encoded, the
handler computes the memory offset for that register (using the
`reg_*` block) and reads/writes the low/high bytes as appropriate for
the operand width.

## Stack model

- The interpreter maintains a VM stack via `reg_stack` and `reg_stack_page`. Push and pull operations (`op_push`, `op_pull`) write register low/high bytes to the stack page and decrement/increment the `reg_stack` pointer, with page overflow/underflow checks handled by `.stack_page_increase` / `.stack_page_decrease`.
- Stack overflow or underflow triggers error reporting (`.stack_error`) and halts the VM.

## Status flags and conditionals

- A VM status register (`reg_status`) contains typical condition flags: carry, zero, negative, overflow. Several helper routines set/clear flags (e.g., `.set_carry`, `.clear_zero`, etc.).
- Arithmetic and logical handlers translate between the VM internal status and the 6502 emulation helpers using `.vcpu_to_cpu_status` and `.cpu_to_vcpu_status` where necessary.
- Branch opcodes (e.g., `JEQ`, `JNE`, `JMI`, `JPL`, `JVS`, `JVC`, `JCS`, `JCC`) check `reg_status` flags and perform a `JMP` via `op_jump` when the condition matches.

## Opcode categories and examples

- Stack operations: `PSH` (push), `PUL` (pull) — push/pull register
  or immediate values.
- Arithmetic: `ADD`, `SUB`, `MUL`, `DIV` — `ADD` and `SUB` are
  implemented (see `.add_value`, `.sub_value`) and support variable
  widths; `MUL` and `DIV` are not implemented yet.
- Logical & bit ops: `AND`, `OR`, `XOR`, `ROL`, `ROR`, `ASL`, `LSR`
  — operate per-width and update status flags.
- Control flow: `JMP`, conditional jumps (`JEQ`, `JNE`, etc.),
  `JSR`/`RTS` (`op_call`/`op_ret`) — `op_call` pushes a return
  address on the VM stack and adjusts mode/operand state; `op_ret`
  pulls the return address.
- I/O and system: `PRN` (print), `OUT` (stream write), `SYS` (call
  external host routine) — `SYS` marshals arguments into
  `ext_call_*` locations and calls out to an external address from
  the register file.
- Misc: `RND` (random), `COPY` (memcopy between VM memory ranges),
  `HLT` and error opcodes.

### Example: ADD (high-level)

- Fetch operand descriptor(s) (source and destination).
- Decode operand width via `cpu_current_mode`.
- Loop over bytes/words/longs: read operand, add into accumulator
 (`cpu_work_value`), propagate carry into next byte, write result back
 to destination register bytes.
- Update VM status flags via `.cpu_to_vcpu_status` and return.

## External calls and `SYS`

`op_sys` provides a controlled way for bytecode to call host/kernel functions. The handler:

- Decodes a function selector operand and zeroes arg slots.
- Extracts one-to-three optional operands (based on operand descriptors) and stores them into `ext_call_arg0..arg2`.
- Transfers status from VM to CPU (`.vcpu_to_cpu_status`) and jumps to the address stored in `ext_call_address` (indirect call into host/kernel).
- On return, the handler reads results back into VM registers and restores VM status (`.cpu_to_vcpu_status`).

This design cleanly separates VM and host contexts while preserving simple argument/result passing.

## Error handling and invalid encodings

- The interpreter validates opcode ranges and operand encodings; `.invalid_opcode` and `.invalid_operand` report the `reg_pc` location and halt the VM via `op_halt`.
- Stack errors (overflow/underflow) are detected and reported by `.stack_error` before halting.

## Implementation notes

- The interpreter uses compact tables (`opcode_handlers`) containing word-sized addresses of handler routines; dispatch is implemented via an indirect `jmp (current_opcode_handler)` after table lookup.
- Mode decoding uses bit masks and shifts inside `.fetch_operand` so handlers can operate generically across widths; helper routines (e.g., `.read_gp_register_low`, `.read_gp_register_high`, `.write_gp_register_low`) provide consistent byte-level access to register storage.
- Where performance matters the inner loops use explicit 6502 `php`/`plp` and arithmetic instructions to leverage 6502 carry behavior while preserving the VM status register.

## Appendix — opcode & register references

[docs/dev/BCI-OPCODES-TABLE.md](docs/dev/BCI-OPCODES-TABLE.md). It lists all opcode names with hex/decimal codes, and the register encodings used by operand descriptors (e.g., `REG_IMM8`, `REG_A8`, `REG_A16`, ...).

## Short reading guide to `kernel/bci.asm`

- `bci_run` / `vcpu_reset` — VM boot and initialization.
- `interpreter_loop` — main fetch/execute loop.
- `.fetch_opcode`, `.fetch_operand`, `.get_register_index` — decode helpers.
- `opcode_handlers` table — mapping opcode → handler entry.
- `op_*` labels — public opcode handlers (each implements the high-level behaviour).

If you want, I can also generate a small annotated diagram of the fetch/decode/execute flow or add a concrete multi-instruction example showing bytes, operand descriptors, and the exact register changes.

## Annotated example — byte-level walkthrough

The following minimal example shows a short bytecode sequence, how
operands are encoded, and the expected effect on `reg_pc` and VM
registers. This is illustrative; actual byte encodings are generated
from `include/bcibytecode.asm` and listed in
[docs/dev/BCI-OPCODES-TABLE.md](docs/dev/BCI-OPCODES-TABLE.md).

Assume the following (pseudobytes where `0xNN` are byte values):

- Opcode `ADD` = `0x03`
- Register encodings: `REG_IMM8` = `0x00`, `REG_A8` = `0x01`
  (low 3 bits contain register index); `FLAG_LAST` = `0x80`.

Sequence (bytes) — `ADD` using `A` as accumulator and a single immediate source:

```asm
0x03 0x01 0x80 0x2A   ; ADD REG_A8, REG_IMM8|FLAG_LAST(0x2A)
0x80                  ; HLT
```

Step-by-step interpretation:

- `0x03` (ADD): `.fetch_operand` reads the first operand byte `0x01` (register A low byte) — this is the accumulator operand. Since its FLAG_LAST bit is clear, the interpreter continues to read the next operand descriptor.
- Next operand descriptor `0x80` indicates an immediate (low 3 bits = 0) with `FLAG_LAST` set. `.fetch_operand` reads the following immediate byte `0x2A` into `reg_imm`.
- The handler for `ADD` treats the first operand as the accumulator (read `A`), then iterates over the remaining operands (here the single immediate 0x2A) and adds each into the accumulator (`cpu_work_value`) with proper carry propagation across the selected width. After processing the `FLAG_LAST` operand the handler writes the result back to the destination register (`A`) and updates VM status via `.cpu_to_vcpu_status`.
- `reg_pc` advances by 4 bytes for the `ADD` instruction (opcode + accumulator operand + operand descriptor with FLAG_LAST + immediate byte). The subsequent `0x80` (HLT) halts the interpreter.

After execution `reg_a` low byte will have been incremented by `0x2A` (with carry propagated via status flags if needed). This example shows the preferred operand form for `ADD`: the first operand is the accumulator, and the operand list ends when an operand descriptor with `FLAG_LAST` set is encountered.

Notes on exact encoding and widths:

- The operand descriptor's high bits select immediate width (8/16/24/32) or register width; handlers check `cpu_current_mode` and may fetch additional immediate bytes into `reg_imm+1..+3`.
- Multi-byte registers (16/24/32-bit) are stored low byte first in the per-VM register block and handlers perform per-byte loops to implement arithmetic with carry propagation.

If you'd like, I can add a hex dump example showing exact addresses and `reg_pc` values before/after each instruction for easier testing with the assembler output.
