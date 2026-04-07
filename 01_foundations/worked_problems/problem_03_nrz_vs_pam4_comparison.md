# Worked Problem 03: NRZ vs PAM4 Comparison

## Problem Statement

A design team must choose between NRZ and PAM4 signaling for a 112 Gbps per lane serial link in an AI training cluster. The channel has the following measured insertion loss profile:

- At 28 GHz: 18 dB
- At 56 GHz: 38 dB

The SerDes has the following equalization capabilities:
- TX FIR: up to 8 dB effective boost
- CTLE: up to 15 dB peaking (with 5 dB noise enhancement penalty at maximum gain)
- DFE: up to 10 dB ISI cancellation

The transmitter swing is 900 mVpp differential, and the receiver requires a minimum of 15 mV of inner eye height at BER = 1e-6 (PAM4) or 25 mV of eye height at BER = 1e-12 (NRZ without FEC).

Compare the two signaling options and determine which is feasible.

---

## Worked Solution

### Step 1: Define the two options

**Option 1: NRZ at 112 Gbaud** (1 bit/symbol)
- Baud rate: 112 GBd
- Nyquist frequency: 56 GHz
- Channel loss at Nyquist: 38 dB

**Option 2: PAM4 at 56 Gbaud** (2 bits/symbol)
- Baud rate: 56 GBd
- Nyquist frequency: 28 GHz
- Channel loss at Nyquist: 18 dB
- PAM4 penalty: 9.54 dB

### Step 2: Calculate signal at receiver for NRZ option

Transmitter swing: 900 mVpp differential = 450 mV zero-to-peak differential

The NRZ eye amplitude at the receiver before equalization:

```
V_rx_nrz = 450 mV * 10^(-38/20)
V_rx_nrz = 450 mV * 0.0126
V_rx_nrz = 5.66 mV
```

This is the amplitude of the fundamental component at Nyquist. The actual eye opening is worse because the channel also attenuates frequencies between DC and Nyquist.

### Step 3: Apply equalization for NRZ

Maximum equalization capability:

```
TX FIR:           8 dB
CTLE (effective): 15 - 5 = 10 dB (after noise penalty)
DFE:              10 dB
Total:            28 dB
```

But the channel loss is 38 dB at Nyquist. The equalization deficit is:

```
Deficit = 38 - 28 = 10 dB
```

The equalized signal at the receiver (approximate):

```
V_rx_nrz_eq = 450 mV * 10^(-(38-28)/20)
V_rx_nrz_eq = 450 mV * 10^(-10/20)
V_rx_nrz_eq = 450 mV * 0.316
V_rx_nrz_eq = 142 mV (very rough approximation)
```

However, this simplified calculation is misleading because: (a) at 38 dB loss, the pulse response is severely smeared across many UI, creating ISI that exceeds the DFE's ability to cancel (DFE only cancels post-cursor ISI, not the fundamental attenuation), and (b) the noise enhancement from high-gain CTLE significantly degrades the SNR.

A more realistic assessment: at 38 dB of channel loss, the alternating bit pattern is attenuated by a factor of 79. No practical CTLE can boost this by more than 15 dB without catastrophic noise amplification. The effective eye opening after equalization would be well below the 25 mV requirement.

**Conclusion: NRZ at 112 GBd is NOT feasible for this channel.**

### Step 4: Calculate signal at receiver for PAM4 option

PAM4 eye amplitude before equalization. The inner eye height for PAM4 is 1/3 of the equivalent NRZ eye:

```
V_full_swing = 450 mV (zero-to-peak)
V_inner_eye_pam4 = V_full_swing / 3 = 150 mV (before channel loss)

V_rx_pam4 = 150 mV * 10^(-18/20)
V_rx_pam4 = 150 mV * 0.126
V_rx_pam4 = 18.9 mV (at Nyquist component)
```

### Step 5: Apply equalization for PAM4

The equalization must compensate for 18 dB of channel loss:

```
TX FIR:           8 dB
CTLE (effective): 15 - 3 = 12 dB (lower CTLE gain needed, less noise penalty)
DFE:              10 dB
Total available:  30 dB
```

Since only 18 dB of equalization is needed, the system has headroom:

```
Equalization margin = 30 - 18 = 12 dB
```

With optimized equalization settings (not maximum gain):

```
TX FIR: 5 dB (conservative setting)
CTLE: 8 dB peaking (with only 2 dB noise penalty, so 6 dB effective)
DFE: 7 dB
Total applied: 18 dB (matching channel loss)
```

Estimated equalized inner eye height:

```
V_equalized = 150 mV * 10^(-(18-18)/20) = 150 mV
```

In practice, residual ISI, noise, and jitter reduce the eye. A realistic estimate:

```
V_equalized_practical ≈ 150 mV * 0.25 = 37.5 mV
```

This accounts for residual ISI, crosstalk, and noise that the equalization cannot fully cancel.

### Step 6: Check margin

```
PAM4 inner eye height: ~37.5 mV
Receiver requirement: 15 mV at BER = 1e-6

Voltage margin = 37.5 - 15 = 22.5 mV
Margin ratio = 20 * log10(37.5 / 15) = 8.0 dB
```

### Step 7: Summary comparison

| Parameter | NRZ (112 GBd) | PAM4 (56 GBd) |
|-----------|---------------|----------------|
| Nyquist frequency | 56 GHz | 28 GHz |
| Channel loss at Nyquist | 38 dB | 18 dB |
| PAM4 penalty | N/A | 9.54 dB |
| Effective impairment | 38 dB | 27.54 dB |
| Max equalization | 28 dB | 30 dB |
| Feasible? | No | Yes |
| Target BER (pre-FEC) | 1e-12 | 1e-6 |
| FEC required? | N/A | Yes (mandatory) |
| Estimated margin | Negative | ~8 dB |

### Result

PAM4 at 56 GBd is the only feasible option for this channel. Despite the 9.54 dB PAM4 penalty, the 20 dB reduction in channel loss (38 dB to 18 dB) more than compensates. The "crossover point" where PAM4 becomes advantageous occurs when:

```
IL(2 * f_Nyquist_PAM4) - IL(f_Nyquist_PAM4) > 9.54 dB
38 - 18 = 20 dB > 9.54 dB (satisfied by a large margin)
```

### Key Takeaways

1. The channel loss slope between the NRZ and PAM4 Nyquist frequencies is the deciding factor.
2. For this channel, the loss increases by 20 dB from 28 to 56 GHz, far exceeding the 9.54 dB PAM4 penalty.
3. PAM4 requires FEC, adding approximately 100 ns latency and 5.8% bandwidth overhead, but this is universally accepted for 112G links.
4. The PAM4 link has sufficient margin (~8 dB) to absorb manufacturing variation and temperature effects.
5. In AI systems, this tradeoff explains why all 112G interconnects (NVLink, PCIe Gen5/6, UCIe) use PAM4.
