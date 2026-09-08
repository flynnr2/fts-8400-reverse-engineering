# Recovered Data Structures

[Project index](../../README.md) · [Complete edition](../How%20the%20FTS%208400%20Worked%20-%20Complete.md)

These are global/record bases inferred from named report fields and algorithmic use; array elements commonly use a `$108` stride.

| Offset                 | Field                       | Offset           | Field               |
| ---------------------: | --------------------------- | ---------------: | ------------------- |
| `-$01AF`               | ephemeris WN                | `-$02B7`         | Z-count/TOW-related |
| `-$0571`               | IODC                        | `-$0A99`         | IODE                |
| `-$0991`               | TOC                         | `-$1B19`         | TOE                 |
| `-$0889/-$0781/-$0679` | AF0/AF1/AF2                 | `-$0469`         | TGD                 |
| `-$0EB9`               | sqrt(A)                     | `-$0DB1`         | eccentricity        |
| `-$0CA9`               | delta-n                     | `-$0BA1`         | M0                  |
| `-$12D9/-$13E1`        | i0/IDOT                     | `-$10C9`         | omega               |
| `-$0FC1/-$11D1`        | Omega0/Omega-dot            | `-$1801/-$16F9`  | CRS/CRC             |
| `-$15F1/-$14E9`        | CUS/CUC                     | `-$1A11/-$1909`  | CIS/CIC             |
| `-$1C43/-$1F5B`        | almanac week/TOA            | `-$2273`         | almanac sqrt(A)     |
| `-$284B..-$2863`       | ALPHA0..3                   | `-$286B..-$2883` | BETA0..3            |
| `-$288B/-$2893`        | UTC A0/A1                   | `-$28AB`         | T_ot                |
| `-$289B/-$28A3`        | current/future leap seconds | `-$28AC..-$28AE` | WNt/WNLSF/DN        |

---

The recovered design is unusually coherent for its era: GPS-derived clock harmonics connect code tracking, timing measurement, and pulse generation; a compact Pascal application expresses the navigation and instrument policy; native 68000 code supplies deterministic hardware service and expensive numerical primitives; and external-reference comparison is kept conceptually separate from internal receiver-frequency control. That separation—more than any individual opcode—is the key to understanding how the FTS 8400 worked.
