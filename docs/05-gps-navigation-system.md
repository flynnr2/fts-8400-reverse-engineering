# GPS Navigation System

[Project index](../README.md) · [Complete edition](How%20the%20FTS%208400%20Worked%20-%20Complete.md)

### Broadcast data model

**Confirmed:** the application maintains separate records/arrays for ephemeris, almanac, ionosphere, UTC/leap-second data, validity, health, and use history. Many satellite fields use a `$108`-byte stride—33 eight-byte slots—consistent with indexed PRN storage.

The ephemeris model includes:

```text
WN, Z-count, IODC, IODE, TOC, TOE
AF0, AF1, AF2, TGD
sqrt(A), eccentricity, M0, delta-n
i0, IDOT, omega, Omega0, Omega-dot
CRS, CRC, CUS, CUC, CIS, CIC
health, alert, fit interval, validity, merit, last-use time
```

The almanac model separately stores week/time, TOA, `sqrt(A)`, eccentricity, `M0`, `i0`, `omega`, `Omega0`, `Omega-dot`, and validity. The receiver can therefore select full ephemeris, fall back to almanac for prediction, or report that orbit data is unavailable.

The shared ionosphere/UTC record contains `ALPHA0..3`, `BETA0..3`, UTC `A0/A1`, `T_ot`, current and future leap seconds, `WN_t`, `WN_LSF`, and day number. A database checksum routine is also present.

### GPS week and calendar handling

**Confirmed:** current expanded GPS week and day-of-week are held near global offsets `-$2EB9` and `-$2EBA`. The calendar pipeline uses:

```text
MJD = GPS_week * 7 + day_of_week + 44244
```

where 44244 is the MJD of the GPS epoch, 6 January 1980. The MJD-to-calendar routine at about `$0963D2` starts from 17 November 1858 and implements the full Gregorian divisible-by-4/100/400 leap-year rule.

The NAV decoder at about `$0928FC` expands the transmitted ten-bit week approximately as:

```text
if received_week <= 453:
    expanded_week = received_week + 1024
else:
    expanded_week = received_week
```

It also increments the expanded week when seconds-of-week crosses 604800.

This **supersedes an earlier tentative explanation** of the receiver's reported 1999 rollover trouble: this ROM revision deliberately handles the main ten-bit rollover. A different firmware revision or another truncated week field—UTC/leap-second or almanac fields are candidates—would be required to explain such a failure.

## Orbit propagation and position solution

### Satellite orbit and clock

The ephemeris propagator at about `$09C816` follows the broadcast GPS model:

```text
tk = wrap_half_week(t - TOE)
mean motion = nominal(sqrt(A)) + delta-n
M = M0 + mean_motion * tk
solve E - e*sin(E) = M iteratively
derive true anomaly and argument of latitude
apply CUS/CUC, CRS/CRC, CIS/CIC corrections
derive corrected radius, inclination and node
transform orbital-plane coordinates to ECEF X,Y,Z
optionally derive Vx,Vy,Vz
```

The half-week wrap uses ±302400 s and 604800 s. A separate, simplified almanac propagator is at about `$09C372`.

The satellite-clock path around `$09AFFE` evaluates:

```text
AF0 + AF1*dt + AF2*dt^2
```

with `TGD` and the relativistic eccentric-orbit term:

```text
delta_tr = F * e * sqrt(A) * sin(E)
F = -4.442809305e-10
```

**Confirmed:** the Klobuchar ionosphere routine at about `$09AE68` follows the broadcast equation, including `psi = 0.0137/(E+0.11)-0.022`, the ±0.416 semicircle pierce-point latitude clamp, local-time wrap, and the daytime polynomial. One apparent period-limit comparison should be rechecked against VM comparison semantics before claiming that FTS used a nonstandard minimum.

### Receiver position and DOP

The solver around `$098088` is an iterative, exactly determined four-satellite solution. It converts a latitude/longitude/height estimate to WGS-84 ECEF, predicts each range, applies Earth-rotation correction during signal transit, forms a 4×4 design matrix and residual vector, and solves for corrections to latitude, longitude, height, and receiver clock/range bias.

```text
four observations + four satellite states
                    |
                    v
       predicted geometric ranges
       + Earth rotation during flight
       + satellite/propagation corrections
                    |
                    v
        H and observation residuals
                    |
                    v
             delta-x = H^-1 r
                    |
                    v
       update position and clock; iterate
```

WGS-84 constants `a = 6378137 m` and `e^2 = 0.00669437999...` are present. Angles are commonly represented in semicircles, explaining explicit factors of pi. The Earth-rotation constant is stored as `2.32115234247e-5` semicircles/s, equal after multiplication by pi to approximately `7.292115147e-5 rad/s`.

Reusable native/P-code numerical routines include matrix-vector multiplication at `$0035FE`, 4×4 multiplication at `$003766`, inversion at `$003AA2`, and transpose at `$003E8A`.

After the fix, the firmware derives `Q = inverse(H^T H)` and computes PDOP, HDOP, VDOP, and TDOP. The reported PDOP participates in acceptance criteria. Accepted stationary fixes feed sums and squared sums for position averaging and scatter estimates.
