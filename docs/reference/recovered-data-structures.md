# Recovered Data Structures

[Project index](../../README.md) · [Complete edition](../How%20the%20FTS%208400%20Worked%20-%20Complete.md)

The original Pascal declarations are lost, but report routines print named GPS fields immediately after loading their corresponding globals. Combined with algorithmic use and the recurring `$108`-byte satellite stride, this recovers much of the logical database layout.

Offsets below are Pascal global or record bases, not absolute CPU addresses. Satellite arrays commonly contain 33 eight-byte slots.

## Broadcast ephemeris

| Offset                 | Field            | Offset          | Field        |
| ---------------------: | ---------------- | --------------: | ------------ |
| `-$01AF`               | GPS week         | `-$02B7`        | Z-count/TOW  |
| `-$0571`               | IODC             | `-$0A99`        | IODE         |
| `-$0991`               | TOC              | `-$1B19`        | TOE          |
| `-$0889/-$0781/-$0679` | AF0/AF1/AF2      | `-$0469`        | TGD          |
| `-$0EB9`               | sqrt(A)          | `-$0DB1`        | eccentricity |
| `-$0CA9`               | delta-n          | `-$0BA1`        | M0           |
| `-$12D9/-$13E1`        | i0/IDOT          | `-$10C9`        | omega        |
| `-$0FC1/-$11D1`        | Omega0/Omega-dot | `-$1801/-$16F9` | CRS/CRC      |
| `-$15F1/-$14E9`        | CUS/CUC          | `-$1A11/-$1909` | CIS/CIC      |

Additional nearby fields hold health, alert, fit interval, merit, validity, and last-use state.

## Almanac

| Offset   | Field         | Offset   | Field        |
| -------: | ------------- | -------: | ------------ |
| `-$1C43` | Almanac week  | `-$1D4B` | Almanac time |
| `-$1F5B` | TOA           | `-$2273` | sqrt(A)      |
| `-$1E53` | eccentricity  | `-$258B` | M0           |
| `-$2063` | i0            | `-$2483` | omega        |
| `-$237B` | Omega0        | `-$216B` | Omega-dot    |
| `-$25AD` | validity flag |          |              |

## Ionosphere and UTC

| Offset           | Field                      | Offset           | Field        |
| ---------------: | -------------------------- | ---------------: | ------------ |
| `-$284B..-$2863` | ALPHA0..ALPHA3             | `-$286B..-$2883` | BETA0..BETA3 |
| `-$288B/-$2893`  | UTC A0/A1                  | `-$28AB`         | T_ot         |
| `-$289B/-$28A3`  | current/future leap second | `-$28AC..-$28AE` | WNt/WNLSF/DN |

This layout is sufficient to follow the data from NAV-message decoding into orbit propagation, satellite-clock correction, ionosphere modelling, UTC conversion, and reporting. Exact Pascal type declarations and a few administrative fields remain to be reconstructed.
