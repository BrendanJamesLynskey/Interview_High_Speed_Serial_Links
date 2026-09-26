# Worked Problem 03: PLL Jitter Budgeting

## Problem Statement

A 56 GBd SerDes PLL for a 112G PAM4 link has the following specifications:

- Output frequency: 28 GHz (half-rate clock, doubled locally per lane)
- Reference clock: 156.25 MHz
- PLL loop bandwidth: 4 MHz
- VCO phase noise: -108 dBc/Hz at 1 MHz offset (20 dB/decade slope)
- Reference clock jitter: 200 fs RMS (integrated 12 kHz to 20 MHz)
- Charge pump noise contribution: -95 dBc/Hz (flat within loop bandwidth)

The total transmitter jitter budget allocates 120 fs RMS for the PLL contribution (integrated from 100 kHz to 28 GHz, the half-rate Nyquist). Determine whether the PLL meets the jitter budget.

---

## Worked Solution

### Step 1: Identify the noise sources and their frequency behavior

The PLL output phase noise has three main contributors:

1. **Reference noise (multiplied):** Appears within the PLL bandwidth, multiplied by N^2 in power (20*log10(N) in dB)
2. **Charge pump noise:** Appears within the PLL bandwidth, shaped by the loop filter
3. **VCO noise:** Dominates outside the PLL bandwidth, attenuated within the bandwidth

Division ratio: N = 28 GHz / 156.25 MHz = 179.2 (fractional-N)

### Step 2: Calculate reference noise contribution at the output

The reference clock jitter of 200 fs RMS at 156.25 MHz corresponds to a phase noise level. First, convert RMS jitter to phase noise power:

```
Total phase noise power = (2*pi*f_ref * sigma_jitter)^2
= (2*pi*156.25e6 * 200e-15)^2
= (1.963e-4)^2
= 3.855e-8 rad^2
```

Assuming this is integrated from 12 kHz to 20 MHz (the spec bandwidth), the average spectral density within this band is approximately:

```
S_ref(f) ≈ 3.855e-8 / (20e6 - 12e3) ≈ 1.93e-15 rad^2/Hz
L_ref = 10*log10(1.93e-15 / 2) = -150.2 dBc/Hz (approximate average)
```

At the PLL output, this is multiplied by N^2:

```
L_ref_out = L_ref + 20*log10(N) = -150.2 + 20*log10(179.2) = -150.2 + 45.1 = -105.1 dBc/Hz
```

This contributes within the PLL bandwidth (DC to 4 MHz).

### Step 3: Calculate charge pump noise contribution

The charge pump contributes -95 dBc/Hz at the output (within the loop bandwidth). This is higher than the reference noise contribution, so it dominates within the loop bandwidth.

Within the PLL bandwidth (100 kHz to 4 MHz), the total in-band noise is approximately the charge pump level:

```
L_in_band ≈ -95 dBc/Hz
```

Integrated over 100 kHz to 4 MHz:

```
P_in_band = 10^(-95/10) * (4e6 - 100e3) * 2 (single-sideband to double-sideband)
= 3.162e-10 * 3.9e6 * 2
= 2.466e-3 rad^2
```

Wait, this is unreasonably large. Let me reconsider. The -95 dBc/Hz is likely the charge pump noise referred to the PLL output (already including the N^2 multiplication). Let me recalculate:

```
P_in_band = 2 * integral from 100 kHz to 4 MHz of 10^(-95/10) df
= 2 * 3.162e-10 * 3.9e6
= 2.466e-3 rad^2
```

This is still too large. The issue is that -95 dBc/Hz is the phase noise spectral density, and the integration should be:

```
sigma_phase^2 = 2 * integral of L(f) df  (where L is in linear units)
```

Actually, L(f) in dBc/Hz is the single-sideband spectral density:

```
sigma_phase^2 = 2 * integral from f_low to f_high of 10^(L(f)/10) df
```

For L = -95 dBc/Hz constant from 100 kHz to 4 MHz:

```
sigma_phase^2 = 2 * 10^(-9.5) * (4e6 - 1e5)
= 2 * 3.162e-10 * 3.9e6
= 2.466e-3 rad^2
sigma_phase = 0.0497 rad
```

Convert to jitter:

```
sigma_jitter_in_band = sigma_phase / (2*pi*f_out) = 0.0497 / (2*pi*28e9)
= 282 fs RMS
```

This already exceeds the 120 fs budget. Adding the reference contribution from Step 2 (-105.1 dBc/Hz) raises the in-band floor to -94.6 dBc/Hz and the in-band jitter to 296 fs. (Reading -95 dBc/Hz as input-referred would be far worse: +45 dB at the output.)

**With the stated -95 dBc/Hz charge-pump floor, the PLL fails the budget by about 2.5x.**

### Step 3 (continued): In-band floor needed to meet the budget

Allowing for the VCO (16 fs, Step 4) and the doubler (20 fs, Step 7), the in-band part may use:

```
sigma_in_band <= sqrt(120^2 - 16^2 - 20^2) = 117 fs
sigma_phase <= 117e-15 * 2*pi*28e9 = 0.0206 rad
L_in_band <= 10*log10(0.0206^2 / (2 * 3.9e6)) = -102.6 dBc/Hz
```

With the reference already contributing -105.1 dBc/Hz, the charge pump must be at or below about -106.3 dBc/Hz — roughly 11 dB better than specified. As a what-if, a charge-pump floor of -105 dBc/Hz alone gives:

```
sigma_phase_in_band^2 = 2 * 10^(-105/10) * (4e6 - 1e5)
= 2 * 3.162e-11 * 3.9e6
= 2.466e-4 rad^2
sigma_phase_in_band = 0.0157 rad

sigma_jitter_in_band = 0.0157 / (2*pi*28e9) = 89.2 fs RMS
```

but together with the -105.1 dBc/Hz reference this becomes -102.0 dBc/Hz (126 fs), still just over the allowance. The steps below use the 89.2 fs charge-pump-only figure to show how the remaining contributions combine.

### Step 4: Calculate VCO noise contribution (out-of-band)

The VCO phase noise at 1 MHz offset is -108 dBc/Hz and follows a 20 dB/decade slope (1/f^2 behavior):

```
L_VCO(f) = -108 + 20*log10(1e6/f) dBc/Hz for f > PLL bandwidth
```

Outside the PLL bandwidth (f > 4 MHz), the PLL loop attenuates the reference/CP noise but the VCO noise dominates. The VCO noise at the output is:

```
L_out(f) ≈ L_VCO(f) for f >> loop_BW
```

At 4 MHz (loop bandwidth edge):

```
L_VCO(4 MHz) = -108 + 20*log10(1e6/4e6) = -108 + 20*(-0.602) = -108 - 12.04 = -120 dBc/Hz
```

Integrating VCO noise from 4 MHz to 28 GHz with 1/f^2 slope:

```
sigma_phase_VCO^2 = 2 * integral from 4e6 to 28e9 of 10^(L_VCO(f)/10) df
```

For a 1/f^2 phase noise profile, L(f) = L_0 * (f_0/f)^2 where L_0 = 10^(-108/10) at f_0 = 1 MHz:

```
sigma_phase_VCO^2 = 2 * L_0 * f_0^2 * integral from 4e6 to 28e9 of (1/f^2) df
= 2 * 1.585e-11 * (1e6)^2 * [1/4e6 - 1/28e9]
= 2 * 1.585e-11 * 1e12 * [2.5e-7 - 3.57e-11]
= 2 * 1.585e-11 * 1e12 * 2.4996e-7
= 2 * 1.585e-11 * 2.4996e5
= 7.93e-6 rad^2

sigma_phase_VCO = 2.82e-3 rad

sigma_jitter_VCO = 2.82e-3 / (2*pi*28e9) = 16.0 fs RMS
```

### Step 5: Calculate total PLL jitter

The in-band and out-of-band contributions are approximately independent, so they add in power:

```
sigma_total^2 = sigma_in_band^2 + sigma_VCO^2
= (89.2 fs)^2 + (16.0 fs)^2
= 7957 + 256
= 8213 fs^2

sigma_total = 90.6 fs RMS
```

### Step 6: Compare to budget

```
PLL jitter budget: 120 fs RMS
Calculated PLL jitter: 90.6 fs RMS
Margin: 120 - 90.6 = 29.4 fs (24% margin)
```

### Step 7: Frequency doubler contribution

Since the PLL generates a 28 GHz half-rate clock that is doubled to 56 GHz per lane, the frequency doubler adds additional jitter. A typical doubler contributes 10-30 fs RMS. Including 20 fs for the doubler:

```
sigma_total_with_doubler = sqrt(90.6^2 + 20^2) = sqrt(8208 + 400) = sqrt(8608) = 92.8 fs RMS
```

Still within the 120 fs budget with 27 fs margin.

### Result

| Contribution | Jitter (fs RMS) | Fraction of Budget |
|-------------|----------------|--------------------|
| In-band (CP + reference) | 89.2 | 74.3% |
| VCO (out-of-band) | 16.0 | 13.3% |
| Frequency doubler | 20.0 | 16.7% |
| **Total** | **92.8** | **77.3%** |
| **Budget** | **120** | **100%** |
| **Margin** | **27.2** | **22.7%** |

**This table is the what-if case (charge pump improved to -105 dBc/Hz, reference noise ignored).** As specified (-95 dBc/Hz), the total is about 297 fs RMS and the PLL fails the 120 fs budget; including the reference noise, even the -105 dBc/Hz case reaches ~128 fs. The in-band floor must be at or below -102.6 dBc/Hz in total. The jitter is dominated by the in-band contribution (charge pump and reference noise), which is typical for LC-PLLs with a relatively wide 4 MHz loop bandwidth.

### Key Takeaways

1. In-band noise (charge pump and reference) dominates the total PLL jitter for LC-PLLs.
2. The VCO out-of-band contribution is small because the LC-VCO has excellent phase noise and the 1/f^2 noise integrates to a finite value.
3. Reducing the loop bandwidth would reduce the in-band contribution but increase the VCO contribution; the 4 MHz bandwidth is near the optimal crossover.
4. The frequency doubler adds a modest but non-negligible jitter contribution.
5. As specified the PLL fails; closing the budget needs an in-band floor of about -102.6 dBc/Hz or better (charge pump at or below about -106 dBc/Hz).
