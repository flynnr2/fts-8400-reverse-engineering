# Diagnostics and Service Modes

[Project index](../README.md) · [Complete edition](How%20the%20FTS%208400%20Worked%20-%20Complete.md)

Three service layers are visible:

1. **Confirmed — native power-on checks:** RAM, EPROM identity/signature/checksum, peripheral initialization, and custom hardware exercise.
2. **Confirmed — strap-selected factory diagnostics:** the `$A603/$A605/$A619` conditions select a deeper test of all major I/O blocks.
3. **Confirmed in software, high confidence physically — hidden front-panel DAC mode:** two key-state bytes corresponding to ASCII `N` and `P` must remain asserted for five scheduler passes before `SOFTWARE DAC` is enabled. This looks like a held key combination rather than the typed password `NP`, but the physical key labels and safe operating procedure are unverified.

The DAC service page permits direct manipulation of a control that may affect receiver lock or calibration. It should not be used on surviving hardware without recording the original code and understanding the analog destination.
