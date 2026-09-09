# What Remains Unresolved

[Project index](../README.md) · [Complete edition](How%20the%20FTS%208400%20Worked%20-%20Complete.md)

The broad design is now understandable. Most remaining questions lie where software-visible values meet custom hardware, in edge-case runtime semantics, or in the provenance of the Pascal toolchain.

## Timing hardware

The `$A3xx` block is functionally identifiable as the time-interval counter and output-phase generator, but its exact register names, bit definitions, and electrical implementation are not known. A schematic or targeted measurements would help answer:

- What interpolation technique is used for TIC start and stop events?
- How is it calibrated, and what are its single-shot resolution and accuracy?
- Which registers hold coarse output phase, fine phase, status, and latch control?
- What physical path requires the fixed 400 ns correction?
- What circuit quantity is represented by the likely 26-bit, 240 ms accumulator?

The output firmware's nominal 4 ns fine count should not be used as a substitute for measured TIC performance.

## Receiver and analog control

The software proves that `$A200` is a 16-bit DAC driven by the internal carrier-frequency residual. It does not show which analog node receives the voltage. The leading hypothesis is an internal receiver oscillator or frequency-control element, but the following still need physical evidence:

- Does the service page's ±5.000 scale correspond directly to volts?
- What are the DAC polarity, gain, and neutral operating point in the circuit?
- How are acquisition, code tracking, and the Costas/carrier loop divided between software and custom logic?
- What are the roles of the remaining `$A4xx-$A7xx` registers?

## Protocols and historical behavior

The serial hardware is now largely identified: Port 1 is the MC6850-compatible ACIA driven from Timer C and entering through GPIP5; Port 2 is the MC68901 USART driven from Timer D. What remains is the protocol layer. The full command grammar, the intended meaning of `CONTROL CHARS`, the compact six-byte real-number format, the actual MFP clock and baud-rate table, and the exact purpose of Port 1's RTS/transmit-control manipulation are still unknown.

OMEGA appears as a genuine time-source selection, but its complete signal and software path remains to be traced.

This ROM expands the principal ten-bit GPS week across the 1999 rollover. Reports of rollover failure may concern another software revision or one of the narrower week fields in almanac or UTC/leap-second data. Reproducing the behavior with archived NAV data would distinguish those possibilities.

## VM and provenance

The P-code byte fields, primitive dispatch table, indexed operations, and
reference/update convention are now decoded and recorded in the
[opcode reference](reference/vm-opcodes.md). The remaining VM questions concern
original mnemonic names, reserved size combinations, unusual block lengths,
and floating-point edge behavior rather than instruction boundaries.

The exact Pascal compiler/runtime lineage is also unknown. Similarities to period Microware technology are suggestive, not conclusive. No evidence yet expands the `NP` and `SP` version labels.

Finally, the satellite scheduler advances certain events by 86,160 seconds, close to a sidereal day. Reuse of repeating sky geometry is a strong interpretation, but an original module name or design note would turn that inference into a firm conclusion.

These are good targets for the next round of work because each has a discriminating test: trace a DAC net, capture timing-register behavior, complete a rare VM operand, replay rollover NAV data, or find an original schematic/manual. None requires reopening the already coherent high-level architecture.
