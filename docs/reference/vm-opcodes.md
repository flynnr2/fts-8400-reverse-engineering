# Pascal VM Opcode Reference

[Project index](../../README.md) · [Architecture](../03-pascal-pcode-architecture.md) · [Address map](address-map.md)

This reference describes the bytecode interpreter at `$002000`. Names are
reconstructed mnemonics, not original symbols. Unless noted otherwise, the
semantics below are confirmed directly from the 68000 handlers.

## Why the Trimble 4000SX is a valid cross-reference

The public [Trimble 4000SX VM notes](https://beefchicken.com/gps/trimble/4000/sxdeepdive/virtualmachine)
list the same dispatcher and primitive-handler addresses found in the FTS
image. A byte comparison makes the relationship decisive: the FTS low ROM and
the site's [4000SX `ROM0.bin`](https://beefchicken.com/gps/trimble/4000/sxdeepdive/ROM0.bin)
are identical from `$002000` through `$003FF7`. Only the final eight bytes,
which include image-specific integrity data, differ in the upper half of the
16 KiB bank.

The complete VM, floating-point package, and the native routines following it
are therefore shared runtime code, not merely similar designs. The FTS merged
low-ROM SHA-256 is
`6c83f23ae27b56f331e9bf35a87ff7a9baf30427ab8a66921c30eb3fb98e5deb`;
the downloaded 4000SX image is
`940a08c8ad85f9ae2857504d69959a0caeda02f5336e94a307dffa492f61d2af`.

The cross-reference was useful but incomplete. In particular, its `$08-$0F`
range is not a jump family, `$03` is an extended-opcode prefix, and its
disassembler reverses the names of the four unsigned long comparisons. The
tables below follow the observed 68000 comparisons and an `a b -> result`
stack convention, where `b` is at the top of the stack.

## Machine model and notation

- `A5` is the virtual program counter (VPC).
- `A6` is the downward-growing data-stack pointer.
- `A7` is the native 68000 call/control stack.
- `A4` is the effective-address register used by variable operations.
- Long integers and addresses occupy four stack bytes; reals occupy eight.
- Boolean true is `$FFFFFFFF`; false is `$00000000`. Conditional branch tests
  only the low byte, so any value with a nonzero low byte is true.
- Multi-byte bytecode operands are big-endian.
- `F` below means the runtime's eight-byte real format. Its ordinary finite
  values use the IEEE-754 binary64 layout, although the runtime predates and
  does not necessarily implement all modern exceptional-value behavior.

## Top-level decoding

| Encoding            | Reconstructed operation | Operand / effect                                                                                                                                                            |
| ------------------- | ----------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `$00`               | `INVALID`               | Raises `TRAP #5` when decoded as an opcode. It also has a contextual role after certain reads; see below.                                                                   |
| `$01`               | `NOP`                   | No effect.                                                                                                                                                                  |
| `$02`               | `INLINE.68K`            | Align VPC with `(VPC+1)&~1`, then execute 68000 code there. The helper must set `A5` to the following bytecode before its terminating `RTS`.                                |
| `$03 zz`            | `EXT zz`                | Dispatch primitive-table slot `$00-$30`; values above `$30` raise `TRAP #5`. Slots `$00-$1F` alias `$20-$3F`; `$20-$30` are the actual extensions.                          |
| `$04-$07 xx`        | `PUSH.U10`              | Push unsigned `((opcode & 3)<<8) \| xx`.                                                                                                                                    |
| `$08-$0B dddd nnnn` | `READ.BLOCK.l`          | Copy `nnnn` bytes from lexical level `l=opcode&3`, displacement `dddd`, to the stack. Values are copied as words, so the demonstrated form requires a positive even length. |
| `$0C-$0F dddd nnnn` | `WRITE.BLOCK.l`         | Copy `nnnn` bytes from the stack to lexical level `l=opcode&3`, displacement `dddd`.                                                                                        |
| `$10-$17 yy`        | `BR s11`                | Add the signed 11-bit displacement `((opcode&7)<<8)\|yy` to VPC after the operand.                                                                                          |
| `$18-$1F yy`        | `BRZ s11`               | Pop a long flag and take the same relative branch when its low byte is zero.                                                                                                |
| `$20-$3F`           | direct primitives       | See the next table.                                                                                                                                                         |
| `$40-$7F`           | `PUSH.U6`               | Push `opcode & $3F`.                                                                                                                                                        |
| `$80-$FF`           | lexical variable family | Compact read/write, indexed, size, scope, and displacement fields; decoded below.                                                                                           |

The 11-bit branch displacement is sign-extended. A displacement of zero points
to the instruction immediately after its operand.

## Direct primitives `$20-$3F`

| Opcode                  | Handler | Reconstructed mnemonic | Operands and effect                                                                                                                                     |
| ----------------------: | ------: | ---------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `$20`                   | `$279A` | `ADD.F`                | `a:F b:F -> a+b`                                                                                                                                        |
| `$21`                   | `$2792` | `SUB.F`                | `a:F b:F -> a-b`                                                                                                                                        |
| `$22`                   | `$28D6` | `MUL.F`                | `a:F b:F -> a*b`                                                                                                                                        |
| `$23`                   | `$2A1C` | `DIV.F`                | `a:F b:F -> a/b`                                                                                                                                        |
| `$24`                   | `$24E4` | `GT.F`                 | `a:F b:F -> a>b`                                                                                                                                        |
| `$25`                   | `$24F0` | `GTE.F`                | `a:F b:F -> a>=b`                                                                                                                                       |
| `$26`                   | `$24D8` | `EQ.F`                 | `a:F b:F -> a==b`                                                                                                                                       |
| `$27`                   | `$2566` | `ADD.L`                | `a:L b:L -> (a+b) mod 2^32`                                                                                                                             |
| `$28`                   | `$256C` | `SUB.L`                | `a:L b:L -> (a-b) mod 2^32`                                                                                                                             |
| `$29`                   | `$2572` | `MUL.L`                | Unsigned multiply; retain the low 32 bits.                                                                                                              |
| `$2A`                   | `$249E` | `EQ.L`                 | `a:L b:L -> a==b`                                                                                                                                       |
| `$2B`                   | `$24A6` | `NEQ.L`                | `a:L b:L -> a!=b`                                                                                                                                       |
| `$2C`                   | `$2554` | `AND.L`                | Bitwise AND.                                                                                                                                            |
| `$2D`                   | `$255A` | `OR.L`                 | Bitwise OR.                                                                                                                                             |
| `$2E`                   | `$2550` | `NOT.L`                | Bitwise complement in place.                                                                                                                            |
| `$2F`                   | `$25B4` | `MOD.L`                | Unsigned `a % b`. A zero divisor returns zero rather than trapping.                                                                                     |
| `$30 ww`                | `$212C` | `BSR.68K`              | Read unsigned 16-bit `ww`, then call native address `VPC-ww`; resume P-code after the call.                                                             |
| `$31 bb`                | `$2444` | `SUBSP.B`              | Allocate `bb` stack bytes (`A6 -= bb`).                                                                                                                 |
| `$32 bb`                | `$2458` | `ADDSP.B`              | Release `bb` stack bytes (`A6 += bb`).                                                                                                                  |
| `$33`                   | `$2300` | `STORE.INDIRECT`       | Pop an address/size descriptor, then store and pop its value. Descriptors are produced by the contextual-zero read form described below.                |
| `$34`                   | `$213C` | `RETURN`               | Restore the caller's lexical-display entry and return through the native control stack to the saved VPC.                                                |
| `$35 ...`               | `$2B48` | `SWITCH`               | Pop a selector; compare its low byte against case keys in variable-length records.                                                                      |
| `$36 dd`                | `$2362` | `STORE.KEEP dd`        | Store the top byte, word, or long to the direct lexical address encoded by `dd`, without popping it. Used to initialize loop-control variables.         |
| `$37 dd bb`             | `$23D2` | `FOR.UP.NEXT`          | Increment variable `dd`; if it did not wrap to zero, push the new value and branch backward by unsigned 16-bit `bb`. Otherwise skip `bb`.               |
| `$38 tt`                | `$237A` | `FOR.UP.TEST`          | Pop `current, limit`; continue when unsigned `current<=limit`, otherwise skip forward by `tt&$3FFF`. Top bits of `tt` select byte/word/long comparison. |
| `$39 dd bb`             | `$23FC` | `FOR.DOWN.NEXT`        | Decrement variable `dd`; if it did not borrow, push the new value and branch backward by unsigned 16-bit `bb`. Otherwise skip `bb`.                     |
| `$3A tt`                | `$23A6` | `FOR.DOWN.TEST`        | Pop `current, limit`; continue when unsigned `current>=limit`, otherwise skip forward by `tt&$3FFF`.                                                    |
| `$3B ffffffff ffffffff` | `$232A` | `PUSH.F`               | Push an inline eight-byte real bit pattern.                                                                                                             |
| `$3C wwww`              | `$2356` | `PUSH.U16`             | Zero-extend and push an inline 16-bit value as a long.                                                                                                  |
| `$3D llllllll`          | `$2348` | `PUSH.L`               | Push an inline 32-bit value.                                                                                                                            |
| `$3E`                   | `$2340` | `PUSH.F0`              | Push eight zero bytes: real `+0.0`.                                                                                                                     |
| `$3F nn bytes...`       | `$2B32` | `DISP`                 | Pop a display-buffer offset and copy `nn` inline bytes to `$017FE0+offset`.                                                                             |

The `dd` operand used by `$36`, `$37`, and `$39` has this layout:

```text
15           14 13       12 11                         0
+---------------+-----------+----------------------------+
| size          | level     | 12-bit subtractive offset  |
+---------------+-----------+----------------------------+
```

`size=00` selects byte, `01` word, and `10` long; `11` follows the long path.
`level` selects one of the four display pointers at `$4000,$4004,$4008,$400C`.
The effective address is `display[level]-offset`.

For `$38/$3A`, `tt` is a 16-bit test-and-skip operand. Its top bits select byte
(`00`), word (`01`), or long (`10`) unsigned comparison, and its low 14 bits
are the forward displacement. The observed compiler sequence is
`initial; STORE.KEEP; limit; FOR.*.TEST; body; FOR.*.NEXT`.

### `SWITCH` records

After `$35`, each non-default record begins with a 16-bit header:

```text
15                         12 11                         0
+----------------------------+----------------------------+
| number of one-byte keys    | record length after header |
+----------------------------+----------------------------+
```

The keys immediately follow the header, followed by that case's bytecode. The
record length is measured from the first key to the next record header. On a
match the VM skips any remaining keys and executes the case body. On no match
it jumps to the next header. A one-byte `$00` header terminates the search and
execution continues immediately after it as the default body. This agrees with
the worked example in the 4000SX notes.

## Extended primitives `$03 $20-$30`

| Encoding       | Handler | Reconstructed mnemonic | Operands and effect                                                                                                                          |
| -------------: | ------: | ---------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| `$03 $20`      | `$2788` | `NEG.F`                | If the high 16 bits are nonzero, toggle the binary64 sign bit; positive zero remains positive.                                               |
| `$03 $21`      | `$24EA` | `LT.F`                 | `a:F b:F -> a<b`                                                                                                                             |
| `$03 $22`      | `$24F6` | `LTE.F`                | `a:F b:F -> a<=b`                                                                                                                            |
| `$03 $23`      | `$24DE` | `NEQ.F`                | `a:F b:F -> a!=b`                                                                                                                            |
| `$03 $24`      | `$25AE` | `DIV.L`                | Unsigned quotient `a / b`. A zero divisor returns `$FFFFFFFF` rather than trapping.                                                          |
| `$03 $25`      | `$24AE` | `GT.L`                 | Unsigned `a>b`. The old 4000SX disassembler calls this `LT.L`; the handler's operand order proves the reverse.                               |
| `$03 $26`      | `$24B6` | `LT.L`                 | Unsigned `a<b`; called `GT.L` by the old disassembler.                                                                                       |
| `$03 $27`      | `$24BE` | `GTE.L`                | Unsigned `a>=b`; called `LTE.L` by the old disassembler.                                                                                     |
| `$03 $28`      | `$24C6` | `LTE.L`                | Unsigned `a<=b`; called `GTE.L` by the old disassembler.                                                                                     |
| `$03 $29`      | `$25EC` | `F2U.L`                | Convert a nonnegative real to unsigned long, truncating toward zero; saturate negative values to zero and values above range to `$FFFFFFFF`. |
| `$03 $2A`      | `$263C` | `U.L2F`                | Convert an unsigned long to an eight-byte real.                                                                                              |
| `$03 $2B nnnn` | `$246C` | `EQ.BLOCK`             | Compare two adjacent `nnnn`-byte stack blocks, discard both, and push equality. Comparison proceeds wordwise.                                |
| `$03 $2C nnnn` | `$2498` | `NEQ.BLOCK`            | As above, with inverted result.                                                                                                              |
| `$03 $2D wwww` | `$244C` | `SUBSP.W`              | Allocate an unsigned 16-bit number of stack bytes.                                                                                           |
| `$03 $2E wwww` | `$2460` | `ADDSP.W`              | Release an unsigned 16-bit number of stack bytes.                                                                                            |
| `$03 $2F`      | `$2560` | `XOR.L`                | Bitwise exclusive OR.                                                                                                                        |
| `$03 $30 ...`  | `$219A` | `TRAP1`                | Invoke `TRAP #1`. In this firmware the trap consumes an inline length-prefixed string and emits it through the shared output path.           |

The `$03 $00-$1F` encodings dispatch to the same handlers as `$20-$3F`.
They are valid but waste one byte and have not been seen as the compiler's
normal encoding.

## Lexical variable family `$80-$FF`

Every high-bit opcode is decoded by one common path:

```text
 7   6       5       4       3 2       1 0
+---+-------+-------+-------+-----------+-------+
| 1 | disp  | index | write | level     | size  |
+---+-------+-------+-------+-----------+-------+
```

| Field   | Meaning                                                                                                                                                                            |
| ------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `disp`  | `0`: one-byte displacement (`$80-$BF`); `1`: two-byte displacement (`$C0-$FF`).                                                                                                    |
| `index` | Add a stack-supplied index after scaling it by `1<<size`. Reads pop the index. Writes remove an index stored immediately below the value while preserving the value for the store. |
| `write` | `0`: read and push; `1`: pop and write.                                                                                                                                            |
| `level` | Select lexical display pointer 0 through 3.                                                                                                                                        |
| `size`  | `00`: byte, `01`: word, `10`: long, `11`: eight-byte real/block. It is also the index scale 1, 2, 4, or 8.                                                                         |

The base effective address is `display[level]-displacement`. A two-byte
displacement at or above `$4000` is translated by adding `$C000` before the
subtraction. This is how the Pascal address model crosses the physical hole
between the two RAM banks.

Representative non-indexed one-byte forms are:

| Range                   | Operations                          |
| ----------------------- | ----------------------------------- |
| `$80-$83`               | `READ.B/W/L/F` at lexical level 0   |
| `$84-$87`               | `READ.B/W/L/F` at lexical level 1   |
| `$88-$8B`               | `READ.B/W/L/F` at lexical level 2   |
| `$8C-$8F`               | `READ.B/W/L/F` at lexical level 3   |
| `$90-$93` ... `$9C-$9F` | Corresponding `WRITE.B/W/L/F` forms |

Setting bit 5 produces indexed forms (`$A0-$BF`), and setting bit 6 changes
the displacement to two bytes (`$C0-$FF`); all other fields retain the same
meaning.

### Contextual zero and deferred stores

After a non-indexed or indexed read, the handler peeks at the next byte. If it
is nonzero, decoding continues normally. If it is `$00`, the read consumes
that byte and additionally pushes a descriptor containing the effective
address and value size. `$33 STORE.INDIRECT` later consumes this descriptor to
write the value back. `$08-$0B READ.BLOCK` has the same suffix behavior, using
the block length as the descriptor's size.

Thus `$00` is normally invalid, but is meaningful as a contextual suffix to a
read. Treating every zero byte as a standalone opcode is the main reason a
linear decoder loses synchronization on these reference/update forms.

## Remaining limits

The byte-level decoder and all primitive-table slots are now accounted for.
What remains uncertain is narrower:

- the original compiler's names for the store-keep, loop, block, and deferred
  store operations;
- whether reserved size combinations or zero/odd block lengths were ever
  emitted;
- exceptional and denormal behavior of the real-number package;
- the original compiler/runtime lineage.

Those gaps should not be represented as unknown opcode boundaries. They are
edge semantics or provenance questions, and do not prevent deterministic
disassembly of compiler-generated code.
