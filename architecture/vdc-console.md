# VDC display and console architecture

This document explains how OS128 uses the VDC display hardware and how that design feeds into the virtual console system.

## Overview

The OS does not treat the display as a simple peripheral. It treats the VDC as an active part of the kernel’s runtime environment. The VDC is used for:

- console output
- multiple virtual terminal windows
- screen memory management
- character/attribute base configuration
- display timing and memory detection

The main implementation is in:

- [../../kernel/hal/display/vdc.asm](../../kernel/hal/display/vdc.asm)
- [../../kernel/dev/console.asm](../../kernel/dev/console.asm)
- [../../kernel/vtty.asm](../../kernel/vtty.asm)

## VDC register model

The VDC driver directly manipulates VDC control and data registers. It includes functions for:

- waiting for vertical blanking
- writing registers
- reading registers
- setting character base
- setting display base
- setting attribute base
- detecting memory size
- enabling or disabling 64K mode

This part of the kernel is carefully tuned to the hardware quirks of the VDC chip and therefore looks more like a driver layer than a generic drawing API.

## Virtual console model

The OS tracks multiple consoles and virtual terminal sessions. `virtual_console` and the console tables in the runtime data are used to switch between active terminal views and backgrounds.

This is a key architectural choice: the OS is designing around a multi-console environment even on a small retro machine.

## Why this matters

The VDC console subsystem is where the OS crosses from raw hardware to user-visible environment. It is one of the clearest examples of the project combining hardware-specific design with higher-level OS concepts.

## Related files

- [../../kernel/hal/display/vdc.asm](../../kernel/hal/display/vdc.asm)
- [../../kernel/hal/display.asm](../../kernel/hal/display.asm)
- [../../kernel/dev/console.asm](../../kernel/dev/console.asm)
- [../../kernel/vtty.asm](../../kernel/vtty.asm)
- [../../include/ioregisters.asm](../../include/ioregisters.asm)
