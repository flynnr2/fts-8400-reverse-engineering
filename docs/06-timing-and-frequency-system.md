# The Timing and Frequency System

[Project index](../README.md) · [Complete edition](How%20the%20FTS%208400%20Worked%20-%20Complete.md)

## From a GPS signal to a clock observation

The timing path begins with a raw receiver measurement rather than with an already solved timestamp. Code near `$09B3E0` assembles several hardware fields, including a three-byte signed quantity at `$484E-$4850`. The signed interpretation is unambiguous: values above `2^23-1` are reduced by `2^24`.

One conversion constant reveals the counter behind the measurement:

```text
299792458 / 16368000 = 18.3157660068 metres
```

One 16.368 MHz cycle lasts 61.094819 ns, during which light travels 18.315766 m. Firmware scales the reconstructed count by exactly that distance and reduces it modulo 299,792.458 m—the distance light travels in one millisecond, matching one C/A-code period.

The next stage predicts the geometric satellite-to-receiver range, reduces that prediction to the same one-millisecond phase interval, and subtracts it from the observation:

```text
predicted phase range = c_ms × frac(geometric_range / c_ms)

clock phase range = observed code range - predicted phase range

if clock phase range < 0:
    clock phase range += c_ms

c_ms = 299792.458 metres per millisecond
```

The result at `$4A15` is therefore a receiver clock/code-phase observation expressed as range modulo one millisecond. This is more precise than the early description of it as a generic range or time correction.

The unit system is consistent throughout the timing code:

```text
geometric and phase range       metres
fine time                       milliseconds
cable and user delay            nanoseconds × 10^-6 -> milliseconds
GPS/UTC corrections             seconds × 1000 -> milliseconds
```

A fixed 0.0004 ms, or 400 ns, correction is present in this path. The code proves its value but does not identify the physical delay it compensates.

Another scaling relation points to a 26-bit digital quantity:

```text
0.058536529541015... × 2^26 = 3928320
3928320 / 16368000 = 0.240 seconds
```

The arithmetic strongly suggests a 26-bit accumulator or NCO associated with a 240 ms observation interval. Its precise hardware name remains unknown.

## A timing system built around the C/A-code clock family

The receiver's recurring frequencies are related rather than arbitrary:

```text
1.023 MHz   GPS C/A-code chip rate
4.092 MHz =  4 × 1.023 MHz
5.115 MHz =  5 × 1.023 MHz
16.368 MHz = 16 × 1.023 MHz
GPS L1     = 1540 × 1.023 MHz = 1575.42 MHz
```

The 16.368 MHz clock provides a common coarse scale for code measurement, external time-interval counting, and programmed output phase. There are 16,368 cycles per millisecond and each cycle is about 61.1 ns.

That coarse period would not be sufficient by itself for a precision timing instrument. The custom `$A3xx` hardware therefore adds fine interpolation on both measurement and generation paths.

## Programming the output pulse to about 4 ns

For output generation, the firmware multiplies desired phase in milliseconds by 16,368 and splits the result into integer and fractional coarse cycles. It scales the fraction by `15.2737047898`. The apparently odd value has a simple consequence:

```text
16.368 MHz × 15.2737047898 = 250 MHz
1 / 250 MHz = 4 ns
```

The programmed phase representation is therefore:

```text
coarse phase     16.368 MHz, or 61.094819 ns per count
fine phase       nominally about 4 ns per count
```

This is evidence for command granularity, not a measurement of absolute accuracy, jitter, or calibration.

The code also manages the awkward point where a modulo phase crosses the hardware epoch. Values near half of 16,368 (`8177`, `8188`, and `8206`) and two alternatives separated by exactly one full epoch (`8206` and `24574`) allow the requested output phase to remain continuous while the underlying counter wraps.

Output values are staged in RAM and later written to `$A311`, `$A317`, and `$A319` after checking status at `$A315`. The exact electrical names of these registers are not recovered, but their role in coarse/fine phase programming is clear.

## GPS, UTC, and OMEGA output time

The application can align its time/frequency outputs to GPS, UTC, or OMEGA. The actual selector is held at global offset `-$28F8`; a nearby three-state value once mistaken for this choice instead controls initialization and averaging progress.

For GPS, no UTC offset is applied. For UTC, the firmware evaluates the broadcast conversion polynomial using `A0`, `A1`, `T_ot`, and `WN_t`, including week wrapping and leap-second information. Cable delay and the configured user `TIME OFFSET` are then included before the fractional output phase is programmed.

OMEGA support is real at the application level—the menus and selector are explicit—but its complete physical input and processing path have not been reconstructed.

The clock update is deliberately conservative. A running average accepts nearby observations and restarts when a gross step is seen. The user can require a chosen number of acceptable averages before the clock is updated. The recovered pattern is:

```text
measure -> validate -> average -> test quality -> update output time
```

That behavior matters in a metrology instrument: a single noisy satellite observation should not move the generated timescale.

## Measuring an external reference

The same `$A3xx` block contains the external time-interval counter. Procedure `$09D9E0` commands a latch through `$A313`, reads a three-byte coarse value from roughly `$A31B/$A31D/$A31F`, and obtains fine start/stop samples through `$A30B/$A30D`.

Conceptually it reconstructs:

```text
coarse count + start interpolation fraction - stop interpolation fraction
```

The fine fractions are ratios of captured sample and calibration quantities. Once combined, the count is divided by:

```text
0.016368 cycles per nanosecond
```

to produce a time interval in nanoseconds. The software rejects inconsistent snapshots and explicitly reports when the external pulse is not connected.

The TIC clearly resolves fractions of the 61.1 ns coarse clock. The ROM alone does not establish its analog interpolation topology, calibrated single-shot resolution, or accuracy, so those should not be inferred from the output generator's separate 4 ns command scale.

## Why the receiver fits a straight line

Each valid external measurement is paired with absolute GPS time:

```text
x = GPS_week × 604800 + seconds_of_week       seconds
y = measured external time interval           nanoseconds
```

The first pair becomes a local origin. Later phase differences are unwrapped at half a second:

```text
if dy > +500000000 ns: dy -= 1000000000 ns
if dy < -500000000 ns: dy += 1000000000 ns
```

The firmware accumulates `N`, `sum(x)`, `sum(x^2)`, `sum(y)`, and `sum(xy)` and evaluates ordinary least squares:

```text
        N sum(xy) - sum(x) sum(y)
slope = ---------------------------
        N sum(x^2) - sum(x)^2

intercept = (sum(y) - slope sum(x)) / N
```

The resulting displays have distinct meanings:

| Display   | Meaning                                                                    |
| --------- | -------------------------------------------------------------------------- |
| `TI`      | Current external-pulse time interval relative to the GPS-derived timescale |
| `TI FIT`  | Time interval predicted by the least-squares line                          |
| `TI RATE` | Slope of the fitted line, converted to picoseconds per second              |

A phase slope measured in ns/s is fractional frequency scaled by `10^9`. Multiplying by 1000 expresses the same value in ps/s, or parts in `10^12`. `TI RATE` is therefore the external reference's fractional-frequency difference from GPS.

This statistical estimate is one of the more sophisticated features of the instrument. The 8400 asks which frequency difference best explains a series of phase measurements, rather than treating the most recent interval as definitive.

## The separate internal frequency loop

The internal path starts from the receiver's carrier measurement. Code near `$09B314` predicts satellite Doppler from the line-of-sight velocity and forms approximately:

```text
internal frequency error =
    measured internal frequency
    - 4092000 Hz
    + predicted satellite Doppler
```

The residual at global offset `-$4A1D` is shown as `INTRNL DF/F`; a running mean is shown as `AVG DF/F`. Division by `0.00157542` converts an L1-equivalent hertz residual to parts in `10^12`, because:

```text
1575420000 × 10^-12 = 0.00157542
```

The control procedure around `$092108` applies a direct incremental correction:

```text
DAC request += 290.5 × internal frequency error
```

The commit routine bounds the request to `0..65535`, substitutes the neutral midpoint `32768` for an invalid value, rounds it to the nearest word, and marks an update pending. Native periodic code then copies the word from RAM `$7650` to the hardware register at `$A200` and clears the flag at `$764F`.

This proves that `$A200` is a 16-bit DAC controlled by the internal carrier/Doppler residual. It does not prove the analog destination. Tuning a receiver oscillator or related frequency-control element is the strongest interpretation; steering the user's external reference is not supported by this path.

The hidden service page displays the DAC using:

```text
displayed value = 5.0 - DAC_code / 6553.5
```

so code 0 appears as about `+5.000`, midpoint as zero, and 65535 as `-5.000`. A bipolar ±5 V control is plausible, but only a schematic or measurement can show whether the displayed units are volts at the controlled node.

## The recovered timing architecture

The main timing paths can now be stated without conflating them:

```text
GPS code phase + orbit model
          |
          v
receiver clock phase modulo 1 ms
          |
          +-> GPS/UTC delay model -> coarse/fine phase -> output pulses

GPS carrier + predicted Doppler
          |
          v
INTRNL DF/F -> averaging -> internal DAC control

external reference pulse
          |
          v
interpolating TIC -> TI samples -> least-squares fit -> TI RATE
```

That separation is the key to the instrument. Navigation supplies a trustworthy GPS time solution; the output hardware realizes it; the internal loop keeps the receiver's own frequency chain under control; and the TIC compares an independent external standard with GPS.
