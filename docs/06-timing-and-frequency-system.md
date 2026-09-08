# Timing and Frequency System

[Project index](../README.md) · [Complete edition](How%20the%20FTS%208400%20Worked%20-%20Complete.md)

### 8.1 GPS code-phase time observation

The raw measurement conversion around `$09B3E0` reconstructs hardware fields including the three bytes `$484E-$4850`. **Confirmed:** those bytes form a signed 24-bit value: the code tests against `2^23-1` and subtracts `2^24` when required.

The scale factor is:

```text
299792458 / 16368000 = 18.3157660068 metres per coarse cycle
```

Thus each 16.368 MHz cycle is 61.094819 ns or 18.315766 m at light speed. The resulting observed code-phase range is reduced modulo:

```text
299792.458 m = light travel in 1 ms
```

The routine around `$09B314` then calculates the receiver clock/code-phase observation stored at `$4A15`:

```text
predicted_phase_range = c_ms * frac(geometric_range / c_ms)
clock_phase_range     = observed_code_range - predicted_phase_range
if clock_phase_range < 0:
    clock_phase_range += c_ms

c_ms = 299792.458 metres/millisecond
```

**Confirmed:** `$4A15` is therefore observed-minus-predicted GPS C/A code phase expressed as range, modulo one millisecond—not a generic range or time correction.

The timing arithmetic is internally coherent:

```text
range                  metres
fine time              milliseconds
cable/time offsets     nanoseconds * 1e-6 -> milliseconds
GPS/UTC corrections    seconds * 1000 -> milliseconds
```

A fixed `0.0004 ms` (400 ns) correction is present, but its physical source is unresolved.

A second raw scale satisfies:

```text
0.058536529541015... * 2^26 = 3928320
3928320 / 16368000 = 0.240 s
```

**High confidence:** this indicates a 26-bit accumulator/NCO quantity referenced to a 240 ms observation interval. Its precise circuit-level meaning is not established.

### 8.2 The 16.368 MHz timing architecture

The recovered frequencies share the GPS C/A base:

```text
1.023 MHz   C/A chip rate
4.092 MHz =  4 * 1.023 MHz
5.115 MHz =  5 * 1.023 MHz
16.368 MHz = 16 * 1.023 MHz
L1         = 1540 * 1.023 MHz = 1575.42 MHz
```

**High confidence:** this is the organizing clock family for the digital receiver and precision timing logic.

For pulse generation, firmware converts desired phase in milliseconds to coarse cycles by multiplying by 16368. It multiplies the fractional cycle by `15.2737047898`, for which:

```text
16.368 MHz * 15.2737047898 = 250 MHz
1 / 250 MHz = 4 ns
```

Thus the programmed output phase has:

```text
coarse count      61.094819 ns per 16.368 MHz cycle
fine count        nominally about 4 ns
```

This establishes the firmware's commanded resolution, not the hardware's absolute timing accuracy.

The code deliberately manages a one-millisecond, 16,368-count wrap. Constants near half-epoch (`8177`, `8188`, `8206`) and alternative offsets separated by exactly 16,368 (`8206` and `24574`) avoid a discontinuity as the desired output phase crosses the counter boundary.

Output parameters are staged in RAM and written through `$A311`, `$A317`, and `$A319` after status at `$A315` is checked.

### 8.3 GPS/UTC pulse generation

A three-state clock-update machine initializes, establishes a solution, then performs normal averaging/update operation. This state is distinct from the source selector, whose recovered values are:

```text
0 = GPS
1 = UTC
2 = OMEGA
```

For UTC, firmware evaluates the broadcast GPS-to-UTC polynomial using `A0`, `A1`, `T_ot`, and `WN_t`, with week wrapping. Cable delay and configured `TIME OFFSET` are included before output phase is programmed. OMEGA support is visible in the UI and state selection, but its complete external hardware path is not reconstructed.

A running observation average rejects large discontinuities (threshold approximately 100,000 in the working unit) by restarting the average. The user can request a clock update only after a configured number of acceptable averages. This is consistent with a metrology instrument that validates and averages before moving an output timescale.

### 8.4 External TIC, regression, and `TI RATE`

Procedure `$09D9E0` latches `$A3xx`, reads a three-byte coarse counter from approximately `$A31B/$A31D/$A31F`, and obtains fine start/stop information through `$A30B/$A30D`. It reconstructs:

```text
coarse count + start fraction - stop fraction
```

and divides cycles by `0.016368 cycles/ns` to produce nanoseconds. The fine fractions are calculated from ratios of captured calibration/sample quantities. The exact analog interpolation circuit and calibrated single-shot resolution are unresolved.

The code explicitly detects a missing external pulse and displays `EXTERNAL PULSE / NOT CONNECTED`.

For valid samples:

```text
x = GPS_week * 604800 + seconds_of_week       [seconds]
y = external time interval                    [nanoseconds]
```

After subtracting the first `(x0,y0)`, phase differences are unwrapped at ±500,000,000 ns by adding or subtracting 1,000,000,000 ns. The program accumulates `N`, `sum(x)`, `sum(x^2)`, `sum(y)`, and `sum(xy)`, then computes:

```text
        N*sum(xy) - sum(x)*sum(y)
slope = --------------------------------
        N*sum(x^2) - sum(x)^2

intercept = (sum(y) - slope*sum(x)) / N
```

The UI names the results:

| Display   | Meaning                                                                    |
| --------- | -------------------------------------------------------------------------- |
| `TI`      | current external pulse time interval relative to the GPS-derived timescale |
| `TI FIT`  | least-squares fitted interval                                              |
| `TI RATE` | fitted slope in ps/s after ×1000 scaling                                   |

Because a slope in ns/s is fractional frequency scaled by `10^9`, converting it to ps/s gives parts in `10^12`. **Confirmed:** `TI RATE` is the external reference's fractional-frequency difference versus GPS.

### 8.5 Internal `DF/F` and DAC

The internal path is separate. Near `$09B314`, firmware predicts L1 Doppler from line-of-sight satellite velocity and evaluates approximately:

```text
internal_frequency_error =
    measured_internal_frequency
    - 4,092,000 Hz
    + predicted_Doppler
```

The value at global offset `-$4A1D` is displayed as `INTRNL DF/F`; a running mean is `AVG DF/F`. Division by `0.00157542` converts an L1-equivalent hertz residual to parts in `10^12`, because `1575420000 * 10^-12 = 0.00157542`.

The internal control routine at about `$092108` incrementally changes the DAC request by approximately:

```text
DAC_request += 290.5 * internal_frequency_error
```

The commit routine at `$0920C4` accepts codes `0..65535`, substitutes midpoint `32768` for an out-of-range request, rounds to the nearest word, and sets a pending flag. Native periodic code performs:

```text
RAM shadow $7650 -> hardware word $A200
pending flag $764F is then cleared
```

**Confirmed:** `$A200` is a 16-bit DAC controlled by the internal carrier-frequency residual. **High confidence:** it tunes an internal receiver oscillator or frequency-control element. It is not proven to steer the user's external reference.

The hidden service display maps code to:

```text
engineering_value = 5.0 - DAC_code / 6553.5
```

giving approximately `+5.000`, `0.000`, and `-5.000` at codes 0, 32768, and 65535. **High confidence:** the representation is bipolar ±5 engineering units. Direct volts at the controlled node remain a hypothesis until the PCB is traced or measured.
