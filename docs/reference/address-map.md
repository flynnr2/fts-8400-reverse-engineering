# Address Map

[Project index](../../README.md) · [Complete edition](../How%20the%20FTS%208400%20Worked%20-%20Complete.md)

This is the address-level reference behind the narrative chapters. Names are reconstructed from behavior; they are not claimed original symbols. “Function confirmed” means the software-visible role is clear even when the physical register name is not.

## Main memory and I/O regions

| Address range          | Recovered function                                        | Confidence                                                     |
| ---------------------- | --------------------------------------------------------- | -------------------------------------------------------------- |
| `$000000-$003FFF`      | Native ROM: boot, runtime, VM, maths, and drivers         | Confirmed                                                      |
| `$004000-$007FFF`      | First 16 KiB RAM bank                                     | Confirmed                                                      |
| `$008000-$013FFF`      | I/O and unused space between the RAM banks                | Confirmed as non-RAM; individual holes are not all decoded     |
| `$00A001-$00A00F`, odd | TMS9914A GPIB registers                                   | Confirmed                                                      |
| `$00A101/$00A103`      | RS-232 Port 1, MC6850-compatible ACIA data/control/status | Confirmed                                                      |
| `$00A200`              | 16-bit DAC                                                | Confirmed                                                      |
| `$00A3xx`              | TIC and programmable output-phase hardware                | Function confirmed; individual register naming remains partial |
| `$00A4xx-$00A7xx`      | Receiver, time, status, diagnostic, and control blocks    | Partly understood                                              |
| `$00C011-$00C03F`, odd | MC68901 with inverted register-select ordering            | Confirmed                                                      |
| `$014000-$017FFF`      | Second 16 KiB RAM bank                                    | Confirmed                                                      |
| `$090000-$09FFFF`      | Main application ROM                                      | Confirmed                                                      |

## Native runtime and application routines

| Address              | Recovered meaning                                          | Confidence         |
| -------------------: | ---------------------------------------------------------- | ------------------ |
| `$000140`            | Reset entry                                                | Confirmed          |
| `$0004FC`            | Per-EPROM integrity calculation                            | Confirmed          |
| `$00089C`            | MC68901 Timer-B ISR / periodic service                     | Confirmed          |
| `$00103A`            | Enter Pascal outer block through `$2102`                   | Confirmed          |
| `$00127A`            | Application self-test and initialization                   | Confirmed          |
| `$002000`            | P-code interpreter core                                    | Confirmed          |
| `$002102`            | Establish outer Pascal environment                         | Confirmed          |
| `$002144`, `$002164` | P-code procedure-entry variants                            | Confirmed          |
| `$0034AA`, `$0034C8` | Clear and transfer the 32-character display                | Confirmed          |
| `$0035FE`            | Matrix-vector multiplication                               | High confidence    |
| `$003766`            | 4×4 matrix multiplication                                  | High confidence    |
| `$003AA2`            | 4×4 matrix inversion                                       | High confidence    |
| `$003E8A`            | 4×4 matrix transpose                                       | High confidence    |
| `$0920C4`            | Validate, round, and stage the DAC word                    | Confirmed          |
| `$092108`            | Incremental internal-frequency DAC loop                    | Confirmed          |
| `$0928FC`            | NAV week extraction and rollover expansion                 | Confirmed          |
| `$0962BE`            | GPS week/time maintenance                                  | High confidence    |
| `$0963D2`            | MJD-to-Gregorian calendar conversion                       | Confirmed          |
| `$09716A`            | Character-output fan-out                                   | Confirmed          |
| `$0971BC`            | `TRAP #1` inline-output handler                            | Confirmed          |
| `$097E82`            | Form navigation geometry/design matrix                     | High confidence    |
| `$098088`            | Four-satellite iterative position solution                 | Confirmed          |
| `$09AE68`            | Klobuchar ionosphere correction                            | Confirmed          |
| `$09AFFE`            | Satellite clock, relativity, and group-delay processing    | Confirmed          |
| `$09B314`            | Predicted range/Doppler, code phase, and internal residual | Confirmed          |
| `$09B3E0`            | Raw code/carrier measurement conversion                    | Function confirmed |
| `$09C372`            | Almanac orbit propagation                                  | Confirmed          |
| `$09C816`            | Ephemeris orbit propagation                                | Confirmed          |
| `$09D8BC`            | External-TI least-squares processing                       | Confirmed          |
| `$09D9E0`            | TIC latch, read, and interpolation                         | Function confirmed |

## Important RAM locations

| Address / offset          | Recovered meaning                                     | Confidence       |
| ------------------------: | ----------------------------------------------------- | ---------------- |
| `$4000-$400C`             | Pascal lexical-display pointers                       | Confirmed        |
| `$4043`                   | Four pseudorange-like observations                    | High confidence  |
| `$40A7/$40C7/$40E7`       | Four satellite ECEF X/Y/Z arrays                      | High confidence  |
| `$4127/$4147`             | Four elevation/azimuth arrays                         | High confidence  |
| `$42C5/$42CD/$42D5`       | Receiver latitude, longitude, and altitude            | Confirmed by use |
| `$42F5`                   | Receiver clock/range bias                             | High confidence  |
| `$42FD/$4305/$430D/$4315` | PDOP, HDOP, VDOP, and TDOP                            | Confirmed        |
| `$484E-$4850`             | Signed 24-bit raw fine measurement                    | Confirmed        |
| `$4A15`                   | Modulo-1 ms observed-minus-predicted code-phase range | Confirmed        |
| global `-$4A1D`           | Internal Doppler-corrected frequency residual         | Confirmed        |
| global `-$28F8`           | GPS/UTC/OMEGA source selection                        | Confirmed        |
| `$764F/$7650`             | DAC update-pending flag and 16-bit shadow             | Confirmed        |
| `$17FE0-$17FFF`           | Two-by-16-character display buffer                    | Confirmed        |

## Timing hardware landmarks

| Address             | Recovered meaning             | Confidence         |
| ------------------: | ----------------------------- | ------------------ |
| `$A200`             | Physical 16-bit DAC write     | Confirmed          |
| `$A30B/$A30D`       | TIC fine-interpolator samples | High confidence    |
| `$A311/$A317/$A319` | Output-phase parameters       | High confidence    |
| `$A313`             | Timing latch/control strobes  | Function confirmed |
| `$A315`             | Timing status                 | High confidence    |
| `$A31B/$A31D/$A31F` | Three-byte TIC coarse count   | High confidence    |

## Serial hardware landmarks

| Address / vector      | Recovered meaning                                        | Confidence                                |
| --------------------: | -------------------------------------------------------- | ----------------------------------------- |
| `$A101`               | RS-232 Port 1 MC6850-compatible ACIA data register       | Confirmed                                 |
| `$A103`               | Port 1 ACIA status/control; master reset and RTS control | Confirmed for MC6850-compatible semantics |
| `$C011`               | RS-232 Port 2 MC68901 USART data register                | Confirmed                                 |
| `$C013/$C015`         | Port 2 USART transmit/receive status                     | Confirmed                                 |
| `$C017`               | Port 2 USART control                                     | Confirmed                                 |
| `$C01D`               | Timer C data; Port 1 baud-clock programming              | High confidence                           |
| `$C01B`               | Timer D data; Port 2 baud-clock programming              | High confidence                           |
| vector `$47`          | MFP GPIP5, Port 1 ACIA interrupt                         | High confidence                           |
| vectors `$4A` / `$4C` | Port 2 USART transmit-empty / receive-full interrupts    | Confirmed                                 |
