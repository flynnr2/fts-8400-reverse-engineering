# Diagnostics and Service Modes

[Project index](../README.md) · [Complete edition](How%20the%20FTS%208400%20Worked%20-%20Complete.md)

The 8400 has more than one definition of “self-test.” Its diagnostic design is layered, with each stage assuming a little more of the machine is working.

## Power-on integrity checks

The native reset path establishes the processor state and tests the foundations needed by the Pascal runtime. It clears and verifies both RAM banks, initializes the principal interfaces, and validates every EPROM independently.

The five-byte trailer on each EPROM records its socket number, the shared `$DBEB` set signature, and a 16-bit one's-complement checksum. Checking each byte lane separately catches both a damaged device and a ROM placed in the wrong socket. Only after these checks does boot construct the interpreted application environment.

The Pascal self-test then initializes higher-level receiver state and presents `SELF TEST OK`. It can also announce that test routines are enabled. Keeping low-level integrity checks native while leaving operator-facing setup in Pascal matches the architecture used elsewhere in the instrument.

## Strap-selected factory diagnostics

During boot, three masked inputs are tested:

```text
($A603 & $1E) == $1C
($A605 & $1E) == $1A
($A619 & $1E) == $1E
```

When all three match, the firmware enters a deeper native diagnostic. It exercises hardware blocks at `$A000`, `$A100`, `$A300`, `$A400`, `$A500`, `$A600`, and `$A700`, together with ROM and RAM.

The behavior is confirmed in code. Calling the inputs “factory straps” is a high-confidence interpretation: the board wiring or external test fixture that creates those values has not been traced.

## The hidden software-DAC page

A separate service path is embedded in the front-panel application. Two key-state bytes corresponding to ASCII `N` and `P` must remain asserted for five consecutive scheduler passes before a `SOFTWARE DAC` page is enabled. This looks more like a deliberately held key combination than a typed `NP` password, although the physical key labeling has not been verified.

The page allows the 16-bit DAC request to be viewed and altered using its bipolar ±5.000 engineering scale. Since the DAC probably controls an internal receiver-frequency element, direct adjustment could disturb lock or calibration. On surviving hardware, the original value should be recorded and the analog destination established before this mode is used.

The coincidence between the `N`/`P` key states and the firmware version label `NP 120` is interesting but not evidence that the initials have the same meaning.
