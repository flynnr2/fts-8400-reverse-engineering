# Unresolved Questions

[Project index](../README.md) · [Complete edition](How%20the%20FTS%208400%20Worked%20-%20Complete.md)

The architecture is recoverable without answers to these questions, but they limit circuit-level precision:

1. What are the exact schematic names and bit definitions of every `$A3xx` timing register?
2. What physical delay does the fixed 400 ns correction represent?
3. What hardware quantity is represented by the likely 26-bit, 240 ms accumulator?
4. What circuit does the `$A200` DAC drive, and are its ±5.000 units volts at that node?
5. What are the precise roles of the `$A4xx`, remaining `$A5xx`, `$A6xx`, and `$A7xx` registers?
6. How are the acquisition, code-tracking, and Costas/carrier-loop state machines divided between custom logic and software?
7. What is the TIC interpolator topology, its calibration method, and its accuracy—not merely its numerical reconstruction and output command resolution?
8. What is the full RS-232/GPIB command grammar, and how is the six-byte real format encoded?
9. How complete is the OMEGA synchronization path in this ROM and fitted hardware?
10. Which field or firmware revision caused reported first-rollover failures, given that this ROM expands the main ten-bit GPS week?
11. Does the apparent Klobuchar period floor differ from the standard model, or is that an outstanding VM comparison-decoding error?
12. What do `NP` and `SP` mean, and what exact compiler/runtime lineage produced the P-code?
13. The scheduler advances some satellite-related future events by 86,160 s. Is this deliberately a near-sidereal repeat interval for sky geometry? **High confidence as an interpretation, not confirmed nomenclature.**
