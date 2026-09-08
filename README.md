# FTS 8400 reverse engineering

Recovered software and hardware architecture of the FTS 8400 Satellite Timing Receiver, based on the four EPROM images and processor-board photographs supplied for this investigation.

The analysed firmware identifies itself as `NP 120 10-OCT-87` and `SP 101 26-JUL-87`. Reconstructed names are descriptive rather than claimed original FTS identifiers.

## Start here

- [Overview and major findings](docs/01-overview-and-major-findings.md)
- [Complete single-file edition](docs/How%20the%20FTS%208400%20Worked%20-%20Complete.md)

## Architecture chapters

1. [Overview and major findings](docs/01-overview-and-major-findings.md)
2. [Hardware, memory, and boot](docs/02-hardware-memory-and-boot.md)
3. [Pascal/P-code architecture](docs/03-pascal-pcode-architecture.md)
4. [Executive and communications](docs/04-executive-and-communications.md)
5. [GPS navigation system](docs/05-gps-navigation-system.md)
6. [Timing and frequency system](docs/06-timing-and-frequency-system.md)
7. [Diagnostics and service modes](docs/07-diagnostics-and-service-modes.md)
8. [Unresolved questions](docs/08-unresolved-questions.md)

## Reference

- [Address map](docs/reference/address-map.md)
- [Key constants](docs/reference/constants.md)
- [Recovered data structures](docs/reference/recovered-data-structures.md)

## Evidence and artifacts

The `artifacts/` directory is reserved for original ROM images, board photographs, hashes, disassembly exports, and other primary evidence. Derived architectural claims live in `docs/`; raw evidence should not be silently modified.

## Confidence terminology

- **Confirmed** — directly established by coherent executable code, data layout, constants, strings, or visible hardware evidence.
- **High confidence** — strongly constrained by the evidence, but still awaiting schematic, signal-trace, or original-document confirmation.
- **Hypothesis** — plausible and useful for further investigation, but not established.
