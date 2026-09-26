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

### Step 3: Include the degeneration pole

A source-degenerated stage does not have a single zero and a single pole. The R_s-C_s network gives a zero *and* a second pole:

```
H(f) / A_DC = (1 + j f/f_z) / ((1 + j f/f_p2) * (1 + j f/f_p))

f_z  = 1 / (2 * pi * R_s * C_s)
f_p2 = (1 + gm * R_s) * f_z          (degeneration pole)
f_p  = 63.7 GHz                       (load pole, Step 2)
```

The ratio A_HF / A_DC = 1 + gm * R_s is the most peaking the stage can ever give; the value at 28 GHz is lower. (Setting 1 + gm * R_s = 3.98, i.e. 12 dB, with f_z = 6.62 GHz would put f_p2 at 26.4 GHz and deliver only 8.7 dB at 28 GHz.)

### Step 4: Calculate the degeneration components

Choose 1 + gm * R_s = 5, which places the peak close to Nyquist:

```
R_s = (5 - 1) / 40e-3 = 100 ohms
A_DC = gm * R_L / (1 + gm * R_s) = 4.0 / 5 = 0.8 (-1.9 dB)
A_HF = gm * R_L = 4.0 (12 dB)
```

Solving |H(28 GHz)| / A_DC = 10^(12/20) = 3.981 for f_z (with f_p2 = 5 * f_z and f_p = 63.7 GHz) gives:

```
f_z  = 3.26 GHz
f_p2 = 5 * 3.26 = 16.3 GHz
C_s = 1 / (2 * pi * f_z * R_s) = 1 / (2 * pi * 3.26e9 * 100) = 488 fF
```

### Step 5: Verify the response

```
|H(28 GHz)| / A_DC = |1 + j28/3.26| / (|1 + j28/16.3| * |1 + j28/63.7|)
                   = 8.65 / (1.99 * 1.09) = 3.98  (12.0 dB)
```

The peak of the response is 12.0 dB near 32 GHz — just above Nyquist, as intended. Absolute gain at 28 GHz: -1.9 + 12.0 = 10.1 dB.

### Step 6: Calculate the noise enhancement factor

The noise enhancement factor (NEF) compares the output noise with the CTLE response to the output noise with a flat gain of A_DC, for white input noise over the receiver's noise bandwidth (taken here as DC to the 28 GHz Nyquist frequency):

```
NEF = (1 / 28 GHz) * integral from 0 to 28 GHz of |H(f) / A_DC|^2 df
```

Numerical integration of this design's response gives:

```
NEF ≈ 9.5
NEF_dB = 10*log10(9.5) = 9.8 dB
```

(Neither shortcut — (f_p/f_z + 1)/2 or sqrt(f_p/f_z) — is a substitute for doing the integral.)

### Step 7: Calculate effective equalization gain

```
Signal gain at Nyquist (relative to DC): 12 dB
Noise enhancement (0-28 GHz): ~9.8 dB
Effective SNR improvement (crude measure): 12 - 9.8 ≈ 2 dB
```

This crude measure ignores the CTLE's main benefit — removing ISI — so it understates the CTLE's value; it does show why high peaking is expensive in noise.

### Result

| Parameter | Value |
|-----------|-------|
| R_s (degeneration resistor) | 100 ohms |
| C_s (degeneration capacitor) | 488 fF |
| Zero frequency (f_z) | 3.26 GHz |
| Degeneration pole (f_p2) | 16.3 GHz |
| Load pole (f_p) | 63.7 GHz |
| DC gain | -1.9 dB |
| Peaking at 28 GHz | 12.0 dB |
| Noise enhancement (0-28 GHz) | ~9.8 dB |
| Effective SNR improvement (crude) | ~2 dB |

### Key Takeaways

1. The degeneration network adds a pole at (1 + gm*R_s)*f_z; ignoring it overstates the peaking (8.7 dB, not 12 dB, for the naive design). Here f_z = 3.3 GHz and f_p2 = 16.3 GHz put the peak near Nyquist.
2. The noise enhancement (~9.8 dB over 0-28 GHz) consumes most of the 12 dB peaking.
3. By the crude signal-minus-noise measure only ~2 dB remains; the CTLE's real value is ISI removal, which needs a full link (pulse-response/COM) analysis to quantify.
4. The degeneration capacitance (488 fF) is relatively large and may be challenging to implement on-die with good quality factor.
5. A multi-stage CTLE could achieve the same total peaking with less noise enhancement per stage.
