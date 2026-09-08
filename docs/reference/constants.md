# Key Constants

[Project index](../../README.md) · [Complete edition](../How%20the%20FTS%208400%20Worked%20-%20Complete.md)

Constants were among the most reliable clues in the ROM. Several apparently arbitrary values become exact GPS, geodetic, or timing quantities when the firmware's units are understood.

## GPS and geodesy

| Constant           | Interpretation                             | Confidence |
| -----------------: | ------------------------------------------ | ---------- |
| `299792458`        | Speed of light in metres per second        | Confirmed  |
| `299792.458`       | Light travel in one millisecond, in metres | Confirmed  |
| `1575420000`       | GPS L1 carrier frequency in hertz          | Confirmed  |
| `6378137`          | WGS-84 semi-major axis in metres           | Confirmed  |
| `0.00669437999...` | WGS-84 eccentricity squared                | Confirmed  |
| `2.32115234247e-5` | Earth rotation in semicircles per second   | Confirmed  |
| `-4.442809305e-10` | GPS relativistic satellite-clock constant  | Confirmed  |
| `604800`, `302400` | GPS week and half-week in seconds          | Confirmed  |
| `44244`            | Modified Julian Date of the GPS epoch      | Confirmed  |

## Receiver and timing scales

| Constant                  | Interpretation                                               | Confidence                                       |
| ------------------------: | ------------------------------------------------------------ | ------------------------------------------------ |
| `4092000`                 | Internal nominal frequency, `4 × 1.023 MHz`                  | Numerical role confirmed; hardware label unknown |
| `16368000` / `16368`      | Timing-clock hertz / cycles per millisecond                  | Confirmed                                        |
| `18.3157660068`           | Metres travelled by light in one 16.368 MHz cycle            | Confirmed                                        |
| `0.016368`                | 16.368 MHz expressed as cycles per nanosecond                | Confirmed                                        |
| `15.2737047898`           | Fine-phase ratio corresponding to 250 MHz, or 4 ns per count | Arithmetic confirmed                             |
| `3928320`                 | Number of 16.368 MHz cycles in 240 ms                        | Arithmetic confirmed                             |
| `2^23-1`, `2^24`          | Limits used to sign-extend a 24-bit measurement              | Confirmed                                        |
| `2^26` relation           | Likely modulus of an accumulator/NCO quantity                | High confidence                                  |
| `500000000`, `1000000000` | Half/full-second time-interval phase unwrap in nanoseconds   | Confirmed                                        |
| `0.0004 ms`               | Fixed 400 ns correction                                      | Value confirmed; physical source unresolved      |

## Control and scheduling

| Constant | Interpretation                                            | Confidence                                  |
| -------: | --------------------------------------------------------- | ------------------------------------------- |
| `290.5`  | Internal-loop gain in DAC codes per hertz of residual     | Confirmed in software                       |
| `32768`  | Neutral/default 16-bit DAC code                           | Confirmed                                   |
| `86160`  | Satellite-event recurrence increment, near a sidereal day | Value confirmed; purpose is high confidence |
