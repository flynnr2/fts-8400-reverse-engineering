# Hardware, Memory, and Boot

[Project index](../README.md) · [Complete edition](How%20the%20FTS%208400%20Worked%20-%20Complete.md)

## Reassembling the ROM image

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

## A memory map shaped by the board

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

## What happens at reset

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

## The EPROM trailers are service metadata

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

## From native self-test to the application

After the low-level tests, the runtime establishes the Pascal outer block through the entry point at `$002102`. A header value of `$5800` reserves 22,528 bytes of application working space, and the cooperative main loop begins.

The visible `SELF TEST OK` sequence belongs to a Pascal procedure at about `$00127A`, rather than to the earliest boot code. That procedure initializes higher-level receiver state and sets the DAC request to its neutral midpoint of `32768.0`. It can also report `TEST ROUTINES ENABLED`.

Boot tests three values at `$A603`, `$A605`, and `$A619`. A particular masked pattern selects a deeper native diagnostic which exercises the hardware blocks from `$A000` through `$A700`, as well as ROM and RAM. The code proves the path exists; without a schematic or measurements, the physical straps or signals that request it remain unknown.

This layered startup is typical of serious instrumentation: native code establishes electrical and memory integrity first, then the interpreted application performs the operator-facing self-test and state initialization.
