# OS128 hardware requirements and support

This document describes the hardware requirements and supported optional hardware for OS128

## 1. Minimum hardware requirements

OS128 requires the following hardware to work. 

- C128 (any model)
- 80 column display
- 64kb VDC memory (so you'll have to upgrade video ram on a 'flat' 128 and 'plastic' 128D)
- Ram Expansion Module with at least 512kb ram.
- 1571 floppy disk drive

### 1.1

OS128 requires the use of VDC, both because of the functionality VDC has, and because VIC2 would get quite
in the way for using 'fast mode', while fast mode is highly beneficial for OS128. 

### 1.2 VDC memory

OS128 uses VDC video ram for supporting multiple virtual consoles. This does not fit in 16kb video ram.
OS128 supports both the 8563 and 8568 VDC chips

### 1.3 Ram Expansion Module

OS128 uses both the ram and the dma controller provided by the REU, for this reason GEORAM and alternatives are not supported.

Most REU clones which provide DMA should work, tested are

- original Commodore 17xx REUs with at least 512kb
- CMD 1750 and 1750XL
- VICE's REU emulation
- Ultimate II+
- MiSTer REU

RAD is untested so far, and has a slight incompatibility with the 17xx REU but should work.

### 1.4 Storage devices

OS128 can boot from most floppy disk drives, and from mass storage devices with a CMD HDD
compatible DOS (either from a 'native partition' or .dnp image. 

Using a device with burst mode support is highly recommended.

## 2. Optional hardware

OS128 also supports the following hardware:

- Internal DSC12887 or compatible mapped at $d700
- 6551 ACIA mapped at $DE00 (using irq or nmi)
- 1581 floppy disk drive (can be used as boot device)
- All CMD FDD devices (can be used as boot device)
- CMD HDD devices (OS128 supports booting from a partition)
- Ultimate II+ cartridge with command interface enabled
- 1540 and 1541 drives can be used but their use is not recommended because they do not support burst mode
- SD2IEC is supported and OS128 can boot from a .dnp image, but due to lack of burst mode support this is not recommended
- pi1541 is supported but not recommended (due to problems with jiffydos and lack of burst mode support, also when using 1581 emulation)

## 3. Summary

OS128 requires a hardware configuration you could buy in 1987 from Commodore without having to do any modifications, but... it was the 'biggest' C128 configuration you could buy.
