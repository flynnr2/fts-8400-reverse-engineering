# How the FTS 8400 Worked

## A recovered software and hardware architecture

[Project index](../README.md) · [VM opcode reference](reference/vm-opcodes.md)

This is the complete narrative edition of the FTS 8400 reverse-engineering results. It follows the investigation from the four physical EPROMs through the Pascal virtual machine, GPS navigation solution, and precision timing system. Detailed addresses, constants, and reconstructed data layouts are retained as appendices rather than interrupting the main account.

The analysed firmware identifies itself as `NP 120 10-OCT-87` and `SP 101 26-JUL-87`. Reconstructed routine and structure names describe observed behavior; they are not claimed original FTS symbols.

## Contents

1. [Overview and major findings](#1-overview-and-major-findings)
2. [Hardware, memory, and boot](#2-hardware-memory-and-boot)
3. [The Pascal/P-code architecture](#3-the-pascalp-code-architecture)
4. [Executive and communications](#4-executive-and-communications)
5. [The GPS navigation system](#5-the-gps-navigation-system)
6. [The timing and frequency system](#6-the-timing-and-frequency-system)
7. [Diagnostics and service modes](#7-diagnostics-and-service-modes)
8. [What remains unresolved](#8-what-remains-unresolved)
9. [Appendix A: Address map](#appendix-a-address-map)
10. [Appendix B: Key constants](#appendix-b-key-constants)
11. [Appendix C: Recovered data structures](#appendix-c-recovered-data-structures)

## 1. Overview and major findings

### Four ROMs and a timing receiver

The FTS 8400 looks, at first, like a fairly conventional piece of mid-1980s instrumentation. Its processor board carries a Hitachi HD63HC000, the CMOS version of the Motorola 68000, together with an MC68901 multifunction peripheral, a TMS9914A IEEE-488 controller, RAM, EPROMs, and a collection of custom receiver and timing logic.

The first ROM dump did not look like valid 68000 firmware. The reason was physical rather than cryptographic: it contained only the even byte lane of a 16-bit program. Interleaving the four EPROMs as two pairs produced a 16 KiB low ROM and a 64 KiB application ROM. A valid reset vector appeared immediately:

```text
initial supervisor stack pointer = $00004400
reset program counter            = $00000140
```

That solved the first puzzle. The second was more interesting. Much of the larger image still was not 68000 machine code.

### The unexpected software architecture

The 8400 runs a small native 68000 kernel, but most of the instrument is a Pascal-family program executed by a compact P-code virtual machine. The native layer handles reset, interrupts, memory tests, hardware access, floating-point primitives, and the interpreter itself. The P-code layer contains the executive, user interface, navigation algorithms, timing policy, reporting, and most of the instrument's state machines.

Roughly 382 P-code procedure entries can be identified, arranged behind about 60 groups of linkage stubs. The precise original module count is not recoverable from those stubs alone, but this was plainly a modular application rather than one large assembly-language program.

The VM is recognizably Pascal because it maintains a lexical display for nested procedure scopes. It supports compact constants, branches, integer and eight-byte real operations, Pascal strings, procedure calls, local frames, and escapes into embedded native 68000 code.

For a receiver of this age, that choice is striking but sensible. Compact bytecode conserved ROM space. Pascal made a large mathematical application manageable. Native code remained available where deterministic hardware service or expensive arithmetic justified it.

### What the application actually does

The ROM contains far more than menus and hardware drivers. It holds the GPS navigation database and the mathematics needed to use it:

- broadcast ephemeris and almanac records;
- GPS satellite clock and relativistic corrections;
- Klobuchar ionospheric correction;
- satellite orbit and velocity propagation;
- WGS-84 coordinate conversion;
- a four-satellite iterative position and clock solution;
- PDOP, HDOP, VDOP, and TDOP calculation;
- position averaging for a stationary timing installation;
- GPS week, Modified Julian Date, calendar, UTC, and leap-second handling.

The algorithms use software eight-byte floating point and a small reusable matrix library. This is not a controller asking a separate navigation engine for a finished answer: the 68000 application performs the navigation solution itself.

### The machine is better understood as a comparator

An early interpretation treated the 8400 as a straightforward GPS-disciplined oscillator:

```text
GPS phase error -> regression -> DAC -> oscillator
```

Later tracing showed that this combined two separate paths. The corrected architecture is:

```text
                         GPS receiver
                              |
               +--------------+--------------+
               |                             |
        navigation and time           carrier residual
               |                             |
      GPS- or UTC-aligned phase          INTRNL DF/F
               |                         AVG DF/F
      16.368 MHz timing logic                |
               |                       internal DAC
       second/minute outputs               $A200
               |
               +------------------------------+
                                              |
external reference pulse ------------> interpolating TIC
                                              |
                                     TI, TI FIT, TI RATE
                                              |
                              external frequency versus GPS
```

The internal path removes predicted satellite Doppler from a receiver frequency measurement. Its residual is displayed as `INTRNL DF/F` and drives a 16-bit DAC through a simple incremental control law. The evidence strongly suggests that the DAC tunes an internal receiver oscillator or related frequency-control element; it is not proven to steer the user's external standard.

The external path measures a reference pulse with a high-resolution time-interval counter. Repeated measurements are fitted by ordinary least squares. The fitted slope, displayed as `TI RATE` in picoseconds per second, is the external reference's fractional-frequency difference from GPS in parts in `10^12`.

This makes the 8400 primarily a GPS time/frequency comparator and pulse generator, not merely an early GPSDO.

### Why 16.368 MHz appears everywhere

Several initially mysterious constants collapse into one clock family:

```text
1.023 MHz   GPS C/A-code chip rate
4.092 MHz =  4 × 1.023 MHz
5.115 MHz =  5 × 1.023 MHz
16.368 MHz = 16 × 1.023 MHz
GPS L1     = 1540 × 1.023 MHz = 1575.42 MHz
```

The raw code-phase conversion uses exactly the distance light travels during one 16.368 MHz cycle:

```text
299792458 / 16368000 = 18.3157660068 metres
```

The time-interval counter also measures coarse cycles at 16.368 MHz and supplements them with fractional start/stop interpolation. The output generator independently uses 16,368 cycles per millisecond and a fine scale whose arithmetic corresponds to a nominal 250 MHz, or about 4 ns per fine count.

The commanded 4 ns increment is not a claim about absolute output accuracy. It does show how a 61.1 ns coarse clock supported much finer phase placement.

### The most important lessons from the ROM

The main surprises are architectural rather than incidental:

1. A Pascal virtual machine is the foundation of the application.
2. The receiver performs a complete GPS orbit, clock, position, and time solution in software.
3. Software double precision and reusable matrix routines make that practical on a 68000-class processor.
4. External frequency comparison is based on a regression through many phase measurements, not on the latest sample alone.
5. Internal receiver-frequency control and external-reference measurement are deliberately separate.
6. GPS-derived clock harmonics connect code tracking, time measurement, and pulse generation into one coherent timing architecture.
7. The firmware was designed to validate, average, and reject suspect observations before changing the instrument's time outputs.

### Corrections made as the analysis improved

Several early ideas were useful stepping stones but did not survive deeper tracing:

- The external TIC regression does not primarily drive the DAC. It produces `TI FIT` and `TI RATE`; the DAC uses the separate internal carrier/Doppler residual.
- `$4A15` is specifically an observed-minus-predicted C/A-code phase expressed as range modulo one millisecond, not a generic range correction.
- The state at global offset `-$4A42` controls clock-update averaging; GPS/UTC/OMEGA selection is held separately at `-$28F8`.
- This ROM deliberately expands the main ten-bit GPS week across the first rollover. That field alone cannot explain reports of an FTS 8400 failing in 1999.
- The MC68901 registers appear in reversed select-line order in the CPU address map. A conventional ascending register assignment gives incorrect names.

The remaining unknowns are mostly at the hardware boundary: exact custom-register names, the TIC's interpolation circuitry, the analog destination of the DAC, and the division of acquisition and tracking between software and custom logic. None of those prevents a useful reconstruction of the overall design.

## 2. Hardware, memory, and boot

### Reassembling the ROM image

The first useful result came from treating the EPROMs as hardware rather than four independent files. The HD63HC000 has a 16-bit external data bus, while each fitted EPROM supplies eight bits. The devices therefore work in even/odd pairs:

```text
                    68000 data bus
                       /      \
                  D15..D8    D7..D0
                    even       odd

low ROM             U36        U44      $000000-$003FFF
application ROM     U37        U45      $090000-$09FFFF
```

Interleaving `U36` with `U44` produces a 16 KiB native ROM beginning at address zero. Interleaving `U37` with `U45` produces the 64 KiB application image at `$090000`. The combined low image opens with a valid 68000 stack pointer and reset vector, followed by coherent initialization code at `$000140`.

The board photograph independently agrees with the software: it shows a Hitachi `HD63HC000P12`, an MC68901, and a TMS9914A. The `P12` suffix is a processor speed grade, not proof of the clock fitted in this particular instrument.

### A memory map shaped by the board

The 8400 does not present the Pascal application with one continuous RAM region. Startup clears and tests two 16 KiB windows at `$004000-$007FFF` and `$014000-$017FFF`, separated by I/O and unused address space.

| Address range          | Recovered function                                                                 | Status                                                         |
| ---------------------- | ---------------------------------------------------------------------------------- | -------------------------------------------------------------- |
| `$000000-$003FFF`      | Native bootstrap, runtime, VM, maths, and drivers; U36/U44                         | Confirmed                                                      |
| `$004000-$007FFF`      | First 16 KiB RAM bank                                                              | Confirmed by startup test and clear                            |
| `$008000-$013FFF`      | Non-RAM space containing I/O decodes and unused regions                            | Confirmed as a RAM discontinuity; not every address is decoded |
| `$00A001-$00A00F`, odd | TMS9914A IEEE-488 registers                                                        | Confirmed                                                      |
| `$00A101/$00A103`      | RS-232 Port 1, MC6850-compatible ACIA data/control/status                          | Confirmed                                                      |
| `$00A200`              | 16-bit DAC                                                                         | Confirmed                                                      |
| `$00A3xx`              | Time-interval and output-phase hardware                                            | Function confirmed; individual registers only partly named     |
| `$00A4xx`              | Receiver/timing hardware                                                           | Unresolved                                                     |
| `$00A5xx`              | Clock/time interface                                                               | High confidence                                                |
| `$00A6xx`              | Status, configuration, and diagnostic inputs                                       | High confidence                                                |
| `$00A7xx`              | Custom control/reset interface                                                     | Unresolved                                                     |
| `$00C011-$00C03F`, odd | MC68901, including the RS-232 Port 2 USART, with inverted register-select ordering | Confirmed                                                      |
| `$014000-$017FFF`      | Second 16 KiB RAM bank                                                             | Confirmed by startup test and clear                            |
| `$090000-$09FFFF`      | Main application ROM; U37/U45                                                      | Confirmed                                                      |

The VM knows about the discontinuity. When a calculated Pascal address crosses the first bank, the interpreter adds `$C000` and lands in the second. The application can therefore use a convenient logical memory model even though the PCB presents two separated physical banks.

The first 16 bytes of RAM, `$4000-$400C`, are especially important: they hold the lexical-display pointers used by the Pascal runtime. The 32 bytes at `$17FE0-$17FFF` form the two-line front-panel display buffer.

### What happens at reset

Boot remains native until the hardware is checked and stable:

```text
RESET at $000140
    |
    +-- mask interrupts and execute the 68000 RESET instruction
    +-- establish SSP=$4400 and initial USP=$4200
    +-- initialize the MC68901, GPIB, serial, and custom interfaces
    +-- clear and test both RAM windows
    +-- identify and checksum all four EPROMs
    +-- initialize display and front-panel state
    +-- test for the factory-diagnostic conditions
    `-- create the outer Pascal environment and start the executive
```

The processor does not jump blindly into the application ROM. It establishes a known interrupt and peripheral state, verifies the memory on which the interpreter depends, and checks that the four EPROMs belong together.

### The EPROM trailers are service metadata

Each physical device ends with five programmed bytes:

```text
U36  36 DB EB 0E 9E
U44  44 DB EB CD DE
U37  37 DB EB EC 9D
U45  45 DB EB 14 D1
```

The boot code explains all five bytes. The first is the socket number, `$DBEB` is a shared ROM-set signature, and the last word is an integrity value stored low byte first.

The checksum routine at about `$0004FC` works on each physical eight-bit EPROM separately. It advances by two CPU addresses to remain on one byte lane, performs a 16-bit one's-complement/end-around-carry byte sum, and complements the result. The four calculated words match their trailers exactly:

```text
U36  $9E0E       U44  $DECD
U37  $9DEC       U45  $D114
```

This is not a conventional CRC. It is nevertheless fully recovered and could be reproduced when preserving or deliberately modifying an image.

### From native self-test to the application

After the low-level tests, the runtime establishes the Pascal outer block through the entry point at `$002102`. A header value of `$5800` reserves 22,528 bytes of application working space, and the cooperative main loop begins.

The visible `SELF TEST OK` sequence belongs to a Pascal procedure at about `$00127A`, rather than to the earliest boot code. That procedure initializes higher-level receiver state and sets the DAC request to its neutral midpoint of `32768.0`. It can also report `TEST ROUTINES ENABLED`.

Boot tests three values at `$A603`, `$A605`, and `$A619`. A particular masked pattern selects a deeper native diagnostic which exercises the hardware blocks from `$A000` through `$A700`, as well as ROM and RAM. The code proves the path exists; without a schematic or measurements, the physical straps or signals that request it remain unknown.

This layered startup is typical of serious instrumentation: native code establishes electrical and memory integrity first, then the interpreted application performs the operator-facing self-test and state initialization.

## 3. The Pascal/P-code architecture

### Why the application ROM looked wrong

Once the EPROM byte lanes were interleaved, the reset and hardware code disassembled normally. Much of the application ROM did not. Repeated sequences such as `JSR $2144` provided the clue: `$2144` is not an ordinary application routine but an entry into a bytecode interpreter.

The 68000 return address becomes the P-code instruction pointer. Procedure metadata immediately following the call describes parameter storage and lexical level; a second entry at `$2164` also allocates local storage. The interpreter itself is centred around `$002000`.

This hybrid arrangement divides the firmware cleanly:

```text
native 68000                         Pascal-family P-code
-------------                        --------------------
reset and exceptions                 executive state machines
interrupt handlers                   tracking and NAV policy
ROM and RAM tests                    orbit and position algorithms
peripheral access                    timing and averaging policy
floating-point primitives            menus and reports
matrix and maths helpers             communications control
P-code interpreter                   service workflows
```

### Why it is Pascal

The strongest evidence is not the style of the bytecode but the handling of nested scope. Locations `$4000`, `$4004`, `$4008`, and `$400C` hold pointers to active frames at lexical levels 0 through 3. Procedure entry saves and replaces the relevant pointer; return restores it. Procedures at levels 1, 2, and 3 are all present.

That lexical-display mechanism is characteristic of compiled Pascal. Pascal strings, local-frame allocation, nested procedures, and the separation between imported linkage stubs and procedure bodies reinforce the identification.

About 382 procedure entries can be recognized—360 in the large ROM and 22 in the low ROM. Roughly 60 runs of absolute-jump stubs look like per-unit import tables. They show that the source was modular, although they do not prove the original source contained exactly 60 named units.

The compiler itself remains unidentified. Period Microware Pascal systems used a broadly similar mixture of P-code, nested scopes, native support, floating point, and I/O routines, but this image has its own bare-metal startup and interrupt architecture. It is not a conventional OS-9 system.

### A compact instruction set

The bytecode was designed to make common operations cheap in ROM space:

```text
$00          invalid instruction -> TRAP #5
$01          NOP
$02          execute an embedded native 68000 helper
$03 xx       extended primitive
$04-$07 xx   push a 10-bit unsigned constant
$08-$0F ...  block, reference, and address forms
$10-$17 xx   relative branch
$18-$1F xx   branch if false
$20-$3F      direct primitive operation
$40-$7F      push the integer 0..63
$80-$FF ...  lexical variable load/store/reference family
```

The variable family encodes lexical level and object size. Byte, word, longword, and eight-byte values can be loaded or stored. The primitive library covers 32-bit arithmetic and comparisons, eight-byte real arithmetic and comparisons, call and return, stack allocation, `CASE`, loop control, conversion, block comparison, string literals, and compact inline output.

Early analysis stopped short of a complete table because rare indexed and
reference forms have contextual operands that a linear decoder can mistake for
opcodes. The interpreter has since been decoded at the byte-field level. The
[opcode reference](reference/vm-opcodes.md) records every top-level family, all
49 primitive-table slots, the lexical-variable bit fields, loop forms, block
operations, and the contextual-zero deferred-store convention. Original
compiler mnemonic names and a few reserved edge cases remain unknown, but the
instruction boundaries used by compiler-generated code are now deterministic.

The decisive cross-check is the Trimble 4000SX runtime: its public low-ROM
image is byte-identical to this firmware from `$002000` through `$003FF7`,
apart from the final eight integrity bytes. The two products therefore share
the interpreter and numerical runtime, rather than merely using similar
Pascal VM designs.

### Native code where it matters

Opcode `$02` aligns the instruction stream, executes embedded 68000 code, and returns to P-code. At least 26 procedure paths use such an escape. Larger native services are also reached through linkage stubs.

The numerical library is particularly important. It supplies eight-byte floating-point arithmetic, square root, sine and cosine, fractional/modulo operations, and reusable 4×4 matrix functions. Constants and exponent/mantissa handling are consistent with an IEEE-754-like double representation, although every exceptional case has not been tested.

That library explains how a 68000-class processor without an identified FPU could carry out orbit propagation, geodesy, clock modelling, ionosphere correction, regression, and matrix inversion. The VM did not make the receiver mathematically primitive; it gave a large mathematical program a compact, structured home.

### The application as a recovered program

The outer Pascal environment begins through a call at `$00103A`, reserves `$5800` bytes, performs the application self-test, and enters a permanent cooperative loop. Imported procedure stubs around `$0FF2-$1034` dispatch to the major subsystems.

The most useful way to read this firmware is consequently not as thousands of anonymous 68000 instructions. It is closer to a lost Pascal application with a surviving runtime. Procedure names have to be reconstructed from their inputs, outputs, constants, strings, and callers, but the original design boundaries remain surprisingly visible.

## 4. Executive and communications

### A small executive rather than a full operating system

There is no evidence that the 8400 runs a conventional RTOS. The application instead uses a cooperative executive: interrupts capture urgent hardware events and move bytes or state, while a Pascal loop repeatedly advances the slower receiver, navigation, timing, display, and communications tasks.

```text
hardware event
      |
      v
short native interrupt handler
      |
      v
RAM flag, buffer, count, or captured measurement
      |
      v
next pass through the Pascal executive
      |
      v
advance the relevant subsystem state machine
```

The outer block calls a set of subsystem entry points through linkage stubs near `$0FF2-$1034`. Conditional flags determine whether the next pass performs initial acquisition, normal receiver service, position work, time work, display work, or communications processing. This arrangement keeps hard timing out of the interpreter without forcing the whole application into assembly.

### Interrupts reveal the real-time boundaries

The MC68901 supplies interrupt control, event-count timers, GPIO, and one serial channel. Its five register-select lines appear inverted in the CPU address map:

```text
CPU address = $C001 + 2 × (31 - MC68901 register number)
```

That makes `$C011` the USART data register, `$C015` receiver status, `$C017` USART control, `$C029` the vector register, and `$C03F` GPIP. This corrected an early attempt to name the registers in ascending order.

The firmware programs vector base `$40`. The active assignments show what FTS considered time-critical:

| Vector | Handler   | Recovered use                                |
| ------ | --------- | -------------------------------------------- |
| `$43`  | `$000D3E` | GPIP3 timing/control event                   |
| `$47`  | `$000C12` | GPIP5, external serial-interface interrupt   |
| `$48`  | `$00089C` | Timer B, periodic receiver and clock service |
| `$4A`  | `$000D4E` | MFP transmitter-buffer empty                 |
| `$4C`  | `$000D7C` | MFP receiver-buffer full                     |
| `$4E`  | `$000EF8` | GPIP6, TMS9914A GPIB interrupt               |

The remaining vectors in that range use a default handler. Timer A and Timer B operate in event-count mode rather than simply dividing the processor clock, which fits an instrument organized around external timing edges.

Timer B maintains a hierarchy of software time fields and periodically copies them to hardware around `$A500`. That makes the `$A5xx` block a clock/time interface with high confidence, although its full register naming remains open.

### Two RS-232 ports and IEEE-488

The rear DB-25s are not implemented by two interchangeable UARTs. Further tracing assigns them confidently:

| Rear port      | Hardware path            | CPU addresses                        | Baud-clock source |
| -------------- | ------------------------ | ------------------------------------ | ----------------- |
| `RS232 Port 1` | MC6850-compatible ACIA   | `$A101` data; `$A103` status/control | MC68901 Timer C   |
| `RS232 Port 2` | MC68901's built-in USART | `$C011` data; `$C013/$C015/$C017`    | MC68901 Timer D   |

The Port 1 identification is particularly strong. When its parameters change, firmware first sets the low two bits of the saved control byte and writes it to `$A103`, then writes the actual configuration. On an MC6850, `CR1:CR0 = 11` is the master-reset command. The same code tests bit 1 as transmitter-data-register empty and bit 0 as receiver-data-register full, then reads or writes `$A101` exactly as an ACIA data register.

Port 2 uses the 68901 USART directly: the transmit and receive paths wait on bit 7 in `$C013` and `$C015`, respectively, then transfer a byte through `$C011`. Its framing and baud settings are programmed independently from Port 1. Timer C's data register at `$C01D` is adjusted with the Port 1 ACIA configuration; Timer D at `$C01B` is adjusted with the Port 2 USART configuration. The 68901 therefore provides both its own serial port and the baud clock for the separate ACIA.

The boot values for Timer C and D are both eight, so both ports start at the same rate. If the MFP were driven from the conventional 2.4576 MHz serial clock, the arithmetic would imply a 4800-baud default and a likely 300–9600 baud selection range. That clock has not been proved, so the actual baud-rate table remains an informed hypothesis rather than a confirmed specification.

Both ports use RAM buffering and interrupt-driven producer/consumer state during normal operation. Port 2 uses the MFP's transmit-buffer-empty and receive-buffer-full vectors, `$4A` and `$4C`. Port 1's ACIA interrupt enters through MFP GPIP5 at vector `$47`.

The front-panel menus expose independent framing choices for both connectors:

```text
7-EVEN-2   7-ODD-2    7-EVEN-1   7-ODD-1
8-NONE-2   8-NONE-1   8-EVEN-1   8-ODD-1
```

The firmware also saves the Port 1 control byte and sometimes ORs in `$20` before rewriting `$A103`. That is in the MC6850's RTS/transmit-control field, showing that Port 1 has software-controlled handshake or control-line behavior. Whether FTS uses it as conventional RTS/CTS flow control or for another purpose remains unresolved.

Both ports can report `PORT 1 NOT READY` or `PORT 2 NOT READY`. The machine-readable interface supports the native eight-byte real representation and a compact six-byte form that has not yet been decoded.

The TMS9914A GPIB controller occupies odd addresses `$A001-$A00F`. Its setup includes configurable addressing and `TALK_ONLY`, and GPIP6 carries its interrupt into the MC68901. The ROM's `GPIB NOT READY` path and the board photograph independently confirm the identification.

### One formatting system, several destinations

The reporting system is shared rather than duplicated for each interface. A native character routine near `$09716A` fans output to whichever serial ports or GPIB destinations are enabled. Helpers emit common punctuation, spaces, and line endings. `TRAP #1`, handled at `$0971BC`, sends inline literal text efficiently without making the interpreter construct it character by character.

The same application produces tracking reports, navigation-data dumps, position and DOP results, time/frequency reports, scheduled-observation output, and configuration responses. Output destinations can be selected independently among GPIB, Port 1, and Port 2.

The ROM also contains `PORT1 CONTROL`, `CHARS ENTERED`, `CONTROL CHARS`, `CC`, `ID`, `BAUD`, and `TIMER` screens. This confirms a higher-level serial-input and remote-control layer, but not yet whether its control characters are XON/XOFF, command framing, or both. A complete external command grammar is still missing.

### The front panel as another task

The display is a 32-byte buffer at `$17FE0-$17FFF`, naturally arranged as two rows of 16 characters. Native routines clear the buffer and transfer it to the physical display. Bit 7 of a character is removed during transfer and converted into an attribute signal; highlight, blink, reverse video, or cursor indication are possible, but the exact visible effect is not established.

Keyboard handling, menu navigation, password/lock functions, the beeper, status screens, and hidden service pages run as ordinary cooperative application work. This is another benefit of the split architecture: only the electrical transfer needs native code, while almost all user interaction remains in Pascal.

## 5. The GPS navigation system

### A complete NAV database in RAM

The ROM's reporting strings first gave away the scale of the navigation system. Names such as `IODC`, `IODE`, `SQRT A`, `DELTA N`, `OMEGA`, `ALPHA`, `BETA`, and the leap-second fields are not a loose diagnostic vocabulary. Following their report routines leads to arrays holding the decoded GPS broadcast message.

Many satellite arrays use a stride of `$108` bytes—33 slots of eight bytes—providing indexed storage across the GPS satellite set. The full ephemeris records include:

```text
week and Z-count                 IODC, IODE, TOC, TOE
AF0, AF1, AF2, TGD              sqrt(A), eccentricity, M0, delta-n
i0, IDOT, omega                 Omega0, Omega-dot
CRS, CRC, CUS, CUC, CIS, CIC   health, alert, fit and validity state
```

Almanac data is stored separately, with the reduced orbital parameter set expected from the broadcast almanac. This supports the front-panel states `USING EPHEMERIS`, `USING ALMANAC`, and `ORBIT DATA NOT AVAILABLE`: the application can use precise data when available and fall back to the almanac for prediction.

A shared record holds the eight Klobuchar coefficients, the GPS-to-UTC polynomial, current and future leap seconds, and their reference week/day fields. The firmware even maintains a checksum over this database.

### Turning broadcast parameters into a satellite position

The ephemeris routine around `$09C816` follows the GPS broadcast model closely enough to recognize equation by equation:

```text
time and broadcast ephemeris
           |
           v
wrap time from TOE to +/- half a GPS week
           |
           v
mean motion and mean anomaly
           |
           v
iterative solution of Kepler's equation
           |
           v
true anomaly and argument of latitude
           |
           v
harmonic corrections to latitude, radius, and inclination
           |
           v
orbital-plane coordinates -> Earth-fixed X, Y, Z
           |
           `-> optional velocity X, Y, Z
```

Time from ephemeris reference is wrapped at ±302,400 seconds using the 604,800-second GPS week. The routine solves `E - e sin(E) = M`, applies the six harmonic correction terms, includes inclination and node rates, and rotates the result into Earth-centred, Earth-fixed coordinates. A separate routine near `$09C372` performs the simpler almanac propagation.

The constants authenticate the interpretation. `6378137` and `0.00669437999...` are the WGS-84 semi-major axis and eccentricity squared. The stored Earth-rotation value, `2.32115234247e-5` semicircles per second, becomes approximately `7.292115147e-5` radians per second after multiplication by pi. Angles are commonly retained in semicircles, matching the GPS broadcast convention.

### Satellite time is modelled as carefully as position

The clock-correction path near `$09AFFE` evaluates the broadcast polynomial:

```text
AF0 + AF1 × dt + AF2 × dt^2
```

It also incorporates `TGD` and the relativistic eccentric-orbit correction:

```text
delta_tr = F × e × sqrt(A) × sin(E)
F = -4.442809305e-10
```

The Klobuchar routine near `$09AE68` is similarly recognizable. It computes the ionospheric pierce point, clamps its latitude to ±0.416 semicircles, derives local time, forms amplitude and period from `ALPHA0..3` and `BETA0..3`, and uses the standard daytime polynomial. One apparent period-floor comparison remains worth checking against the last undecoded VM comparison details before claiming that FTS deliberately departed from the usual model.

Together, these routines show that the observation model includes geometric range, satellite clock drift, relativistic correction, group delay, ionosphere, and Earth rotation during signal flight.

### Solving the receiver position

The position procedure near `$098088` uses four satellites to solve four unknowns: receiver latitude, longitude, height, and clock/range bias. It begins with a current geodetic estimate, converts it to WGS-84 Earth-centred coordinates, predicts ranges to the selected satellites, and corrects for Earth rotation during each signal's transit time.

For every satellite it forms a residual and one row of a 4×4 geometry matrix. The fourth element is one, corresponding to receiver clock bias. A reusable matrix package then solves the correction:

```text
observed ranges - modelled ranges -> residual vector r
geometry partial derivatives      -> matrix H

position/clock correction         -> delta-x = inverse(H) × r
```

The solution updates latitude, longitude, height, and clock bias, then iterates. This is a proper GPS navigation solution rather than a rough geometric shortcut.

The supporting numerical routines include matrix-vector multiplication around `$0035FE`, 4×4 multiplication around `$003766`, inversion around `$003AA2`, and transpose around `$003E8A`. Their reuse is another sign that the original software was designed as a structured numerical application.

### Geometry quality and stationary operation

After solving the fix, the program forms the usual covariance-like geometry matrix:

```text
Q = inverse(transpose(H) × H)
```

and derives PDOP, HDOP, VDOP, and TDOP from its diagonal terms. The values at `$42FD`, `$4305`, `$430D`, and `$4315` are independently confirmed by the display code. PDOP is compared with the user's acceptance criterion, so satellite geometry affects whether the receiver trusts a fix.

Accepted positions feed sums and squared sums of latitude, longitude, and altitude. For a stationary timing receiver this averaging is valuable: a better antenna-position estimate reduces the coupling between position error and the clock solution.

The scheduler also advances some predicted satellite events by 86,160 seconds—23 h 56 min—until they lie in the future. That is close to a sidereal day and strongly suggests reuse of daily satellite-geometry recurrence. The value and behavior are confirmed; the astronomical intent remains a high-confidence interpretation rather than a recovered FTS name.

### Week numbers, MJD, and a corrected rollover story

The receiver maintains an expanded GPS week and computes:

```text
MJD = GPS_week × 7 + day_of_week + 44244
```

`44244` is the Modified Julian Date of the GPS epoch, 6 January 1980. The calendar routine starts at the MJD epoch, 17 November 1858, and applies the full Gregorian divisible-by-4/100/400 leap-year rule.

More surprisingly, the 1987 NAV decoder anticipates the first ten-bit GPS week rollover:

```text
if received_week <= 453:
    expanded_week = received_week + 1024
else:
    expanded_week = received_week
```

Normal operation also increments the expanded week when seconds-of-week crosses 604,800. An early hypothesis blamed the main ten-bit week field for reported 1999 failures, but this ROM explicitly handles that transition. If an FTS 8400 revision did fail then, another firmware version or a narrower almanac/UTC week field is a better place to look.

## 6. The timing and frequency system

### From a GPS signal to a clock observation

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

### A timing system built around the C/A-code clock family

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

### Programming the output pulse to about 4 ns

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

### GPS, UTC, and OMEGA output time

The application can align its time/frequency outputs to GPS, UTC, or OMEGA. The actual selector is held at global offset `-$28F8`; a nearby three-state value once mistaken for this choice instead controls initialization and averaging progress.

For GPS, no UTC offset is applied. For UTC, the firmware evaluates the broadcast conversion polynomial using `A0`, `A1`, `T_ot`, and `WN_t`, including week wrapping and leap-second information. Cable delay and the configured user `TIME OFFSET` are then included before the fractional output phase is programmed.

OMEGA support is real at the application level—the menus and selector are explicit—but its complete physical input and processing path have not been reconstructed.

The clock update is deliberately conservative. A running average accepts nearby observations and restarts when a gross step is seen. The user can require a chosen number of acceptable averages before the clock is updated. The recovered pattern is:

```text
measure -> validate -> average -> test quality -> update output time
```

That behavior matters in a metrology instrument: a single noisy satellite observation should not move the generated timescale.

### Measuring an external reference

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

### Why the receiver fits a straight line

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

### The separate internal frequency loop

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

### The recovered timing architecture

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

## 7. Diagnostics and service modes

The 8400 has more than one definition of “self-test.” Its diagnostic design is layered, with each stage assuming a little more of the machine is working.

### Power-on integrity checks

The native reset path establishes the processor state and tests the foundations needed by the Pascal runtime. It clears and verifies both RAM banks, initializes the principal interfaces, and validates every EPROM independently.

The five-byte trailer on each EPROM records its socket number, the shared `$DBEB` set signature, and a 16-bit one's-complement checksum. Checking each byte lane separately catches both a damaged device and a ROM placed in the wrong socket. Only after these checks does boot construct the interpreted application environment.

The Pascal self-test then initializes higher-level receiver state and presents `SELF TEST OK`. It can also announce that test routines are enabled. Keeping low-level integrity checks native while leaving operator-facing setup in Pascal matches the architecture used elsewhere in the instrument.

### Strap-selected factory diagnostics

During boot, three masked inputs are tested:

```text
($A603 & $1E) == $1C
($A605 & $1E) == $1A
($A619 & $1E) == $1E
```

When all three match, the firmware enters a deeper native diagnostic. It exercises hardware blocks at `$A000`, `$A100`, `$A300`, `$A400`, `$A500`, `$A600`, and `$A700`, together with ROM and RAM.

The behavior is confirmed in code. Calling the inputs “factory straps” is a high-confidence interpretation: the board wiring or external test fixture that creates those values has not been traced.

### The hidden software-DAC page

A separate service path is embedded in the front-panel application. Two key-state bytes corresponding to ASCII `N` and `P` must remain asserted for five consecutive scheduler passes before a `SOFTWARE DAC` page is enabled. This looks more like a deliberately held key combination than a typed `NP` password, although the physical key labeling has not been verified.

The page allows the 16-bit DAC request to be viewed and altered using its bipolar ±5.000 engineering scale. Since the DAC probably controls an internal receiver-frequency element, direct adjustment could disturb lock or calibration. On surviving hardware, the original value should be recorded and the analog destination established before this mode is used.

The coincidence between the `N`/`P` key states and the firmware version label `NP 120` is interesting but not evidence that the initials have the same meaning.

## 8. What remains unresolved

The broad design is now understandable. Most remaining questions lie where software-visible values meet custom hardware, in edge-case runtime semantics, or in the provenance of the Pascal toolchain.

### Timing hardware

The `$A3xx` block is functionally identifiable as the time-interval counter and output-phase generator, but its exact register names, bit definitions, and electrical implementation are not known. A schematic or targeted measurements would help answer:

- What interpolation technique is used for TIC start and stop events?
- How is it calibrated, and what are its single-shot resolution and accuracy?
- Which registers hold coarse output phase, fine phase, status, and latch control?
- What physical path requires the fixed 400 ns correction?
- What circuit quantity is represented by the likely 26-bit, 240 ms accumulator?

The output firmware's nominal 4 ns fine count should not be used as a substitute for measured TIC performance.

### Receiver and analog control

The software proves that `$A200` is a 16-bit DAC driven by the internal carrier-frequency residual. It does not show which analog node receives the voltage. The leading hypothesis is an internal receiver oscillator or frequency-control element, but the following still need physical evidence:

- Does the service page's ±5.000 scale correspond directly to volts?
- What are the DAC polarity, gain, and neutral operating point in the circuit?
- How are acquisition, code tracking, and the Costas/carrier loop divided between software and custom logic?
- What are the roles of the remaining `$A4xx-$A7xx` registers?

### Protocols and historical behavior

The serial hardware is now largely identified: Port 1 is the MC6850-compatible ACIA driven from Timer C and entering through GPIP5; Port 2 is the MC68901 USART driven from Timer D. What remains is the protocol layer. The full command grammar, the intended meaning of `CONTROL CHARS`, the compact six-byte real-number format, the actual MFP clock and baud-rate table, and the exact purpose of Port 1's RTS/transmit-control manipulation are still unknown.

OMEGA appears as a genuine time-source selection, but its complete signal and software path remains to be traced.

This ROM expands the principal ten-bit GPS week across the 1999 rollover. Reports of rollover failure may concern another software revision or one of the narrower week fields in almanac or UTC/leap-second data. Reproducing the behavior with archived NAV data would distinguish those possibilities.

### VM and provenance

The P-code byte fields, primitive dispatch table, indexed operations, and
reference/update convention are now decoded and recorded in the
[opcode reference](reference/vm-opcodes.md). The remaining VM questions concern
original mnemonic names, reserved size combinations, unusual block lengths,
and floating-point edge behavior rather than instruction boundaries.

The exact Pascal compiler/runtime lineage is also unknown. Similarities to period Microware technology are suggestive, not conclusive. No evidence yet expands the `NP` and `SP` version labels.

Finally, the satellite scheduler advances certain events by 86,160 seconds, close to a sidereal day. Reuse of repeating sky geometry is a strong interpretation, but an original module name or design note would turn that inference into a firm conclusion.

These are good targets for the next round of work because each has a discriminating test: trace a DAC net, capture timing-register behavior, complete a rare VM operand, replay rollover NAV data, or find an original schematic/manual. None requires reopening the already coherent high-level architecture.

## Appendix A. Address map

This is the address-level reference behind the narrative chapters. Names are reconstructed from behavior; they are not claimed original symbols. “Function confirmed” means the software-visible role is clear even when the physical register name is not.

### Main memory and I/O regions

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

### Native runtime and application routines

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

### Important RAM locations

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

### Timing hardware landmarks

| Address             | Recovered meaning             | Confidence         |
| ------------------: | ----------------------------- | ------------------ |
| `$A200`             | Physical 16-bit DAC write     | Confirmed          |
| `$A30B/$A30D`       | TIC fine-interpolator samples | High confidence    |
| `$A311/$A317/$A319` | Output-phase parameters       | High confidence    |
| `$A313`             | Timing latch/control strobes  | Function confirmed |
| `$A315`             | Timing status                 | High confidence    |
| `$A31B/$A31D/$A31F` | Three-byte TIC coarse count   | High confidence    |

### Serial hardware landmarks

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

## Appendix B. Key constants

Constants were among the most reliable clues in the ROM. Several apparently arbitrary values become exact GPS, geodetic, or timing quantities when the firmware's units are understood.

### GPS and geodesy

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

### Receiver and timing scales

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

### Control and scheduling

| Constant | Interpretation                                            | Confidence                                  |
| -------: | --------------------------------------------------------- | ------------------------------------------- |
| `290.5`  | Internal-loop gain in DAC codes per hertz of residual     | Confirmed in software                       |
| `32768`  | Neutral/default 16-bit DAC code                           | Confirmed                                   |
| `86160`  | Satellite-event recurrence increment, near a sidereal day | Value confirmed; purpose is high confidence |

## Appendix C. Recovered data structures

The original Pascal declarations are lost, but report routines print named GPS fields immediately after loading their corresponding globals. Combined with algorithmic use and the recurring `$108`-byte satellite stride, this recovers much of the logical database layout.

Offsets below are Pascal global or record bases, not absolute CPU addresses. Satellite arrays commonly contain 33 eight-byte slots.

### Broadcast ephemeris

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

### Almanac

| Offset   | Field         | Offset   | Field        |
| -------: | ------------- | -------: | ------------ |
| `-$1C43` | Almanac week  | `-$1D4B` | Almanac time |
| `-$1F5B` | TOA           | `-$2273` | sqrt(A)      |
| `-$1E53` | eccentricity  | `-$258B` | M0           |
| `-$2063` | i0            | `-$2483` | omega        |
| `-$237B` | Omega0        | `-$216B` | Omega-dot    |
| `-$25AD` | validity flag |          |              |

### Ionosphere and UTC

| Offset           | Field                      | Offset           | Field        |
| ---------------: | -------------------------- | ---------------: | ------------ |
| `-$284B..-$2863` | ALPHA0..ALPHA3             | `-$286B..-$2883` | BETA0..BETA3 |
| `-$288B/-$2893`  | UTC A0/A1                  | `-$28AB`         | T_ot         |
| `-$289B/-$28A3`  | current/future leap second | `-$28AC..-$28AE` | WNt/WNLSF/DN |

This layout is sufficient to follow the data from NAV-message decoding into orbit propagation, satellite-clock correction, ionosphere modelling, UTC conversion, and reporting. Exact Pascal type declarations and a few administrative fields remain to be reconstructed.

---

The recovered design is coherent in a way that was not obvious from the raw ROMs. GPS-derived clock harmonics connect code tracking, time measurement, and pulse generation; Pascal P-code carries the large navigation and instrument application; native 68000 code handles deterministic service and expensive primitives; and the external-reference comparison remains separate from internal receiver-frequency control. That architecture, rather than any individual opcode, is the central result of the reverse engineering.
