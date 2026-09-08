# Address Map

[Project index](../../README.md) · [Complete edition](../How%20the%20FTS%208400%20Worked%20-%20Complete.md)

| Address / offset          | Recovered meaning                                         | Confidence                 |
| ------------------------: | --------------------------------------------------------- | -------------------------- |
| `$000140`                 | Reset entry                                               | **Confirmed**              |
| `$0004FC`                 | Per-EPROM integrity calculation                           | **Confirmed**              |
| `$00089C`                 | MC68901 Timer-B ISR / periodic service                    | **Confirmed**              |
| `$00103A`                 | Enter Pascal outer block via `$2102`                      | **Confirmed**              |
| `$00127A`                 | Application self-test/initialization                      | **Confirmed**              |
| `$002000`                 | P-code interpreter core                                   | **Confirmed**              |
| `$002102`                 | Establish outer Pascal environment                        | **Confirmed**              |
| `$002144`, `$002164`      | P-code procedure entry variants                           | **Confirmed**              |
| `$0034AA`, `$0034C8`      | Clear/transfer 32-character display                       | **Confirmed**              |
| `$0035FE`                 | Matrix-vector multiply                                    | **High confidence**        |
| `$003766`                 | 4×4 matrix multiply                                       | **High confidence**        |
| `$003AA2`                 | 4×4 matrix inverse                                        | **High confidence**        |
| `$003E8A`                 | 4×4 transpose                                             | **High confidence**        |
| `$0920C4`                 | Validate/round/stage DAC word                             | **Confirmed**              |
| `$092108`                 | Incremental internal-frequency DAC loop                   | **Confirmed**              |
| `$0928FC`                 | NAV week extraction/rollover expansion                    | **Confirmed**              |
| `$0962BE`                 | Week/time maintenance path                                | **High confidence**        |
| `$0963D2`                 | MJD-to-Gregorian calendar                                 | **Confirmed**              |
| `$09716A`                 | Character output fan-out                                  | **Confirmed**              |
| `$0971BC`                 | `TRAP #1` inline-output handler                           | **Confirmed**              |
| `$097E82`                 | Form navigation geometry/design matrix                    | **High confidence**        |
| `$098088`                 | Four-satellite iterative position solution                | **Confirmed**              |
| `$09AE68`                 | Klobuchar ionosphere correction                           | **Confirmed**              |
| `$09AFFE`                 | Satellite clock/relativity/group-delay path               | **Confirmed**              |
| `$09B314`                 | Predicted range/Doppler; code phase and internal residual | **Confirmed**              |
| `$09B3E0`                 | Raw code/carrier measurement conversion                   | **Confirmed** functionally |
| `$09C372`                 | Almanac orbit propagation                                 | **Confirmed**              |
| `$09C816`                 | Ephemeris orbit propagation                               | **Confirmed**              |
| `$09D8BC`                 | External-TI least-squares processing                      | **Confirmed**              |
| `$09D9E0`                 | `$A3xx` TIC latch/read/interpolation                      | **Confirmed** functionally |
| `$4000-$400C`             | Pascal lexical display pointers                           | **Confirmed**              |
| `$4043`                   | Four pseudorange-like observations                        | **High confidence**        |
| `$40A7/$40C7/$40E7`       | Four satellite ECEF X/Y/Z arrays                          | **High confidence**        |
| `$4127/$4147`             | Four elevation/azimuth arrays                             | **High confidence**        |
| `$42C5/$42CD/$42D5`       | Receiver latitude/longitude/altitude                      | **Confirmed** by use       |
| `$42F5`                   | Receiver clock/range bias                                 | **High confidence**        |
| `$42FD/$4305/$430D/$4315` | PDOP/HDOP/VDOP/TDOP                                       | **Confirmed**              |
| `$484E-$4850`             | Signed 24-bit raw fine measurement                        | **Confirmed**              |
| `$4A15`                   | Modulo-1 ms observed-minus-predicted code-phase range     | **Confirmed**              |
| global `-$4A1D`           | Internal Doppler-corrected frequency residual             | **Confirmed**              |
| global `-$28F8`           | GPS/UTC/OMEGA source selection                            | **Confirmed**              |
| `$764F/$7650`             | DAC pending flag / 16-bit shadow                          | **Confirmed**              |
| `$17FE0-$17FFF`           | 2×16 display buffer                                       | **Confirmed**              |
| `$A200`                   | Physical 16-bit DAC write                                 | **Confirmed**              |
| `$A30B/$A30D`             | TIC fine-interpolator samples                             | **High confidence**        |
| `$A311/$A317/$A319`       | Output-phase parameters                                   | **High confidence**        |
| `$A313`                   | Timing latch/control strobes                              | **Confirmed** functionally |
| `$A315`                   | Timing status                                             | **High confidence**        |
| `$A31B/$A31D/$A31F`       | Three-byte TIC coarse count                               | **High confidence**        |
