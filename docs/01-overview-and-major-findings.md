# Overview and Major Findings

[Project index](../README.md) · [Complete edition](How%20the%20FTS%208400%20Worked%20-%20Complete.md)

This document reconstructs the architecture of the FTS 8400 Satellite Timing Receiver from its four EPROM images and processor-board photographs. It describes the design rather than reproducing a disassembly. Names assigned to routines and data structures are descriptive reconstructions; they are not claimed to be the original FTS identifiers.

The analysed ROM set identifies itself on the display as:

```text
NP 120 10-OCT-87
SP 101 26-JUL-87
```

The hardware may date from about 1986, but this particular firmware set is therefore from late 1987 or later. The meanings of `NP` and `SP` have not been established.

### Confidence labels

- **Confirmed** — directly established by coherent executable code, data layout, constants, strings, or a visible board component; normally supported by more than one clue.
- **High confidence** — the evidence strongly constrains the interpretation, but a schematic, signal trace, or original manual would still be needed for literal hardware naming.
- **Hypothesis** — plausible and useful as a target for further work, but not established.

The FTS 8400 was not simply an early GPS-disciplined oscillator. **Confirmed:** it was a GPS time/frequency measurement and generation instrument with two distinct frequency-comparison paths:

```text
                         GPS L1 receiver
                                |
                 +--------------+--------------+
                 |                             |
          navigation/time                carrier residual
                 |                             |
                 |                       INTRNL DF/F
                 |                         AVG DF/F
                 |                             |
                 |                      internal DAC loop
                 |                           $A200
                 |
       GPS or GPS->UTC time
                 |
       delay and phase corrections
                 |
       16.368 MHz timing hardware $A3xx
                 |
       synchronized second/minute pulses
                 |
                 +-----------------------------+
                                               |
 External reference pulse ----------------> interpolating TIC
                                               |
                                      TI, TI FIT, TI RATE
                                               |
                                  external-reference frequency
                                      difference versus GPS
```

The software is unexpectedly elaborate. **Confirmed:** a Hitachi HD63HC000 (68000-compatible) runs a small native kernel containing boot code, interrupts, device drivers, a software floating-point library, and a compact Pascal-family P-code virtual machine. Most application logic—roughly 382 identifiable procedures in about 60 linkage groups—is P-code executed by that VM.

The application decodes GPS navigation data, propagates broadcast orbits, models satellite clocks and the ionosphere, solves a four-satellite position and clock solution, computes DOP, averages stationary fixes, generates GPS/UTC-aligned output phase, and performs time-interval regression against an external reference.

### Major surprises

1. **Confirmed — Pascal is the application architecture, not a curiosity.** Lexical displays, nested procedure levels, compact bytecode, native escapes, and module linkage tables are fundamental to the firmware.
2. **Confirmed — substantial numerical work is done in software double precision.** The ROM contains reusable floating-point, trigonometric, square-root, and 4×4 matrix operations despite having no identified FPU.
3. **Confirmed — the receiver contains essentially a complete GPS navigation solution.** It does not merely receive a precomputed time or position from a separate navigation module.
4. **Confirmed — external frequency comparison is statistical.** `TI RATE` is the least-squares slope of external time interval versus GPS time, not an instantaneous control-loop value.
5. **Confirmed — internal and external `DF/F` concepts are different.** The internal carrier/Doppler residual controls an internal DAC; the external-reference result comes from TIC regression.
6. **High confidence — the timing architecture is organized around harmonics of the 1.023 MHz C/A-code rate.** The recovered values 4.092, 5.115, and 16.368 MHz are respectively 4×, 5×, and 16× 1.023 MHz.
7. **Confirmed — the 16.368 MHz coarse clock is supplemented by fine interpolation.** Output-phase programming has a nominal 4 ns fine increment; the TIC also reconstructs fractional coarse-clock cycles.

## Corrections to earlier interpretations

Later tracing superseded several useful but incorrect early ideas:

- **Corrected:** the external TIC regression does not primarily drive `$A200`. It produces `TI FIT` and external-reference `TI RATE`; the DAC is driven by the separate internal carrier/Doppler residual.
- **Corrected:** the 8400 should be described primarily as a GPS time/frequency comparator and pulse generator, not simply as a GPSDO.
- **Corrected:** `$4A15` is specifically a modulo-1 ms observed-minus-predicted C/A code-phase range.
- **Corrected:** `-$4A42` is a three-state update/averaging state, not the GPS/UTC/OMEGA selector; the latter is at `-$28F8`.
- **Corrected:** the 16.368 MHz factor in the TIC is `0.016368 cycles/ns`; the output generator independently establishes a nominal 4 ns fine phase step.
- **Corrected:** this ROM's primary ten-bit GPS week field contains deliberate rollover expansion. It cannot by itself substantiate a 1999-failure explanation.
- **Corrected:** the MC68901 register select is reversed in the CPU address map; a naive ascending-register map assigns the wrong functions.
