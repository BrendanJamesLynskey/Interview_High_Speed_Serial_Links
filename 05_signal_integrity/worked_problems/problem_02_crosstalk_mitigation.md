# Worked Problem 02: Crosstalk Mitigation

## Problem Statement

A 16-lane NVLink interface on an AI server motherboard routes all lanes as differential stripline pairs on layer 4 of a 24-layer PCB. The parallel routing length is 6 inches. Without any mitigation, FEXT simulation shows -35 dB coupling from nearest neighbors at 28 GHz. The COM analysis shows that the crosstalk noise reduces COM from 5.0 dB (crosstalk-free) to 2.5 dB (below the 3 dB requirement).

Evaluate three mitigation options: (a) increase lane spacing from 20 mils to 30 mils, (b) add guard traces with via stitching every 500 mils, (c) route alternating lanes on different layers. Determine which option(s) restore COM to >= 3 dB.

---

## Worked Solution

### Step 1: Quantify the crosstalk noise impact

The COM reduction from 5.0 to 2.5 dB means the crosstalk added noise that reduced the SNR by 2.5 dB. Converting to linear:

```
SNR_without_xtalk = 10^(5.0/20) = 1.778
SNR_with_xtalk = 10^(2.5/20) = 1.334
```

The crosstalk noise power: signal/noise_total = 1.334, signal/noise_no_xtalk = 1.778.

```
noise_total^2 = noise_no_xtalk^2 + noise_xtalk^2
(S/1.334)^2 = (S/1.778)^2 + noise_xtalk^2
noise_xtalk = S * sqrt(1/1.334^2 - 1/1.778^2)
= S * sqrt(0.562 - 0.316) = S * sqrt(0.246) = S * 0.496
```

So the crosstalk noise is 0.496 times the signal, or -6.1 dB relative to the signal.

### Step 2: Option A - Increase spacing from 20 to 30 mils

FEXT coupling for stripline scales approximately as (spacing)^(-3):

```
FEXT_new = FEXT_old * (20/30)^3 = -35 + 20*log10((20/30)^3)
= -35 + 20*log10(0.296) = -35 + (-10.6) = -45.6 dB
```

This is a 10.6 dB improvement. The crosstalk noise is reduced by 10.6 dB:

```
noise_xtalk_new = noise_xtalk_old * 10^(-10.6/20) = 0.496 * 0.295 = 0.146
```

New COM:
```
noise_total_new = sqrt(noise_no_xtalk^2 + noise_xtalk_new^2)
= sqrt(0.316 + 0.0214) * S = sqrt(0.337) * S = 0.581 * S
COM_new = 20*log10(1/0.581) = 4.71 dB
```

COM restored to 4.71 dB (> 3 dB). This option works.

### Step 3: Option B - Guard traces with via stitching

Guard traces with 500-mil via stitching at 28 GHz: the stitching interval (500 mil = 12.7 mm) compared to quarter wavelength at 28 GHz in FR4 (approximately 1.4 mm). The stitching is much coarser than quarter wavelength, so the guard effectiveness is reduced at 28 GHz.

Estimated FEXT reduction: approximately 5-8 dB (guard traces are partially effective at this stitching interval).

```
FEXT_new = -35 - 6.5 = -41.5 dB (using 6.5 dB average improvement)
noise_xtalk_new = 0.496 * 10^(-6.5/20) = 0.496 * 0.473 = 0.235
noise_total_new = sqrt(0.316 + 0.055) = sqrt(0.371) = 0.609
COM_new = 20*log10(1/0.609) = 4.31 dB
```

COM restored to 4.31 dB (> 3 dB). This option works but with less margin.

### Step 4: Option C - Alternating layers

Routing alternating lanes on different layers (e.g., lanes 1,3,5,7 on layer 4 and lanes 2,4,6,8 on layer 8) increases the effective spacing by the inter-layer distance (typically 10-15 mils for adjacent signal layers). This dramatically reduces FEXT because the coupling decays exponentially with vertical separation.

```
Vertical separation: ~40 mils (4 layers at 10 mil spacing)
FEXT reduction: approximately 20-25 dB (vertical coupling is much weaker)
FEXT_new = -35 - 22 = -57 dB
noise_xtalk_new = 0.496 * 10^(-22/20) = 0.496 * 0.079 = 0.039
COM_new = 20*log10(1/sqrt(0.316 + 0.0015)) = 20*log10(1/0.564) = 4.97 dB
```

COM restored to 4.97 dB, nearly matching the crosstalk-free case. This is the most effective option.

### Result

| Option | FEXT Improvement | New COM | Meets 3 dB? | Routing Cost |
|--------|-----------------|---------|-------------|-------------|
| A: 30 mil spacing | 10.6 dB | 4.71 dB | Yes | 50% wider routing |
| B: Guard traces | 6.5 dB | 4.31 dB | Yes | Guard trace area |
| C: Alternating layers | 22 dB | 4.97 dB | Yes | Uses 2 layers |

### Key Takeaways

1. All three options restore COM to >= 3 dB, but with different margins and routing costs.
2. Layer alternation (Option C) is most effective but uses additional PCB layers.
3. Spacing increase (Option A) is straightforward but requires 50% more routing width.
4. Guard traces (Option B) are least effective at 28 GHz due to inadequate via stitching frequency.
5. Combining options (e.g., B+A or C+A) provides the maximum margin.
