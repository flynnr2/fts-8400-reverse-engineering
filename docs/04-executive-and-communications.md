# Executive and Communications

[Project index](../README.md) · [Complete edition](How%20the%20FTS%208400%20Worked%20-%20Complete.md)

## A small executive rather than a full operating system

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

## Interrupts reveal the real-time boundaries

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

## Two serial paths and IEEE-488

One serial channel uses the MC68901 USART. A second interface at `$A101/$A103` behaves like a Motorola 6850-compatible ACIA: its status bits are tested as interrupt, transmitter-ready, receiver-ready, and error/control conditions, while the companion address carries data. The code proves there are two distinct serial paths, but it does not yet tie each device confidently to the physical `PORT 1` and `PORT 2` labels.

Both paths use RAM buffering and interrupt-driven producer/consumer state. The firmware can report `PORT 1 NOT READY` and `PORT 2 NOT READY`, and the menu offers conventional framing choices. It also supports two binary floating-point transfer formats: the native eight-byte representation and a compact six-byte form that has not yet been decoded.

The TMS9914A GPIB controller occupies odd addresses `$A001-$A00F`. Its setup includes configurable addressing and `TALK_ONLY`, and GPIP6 carries its interrupt into the MC68901. The ROM's `GPIB NOT READY` path and the board photograph independently confirm the identification.

## One formatting system, several destinations

The reporting system is shared rather than duplicated for each interface. A native character routine near `$09716A` fans output to whichever serial ports or GPIB destinations are enabled. Helpers emit common punctuation, spaces, and line endings. `TRAP #1`, handled at `$0971BC`, sends inline literal text efficiently without making the interpreter construct it character by character.

The same application produces tracking reports, navigation-data dumps, position and DOP results, time/frequency reports, scheduled-observation output, and configuration responses. A complete external command grammar is still missing, but the transport and formatting architecture are clear.

## The front panel as another task

The display is a 32-byte buffer at `$17FE0-$17FFF`, naturally arranged as two rows of 16 characters. Native routines clear the buffer and transfer it to the physical display. Bit 7 of a character is removed during transfer and converted into an attribute signal; highlight, blink, reverse video, or cursor indication are possible, but the exact visible effect is not established.

Keyboard handling, menu navigation, password/lock functions, the beeper, status screens, and hidden service pages run as ordinary cooperative application work. This is another benefit of the split architecture: only the electrical transfer needs native code, while almost all user interaction remains in Pascal.
