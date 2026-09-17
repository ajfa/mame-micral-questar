# R2E Micral N and Questar M floppy subsystem for MAME

MAME carries `micral` and `questarm` as a skeleton: they show their banner and
stop, because the driver has no disk subsystem at all. This patch gives them
one, and both machines boot.

The Micral N 80-22G and the Questar M are R2E machines of 1982, 8085 based, with
an 8 inch floppy subsystem that is nothing like a WD1771: there is no formatter
chip. Three memory mapped registers do everything and the processor shifts the
bits itself, one per access.

## Status

`questarm` boots R2E's CP/M 2.23 to the `A>` prompt and `DIR` lists its 23
files. `micral` boots the same way with its own ROM, and QMOS 1.A of October
1982 reaches its `->` prompt.

## What the hardware turned out to be

Read out of the boot ROM disassembly and of the CP/M BIOS on the diskette
itself:

    FFFD  write  drive and sector select, bits 0 to 3 sector, bit 4 drive 0, bit 5 drive 1
          read   status: sector found, track 0, write protect, double sided
    FFFE  write  command: step direction and pulse, motor, data mode, sector seek,
                 write start, side
    FFFF         serial data, one bit per access in bit 0, inverted on read

After a seek in data mode the stream is a pad bit, then `00 TT SS`, 256 data
bytes and two checksum bytes, with `TT` the cylinder and `SS` the logical sector
0 to 31, where 16 to 31 is the second side.

Two details cost time and are worth writing down. The side bit is latched at
seek time, not at data time: the ROM writes a literal `1C` in data mode, which
drops bit 7. And the sync pattern is written by software, 256 zero bits then
`101` then eight zeros, so the detector has to hunt for the last six bits being
`000101`.

**The second checksum.** The dump's own documentation left it open. It is a
Fletcher style pair: `chk1` is a one's complement running sum, `add a,d` then
`adc a,0`, and `chk2` folds the running sums the same way, both seeded with the
track number and then updated with the sector number before the 256 data bytes.
A Python replica of the ROM's block checksum passes on every boot sector of both
diskettes.

## Layout

    patches/micral-floppy.patch   the driver change, against MAME 0.289

The patch adds a floppy image device inside the driver, taking raw 640 KB images
on `-flop1` and `-flop2` and writing back in place, the read and write bit
machinery with its sync hunt, the step pulses and per drive cylinder, and the
CRTC register 6 hardware scroll the console uses.

## Building

    cd <mame>
    patch -p1 < <this>/patches/micral-floppy.patch
    make SUBTARGET=micral SOURCES=src/mame/skeleton/micral.cpp

## Running

    mame questarm -flop1 cpm.disk

Type `B0` at the prompt to boot CP/M; on the Micral, `B0,8` boots QMOS from
cylinder 4. The images are 655360 bytes, 80 cylinders of 32 sectors of 256
bytes, decoded data only.

No R2E software or ROM is included here.

## Why it lives here and not in MAME

It was offered as [mame#15713](https://github.com/mamedev/mame/pull/15713) and
closed without merging, for a good reason: it ignores MAME's floppy
infrastructure. The disk arrives through a private image device that takes one
flat file of already decoded sectors, instead of a `floppy_image_format_t` under
`src/lib/formats` turning a flux level dump into cell data and back, with the
driver taking a `FLOPPY_CONNECTOR` and running its discrete serial controller off
the real cell stream. Done properly, write back and flux dumps work by
construction rather than only for the one file this device understands. That is a
rewrite, not a fixup, so the branch was withdrawn rather than left sitting in the
queue.

What is here works and boots both machines. Anyone wanting it upstream should
start from the format side.

## Known gaps

`STAT` loads but the CCP never jumps to 0100, so it produces no output; `DDT`
and `DIR` are fine. The keyboard microcontroller is not emulated and the machine
is driven through the serial keyboard instead.

## License

BSD-3-Clause, the same as the MAME source it is built against. See LICENSE.
