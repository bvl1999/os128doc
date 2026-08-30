# Device layer and registration model

This document explains how OS128 represents devices, discovers them, and routes requests through the system.

## Overview

The device model is a classic kernel-style abstraction: hardware devices are registered with the OS, assigned classes, and given bus IDs so they can be addressed uniformly from higher-level code.

The key implementation is centered in:

- [../../kernel/devd.asm](../../kernel/devd.asm)
- [../../kernel/init.asm](../../kernel/init.asm)
- [../../kernel/hal/hal.asm](../../kernel/hal/hal.asm)

## Device registration

The kernel creates device classes and registers them early in system startup. In the init path, the code calls device registration routines for several categories, including console, disk, RAM, ROM, and other device families.

The OS stores bookkeeping such as:

- device flags
- bus identifiers
- command queues
- device names
- class assignments

This is not ad hoc hardware access; it is a structured registry for runtime device discovery.

## Device classes

The system uses class-based registration, which means a device may be classified and then handled according to its role. Examples in the project include:

- console class
- disk/file device class
- RAM-backed device class
- ROM-backed device class
- bus-specific device classes

This class model is a strong sign that the OS is trying to expose a consistent abstraction above raw hardware.

## Bus model

The OS supports multiple device transport styles, especially in the HAL layer:

- IEC bus
- UCI/Ultimate device bus
- hardware-specific I/O drivers

The bus and protocol code is kept in the HAL area, while the higher-level device registry remains in the OS runtime.

## Why it is unusual

The device layer is more elaborate than many educational 8-bit projects. It organizes hardware as a system of named, registered, classed entities. That makes it resemble a mini operating-system device manager rather than a set of direct subroutines.

## Related files

- [../../kernel/devd.asm](../../kernel/devd.asm)
- [../../kernel/init.asm](../../kernel/init.asm)
- [../../kernel/hal/hal.asm](../../kernel/hal/hal.asm)
- [../../kernel/hal/bus/iec](../../kernel/hal/bus/iec)
- [../../kernel/hal/iodrv](../../kernel/hal/iodrv)
