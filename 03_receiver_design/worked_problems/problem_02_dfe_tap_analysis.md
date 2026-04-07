# Worked Problem 02: DFE Tap Analysis

## Problem Statement

A 112 Gbps PAM4 receiver (56 GBd) has a 5-tap DFE. After the CTLE, the channel pulse response at the sampling instants is:

- h[0] = 0.45 (main cursor)
- h[1] = -0.18 (first post-cursor)
- h[2] = 0.09 (second post-cursor)
- h[3] = -0.05 (third post-cursor)
- h[4] = 0.03 (fourth post-cursor)
- h[5] = -0.02 (fifth post-cursor)
- h[6] = 0.01 (beyond DFE reach)

The PAM4 transmitter swing is 800 mVpp differential (four levels spanning 800 mV). Calculate: (a) the DFE tap coefficients, (b) the eye opening before and after DFE, (c) the residual ISI, and (d) the risk of error propagation.

---

## Worked Solution

### Step 1: Determine PAM4 level voltages

For 800 mVpp differential swing with four equally spaced levels:

```
Level spacing = 800 mV / 3 = 266.7 mV
Level 0 = -400 mV
Level 1 = -133.3 mV
Level 2 = +133.3 mV
Level 3 = +400 mV
```

Inner eye height (between adjacent levels, at perfect sampling): 266.7 mV.

### Step 2: Calculate eye opening before DFE

Before DFE, the ISI from all post-cursor taps reduces the eye opening. For PAM4, the worst-case ISI occurs when all post-cursor symbols are at maximum distance from the current symbol.

The main cursor voltage:

```
V_main = h[0] * 266.7 mV = 0.45 * 266.7 = 120.0 mV (per-level eye contribution)
```

Wait - let me reconsider. The pulse response h[n] is normalized to the channel. The received voltage for a PAM4 symbol with level a_k is:

```
v[n] = sum over k of a_k * h[n-k]
```

Where a_k is the PAM4 symbol level (one of {-3, -1, +1, +3} in normalized units, or {-400, -133.3, +133.3, +400} mV).

The main cursor contribution to the received voltage is a_n * h[0]. The ISI contribution is sum over k (not equal to n) of a_k * h[n-k].

The inner eye height (for the middle eye, between levels 1 and 2) is:

```
Eye_height = 2 * h[0] * delta_level - 2 * sum(|h[k]| * max_ISI) for k >= 1
```

Where delta_level is the level spacing in normalized units (= 2/3 of the total peak swing in normalized units). Let me use the simplified approach:

The worst-case ISI voltage from each post-cursor tap is the maximum PAM4 level magnitude (which is 3 in normalized units, corresponding to 400 mV) times the tap magnitude:

```
ISI_worst_k = 3 * |h[k]| for levels in {-3,-1,+1,+3}
```

Actually, for the inner eye, the worst-case ISI from each interferer is when the interferer is at its maximum distance (level 3 or 0), adding maximum noise. The per-tap worst-case ISI contribution depends on which eye we analyze. For simplicity, the total worst-case ISI is:

```
Total_worst_ISI = sum over k=1 to 6 of 2 * |h[k]|
```

(Factor of 2 because the interfering symbol can be at +3 or -3, a range of 6 units, but the inner eye height is only 2 units for the worst case.)

Let me use a cleaner formulation. The inner eye height for PAM4 is:

```
Eye_inner = 2/3 * V_pk * h[0] - 2 * V_pk * sum(|h[k]|) for k >= 1
```

Wait, this is getting complex. Let me use voltage directly.

The minimum inner eye height (worst-case ISI):

```
Eye_before_DFE = (2/3) * V_pp * h[0] - (2) * V_pp * sum(|h[k]|, k=1..6) / 3
```

Actually, the simplest correct approach: for PAM4 with levels {-3, -1, 1, 3} * V_unit where V_unit = Vpp/6:

```
V_unit = 800/6 = 133.3 mV
Inner eye height = 2 * V_unit * h[0] - 2 * V_unit * sum(3 * |h[k]|, k=1..6)
```

No wait, the inner eye is reduced by ISI from all interfering symbols. Each interfering symbol adds ISI equal to its level times h[k]. The worst case is when all ISI terms are in the same direction.

The inner eye half-height is V_unit * h[0] (the smallest signal difference). The ISI from each tap k is at most 3 * V_unit * |h[k]| (maximum symbol level times channel tap). The worst-case inner eye:

```
Eye_inner = 2 * (V_unit * h[0] - sum over k=1..6 of 3 * V_unit * |h[k]|)
= 2 * V_unit * (h[0] - 3 * sum(|h[k]|))
= 2 * 133.3 * (0.45 - 3 * (0.18 + 0.09 + 0.05 + 0.03 + 0.02 + 0.01))
= 266.7 * (0.45 - 3 * 0.38)
= 266.7 * (0.45 - 1.14)
= 266.7 * (-0.69)
= -184 mV
```

The eye is completely closed (negative value) before DFE. This is expected for a channel with significant ISI.

### Step 3: Determine DFE tap coefficients

The DFE tap coefficients are set equal to the channel pulse response at each post-cursor position (normalized to the main cursor):

```
DFE h1 = h[1] / h[0] * V_unit = ... 
```

Actually, the DFE tap values are simply set to match the ISI values. The DFE subtracts h[k] * d[n-k] from the received sample, where d[n-k] is the decided PAM4 level. The tap coefficients in voltage terms are:

```
Tap 1: h1 = h[1] = -0.18 (will subtract -0.18 * d[n-1])
Tap 2: h2 = h[2] = 0.09
Tap 3: h3 = h[3] = -0.05
Tap 4: h4 = h[4] = 0.03
Tap 5: h5 = h[5] = -0.02
```

In absolute voltage terms (per V_unit):

```
h1_voltage = -0.18 * 133.3 = -24.0 mV (per unit level)
h2_voltage = 0.09 * 133.3 = 12.0 mV
h3_voltage = -0.05 * 133.3 = -6.67 mV
h4_voltage = 0.03 * 133.3 = 4.0 mV
h5_voltage = -0.02 * 133.3 = -2.67 mV
```

### Step 4: Calculate eye opening after DFE

After DFE, the taps h[1] through h[5] are canceled. The residual ISI is from h[6] onward:

```
Eye_after_DFE = 2 * V_unit * (h[0] - 3 * sum(|h[k]|, k=6..))
= 2 * 133.3 * (0.45 - 3 * 0.01)
= 266.7 * (0.45 - 0.03)
= 266.7 * 0.42
= 112.0 mV
```

### Step 5: Calculate the improvement

```
Eye before DFE: closed (negative)
Eye after DFE: 112.0 mV
Total ISI canceled: 3 * (0.18+0.09+0.05+0.03+0.02) * 266.7 = 3 * 0.37 * 266.7 = 296 mV
```

### Step 6: Analyze error propagation risk

For PAM4, the error propagation risk is highest for the first DFE tap (h1). If a decision error occurs, the DFE subtracts the wrong ISI for the next sample:

```
Error in DFE correction for h1 = h1 * (d_correct - d_wrong)
```

The minimum error is one level spacing (when the decision is off by one level):

```
Error = |h1| * 2 * V_unit = 0.18 * 2 * 133.3 = 48.0 mV
```

This error of 48.0 mV compared to the eye opening of 112.0 mV represents:

```
Error / Eye = 48.0 / 112.0 = 42.9%
```

This means a single decision error shifts the next sample by 42.9% of the eye height, which significantly increases the probability of error propagation. The probability that the next sample is also incorrect (given the error) depends on the noise distribution, but with 42.9% eye reduction, the propagation probability is approximately 10-20% for typical noise levels.

### Step 7: Calculate effective BER impact of error propagation

If isolated error rate is BER_0 and propagation probability is P_prop:

```
Effective BER ≈ BER_0 * (1 + P_prop + P_prop^2 + ...) = BER_0 / (1 - P_prop)
```

For P_prop = 0.15 (estimated):

```
BER_effective = BER_0 / 0.85 = 1.18 * BER_0
```

The error propagation increases the effective BER by about 18%, which is approximately 0.7 dB of SNR penalty.

### Result

| Metric | Before DFE | After DFE |
|--------|-----------|-----------|
| Inner eye height | Closed | 112.0 mV |
| ISI canceled | N/A | 296 mV (worst-case) |
| Residual ISI | 0.38 (normalized) | 0.01 (normalized) |
| Error propagation risk | N/A | 42.9% eye reduction |

DFE tap coefficients (normalized):

| Tap | Value | ISI Canceled (mV per unit) |
|-----|-------|---------------------------|
| h1 | -0.18 | 24.0 |
| h2 | +0.09 | 12.0 |
| h3 | -0.05 | 6.67 |
| h4 | +0.03 | 4.0 |
| h5 | -0.02 | 2.67 |

### Key Takeaways

1. The eye is completely closed before DFE, confirming that DFE is essential for this channel.
2. The 5-tap DFE recovers 112 mV of inner eye height from a closed eye.
3. The h1 error propagation risk is significant (42.9% eye reduction), which is characteristic of PAM4.
4. Residual ISI from h[6] and beyond is small (0.01) and contributes only 3% eye reduction.
5. The error propagation BER penalty (~0.7 dB) must be accounted for in the link budget.
