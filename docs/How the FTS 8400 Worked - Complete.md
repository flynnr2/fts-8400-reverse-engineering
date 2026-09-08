# How the FTS 8400 Worked

## Recovered software and hardware architecture

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

## 1. Executive summary

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

## 2. Hardware and ROM organization

### Processor and major visible devices

| Device                          | Recovered role                                    | Confidence                                                                                            |
| ------------------------------- | ------------------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| Hitachi `HD63HC000P12`          | 68000-compatible main processor                   | **Confirmed** from board image and valid 68000 reset/code sequences                                   |
| TMS9914A                        | IEEE-488/GPIB controller                          | **Confirmed** from board image and register access pattern                                            |
| MC68901                         | interrupt controller, timers, GPIO, and one USART | **Confirmed** from board image, initialization, and interrupt use                                     |
| 6850-like ACIA at `$A101/$A103` | second serial channel                             | **High confidence** from status/data semantics; exact device identity and port number remain unproved |
| Custom logic at `$A3xx`         | precision TIC and programmable output phase       | **Confirmed** functionally; exact chip/register names remain unresolved                               |
| 16-bit interface at `$A200`     | software-controlled DAC                           | **Confirmed** from the write path                                                                     |

`P12` is a device speed grade; it does not by itself prove the CPU's fitted clock frequency.

### EPROM pairing

The four 8-bit EPROMs form two 16-bit, even/odd byte-lane images:

```text
                    68000 16-bit data bus
                         /          \
                    D15..D8        D7..D0
                      even           odd

 $000000 image:        U36            U44      16 KiB combined
 $090000 image:        U37            U45      64 KiB combined
```

**Confirmed:** interleaving `U36/U44` exactly produces the supplied low ROM image, while `U37/U45` produces the application ROM. The low image begins with a valid reset state:

```text
initial supervisor SP = $00004400
reset PC              = $00000140
```

Coherent 68000 initialization begins at `$000140`.

### Recovered memory and I/O map

| Address range          | Function                                                      | Confidence / qualification                                         |
| ---------------------- | ------------------------------------------------------------- | ------------------------------------------------------------------ |
| `$000000-$003FFF`      | Bootstrap, native runtime, VM, maths and drivers; U36/U44     | **Confirmed**                                                      |
| `$004000-$007FFF`      | RAM bank 0, 16 KiB                                            | **Confirmed** by startup test/clear                                |
| `$008000-$013FFF`      | Non-RAM address space containing I/O decodes and unused areas | **Confirmed** as a RAM discontinuity; not every address is decoded |
| `$00A001-$00A00F`, odd | TMS9914 GPIB registers                                        | **Confirmed**                                                      |
| `$00A101/$00A103`      | ACIA-like status/control and data                             | **High confidence**                                                |
| `$00A200`              | 16-bit DAC register                                           | **Confirmed**                                                      |
| `$00A3xx`              | TIC/output-phase timing block                                 | **Confirmed** functionally                                         |
| `$00A4xx`              | receiver/timing hardware                                      | **Unresolved**                                                     |
| `$00A5xx`              | clock/time interface                                          | **High confidence**                                                |
| `$00A6xx`              | status/configuration/diagnostic inputs                        | **High confidence**                                                |
| `$00A7xx`              | custom control/reset interface                                | **Unresolved**                                                     |
| `$00C011-$00C03F`, odd | MC68901, with inverted register-select order                  | **Confirmed**                                                      |
| `$014000-$017FFF`      | RAM bank 1, 16 KiB                                            | **Confirmed** by startup test/clear                                |
| `$090000-$09FFFF`      | Main P-code/native application; U37/U45                       | **Confirmed**                                                      |

The Pascal runtime hides the split RAM arrangement: when a calculated application address crosses the first bank, the VM adds `$C000` to jump over the physical hole. The two physical banks can therefore serve one logical application-memory scheme.

## 3. Boot, integrity checks, and self-test

### Native boot sequence

**Confirmed:** reset enters native code, not the P-code application.

```text
RESET at $000140
    |
    +-- mask interrupts; execute 68000 RESET
    +-- set SSP=$4400 and initial USP=$4200
    +-- initialize MC68901, GPIB, serial and custom interfaces
    +-- clear/test both 16 KiB RAM windows
    +-- validate all four physical EPROMs independently
    +-- initialize display/front-panel state
    +-- optionally enter deeper hardware diagnostics
    `-- establish Pascal outer block and enter main scheduler
```

The normal application self-test at about `$00127A` is itself a Pascal procedure. It initializes application state—including the DAC's neutral `32768.0` value—and displays `SELF TEST OK`; it can also report `TEST ROUTINES ENABLED`.

### EPROM identity and checksum

Each physical EPROM ends with five bytes:

```text
U36  36 DB EB 0E 9E
U44  44 DB EB CD DE
U37  37 DB EB EC 9D
U45  45 DB EB 14 D1
```

**Confirmed:** byte `-5` is the physical socket ID; bytes `-4..-3` are the common set signature `$DBEB`; bytes `-2..-1` hold a little-endian 16-bit integrity result. The routine around `$0004FC` walks each byte lane separately, uses one's-complement/end-around-carry byte accumulation, complements the result, and reproduces:

```text
U36 $9E0E   U44 $DECD   U37 $9DEC   U45 $D114
```

This is not a conventional CRC. It is sufficient information to create a modified image that passes the ROM's original integrity calculation.

### Factory/service diagnostic path

**Confirmed:** boot tests three status fields:

```text
($A603 & $1E) == $1C
($A605 & $1E) == $1A
($A619 & $1E) == $1E
```

When all match, it enters a deeper native diagnostic which exercises blocks from `$A000` through `$A700` as well as ROM and RAM. **High confidence:** these bits are factory straps or externally driven test conditions. Their physical source is not known.

## 4. Pascal/P-code virtual machine

### Division of responsibility

```text
 Native 68000                         Pascal-family P-code
 ----------------                    --------------------
 reset and exceptions                executive state machines
 interrupt handlers                  acquisition/tracking policy
 memory and ROM tests                NAV database management
 peripheral access                   orbit and position algorithms
 software floating point             timing policy and averaging
 matrix/numerical primitives         menus, reports, diagnostics
 VM interpreter                      communications control
```

The VM interpreter is centered around `$002000`. Ordinary compiled procedures enter through `$002144`; procedures needing local allocation also use `$002164`. The application's outermost environment is established at `$002102` by a call at `$00103A`; its header allocates `$5800` (22,528) bytes of working space.

### Why the language identification is strong

**Confirmed:** RAM locations `$4000`, `$4004`, `$4008`, and `$400C` form a lexical display pointing to current activation records at nesting levels 0–3. Procedure entry replaces the appropriate display entry and return restores it. Identified procedures use lexical levels 1, 2, and 3. This is characteristic compiled-Pascal machinery.

The ROM contains about 382 recognizable P-code procedures: 360 in the large ROM and 22 in the low ROM. About 60 runs of absolute-jump linkage stubs resemble separately linked units or modules. The latter count is architectural evidence, not proof that the original source contained exactly 60 named Pascal units.

The exact compiler is unresolved. Similarity to period Microware Pascal technology is plausible, but the ROM is a bare-metal FTS runtime, not a conventional OS-9 image.

### Instruction model

The recovered major opcode families are:

```text
$00          invalid -> TRAP #5
$01          NOP
$02          execute embedded native 68000 helper
$03 xx       extended primitive
$04-$07 xx   push 10-bit unsigned constant
$08-$0F ...  reference/block/address forms [partly decoded]
$10-$17 xx   relative branch
$18-$1F xx   branch if false
$20-$3F      direct primitive operations
$40-$7F      push integer 0..63
$80-$FF ...  lexical variable load/store/reference family
```

The variable family encodes lexical level and byte, word, longword, or eight-byte object size. Primitive operations include 32-bit integer arithmetic and comparisons, 64-bit real arithmetic and comparisons, call/return, stack allocation, case dispatch, Pascal string literals, block comparison, conversions, and inline text output.

**Caution:** obscure indexed/reference forms are not fully decoded. A linear bytecode listing can mistake their inline operands for opcodes. The architectural interpretation and algorithms in this report were accepted only where constants, data flow, and coherent control flow agreed; this document does not claim a complete 256-entry VM specification.

### Native escapes and numerical library

Opcode `$02` temporarily executes embedded 68000 code and returns to the interpreter. At least 26 procedures reach such an escape. The native runtime supplies eight-byte floating-point operations, square root, sine/cosine, fractional/modulo operations, and matrix routines. **Confirmed:** the eight-byte constants and exponent/mantissa behavior are IEEE-754-like double precision; exact compiler-level conformance in every exceptional case has not been tested.

## 5. Executive and interrupts

### Cooperative executive

The main program is a polling/event executive rather than a recovered pre-emptive RTOS:

```text
                         interrupt handlers
                  (capture events, move bytes/state)
                                  |
                                  v
 initialize -> forever:
                 advance hardware/receiver state
                 conditionally run navigation/position
                 update time and frequency processing
                 service display and keyboard
                 service serial and GPIB work
                 publish outputs and loop
```

**Confirmed:** the outer block repeatedly calls conditional subsystem procedures through stubs at `$0FF2-$1034`. **High confidence:** interrupts handle bounded time-critical transfers while the Pascal tasks advance longer state machines cooperatively.

### MC68901 mapping and interrupts

Later analysis corrected an earlier straightforward-register-order assumption. **Confirmed:** the five MC68901 register-select lines appear inverted from the CPU's point of view:

```text
CPU address = $C001 + 2 * (31 - MC68901 register number)
```

Key examples are `$C011` USART data, `$C015` receiver status, `$C017` USART control, `$C029` vector register, and `$C03F` GPIP. The firmware programs vector base `$40`.

| Vector | Handler   | Recovered use                                     |
| -----: | --------: | ------------------------------------------------- |
| `$43`  | `$000D3E` | GPIP3 timing/control event                        |
| `$47`  | `$000C12` | GPIP5, external serial-interface interrupt        |
| `$48`  | `$00089C` | Timer B, periodic receiver service/software clock |
| `$4A`  | `$000D4E` | MFP transmitter-buffer empty                      |
| `$4C`  | `$000D7C` | MFP receiver-buffer full                          |
| `$4E`  | `$000EF8` | GPIP6, TMS9914 GPIB interrupt                     |

Other vectors in `$40-$4F` lead to a default handler. Timer A and B are configured for event-count operation, emphasizing externally timed events rather than ordinary CPU-clock scheduling.

## 6. GPS NAV database and timekeeping

### Broadcast data model

**Confirmed:** the application maintains separate records/arrays for ephemeris, almanac, ionosphere, UTC/leap-second data, validity, health, and use history. Many satellite fields use a `$108`-byte stride—33 eight-byte slots—consistent with indexed PRN storage.

The ephemeris model includes:

```text
WN, Z-count, IODC, IODE, TOC, TOE
AF0, AF1, AF2, TGD
sqrt(A), eccentricity, M0, delta-n
i0, IDOT, omega, Omega0, Omega-dot
CRS, CRC, CUS, CUC, CIS, CIC
health, alert, fit interval, validity, merit, last-use time
```

The almanac model separately stores week/time, TOA, `sqrt(A)`, eccentricity, `M0`, `i0`, `omega`, `Omega0`, `Omega-dot`, and validity. The receiver can therefore select full ephemeris, fall back to almanac for prediction, or report that orbit data is unavailable.

The shared ionosphere/UTC record contains `ALPHA0..3`, `BETA0..3`, UTC `A0/A1`, `T_ot`, current and future leap seconds, `WN_t`, `WN_LSF`, and day number. A database checksum routine is also present.

### GPS week and calendar handling

**Confirmed:** current expanded GPS week and day-of-week are held near global offsets `-$2EB9` and `-$2EBA`. The calendar pipeline uses:

```text
MJD = GPS_week * 7 + day_of_week + 44244
```

where 44244 is the MJD of the GPS epoch, 6 January 1980. The MJD-to-calendar routine at about `$0963D2` starts from 17 November 1858 and implements the full Gregorian divisible-by-4/100/400 leap-year rule.

The NAV decoder at about `$0928FC` expands the transmitted ten-bit week approximately as:

```text
if received_week <= 453:
    expanded_week = received_week + 1024
else:
    expanded_week = received_week
```

It also increments the expanded week when seconds-of-week crosses 604800.

This **supersedes an earlier tentative explanation** of the receiver's reported 1999 rollover trouble: this ROM revision deliberately handles the main ten-bit rollover. A different firmware revision or another truncated week field—UTC/leap-second or almanac fields are candidates—would be required to explain such a failure.

## 7. Orbit propagation and position solution

### Satellite orbit and clock

The ephemeris propagator at about `$09C816` follows the broadcast GPS model:

```text
tk = wrap_half_week(t - TOE)
mean motion = nominal(sqrt(A)) + delta-n
M = M0 + mean_motion * tk
solve E - e*sin(E) = M iteratively
derive true anomaly and argument of latitude
apply CUS/CUC, CRS/CRC, CIS/CIC corrections
derive corrected radius, inclination and node
transform orbital-plane coordinates to ECEF X,Y,Z
optionally derive Vx,Vy,Vz
```

The half-week wrap uses ±302400 s and 604800 s. A separate, simplified almanac propagator is at about `$09C372`.

The satellite-clock path around `$09AFFE` evaluates:

```text
AF0 + AF1*dt + AF2*dt^2
```

with `TGD` and the relativistic eccentric-orbit term:

```text
delta_tr = F * e * sqrt(A) * sin(E)
F = -4.442809305e-10
```

**Confirmed:** the Klobuchar ionosphere routine at about `$09AE68` follows the broadcast equation, including `psi = 0.0137/(E+0.11)-0.022`, the ±0.416 semicircle pierce-point latitude clamp, local-time wrap, and the daytime polynomial. One apparent period-limit comparison should be rechecked against VM comparison semantics before claiming that FTS used a nonstandard minimum.

### Receiver position and DOP

The solver around `$098088` is an iterative, exactly determined four-satellite solution. It converts a latitude/longitude/height estimate to WGS-84 ECEF, predicts each range, applies Earth-rotation correction during signal transit, forms a 4×4 design matrix and residual vector, and solves for corrections to latitude, longitude, height, and receiver clock/range bias.

```text
four observations + four satellite states
                    |
                    v
       predicted geometric ranges
       + Earth rotation during flight
       + satellite/propagation corrections
                    |
                    v
        H and observation residuals
                    |
                    v
             delta-x = H^-1 r
                    |
                    v
       update position and clock; iterate
```

WGS-84 constants `a = 6378137 m` and `e^2 = 0.00669437999...` are present. Angles are commonly represented in semicircles, explaining explicit factors of pi. The Earth-rotation constant is stored as `2.32115234247e-5` semicircles/s, equal after multiplication by pi to approximately `7.292115147e-5 rad/s`.

Reusable native/P-code numerical routines include matrix-vector multiplication at `$0035FE`, 4×4 multiplication at `$003766`, inversion at `$003AA2`, and transpose at `$003E8A`.

After the fix, the firmware derives `Q = inverse(H^T H)` and computes PDOP, HDOP, VDOP, and TDOP. The reported PDOP participates in acceptance criteria. Accepted stationary fixes feed sums and squared sums for position averaging and scatter estimates.

## 8. Timing and frequency subsystem

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

## 9. Communications, display, and operator interface

### Display

**Confirmed:** `$17FE0-$17FFF` is a 32-byte display buffer, naturally arranged as two 16-character rows. Native routines around `$0034AA` and `$0034C8` clear and transfer it. Bit 7 of each character is stripped during output and drives an attribute; reverse video, blink, highlight, or cursor is plausible, but the exact visible effect is not proven.

### Serial and GPIB

- **Confirmed:** TMS9914A registers occupy odd addresses `$A001-$A00F`; firmware supports configurable GPIB address and `TALK_ONLY`, handles its interrupt, and can report `GPIB NOT READY`.
- **Confirmed:** the MC68901 USART implements one buffered serial path.
- **High confidence:** `$A101/$A103` is a 6850-compatible second serial path. Which physical connector is `PORT 1` or `PORT 2` has not been assigned.
- **Confirmed:** RAM producer/consumer buffers support interrupt-driven byte movement, and both port-not-ready errors exist.
- **Confirmed:** a character fan-out near `$09716A` can send formatted output to enabled serial channels and GPIB. `TRAP #1`, whose handler is at `$0971BC`, efficiently emits inline text.
- **Confirmed:** operator choices include ordinary framing formats and an 8-byte versus 6-byte binary-real transfer format. The 6-byte representation remains undecoded.

The ROM clearly contains reports and control menus for tracking state, NAV updates, position, DOP, time/frequency results, timing source, scheduled observations, RS-232, GPIB, keyboard lock, and automatic printing. A complete external command grammar has not yet been recovered.

## 10. Diagnostics and service modes

Three service layers are visible:

1. **Confirmed — native power-on checks:** RAM, EPROM identity/signature/checksum, peripheral initialization, and custom hardware exercise.
2. **Confirmed — strap-selected factory diagnostics:** the `$A603/$A605/$A619` conditions select a deeper test of all major I/O blocks.
3. **Confirmed in software, high confidence physically — hidden front-panel DAC mode:** two key-state bytes corresponding to ASCII `N` and `P` must remain asserted for five scheduler passes before `SOFTWARE DAC` is enabled. This looks like a held key combination rather than the typed password `NP`, but the physical key labels and safe operating procedure are unverified.

The DAC service page permits direct manipulation of a control that may affect receiver lock or calibration. It should not be used on surviving hardware without recording the original code and understanding the analog destination.

## 11. Corrected interpretations

Later tracing superseded several useful but incorrect early ideas:

- **Corrected:** the external TIC regression does not primarily drive `$A200`. It produces `TI FIT` and external-reference `TI RATE`; the DAC is driven by the separate internal carrier/Doppler residual.
- **Corrected:** the 8400 should be described primarily as a GPS time/frequency comparator and pulse generator, not simply as a GPSDO.
- **Corrected:** `$4A15` is specifically a modulo-1 ms observed-minus-predicted C/A code-phase range.
- **Corrected:** `-$4A42` is a three-state update/averaging state, not the GPS/UTC/OMEGA selector; the latter is at `-$28F8`.
- **Corrected:** the 16.368 MHz factor in the TIC is `0.016368 cycles/ns`; the output generator independently establishes a nominal 4 ns fine phase step.
- **Corrected:** this ROM's primary ten-bit GPS week field contains deliberate rollover expansion. It cannot by itself substantiate a 1999-failure explanation.
- **Corrected:** the MC68901 register select is reversed in the CPU address map; a naive ascending-register map assigns the wrong functions.

## 12. Unresolved questions

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

## Appendix A. Key addresses

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

## Appendix B. Key constants and what they reveal

| Constant                  | Meaning                                                      | Confidence                                        |
| ------------------------: | ------------------------------------------------------------ | ------------------------------------------------- |
| `299792458`               | speed of light, m/s                                          | **Confirmed** by algorithm                        |
| `299792.458`              | light travel, m/ms; one C/A-code millisecond                 | **Confirmed**                                     |
| `1575420000`              | GPS L1 frequency, Hz                                         | **Confirmed**                                     |
| `4092000`                 | internal nominal measurement frequency, Hz (`4 × 1.023 MHz`) | **Confirmed** numerically; hardware label unknown |
| `16368000` / `16368`      | timing clock Hz / cycles per ms                              | **Confirmed**                                     |
| `18.3157660068`           | metres per 16.368 MHz cycle                                  | **Confirmed**                                     |
| `0.016368`                | 16.368 MHz expressed as cycles/ns                            | **Confirmed**                                     |
| `15.2737047898`           | fine-scale ratio producing 250 MHz / 4 ns bins               | **Confirmed** arithmetic                          |
| `3928320`                 | 16.368 MHz cycles in 240 ms                                  | **Confirmed** arithmetic                          |
| `2^23-1`, `2^24`          | signed 24-bit conversion limits                              | **Confirmed**                                     |
| `2^26` relation           | likely accumulator modulus                                   | **High confidence**                               |
| `6378137`                 | WGS-84 semi-major axis, m                                    | **Confirmed**                                     |
| `0.00669437999...`        | WGS-84 eccentricity squared                                  | **Confirmed**                                     |
| `2.32115234247e-5`        | Earth rotation, semicircles/s                                | **Confirmed**                                     |
| `-4.442809305e-10`        | GPS relativistic clock constant                              | **Confirmed**                                     |
| `604800`, `302400`        | GPS week and half-week, s                                    | **Confirmed**                                     |
| `44244`                   | MJD of GPS epoch                                             | **Confirmed**                                     |
| `500000000`, `1000000000` | half/full-second TI phase unwrap, ns                         | **Confirmed**                                     |
| `290.5`                   | internal-loop gain, DAC codes per hertz residual             | **Confirmed** in software                         |
| `32768`                   | neutral/default DAC code                                     | **Confirmed**                                     |
| `86160`                   | recurrence increment, near a sidereal day                    | **Confirmed** value; purpose **high confidence**  |
| `0.0004 ms`               | fixed 400 ns correction                                      | **Confirmed** value; source unresolved            |

## Appendix C. Representative recovered data offsets

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
