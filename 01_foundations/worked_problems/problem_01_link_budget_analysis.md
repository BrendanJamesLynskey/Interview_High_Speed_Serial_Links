# Worked Problem 01: Link Budget Analysis

## Problem Statement

An AI accelerator board uses a 112 Gbps PAM4 serial link (56 GBd) to connect a GPU to a switch ASIC. The channel consists of:

- Transmitter package: 2 dB loss at Nyquist
- PCB trace: 8 inches at 0.9 dB/inch at Nyquist on Megtron 6
- One mid-board connector: 1.5 dB insertion loss at Nyquist
- Receiver package: 1.5 dB loss at Nyquist

The transmitter produces 800 mV peak-to-peak differential swing. The equalization capabilities are:

- TX FIR: 6 dB of effective equalization gain
- CTLE: 10 dB of peaking gain
- DFE (4-tap): 8 dB of ISI cancellation

The receiver sensitivity is -22 dBm differential at BER = 1e-6. Determine whether the link has sufficient margin and compute the approximate Channel Operating Margin.

---

## Worked Solution

### Step 1: Calculate total channel insertion loss at Nyquist

The Nyquist frequency for a 56 GBd link is:

```
f_Nyquist = 56 GBd / 2 = 28 GHz
```

Total channel insertion loss at 28 GHz:

```
IL_total = IL_tx_pkg + IL_pcb + IL_connector + IL_rx_pkg
IL_total = 2.0 + (8 * 0.9) + 1.5 + 1.5
IL_total = 2.0 + 7.2 + 1.5 + 1.5
IL_total = 12.2 dB
```

### Step 2: Convert transmitter swing to dBm

The transmitter output power for an 800 mVpp differential signal into a 100-ohm differential termination:

```
V_rms = 800 mV / (2 * sqrt(2)) = 283 mV_rms
P_tx = V_rms^2 / R_diff = (0.283)^2 / 100 = 0.8 mW
P_tx_dBm = 10 * log10(0.8) = -0.97 dBm
```

### Step 3: Account for PAM4 penalty

PAM4 uses four voltage levels, so the distance between adjacent levels is 1/3 of the full NRZ swing:

```
PAM4_penalty = 20 * log10(3) = 9.54 dB
```

The effective signal power for PAM4 eye analysis:

```
P_effective = P_tx_dBm - PAM4_penalty = -0.97 - 9.54 = -10.51 dBm
```

### Step 4: Calculate signal at receiver before equalization

```
P_rx_before_eq = P_effective - IL_total
P_rx_before_eq = -10.51 - 12.2
P_rx_before_eq = -22.71 dBm
```

This is already below the receiver sensitivity of -22 dBm, confirming that equalization is essential.

### Step 5: Apply equalization gains

The equalization chain recovers signal quality by compensating for channel loss:

```
P_rx_after_eq = P_rx_before_eq + TX_FIR_gain + CTLE_gain + DFE_gain
P_rx_after_eq = -22.71 + 6.0 + 10.0 + 8.0
P_rx_after_eq = 1.29 dBm (equivalent signal quality)
```

Note: This is a simplified linear model. In practice, equalization gains are not simply additive in dBm because TX FIR boosts the signal before the channel, CTLE amplifies noise along with signal, and DFE operates on post-cursor ISI specifically. The COM methodology handles these interactions properly.

### Step 6: Calculate margin

```
Margin = P_rx_after_eq - Rx_sensitivity
Margin = 1.29 - (-22)
Margin = 23.29 dB (simplified)
```

This margin is unrealistically large because the simplified model double-counts equalization effectiveness. A more realistic estimate accounts for noise enhancement from CTLE and partial ISI cancellation:

```
Effective_eq_gain = TX_FIR + CTLE_effective + DFE
```

Where CTLE_effective accounts for noise enhancement penalty (typically 3-5 dB less than the peaking gain):

```
CTLE_effective = 10.0 - 4.0 = 6.0 dB (after noise penalty)
Effective_eq_gain = 6.0 + 6.0 + 8.0 = 20.0 dB
```

### Step 7: Revised margin calculation with noise considerations

```
P_rx_equalized = -22.71 + 20.0 = -2.71 dBm
```

Including crosstalk noise (estimated at -30 dBm from adjacent aggressors) and random noise (estimated at -28 dBm from receiver thermal noise):

```
Total_noise = 10*log10(10^(-30/10) + 10^(-28/10)) = -25.7 dBm
SNR = P_rx_equalized - Total_noise = -2.71 - (-25.7) = 23.0 dB
Required_SNR_for_BER_1e-6 (PAM4) = ~20 dB
```

### Step 8: Approximate COM

```
COM (approximate) = SNR_achieved - SNR_required
COM (approximate) = 23.0 - 20.0 = 3.0 dB
```

### Result

The link achieves approximately 3.0 dB of COM, which meets the minimum requirement of 3 dB specified by IEEE 802.3. The link is feasible but has minimal margin, suggesting that any degradation in channel quality (such as manufacturing variation, temperature effects, or aging) could push the link below the threshold. Design improvements to consider include using a lower-loss PCB material or reducing the trace length by 1-2 inches.

### Key Takeaways

1. The PAM4 penalty (9.54 dB) is the single largest fixed impairment in the budget.
2. CTLE noise enhancement partially offsets the equalization gain.
3. A 3 dB COM represents the minimum acceptable margin for production designs.
4. The simplified linear link budget provides intuition but the COM methodology is needed for accurate results.
