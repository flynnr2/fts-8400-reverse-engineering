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

## Two RS-232 ports and IEEE-488

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

## One formatting system, several destinations

The reporting system is shared rather than duplicated for each interface. A native character routine near `$09716A` fans output to whichever serial ports or GPIB destinations are enabled. Helpers emit common punctuation, spaces, and line endings. `TRAP #1`, handled at `$0971BC`, sends inline literal text efficiently without making the interpreter construct it character by character.

The same application produces tracking reports, navigation-data dumps, position and DOP results, time/frequency reports, scheduled-observation output, and configuration responses. Output destinations can be selected independently among GPIB, Port 1, and Port 2.

The ROM also contains `PORT1 CONTROL`, `CHARS ENTERED`, `CONTROL CHARS`, `CC`, `ID`, `BAUD`, and `TIMER` screens. This confirms a higher-level serial-input and remote-control layer, but not yet whether its control characters are XON/XOFF, command framing, or both. A complete external command grammar is still missing.

## The front panel as another task

The display is a 32-byte buffer at `$17FE0-$17FFF`, naturally arranged as two rows of 16 characters. Native routines clear the buffer and transfer it to the physical display. Bit 7 of a character is removed during transfer and converted into an attribute signal; highlight, blink, reverse video, or cursor indication are possible, but the exact visible effect is not established.

Keyboard handling, menu navigation, password/lock functions, the beeper, status screens, and hidden service pages run as ordinary cooperative application work. This is another benefit of the split architecture: only the electrical transfer needs native code, while almost all user interaction remains in Pascal.
