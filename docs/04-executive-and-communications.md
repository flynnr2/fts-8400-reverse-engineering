# Executive and Communications

[Project index](../README.md) · [Complete edition](How%20the%20FTS%208400%20Worked%20-%20Complete.md)

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

## Communications, display, and operator interface

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
