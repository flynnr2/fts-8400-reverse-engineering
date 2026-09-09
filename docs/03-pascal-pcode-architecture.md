# The Pascal/P-code Architecture

[Project index](../README.md) · [Complete edition](How%20the%20FTS%208400%20Worked%20-%20Complete.md)

## Why the application ROM looked wrong

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

## Why it is Pascal

The strongest evidence is not the style of the bytecode but the handling of nested scope. Locations `$4000`, `$4004`, `$4008`, and `$400C` hold pointers to active frames at lexical levels 0 through 3. Procedure entry saves and replaces the relevant pointer; return restores it. Procedures at levels 1, 2, and 3 are all present.

That lexical-display mechanism is characteristic of compiled Pascal. Pascal strings, local-frame allocation, nested procedures, and the separation between imported linkage stubs and procedure bodies reinforce the identification.

About 382 procedure entries can be recognized—360 in the large ROM and 22 in the low ROM. Roughly 60 runs of absolute-jump stubs look like per-unit import tables. They show that the source was modular, although they do not prove the original source contained exactly 60 named units.

The compiler itself remains unidentified. Period Microware Pascal systems used a broadly similar mixture of P-code, nested scopes, native support, floating point, and I/O routines, but this image has its own bare-metal startup and interrupt architecture. It is not a conventional OS-9 system.

## A compact instruction set

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

## Native code where it matters

Opcode `$02` aligns the instruction stream, executes embedded 68000 code, and returns to P-code. At least 26 procedure paths use such an escape. Larger native services are also reached through linkage stubs.

The numerical library is particularly important. It supplies eight-byte floating-point arithmetic, square root, sine and cosine, fractional/modulo operations, and reusable 4×4 matrix functions. Constants and exponent/mantissa handling are consistent with an IEEE-754-like double representation, although every exceptional case has not been tested.

That library explains how a 68000-class processor without an identified FPU could carry out orbit propagation, geodesy, clock modelling, ionosphere correction, regression, and matrix inversion. The VM did not make the receiver mathematically primitive; it gave a large mathematical program a compact, structured home.

## The application as a recovered program

The outer Pascal environment begins through a call at `$00103A`, reserves `$5800` bytes, performs the application self-test, and enters a permanent cooperative loop. Imported procedure stubs around `$0FF2-$1034` dispatch to the major subsystems.

The most useful way to read this firmware is consequently not as thousands of anonymous 68000 instructions. It is closer to a lost Pascal application with a surviving runtime. Procedure names have to be reconstructed from their inputs, outputs, constants, strings, and callers, but the original design boundaries remain surprisingly visible.
