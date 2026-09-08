# Overview and Major Findings

[Project index](../README.md) · [Complete edition](How%20the%20FTS%208400%20Worked%20-%20Complete.md)

## Four ROMs and a timing receiver

The FTS 8400 looks, at first, like a fairly conventional piece of mid-1980s instrumentation. Its processor board carries a Hitachi HD63HC000, the CMOS version of the Motorola 68000, together with an MC68901 multifunction peripheral, a TMS9914A IEEE-488 controller, RAM, EPROMs, and a collection of custom receiver and timing logic.

The first ROM dump did not look like valid 68000 firmware. The reason was physical rather than cryptographic: it contained only the even byte lane of a 16-bit program. Interleaving the four EPROMs as two pairs produced a 16 KiB low ROM and a 64 KiB application ROM. A valid reset vector appeared immediately:

```text
initial supervisor stack pointer = $00004400
reset program counter            = $00000140
```

That solved the first puzzle. The second was more interesting. Much of the larger image still was not 68000 machine code.

## The unexpected software architecture

The 8400 runs a small native 68000 kernel, but most of the instrument is a Pascal-family program executed by a compact P-code virtual machine. The native layer handles reset, interrupts, memory tests, hardware access, floating-point primitives, and the interpreter itself. The P-code layer contains the executive, user interface, navigation algorithms, timing policy, reporting, and most of the instrument's state machines.

Roughly 382 P-code procedure entries can be identified, arranged behind about 60 groups of linkage stubs. The precise original module count is not recoverable from those stubs alone, but this was plainly a modular application rather than one large assembly-language program.

The VM is recognizably Pascal because it maintains a lexical display for nested procedure scopes. It supports compact constants, branches, integer and eight-byte real operations, Pascal strings, procedure calls, local frames, and escapes into embedded native 68000 code.

For a receiver of this age, that choice is striking but sensible. Compact bytecode conserved ROM space. Pascal made a large mathematical application manageable. Native code remained available where deterministic hardware service or expensive arithmetic justified it.

## What the application actually does

The ROM contains far more than menus and hardware drivers. It holds the GPS navigation database and the mathematics needed to use it:

- broadcast ephemeris and almanac records;
- GPS satellite clock and relativistic corrections;
- Klobuchar ionospheric correction;
- satellite orbit and velocity propagation;
- WGS-84 coordinate conversion;
- a four-satellite iterative position and clock solution;
- PDOP, HDOP, VDOP, and TDOP calculation;
- position averaging for a stationary timing installation;
- GPS week, Modified Julian Date, calendar, UTC, and leap-second handling.

The algorithms use software eight-byte floating point and a small reusable matrix library. This is not a controller asking a separate navigation engine for a finished answer: the 68000 application performs the navigation solution itself.

## The machine is better understood as a comparator

An early interpretation treated the 8400 as a straightforward GPS-disciplined oscillator:

```text
GPS phase error -> regression -> DAC -> oscillator
```

Later tracing showed that this combined two separate paths. The corrected architecture is:

```text
                         GPS receiver
                              |
               +--------------+--------------+
               |                             |
        navigation and time           carrier residual
               |                             |
      GPS- or UTC-aligned phase          INTRNL DF/F
               |                         AVG DF/F
      16.368 MHz timing logic                |
               |                       internal DAC
       second/minute outputs               $A200
               |
               +------------------------------+
                                              |
external reference pulse ------------> interpolating TIC
                                              |
                                     TI, TI FIT, TI RATE
                                              |
                              external frequency versus GPS
```

The internal path removes predicted satellite Doppler from a receiver frequency measurement. Its residual is displayed as `INTRNL DF/F` and drives a 16-bit DAC through a simple incremental control law. The evidence strongly suggests that the DAC tunes an internal receiver oscillator or related frequency-control element; it is not proven to steer the user's external standard.

The external path measures a reference pulse with a high-resolution time-interval counter. Repeated measurements are fitted by ordinary least squares. The fitted slope, displayed as `TI RATE` in picoseconds per second, is the external reference's fractional-frequency difference from GPS in parts in `10^12`.

This makes the 8400 primarily a GPS time/frequency comparator and pulse generator, not merely an early GPSDO.

## Why 16.368 MHz appears everywhere

Several initially mysterious constants collapse into one clock family:

```text
1.023 MHz   GPS C/A-code chip rate
4.092 MHz =  4 × 1.023 MHz
5.115 MHz =  5 × 1.023 MHz
16.368 MHz = 16 × 1.023 MHz
GPS L1     = 1540 × 1.023 MHz = 1575.42 MHz
```

The raw code-phase conversion uses exactly the distance light travels during one 16.368 MHz cycle:

```text
299792458 / 16368000 = 18.3157660068 metres
```

The time-interval counter also measures coarse cycles at 16.368 MHz and supplements them with fractional start/stop interpolation. The output generator independently uses 16,368 cycles per millisecond and a fine scale whose arithmetic corresponds to a nominal 250 MHz, or about 4 ns per fine count.

The commanded 4 ns increment is not a claim about absolute output accuracy. It does show how a 61.1 ns coarse clock supported much finer phase placement.

## The most important lessons from the ROM

The main surprises are architectural rather than incidental:

1. A Pascal virtual machine is the foundation of the application.
2. The receiver performs a complete GPS orbit, clock, position, and time solution in software.
3. Software double precision and reusable matrix routines make that practical on a 68000-class processor.
4. External frequency comparison is based on a regression through many phase measurements, not on the latest sample alone.
5. Internal receiver-frequency control and external-reference measurement are deliberately separate.
6. GPS-derived clock harmonics connect code tracking, time measurement, and pulse generation into one coherent timing architecture.
7. The firmware was designed to validate, average, and reject suspect observations before changing the instrument's time outputs.

## Corrections made as the analysis improved

Several early ideas were useful stepping stones but did not survive deeper tracing:

- The external TIC regression does not primarily drive the DAC. It produces `TI FIT` and `TI RATE`; the DAC uses the separate internal carrier/Doppler residual.
- `$4A15` is specifically an observed-minus-predicted C/A-code phase expressed as range modulo one millisecond, not a generic range correction.
- The state at global offset `-$4A42` controls clock-update averaging; GPS/UTC/OMEGA selection is held separately at `-$28F8`.
- This ROM deliberately expands the main ten-bit GPS week across the first rollover. That field alone cannot explain reports of an FTS 8400 failing in 1999.
- The MC68901 registers appear in reversed select-line order in the CPU address map. A conventional ascending register assignment gives incorrect names.

The remaining unknowns are mostly at the hardware boundary: exact custom-register names, the TIC's interpolation circuitry, the analog destination of the DAC, and the division of acquisition and tracking between software and custom logic. None of those prevents a useful reconstruction of the overall design.
