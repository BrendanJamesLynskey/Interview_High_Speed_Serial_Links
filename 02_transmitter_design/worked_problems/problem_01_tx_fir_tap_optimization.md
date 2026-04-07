# Worked Problem 01: TX FIR Tap Optimization

## Problem Statement

A 112 Gbps PAM4 serial link (56 GBd) uses a 3-tap TX FIR filter with one pre-cursor tap (c[-1]), one main cursor tap (c[0]), and one post-cursor tap (c[1]). The channel pulse response (after normalization) at the sampling instants is:

- h[-1] = 0.08 (pre-cursor ISI)
- h[0] = 0.55 (main cursor)
- h[1] = -0.25 (first post-cursor ISI)
- h[2] = -0.08 (second post-cursor ISI)
- h[3] = -0.04 (third post-cursor ISI)

The tap coefficient constraint is: |c[-1]| + c[0] + |c[1]| = 1.0 (full-scale normalization).

Find the optimal FIR tap coefficients to maximize the main cursor amplitude while zeroing the pre-cursor and first post-cursor ISI. Calculate the resulting equalized pulse response and the residual ISI.

---

## Worked Solution

### Step 1: Understand the equalized pulse response

The equalized pulse response at the receiver is the convolution of the FIR filter coefficients with the channel pulse response:

```
y[n] = sum over k of c[k] * h[n-k]
```

For our 3-tap filter (k = -1, 0, 1):

```
y[n] = c[-1]*h[n+1] + c[0]*h[n] + c[1]*h[n-1]
```

### Step 2: Set up the zero-forcing equations

To zero the pre-cursor ISI at n = -1:

```
y[-1] = c[-1]*h[0] + c[0]*h[-1] + c[1]*h[-2] = 0
```

Since h[-2] = 0 (no energy 2 UI before main cursor):

```
c[-1]*0.55 + c[0]*0.08 = 0
c[-1] = -c[0]*0.08/0.55 = -0.1455*c[0]
```

To zero the first post-cursor ISI at n = 1:

```
y[1] = c[-1]*h[2] + c[0]*h[1] + c[1]*h[0] = 0
c[-1]*(-0.08) + c[0]*(-0.25) + c[1]*0.55 = 0
c[1] = (0.08*c[-1] + 0.25*c[0]) / 0.55
```

### Step 3: Substitute and apply the constraint

Substituting c[-1] = -0.1455*c[0] into the c[1] equation:

```
c[1] = (0.08*(-0.1455*c[0]) + 0.25*c[0]) / 0.55
c[1] = (-0.01164*c[0] + 0.25*c[0]) / 0.55
c[1] = 0.2384*c[0] / 0.55
c[1] = 0.4334*c[0]
```

Apply the normalization constraint:

```
|c[-1]| + c[0] + |c[1]| = 1.0
0.1455*c[0] + c[0] + 0.4334*c[0] = 1.0
1.5789*c[0] = 1.0
c[0] = 0.6333
```

### Step 4: Calculate all tap coefficients

```
c[-1] = -0.1455 * 0.6333 = -0.0921
c[0]  = 0.6333
c[1]  = 0.4334 * 0.6333 = 0.2745
```

Wait - c[1] is positive, which means the post-cursor tap is adding to the signal rather than subtracting. Let us verify: h[1] = -0.25 is negative (the pulse response dips below zero at the first post-cursor), so a positive FIR tap compensates correctly.

Verify constraint: |−0.0921| + 0.6333 + |0.2745| = 0.0921 + 0.6333 + 0.2745 = 0.9999 (approximately 1.0).

### Step 5: Calculate the equalized pulse response

At n = -1 (pre-cursor):
```
y[-1] = (-0.0921)(0.55) + (0.6333)(0.08) + (0.2745)(0)
y[-1] = -0.0507 + 0.0507 + 0 = 0.0000 (zeroed)
```

At n = 0 (main cursor):
```
y[0] = (-0.0921)(−0.25) + (0.6333)(0.55) + (0.2745)(0.08)
y[0] = 0.0230 + 0.3483 + 0.0220 = 0.3933
```

At n = 1 (first post-cursor):
```
y[1] = (-0.0921)(−0.08) + (0.6333)(−0.25) + (0.2745)(0.55)
y[1] = 0.0074 + (−0.1583) + 0.1510 = 0.0001 (approximately zeroed)
```

At n = 2 (second post-cursor, residual ISI):
```
y[2] = (-0.0921)(−0.04) + (0.6333)(−0.08) + (0.2745)(−0.25)
y[2] = 0.0037 + (−0.0507) + (−0.0686) = -0.1156
```

At n = 3 (third post-cursor, residual ISI):
```
y[3] = (-0.0921)(0) + (0.6333)(−0.04) + (0.2745)(−0.08)
y[3] = 0 + (−0.0253) + (−0.0220) = -0.0473
```

### Step 6: Calculate residual ISI and signal-to-ISI ratio

```
Main cursor amplitude: y[0] = 0.3933
Residual ISI: |y[2]| + |y[3]| = 0.1156 + 0.0473 = 0.1629
Signal-to-ISI ratio: 0.3933 / 0.1629 = 2.41 (7.7 dB)
```

### Step 7: Compare to unequalized case

Without FIR (c[0] = 1.0, all others = 0):

```
Main cursor: h[0] = 0.55
Total ISI: |h[-1]| + |h[1]| + |h[2]| + |h[3]| = 0.08 + 0.25 + 0.08 + 0.04 = 0.45
Signal-to-ISI ratio: 0.55 / 0.45 = 1.22 (1.7 dB)
```

### Result

| Metric | Unequalized | FIR Equalized | Improvement |
|--------|-------------|---------------|-------------|
| Main cursor | 0.55 | 0.39 | Reduced (cost of FIR) |
| Total ISI | 0.45 | 0.16 | 64% reduction |
| Signal-to-ISI | 1.22 (1.7 dB) | 2.41 (7.7 dB) | +6.0 dB |
| Pre-cursor ISI | 0.08 | 0.00 | Eliminated |
| 1st post-cursor ISI | 0.25 | 0.00 | Eliminated |

The TX FIR provides a 6 dB improvement in signal-to-ISI ratio by eliminating the pre-cursor and first post-cursor ISI. The main cursor amplitude is reduced from 0.55 to 0.39 (a 3.0 dB penalty in absolute amplitude), but the ISI reduction more than compensates. The remaining residual ISI at n=2 and n=3 must be handled by the receiver's DFE.

### Key Takeaways

1. Zero-forcing the pre-cursor is critical because DFE cannot cancel pre-cursor ISI.
2. The FIR tap coefficients depend on the channel pulse response shape.
3. The main cursor amplitude decreases with more aggressive pre-emphasis.
4. Residual ISI at tap positions beyond the FIR filter length must be handled by the RX DFE.
5. The sign of the optimal tap coefficients depends on the polarity of the channel pulse response at each cursor position.
