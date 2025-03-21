Introduction
============

This document describes a programming language designed for the
DSP inside the AICA sound chip of the SEGA Dreamcast.
It is not really an assembly language, as one instruction won't
map to a DSP "step" directly.

Register names
==============

This document references the following internal DSP registers:

- `MDEC`: 16-bit counter, decremented on every sample.
- `INPUTS`: 24-bit signed input value
- `SHIFTED`: 24-bit signed shifted output value
- `YREG`: 24-bit signed latch register
- `ADRS`: 12-bit address register

Input selection
===============
```
INPUT mems:<arg>
```
Read INPUTS from the corresponding MEMS register (0x00-0x1f).

```
INPUT mixer:<channel>
```
Read INPUTS from the corresponding input mixer channel (0x0-0xf).

```
INPUT cdda:<arg>
```
Read INPUTS from the corresponding input cdda channel (0-1).

Writing registers
=================
```
OUTPUT yreg
```
Store the INPUTS value into the YREG register.

```
OUTPUT adrs
```
Store bits 23-12 of the INPUTS register, right-shifted 12 bits, into the ADRS register.

```
OUTPUT adrs/s
```
Store bits 23-16 of the INPUTS register, right-shifted 16 bits with sign extension, into the ADRS register.

```
OUTPUT mixer:<arg>
```
Write the value of the SHIFTED register, demoted to 16-bits (shifted), to the output mixer channel specified.

Read-write Memory
=================

Data format specification
-------------------------

The DSP uses a 24-bit integer format internally. However, it loads and stores 16-bit words from/to memory. Those 16-bit words can be in two formats:

- 16-bit integers: When loading, the 16-bit value is promoted to 24-bits by simply shifting it left by 8 bits; When storing, the 24-bit value is shifted right by 8 bits.
- 16-bit floating-point values: The data will be converted from/to a
  floating-point format with the following characteristics:
```
  bit 15:      sign
  bits 14-11:  exponent
  bits 10-0:   mantissa
```

Address format specification
----------------------------

These define the format of the `<addr>` token used in the load/store instructions below.
```
madrs:<arg>
```
Read from MADRS buffer at offset <arg> (0x00-0x3f).

Optionally, the value read can be incremented by one by adding a "plus" sign:
```
madrs:<arg>+
```

Optionally, the value read can be incremented by the value in the `ADRS` register by adding the /s suffix:
```
madrs:<arg>/s
```

Note that the plus sign and the /s suffix can be combined:
```
madrs:<arg>+/s
```

Optionally, the address format can be put inside brackets, to specify that the offset is relative to the current sample (in which case the `MDEC` register is added to the value):
```
[madrs:<arg>+/s]
```

Read/write
----------
```
LD <addr>, mems:arg
```
Read a 16-bit value, promoted to 24-bits (shifted), into the given `MEMS` register.

```
LDF <addr>, mems:arg
```
Read a 16-bit floating-point value, converted to a 24-bit integer, into the given `MEMS` register.

```
ST [temp:arg]
```
Store the `SHIFTED` value into the given temporary register (with the offset being relative to the current sample)

```
ST <addr>
```
Store the `SHIFTED` value, demoted to 16-bits (shifted), to the given address.

```
STF <addr>
```
Store the `SHIFTED` value, converted to a 16-bit floating-point value, to the given address.

Multiply-accumulate
===================

Input specifiers
----------------

- `X` is a 24-bit value, which can be one of:
  - `input`: (the value of `INPUTS` signal)
  - `[temp:arg]`: the value of the given temporary register (with the offset being relative to the current sample)

- `Y` is a 13-bit value, which can be one of:
  - `shifted:lo`: Lowest 12 bits of the `SHIFTED` value, with MSB cleared;
  - `shifted:hi`: Bits 23-11 of the `SHIFTED` value, shifted right by 11 bits.
  - `yreg:lo`: Bits 15-4 of the `YREG` register, shifted right by 4 bits and with MSBs cleared;
  - `yreg:hi`: Bits 23-11 of the `YREG` register, shifted right by 11 bits and with MSBs cleared.
  - `$value`: a 13-bit signed immediate value.

- `B` is a 26-bit value, which can be one of:
  - `acc`: The previous value of the accumulator
  - `-acc`: The opposite of the previous value of the accumulator
  - `[temp:arg]`: the value of the given temporary register (with the offset being relative to the current sample)
  - `-[temp:arg]`: the opposite of the given temporary register (with the offset being relative to the current sample)

Multiply-accumulate operation
-----------------------------
```
MAC X, Y
```
Multiply X by Y, divide the result by 4096 (aka. shift right by 12 bits) and store the result into the accumulator.

```
MAC X, Y, B
```
Multiply X by Y, divide the result by 4096 (aka. shift right by 12 bits) then add B, and store the result into the accumulator.

Shift mode value
----------------
```
SMODE <arg>
```
Change the mode in which `SHIFTED` is computed from the accumulator.

With <arg> being one of:

- `sat`: `SHIFTED` = accumulator, saturated to 24-bit (default)
- `trim`: `SHIFTED` = accumulator, with the two MSBs cut off
- `sat2`: `SHIFTED` = accumulator x 2, saturated to 24-bit
- `trim2`: `SHIFTED` = accumulator x 2, with the two MSBs cut off
