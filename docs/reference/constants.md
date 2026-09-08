# Key Constants

[Project index](../../README.md) · [Complete edition](../How%20the%20FTS%208400%20Worked%20-%20Complete.md)

| Constant                  | Meaning                                                      | Confidence                                        |
| ------------------------: | ------------------------------------------------------------ | ------------------------------------------------- |
| `299792458`               | speed of light, m/s                                          | **Confirmed** by algorithm                        |
| `299792.458`              | light travel, m/ms; one C/A-code millisecond                 | **Confirmed**                                     |
| `1575420000`              | GPS L1 frequency, Hz                                         | **Confirmed**                                     |
| `4092000`                 | internal nominal measurement frequency, Hz (`4 × 1.023 MHz`) | **Confirmed** numerically; hardware label unknown |
| `16368000` / `16368`      | timing clock Hz / cycles per ms                              | **Confirmed**                                     |
| `18.3157660068`           | metres per 16.368 MHz cycle                                  | **Confirmed**                                     |
| `0.016368`                | 16.368 MHz expressed as cycles/ns                            | **Confirmed**                                     |
| `15.2737047898`           | fine-scale ratio producing 250 MHz / 4 ns bins               | **Confirmed** arithmetic                          |
| `3928320`                 | 16.368 MHz cycles in 240 ms                                  | **Confirmed** arithmetic                          |
| `2^23-1`, `2^24`          | signed 24-bit conversion limits                              | **Confirmed**                                     |
| `2^26` relation           | likely accumulator modulus                                   | **High confidence**                               |
| `6378137`                 | WGS-84 semi-major axis, m                                    | **Confirmed**                                     |
| `0.00669437999...`        | WGS-84 eccentricity squared                                  | **Confirmed**                                     |
| `2.32115234247e-5`        | Earth rotation, semicircles/s                                | **Confirmed**                                     |
| `-4.442809305e-10`        | GPS relativistic clock constant                              | **Confirmed**                                     |
| `604800`, `302400`        | GPS week and half-week, s                                    | **Confirmed**                                     |
| `44244`                   | MJD of GPS epoch                                             | **Confirmed**                                     |
| `500000000`, `1000000000` | half/full-second TI phase unwrap, ns                         | **Confirmed**                                     |
| `290.5`                   | internal-loop gain, DAC codes per hertz residual             | **Confirmed** in software                         |
| `32768`                   | neutral/default DAC code                                     | **Confirmed**                                     |
| `86160`                   | recurrence increment, near a sidereal day                    | **Confirmed** value; purpose **high confidence**  |
| `0.0004 ms`               | fixed 400 ns correction                                      | **Confirmed** value; source unresolved            |
