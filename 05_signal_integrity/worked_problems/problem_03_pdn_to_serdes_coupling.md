# Worked Problem 03: PDN-to-SerDes Coupling

## Problem Statement

An AI accelerator GPU has a core supply voltage of 0.8V with a PDN impedance profile that peaks at 15 milliohms at the anti-resonance frequency of 200 MHz. The core switching current has a component of 5A peak at 200 MHz (from a particular workload pattern). The SerDes PLL has a supply sensitivity (KVCO_supply) of 5 MHz/mV and operates at 28 GHz. The PLL loop bandwidth is 4 MHz.

Calculate: (a) the supply noise voltage at 200 MHz, (b) the resulting PLL jitter, (c) whether this jitter is within the 100 fs RMS budget, and (d) what PDN impedance target would be needed to meet the budget.

---

## Worked Solution

### Step 1: Calculate supply noise voltage

```
V_noise = Z_PDN(200 MHz) * I_switching(200 MHz)
V_noise = 15e-3 ohms * 5 A = 75 mV peak
V_noise_rms = 75 / sqrt(2) = 53 mV RMS
```

### Step 2: Calculate PLL frequency modulation

The VCO frequency is modulated by the supply noise:
```
delta_f = K_supply * V_noise_peak = 5 MHz/mV * 75 mV = 375 MHz peak deviation
```

### Step 3: Determine PLL response at 200 MHz

The PLL loop bandwidth is 4 MHz, and the noise is at 200 MHz (well outside the loop bandwidth). The PLL loop attenuates noise within the bandwidth but the VCO supply sensitivity operates directly:

Since 200 MHz >> 4 MHz (PLL BW), the PLL does NOT attenuate this noise. The VCO noise at 200 MHz offset passes directly to the output.

### Step 4: Convert frequency modulation to phase jitter

For a sinusoidal frequency modulation at f_mod = 200 MHz:
```
Phase deviation (peak) = delta_f / f_mod = 375e6 / 200e6 = 1.875 radians
```

Converting to time jitter:
```
Jitter_peak = phase_peak / (2*pi*f_carrier) = 1.875 / (2*pi*28e9)
= 1.875 / 175.9e9 = 10.66 ps peak
Jitter_rms = 10.66 / sqrt(2) = 7.53 ps RMS
```

### Step 5: Compare to budget

```
Calculated jitter: 7530 fs RMS
Budget: 100 fs RMS
Ratio: 7530 / 100 = 75.3x over budget (37.5 dB)
```

This is massively over budget, indicating that the PDN impedance at the anti-resonance is far too high for the SerDes PLL.

### Step 6: Calculate required PDN impedance

Working backwards from the 100 fs RMS jitter budget:

```
Jitter_rms_target = 100 fs = 100e-15 s
Phase_rms_target = 100e-15 * 2*pi*28e9 = 17.6e-3 radians
Phase_peak_target = 17.6e-3 * sqrt(2) = 24.9e-3 radians
delta_f_target = phase_peak * f_mod = 24.9e-3 * 200e6 = 4.98 MHz peak
V_noise_target = delta_f_target / K_supply = 4.98 / 5 = 0.996 mV peak
Z_PDN_target = V_noise_target / I_switching = 0.996e-3 / 5 = 0.199 milliohms
```

### Step 7: Assess feasibility

```
Current Z_PDN at 200 MHz: 15 milliohms
Required Z_PDN: 0.2 milliohms
Reduction needed: 15 / 0.2 = 75x (37.5 dB)
```

This is an extremely aggressive target. Achieving 0.2 milliohms at 200 MHz requires either massive improvement in the PDN design (additional decoupling capacitors with low ESL to eliminate the anti-resonance), an on-die LDO regulator for the PLL supply (providing 40+ dB of supply rejection at 200 MHz), or reducing the VCO supply sensitivity from 5 MHz/mV to less than 0.07 MHz/mV (through circuit design techniques).

In practice, the most effective approach is an on-die LDO with 40 dB PSRR at 200 MHz, which reduces the effective supply noise from 75 mV to 0.75 mV, bringing the jitter to approximately 100 fs RMS.

### Result

| Parameter | Value |
|-----------|-------|
| PDN noise at 200 MHz | 75 mV peak |
| PLL jitter (unmitigated) | 7530 fs RMS |
| Jitter budget | 100 fs RMS |
| Required PDN impedance | 0.2 milliohms |
| Required LDO PSRR | 37.5 dB at 200 MHz |

### Key Takeaways

1. PDN anti-resonance at 200 MHz creates 75 mV of supply noise from 5A of switching current.
2. Without mitigation, this creates 7530 fs of PLL jitter, 75x over the 100 fs budget.
3. The required PDN impedance (0.2 milliohms) is impractical without active regulation.
4. An on-die LDO with 40 dB PSRR is the practical solution for PLL supply noise rejection.
5. AI accelerator workload patterns that create correlated switching at specific frequencies are particularly problematic for PLL jitter.
