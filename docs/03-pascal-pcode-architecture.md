# Pascal/P-code Architecture

[Project index](../README.md) · [Complete edition](How%20the%20FTS%208400%20Worked%20-%20Complete.md)

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
