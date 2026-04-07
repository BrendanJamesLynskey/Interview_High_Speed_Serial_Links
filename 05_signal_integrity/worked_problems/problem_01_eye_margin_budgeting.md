# Worked Problem 01: Eye Margin Budgeting

## Problem Statement

A 112 Gbps PAM4 link (56 GBd) has the following measured characteristics after full equalization (TX FIR + CTLE + DFE): inner eye height of 40 mV at BER = 1e-6, random noise (sigma) of 2.5 mV RMS, deterministic ISI residual of 8 mV peak-to-peak. The FEC (RS-544,514) requires a pre-FEC BER of no worse than 2.4e-4. Determine the voltage margin, timing margin, and whether the link meets the COM >= 3 dB requirement.

---

## Worked Solution

### Step 1: Calculate voltage margin at target BER

The inner eye height at BER = 1e-6 is 40 mV. This represents the eye opening after accounting for all noise components at the 1e-6 probability level. The eye height at a different BER can be extrapolated:

```
Eye_height(BER) = Eye_height(0) - 2 * Q(BER) * sigma_RJ - DJ_pp
```

Where Q(1e-6) = 4.75, Q(2.4e-4) = 3.50.

At BER = 1e-6:
```
40 mV = Eye_height(0) - 2 * 4.75 * 2.5 - 8
40 = Eye_height(0) - 23.75 - 8
Eye_height(0) = 71.75 mV (deterministic eye opening)
```

At BER = 2.4e-4 (FEC threshold):
```
Eye_height(2.4e-4) = 71.75 - 2 * 3.50 * 2.5 - 8
= 71.75 - 17.5 - 8 = 46.25 mV
```

Voltage margin at FEC threshold:
```
V_margin = Eye_height(2.4e-4) / 2 = 23.1 mV (from center to edge)
```

### Step 2: Calculate signal-to-noise ratio

The signal amplitude (half the inner eye opening at the deterministic level):
```
A_signal = (71.75 - 8) / 2 = 31.875 mV (half of deterministic eye minus DJ)
```

Actually, let us use the COM definition more directly:
```
A_signal = Eye_height(0) / 2 = 71.75 / 2 = 35.875 mV
A_noise = sqrt(sigma_RJ^2 + (DJ_pp/2)^2 * correction) 
```

Using the simplified COM approach:
```
A_signal = 35.875 mV
A_noise_rms = sqrt(sigma_RJ^2) = 2.5 mV (for the Gaussian component)
```

### Step 3: Calculate COM

COM is defined as:
```
COM = 20 * log10(A_signal / (Q_target * A_noise_total))
```

Where Q_target corresponds to the target BER (Q = 3.50 for BER = 2.4e-4 with RS-FEC):

The total noise at the target BER includes random noise scaled by Q:
```
A_noise_at_BER = Q_target * sigma_RJ + DJ_pp/2
= 3.50 * 2.5 + 8/2
= 8.75 + 4.0 = 12.75 mV
```

```
COM = 20 * log10(35.875 / 12.75) = 20 * log10(2.814) = 8.99 dB
```

Alternatively, using the standard COM formulation:
```
COM = 20 * log10(A_signal / A_noise_total)
```

Where A_noise_total includes all noise at the target BER:
```
A_noise_total = DJ_pp/2 + Q * sigma_noise
= 4.0 + 3.50 * 2.5 = 12.75 mV
COM = 20 * log10(35.875 / 12.75) = 8.99 dB
```

### Step 4: Verify COM >= 3 dB

```
COM = 8.99 dB > 3 dB requirement: PASS
Margin above requirement: 8.99 - 3.0 = 5.99 dB
```

### Step 5: Estimate timing margin

The eye width at BER = 1e-6 can be estimated from the jitter components. Assuming:
- Random jitter: sigma_RJ_timing = 200 fs RMS (typical for 56 GBd)
- Deterministic jitter: DJ_timing = 3.0 ps pp

```
UI = 1/56e9 = 17.86 ps
Eye_width(1e-6) = UI - DJ_timing - 2*Q(1e-6)*sigma_RJ_timing
= 17.86 - 3.0 - 2*4.75*0.2
= 17.86 - 3.0 - 1.9
= 12.96 ps = 0.726 UI
```

This is a generous timing margin, indicating the link is voltage-limited rather than timing-limited.

### Result

| Metric | Value | Requirement | Status |
|--------|-------|-------------|--------|
| Eye height (BER 1e-6) | 40 mV | > 15 mV | Pass |
| Eye height (BER 2.4e-4) | 46.25 mV | > 0 mV | Pass |
| COM | 8.99 dB | >= 3 dB | Pass |
| Eye width (BER 1e-6) | 0.726 UI | > 0.3 UI | Pass |
| Voltage margin | 23.1 mV | > 0 | Pass |

### Key Takeaways

1. The link has excellent COM margin (9 dB vs 3 dB requirement), indicating a short or well-designed channel.
2. Random noise (2.5 mV RMS) and deterministic ISI (8 mV pp) contribute roughly equally to eye closure.
3. The link is voltage-limited (the timing margin of 0.726 UI is much larger than the voltage margin equivalent).
4. FEC relaxes the BER target from 1e-12 to 2.4e-4, providing approximately 10 dB of effective coding gain in the COM calculation.
5. The excess margin (6 dB above requirement) could tolerate significant manufacturing variation.
