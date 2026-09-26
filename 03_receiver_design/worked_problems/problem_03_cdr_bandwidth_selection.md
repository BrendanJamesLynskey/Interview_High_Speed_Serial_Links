# Worked Problem 03: CDR Bandwidth Selection

## Problem Statement

A 112G PAM4 SerDes (56 GBd) CDR must be designed to meet the following specifications:

- Jitter tolerance at 10 MHz: at least 0.15 UI peak-to-peak
- Jitter transfer peaking: less than 0.1 dB
- Must track SSC modulation at 33 kHz with 5000 ppm deviation
- Must tolerate PCIe SKP ordered sets (1180 symbols between SKP insertions)
- Target recovered clock jitter: less than 50 fs RMS (integrated 100 kHz to 28 GHz)

The CDR uses a PI-based (phase interpolator) architecture with a bang-bang phase detector. Determine the optimal CDR bandwidth and loop parameters.

---

## Worked Solution

### Step 1: Determine bandwidth from jitter tolerance requirement

For a second-order CDR with a bang-bang (binary) phase detector, the jitter tolerance at frequency f_j is approximately:

```
JTOL(f_j) ≈ min(A_max, BW_CDR / (pi * f_j) * A_max) for f_j < BW_CDR
JTOL(f_j) ≈ A_max * (BW_CDR / f_j) for f_j near BW_CDR
JTOL(f_j) ≈ eye_width for f_j >> BW_CDR
```

Where A_max is the maximum tracking range and BW_CDR is the CDR bandwidth. At 10 MHz (above the CDR bandwidth), the jitter tolerance depends on the eye opening and the CDR's high-frequency tracking capability.

For a bang-bang CDR, the high-frequency jitter tolerance is approximately:

```
JTOL(f_j >> BW) ≈ Kp / (2 * pi * f_j)
```

Where Kp is the proportional path step size in UI per update. For JTOL(10 MHz) >= 0.15 UI:

```
Kp / (2 * pi * 10e6) >= 0.15 UI / 2 (amplitude to peak)
Kp >= 2 * pi * 10e6 * 0.075 = 4.71e6 UI/s
```

Since the CDR updates at the baud rate (56 GBd), the proportional step size per update is:

```
Kp_per_update = 4.71e6 / 56e9 = 8.42e-5 UI per update
```

This corresponds to approximately 1/11880 of a UI per update, or about 1.5 fs per update at 56 GBd (where 1 UI = 17.86 ps).

### Step 2: Determine bandwidth from SSC tracking requirement

SSC modulation at 33 kHz with 5000 ppm deviation at 56 GBd:

```
delta_f = 0.005 * 56e9 = 280 MHz peak deviation
```

The phase is the integral of the frequency offset, so for sinusoidal modulation the peak phase deviation, in cycles (UI), is:

```
phi_pk = delta_f / (2 * pi * f_SSC) = 280e6 / (2 * pi * 33e3) = 1350 UI   (8,480 radians)
```

The CDR must track this enormous phase deviation. For the CDR to track SSC, its bandwidth must be well above the SSC modulation frequency:

```
BW_CDR >> f_SSC = 33 kHz
```

Typically, BW_CDR >= 10 * f_SSC = 330 kHz. With a 4+ MHz bandwidth, this is easily satisfied.

### Step 3: Determine bandwidth from jitter transfer peaking requirement

For a second-order CDR, the jitter transfer function has the form:

```
|H(f)|^2 = (BW^2 + (f/f_n)^2) / ((f_n^2 - f^2)^2 + (2*zeta*f_n*f)^2)
```

A CDR with proportional and integral paths is a type-II loop: its jitter transfer has a zero as well as two poles,

```
H(s) = (2*zeta*w_n*s + w_n^2) / (s^2 + 2*zeta*w_n*s + w_n^2)
```

so it peaks for *every* damping ratio (the "no peaking above zeta = 0.707" rule is for a pure second-order low-pass without the zero). Evaluating |H| numerically:

| zeta | Peaking |
|---|---|
| 0.707 | 2.1 dB |
| 0.8 | 1.75 dB |
| 1 | 1.25 dB |
| 2 | 0.40 dB |
| 4.3 | 0.10 dB |

For peaking < 0.1 dB the loop must be heavily overdamped, zeta ≈ 4.3 or more — in practice by making the integral path very weak relative to the proportional path.

### Step 4: Set CDR bandwidth considering all constraints

Minimum bandwidth:
- SSC tracking: > 330 kHz (easily met)
- Jitter tolerance at 10 MHz: sets Kp (but BW is set by both Kp and Ki)

Maximum bandwidth:
- Recovered clock jitter: narrower bandwidth filters more CDR noise

Let us target BW_CDR = 6 MHz as a starting point and verify all constraints.

### Step 5: Calculate loop parameters for 6 MHz bandwidth

For a PI-based CDR with bang-bang PD:

The natural frequency of the second-order loop:

```
f_n = BW_CDR / sqrt(1 + 2*zeta^2 + sqrt((1+2*zeta^2)^2 + 1))
```

For zeta = 0.8 and BW_CDR = 6 MHz:

```
f_n = 6 MHz / sqrt(1 + 1.28 + sqrt(2.28^2 + 1)) = 6 / 2.18 = 2.75 MHz
```

The proportional gain Kp and integral gain Ki are related:

```
Kp = 2 * zeta * w_n / K_PD = 2 * 0.8 * 2*pi*2.75e6 / K_PD
Ki = w_n^2 / K_PD
```

For a bang-bang PD, K_PD is the effective gain (which depends on the jitter and signal quality). A typical value is K_PD = 1/(4*sigma_jitter) for a bang-bang detector.

The PI step size for the proportional path:

```
Kp_step = 2 * zeta * w_n * T_update = 2 * 0.8 * 2*pi*2.75e6 / 56e9
= 4.94e-4 (in PI phase units)
```

For a 7-bit PI (128 steps per UI):

```
Kp_step_codes = 4.94e-4 * 128 = 0.063 PI codes per update
```

This is less than 1 code per update, which is implemented by updating the PI code once every ~16 baud periods (56 GBd / 16 = 3.5 GHz effective update rate).

### Step 6: Verify jitter tolerance at 10 MHz

For a 6 MHz CDR with zeta = 0.8:

```
JTOL(10 MHz) ≈ 0.5 * BW / f_j * (1 + zeta) = 0.5 * 6/10 * 1.8 = 0.54 UI pp
```

This rough estimate exceeds the 0.15 UI requirement by a comfortable margin.

### Step 7: Estimate recovered clock jitter

The recovered clock jitter comes from the bang-bang CDR's limit cycle jitter and the PI quantization noise.

Bang-bang limit cycle jitter (RMS):

```
sigma_BB = Kp_step / sqrt(12) ≈ (1/128 UI) / sqrt(12) = 2.26e-3 UI = 40.4 fs
```

PI quantization noise (INL):

```
sigma_PI ≈ 0.5 * LSB / sqrt(12) = 0.5 * (1/128) / sqrt(12) = 1.13e-3 UI = 20.2 fs
```

Total recovered clock jitter:

```
sigma_total = sqrt(40.4^2 + 20.2^2) = sqrt(1632 + 408) = sqrt(2040) = 45.2 fs RMS
```

This is within the 50 fs budget with 4.8 fs margin.

### Step 8: Verify SKP ordered set handling

SKP ordered sets occur every 1180 symbols at 56 GBd:

```
SKP interval: 1180 / 56e9 = 21.1 ns
```

During SKP, the CDR may receive non-standard patterns. With a 6 MHz CDR bandwidth, the phase drift during a 21.1 ns interruption is:

```
Phase drift ≈ delta_f * T_SKP = (frequency offset) * 21.1 ns
```

For 300 ppm frequency offset: delta_f = 300e-6 * 56e9 = 16.8 MHz

```
Phase drift = 16.8e6 * 21.1e-9 = 0.354 cycles = 0.35 UI
```

(Frequency offset times time gives cycles, i.e. UI, not radians.) A 0.35 UI drift is a large fraction of the eye, so the CDR cannot simply coast through the interval; it must keep tracking (SKP ordered sets still carry data transitions).

### Result

| Parameter | Value |
|-----------|-------|
| CDR bandwidth | 6 MHz |
| Damping ratio (zeta) | 0.80 |
| Natural frequency (f_n) | 2.75 MHz |
| PI resolution | 7 bits (128 steps/UI) |
| Jitter tolerance at 10 MHz | ~0.54 UI pp (spec: 0.15 UI) |
| Jitter transfer peaking | 1.75 dB at zeta = 0.8 — FAILS < 0.1 dB (needs zeta ≈ 4.3) |
| Recovered clock jitter | 45.2 fs RMS (budget: 50 fs) |
| Phase drift during SKP | 0.35 UI (must keep tracking) |

### Key Takeaways

1. A 6 MHz bandwidth with zeta = 0.80 meets the tolerance, SSC and jitter targets but not the 0.1 dB peaking limit: a type-II CDR needs zeta ≈ 4.3 for that.
2. The jitter tolerance margin (0.54 vs 0.15 UI) is generous, suggesting bandwidth could be reduced.
3. The recovered clock jitter (45.2 fs) is the tightest constraint, limiting the maximum CDR bandwidth.
4. The bang-bang limit cycle is the dominant jitter contributor, set by the PI step size.
5. SSC tracking is easily met by any bandwidth above approximately 1 MHz; SKP intervals still require continuous tracking (0.35 UI drift at 300 ppm).
