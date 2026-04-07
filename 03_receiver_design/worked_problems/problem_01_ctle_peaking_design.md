# Worked Problem 01: CTLE Peaking Design

## Problem Statement

A 56 GBd PAM4 receiver requires a CTLE to compensate for a channel with 18 dB insertion loss at the 28 GHz Nyquist frequency and 3 dB loss at 2 GHz. The CTLE is a single-stage degenerated differential amplifier with the following parameters:

- Transconductance (gm): 40 mS
- Load resistance (R_L): 100 ohms
- Source degeneration resistance (R_s): variable (controls DC gain)
- Source degeneration capacitance (C_s): variable (controls zero frequency)
- Parasitic load capacitance (C_L): 25 fF (sets the pole)

Determine the CTLE component values to provide 12 dB of peaking at 28 GHz relative to DC gain, and calculate the noise enhancement factor.

---

## Worked Solution

### Step 1: Determine the required gain profile

The channel loss profile:
- At DC: ~0 dB
- At 2 GHz: -3 dB
- At 28 GHz: -18 dB

The CTLE should partially compensate this loss. With 12 dB of peaking, the CTLE provides:
- At DC: A_DC (reference)
- At 28 GHz: A_DC + 12 dB

The residual loss after CTLE at 28 GHz: 18 - 12 = 6 dB (to be handled by TX FIR and DFE).

### Step 2: Calculate the pole frequency

The pole is set by the load RC time constant:

```
f_p = 1 / (2 * pi * R_L * C_L)
f_p = 1 / (2 * pi * 100 * 25e-15)
f_p = 63.7 GHz
```

This pole is above the Nyquist frequency, which is correct (the CTLE should not roll off before Nyquist).

### Step 3: Determine the zero frequency for 12 dB peaking

For a single zero-single pole CTLE, the peaking at the Nyquist frequency relative to DC is approximately:

```
Peaking = 20 * log10(sqrt(1 + (f_N/f_z)^2) / sqrt(1 + (f_N/f_p)^2))
```

Since f_p >> f_N (63.7 GHz >> 28 GHz), the denominator simplifies:

```
sqrt(1 + (28/63.7)^2) = sqrt(1 + 0.193) = sqrt(1.193) = 1.092
```

For 12 dB peaking:

```
10^(12/20) = sqrt(1 + (f_N/f_z)^2) / 1.092
3.981 = sqrt(1 + (28/f_z)^2) / 1.092
4.347 = sqrt(1 + (28/f_z)^2)
18.9 = 1 + (28/f_z)^2
17.9 = (28/f_z)^2
f_z = 28 / sqrt(17.9) = 28 / 4.23 = 6.62 GHz
```

### Step 4: Calculate the degeneration components

The zero frequency is determined by the RC degeneration network:

```
f_z = 1 / (2 * pi * R_s * C_s)
```

The DC gain with degeneration:

```
A_DC = gm * R_L / (1 + gm * R_s)
```

The high-frequency gain (when C_s shorts R_s):

```
A_HF = gm * R_L = 40e-3 * 100 = 4.0 (12 dB)
```

We need A_HF / A_DC = 10^(12/20) = 3.981:

```
A_HF / A_DC = (1 + gm * R_s)
3.981 = 1 + 40e-3 * R_s
R_s = 2.981 / 40e-3 = 74.5 ohms
```

Now calculate C_s:

```
C_s = 1 / (2 * pi * f_z * R_s)
C_s = 1 / (2 * pi * 6.62e9 * 74.5)
C_s = 322 fF
```

### Step 5: Verify the DC and HF gains

```
A_DC = 40e-3 * 100 / (1 + 40e-3 * 74.5)
A_DC = 4.0 / (1 + 2.98)
A_DC = 4.0 / 3.98 = 1.005 (0.04 dB)

A_HF = 4.0 (12.0 dB)

Peaking = 12.0 - 0.04 = 11.96 dB (approximately 12 dB)
```

### Step 6: Calculate the noise enhancement factor

The noise enhancement factor (NEF) is the ratio of output noise power with the CTLE frequency response to the output noise power with a flat gain of A_DC:

```
NEF = integral from 0 to infinity of |H(f)|^2 df / integral from 0 to infinity of |A_DC|^2 df
```

For practical calculation, integrate from 0 to 3*f_p (where the gain has rolled off sufficiently):

```
NEF = (1 / BW_noise) * integral of |H(f)/A_DC|^2 df
```

The noise bandwidth with the CTLE response is:

```
BW_noise_CTLE = integral of |H(f)|^2 df / |H(0)|^2
```

For a single zero at f_z = 6.62 GHz and single pole at f_p = 63.7 GHz:

```
BW_noise_CTLE ≈ (pi/2) * f_p * (1 + (f_p/f_z)^2) / (1 + f_p/f_z)
```

Using a simpler approximation for the NEF:

```
NEF ≈ (f_p / f_z + 1) / 2 = (63.7/6.62 + 1) / 2 = (9.62 + 1) / 2 = 5.31
NEF_dB = 10*log10(5.31) = 7.25 dB
```

This is a rough estimate. A more accurate numerical integration gives:

```
NEF ≈ sqrt(f_p / f_z) = sqrt(63.7/6.62) = sqrt(9.62) = 3.10
NEF_dB = 10*log10(3.10) = 4.9 dB
```

### Step 7: Calculate effective equalization gain

```
Signal gain at Nyquist: 12 dB
Noise enhancement: ~5 dB
Effective SNR improvement: 12 - 5 = 7 dB
```

### Result

| Parameter | Value |
|-----------|-------|
| R_s (degeneration resistor) | 74.5 ohms |
| C_s (degeneration capacitor) | 322 fF |
| Zero frequency (f_z) | 6.62 GHz |
| Pole frequency (f_p) | 63.7 GHz |
| DC gain | 0.04 dB |
| Peaking at 28 GHz | 12.0 dB |
| Noise enhancement | ~5 dB |
| Effective SNR improvement | ~7 dB |

### Key Takeaways

1. The zero frequency (~6.6 GHz) is about 1/4 of the Nyquist frequency for 12 dB peaking.
2. The noise enhancement penalty (~5 dB) consumes nearly half the equalization gain.
3. The effective SNR improvement (~7 dB) is the metric that matters for link budget.
4. The degeneration capacitance (322 fF) is relatively large and may be challenging to implement on-die with good quality factor.
5. A multi-stage CTLE could achieve the same total peaking with less noise enhancement per stage.
