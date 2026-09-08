# Hardware, Memory, and Boot

[Project index](../README.md) · [Complete edition](How%20the%20FTS%208400%20Worked%20-%20Complete.md)

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

## Boot, integrity checks, and self-test

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
