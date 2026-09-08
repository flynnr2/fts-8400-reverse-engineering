# The GPS Navigation System

[Project index](../README.md) · [Complete edition](How%20the%20FTS%208400%20Worked%20-%20Complete.md)

## A complete NAV database in RAM

The ROM's reporting strings first gave away the scale of the navigation system. Names such as `IODC`, `IODE`, `SQRT A`, `DELTA N`, `OMEGA`, `ALPHA`, `BETA`, and the leap-second fields are not a loose diagnostic vocabulary. Following their report routines leads to arrays holding the decoded GPS broadcast message.

Many satellite arrays use a stride of `$108` bytes—33 slots of eight bytes—providing indexed storage across the GPS satellite set. The full ephemeris records include:

```text
week and Z-count                 IODC, IODE, TOC, TOE
AF0, AF1, AF2, TGD              sqrt(A), eccentricity, M0, delta-n
i0, IDOT, omega                 Omega0, Omega-dot
CRS, CRC, CUS, CUC, CIS, CIC   health, alert, fit and validity state
```

Almanac data is stored separately, with the reduced orbital parameter set expected from the broadcast almanac. This supports the front-panel states `USING EPHEMERIS`, `USING ALMANAC`, and `ORBIT DATA NOT AVAILABLE`: the application can use precise data when available and fall back to the almanac for prediction.

A shared record holds the eight Klobuchar coefficients, the GPS-to-UTC polynomial, current and future leap seconds, and their reference week/day fields. The firmware even maintains a checksum over this database.

## Turning broadcast parameters into a satellite position

The ephemeris routine around `$09C816` follows the GPS broadcast model closely enough to recognize equation by equation:

```text
time and broadcast ephemeris
           |
           v
wrap time from TOE to +/- half a GPS week
           |
           v
mean motion and mean anomaly
           |
           v
iterative solution of Kepler's equation
           |
           v
true anomaly and argument of latitude
           |
           v
harmonic corrections to latitude, radius, and inclination
           |
           v
orbital-plane coordinates -> Earth-fixed X, Y, Z
           |
           `-> optional velocity X, Y, Z
```

Time from ephemeris reference is wrapped at ±302,400 seconds using the 604,800-second GPS week. The routine solves `E - e sin(E) = M`, applies the six harmonic correction terms, includes inclination and node rates, and rotates the result into Earth-centred, Earth-fixed coordinates. A separate routine near `$09C372` performs the simpler almanac propagation.

The constants authenticate the interpretation. `6378137` and `0.00669437999...` are the WGS-84 semi-major axis and eccentricity squared. The stored Earth-rotation value, `2.32115234247e-5` semicircles per second, becomes approximately `7.292115147e-5` radians per second after multiplication by pi. Angles are commonly retained in semicircles, matching the GPS broadcast convention.

## Satellite time is modelled as carefully as position

The clock-correction path near `$09AFFE` evaluates the broadcast polynomial:

```text
AF0 + AF1 × dt + AF2 × dt^2
```

It also incorporates `TGD` and the relativistic eccentric-orbit correction:

```text
delta_tr = F × e × sqrt(A) × sin(E)
F = -4.442809305e-10
```

The Klobuchar routine near `$09AE68` is similarly recognizable. It computes the ionospheric pierce point, clamps its latitude to ±0.416 semicircles, derives local time, forms amplitude and period from `ALPHA0..3` and `BETA0..3`, and uses the standard daytime polynomial. One apparent period-floor comparison remains worth checking against the last undecoded VM comparison details before claiming that FTS deliberately departed from the usual model.

Together, these routines show that the observation model includes geometric range, satellite clock drift, relativistic correction, group delay, ionosphere, and Earth rotation during signal flight.

## Solving the receiver position

The position procedure near `$098088` uses four satellites to solve four unknowns: receiver latitude, longitude, height, and clock/range bias. It begins with a current geodetic estimate, converts it to WGS-84 Earth-centred coordinates, predicts ranges to the selected satellites, and corrects for Earth rotation during each signal's transit time.

For every satellite it forms a residual and one row of a 4×4 geometry matrix. The fourth element is one, corresponding to receiver clock bias. A reusable matrix package then solves the correction:

```text
observed ranges - modelled ranges -> residual vector r
geometry partial derivatives      -> matrix H

position/clock correction         -> delta-x = inverse(H) × r
```

The solution updates latitude, longitude, height, and clock bias, then iterates. This is a proper GPS navigation solution rather than a rough geometric shortcut.

The supporting numerical routines include matrix-vector multiplication around `$0035FE`, 4×4 multiplication around `$003766`, inversion around `$003AA2`, and transpose around `$003E8A`. Their reuse is another sign that the original software was designed as a structured numerical application.

## Geometry quality and stationary operation

After solving the fix, the program forms the usual covariance-like geometry matrix:

```text
Q = inverse(transpose(H) × H)
```

and derives PDOP, HDOP, VDOP, and TDOP from its diagonal terms. The values at `$42FD`, `$4305`, `$430D`, and `$4315` are independently confirmed by the display code. PDOP is compared with the user's acceptance criterion, so satellite geometry affects whether the receiver trusts a fix.

Accepted positions feed sums and squared sums of latitude, longitude, and altitude. For a stationary timing receiver this averaging is valuable: a better antenna-position estimate reduces the coupling between position error and the clock solution.

The scheduler also advances some predicted satellite events by 86,160 seconds—23 h 56 min—until they lie in the future. That is close to a sidereal day and strongly suggests reuse of daily satellite-geometry recurrence. The value and behavior are confirmed; the astronomical intent remains a high-confidence interpretation rather than a recovered FTS name.

## Week numbers, MJD, and a corrected rollover story

The receiver maintains an expanded GPS week and computes:

```text
MJD = GPS_week × 7 + day_of_week + 44244
```

`44244` is the Modified Julian Date of the GPS epoch, 6 January 1980. The calendar routine starts at the MJD epoch, 17 November 1858, and applies the full Gregorian divisible-by-4/100/400 leap-year rule.

More surprisingly, the 1987 NAV decoder anticipates the first ten-bit GPS week rollover:

```text
if received_week <= 453:
    expanded_week = received_week + 1024
else:
    expanded_week = received_week
```

Normal operation also increments the expanded week when seconds-of-week crosses 604,800. An early hypothesis blamed the main ten-bit week field for reported 1999 failures, but this ROM explicitly handles that transition. If an FTS 8400 revision did fail then, another firmware version or a narrower almanac/UTC week field is a better place to look.
