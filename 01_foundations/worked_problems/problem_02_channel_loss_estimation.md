# Worked Problem 02: Channel Loss Estimation

## Problem Statement

A design team is evaluating two PCB material options for a 112 Gbps PAM4 (56 GBd) interconnect between two AI accelerators on a server motherboard. The channel consists of a 12-inch differential stripline trace. The two material options are:

- **Option A:** Megtron 6 (Dk = 3.6, Df = 0.004 at 10 GHz)
- **Option B:** Megtron 7 (Dk = 3.4, Df = 0.002 at 10 GHz)

The trace geometry is: 5-mil trace width, 5-mil spacing (edge-coupled differential pair), 4-mil dielectric height, 0.7 oz (1-mil thick) copper.

Estimate the insertion loss at the Nyquist frequency (28 GHz) for both materials and determine the loss difference. Assume conductor loss scales as sqrt(f) and dielectric loss scales linearly with f.

---

## Worked Solution

### Step 1: Identify the loss components

Total insertion loss per unit length is the sum of conductor loss and dielectric loss:

```
alpha_total(f) = alpha_conductor(f) + alpha_dielectric(f)
```

### Step 2: Estimate conductor loss

Conductor loss for a stripline trace is dominated by skin effect and can be approximated as:

```
alpha_conductor(f) = alpha_c0 * sqrt(f / f_ref)
```

For the given geometry (5-mil wide, 1-mil thick copper stripline), a typical conductor loss at a reference frequency of 1 GHz is approximately 0.05 dB/inch. This value is relatively independent of dielectric material.

At 28 GHz:

```
alpha_conductor(28 GHz) = 0.05 * sqrt(28 / 1)
alpha_conductor(28 GHz) = 0.05 * 5.29
alpha_conductor(28 GHz) = 0.265 dB/inch
```

This is the same for both materials since conductor loss depends on trace geometry and copper properties.

### Step 3: Estimate dielectric loss

Dielectric loss scales linearly with frequency and is proportional to the loss tangent:

```
alpha_dielectric(f) = (pi * f * sqrt(Dk) * Df) / c * (conversion to dB/inch)
```

A simpler engineering approximation is:

```
alpha_dielectric(f) = alpha_d_ref * (f / f_ref) * (Df / Df_ref)
```

For typical stripline geometry, the dielectric loss at 1 GHz with Df = 0.020 (standard FR4) is approximately 0.025 dB/inch. Scaling:

**Option A (Megtron 6, Df = 0.004):**

```
alpha_d_A(1 GHz) = 0.025 * (0.004 / 0.020) = 0.005 dB/inch at 1 GHz
alpha_d_A(28 GHz) = 0.005 * (28 / 1) = 0.140 dB/inch
```

Note: At frequencies above 10 GHz, the loss tangent typically increases. Using a frequency-adjusted Df at 28 GHz of approximately 0.005 for Megtron 6:

```
alpha_d_A(28 GHz) = 0.005 * 28 * (correction) ≈ 0.175 dB/inch
```

More practically, Megtron 6 datasheets report approximately 0.45-0.55 dB/inch total loss at 28 GHz for similar geometries. Using conductor loss of 0.265 dB/inch, the dielectric component is:

```
alpha_d_A(28 GHz) ≈ 0.50 - 0.265 ≈ 0.235 dB/inch
```

**Option B (Megtron 7, Df = 0.002):**

The dielectric loss scales proportionally with Df:

```
alpha_d_B(28 GHz) ≈ 0.235 * (0.002 / 0.004) = 0.118 dB/inch
```

### Step 4: Calculate total insertion loss per inch at 28 GHz

**Option A (Megtron 6):**

```
alpha_total_A = 0.265 + 0.235 = 0.50 dB/inch
```

**Option B (Megtron 7):**

```
alpha_total_B = 0.265 + 0.118 = 0.383 dB/inch
```

### Step 5: Calculate total channel loss for 12-inch trace

**Option A (Megtron 6):**

```
IL_A = 12 * 0.50 = 6.0 dB at 28 GHz
```

**Option B (Megtron 7):**

```
IL_B = 12 * 0.383 = 4.6 dB at 28 GHz
```

### Step 6: Calculate the loss difference

```
Delta_IL = IL_A - IL_B = 6.0 - 4.6 = 1.4 dB
```

### Step 7: Add discontinuity losses

In practice, the total channel includes via transitions and package losses. Assuming two via transitions (0.5 dB each) and two package contributions (1.5 dB each):

```
IL_discontinuities = 2 * 0.5 + 2 * 1.5 = 4.0 dB

Total_IL_A = 6.0 + 4.0 = 10.0 dB
Total_IL_B = 4.6 + 4.0 = 8.6 dB
```

### Result

| Parameter | Megtron 6 | Megtron 7 | Difference |
|-----------|-----------|-----------|------------|
| Conductor loss (28 GHz) | 0.265 dB/inch | 0.265 dB/inch | 0 |
| Dielectric loss (28 GHz) | 0.235 dB/inch | 0.118 dB/inch | 0.117 dB/inch |
| Total loss/inch (28 GHz) | 0.50 dB/inch | 0.383 dB/inch | 0.117 dB/inch |
| Trace loss (12 inches) | 6.0 dB | 4.6 dB | 1.4 dB |
| Total channel loss | 10.0 dB | 8.6 dB | 1.4 dB |

The 1.4 dB improvement from Megtron 7 over Megtron 6 translates directly to improved COM margin. For a link that is marginally meeting the 3 dB COM requirement, this improvement could be the difference between pass and fail.

### Key Takeaways

1. At 28 GHz, conductor and dielectric losses are comparable for low-loss materials, but dielectric loss dominates at higher frequencies.
2. Megtron 7 provides approximately 0.12 dB/inch improvement over Megtron 6 at 28 GHz for this geometry.
3. For a 12-inch trace, the material upgrade saves 1.4 dB, which is significant but must be weighed against the 3-4x cost premium.
4. Discontinuity losses (vias, packages) are fixed regardless of material and contribute a significant portion of total loss.
5. At 56 GHz (224G Nyquist), the dielectric loss difference doubles because dielectric loss scales linearly with frequency.
