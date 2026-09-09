# FTS 8400 reverse engineering

This repository reconstructs how the FTS 8400 Satellite Timing Receiver worked from four EPROM images and photographs of its processor board.

The main result is not a commented disassembly. It is a recovered design: a 68000-based instrument running most of its application as Pascal P-code, calculating GPS satellite orbits and clock corrections in software, generating GPS- or UTC-aligned pulses, and comparing an external frequency standard with GPS using an interpolating time-interval counter.

The analysed firmware identifies itself as:

```text
NP 120 10-OCT-87
SP 101 26-JUL-87
```

The meanings of `NP` and `SP` have not yet been established.

## Read the story

- [How the FTS 8400 Worked — complete edition](docs/How%20the%20FTS%208400%20Worked%20-%20Complete.md)
- [Overview and major findings](docs/01-overview-and-major-findings.md)

## Read by subject

1. [Overview and major findings](docs/01-overview-and-major-findings.md)
2. [Hardware, memory, and boot](docs/02-hardware-memory-and-boot.md)
3. [The Pascal/P-code architecture](docs/03-pascal-pcode-architecture.md)
4. [Executive and communications](docs/04-executive-and-communications.md)
5. [The GPS navigation system](docs/05-gps-navigation-system.md)
6. [The timing and frequency system](docs/06-timing-and-frequency-system.md)
7. [Diagnostics and service modes](docs/07-diagnostics-and-service-modes.md)
8. [What remains unresolved](docs/08-unresolved-questions.md)

## Technical reference

- [Address map](docs/reference/address-map.md)
- [Pascal VM opcode reference](docs/reference/vm-opcodes.md)
- [Key constants](docs/reference/constants.md)
- [Recovered data structures](docs/reference/recovered-data-structures.md)

The narrative documents explain the recovered architecture. The reference documents preserve the address-level evidence without forcing every reader through it. Original ROM images, photographs, hashes, and derived analysis belong under `artifacts/`; primary evidence should not be silently modified.

## How certainty is described

The text uses three levels of confidence:

- **Confirmed** means the interpretation follows directly from coherent code, data layout, constants, strings, or visible hardware evidence.
- **High confidence** means the evidence strongly constrains the answer, although a schematic, measurement, or original FTS document would still be useful.
- **Hypothesis** marks a plausible explanation that has not yet been established.

These labels are concentrated where uncertainty matters. Routine, well-supported statements are allowed to read normally.
