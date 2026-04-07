# Worked Problem 02: Driver Impedance Matching

## Problem Statement

A voltage-mode SST (Source-Series Terminated) driver is designed for a 112G PAM4 SerDes lane. The target output impedance is 50 ohms per side (100 ohms differential) to match a 100-ohm differential transmission line. The driver uses NMOS transistors in a 5nm FinFET process with the following parameters:

- Each unit cell has an on-resistance of 400 ohms at nominal PVT (process, voltage, temperature)
- At slow corner (SS, 0.72V, 125C): on-resistance increases by 40%
- At fast corner (FF, 0.88V, -40C): on-resistance decreases by 30%
- The driver has 32 binary-weighted segments per side
- The calibration resolution is 1 segment (1 unit cell can be enabled/disabled)

Determine: (a) the nominal number of segments needed for 50 ohms, (b) the impedance range achievable with calibration, and (c) whether the calibration range covers the full PVT variation.

---

## Worked Solution

### Step 1: Calculate nominal segment count

Each unit cell has R_on = 400 ohms. When N cells are connected in parallel, the total resistance is:

```
R_total = R_unit / N = 400 / N ohms
```

For a target of 50 ohms:

```
50 = 400 / N
N = 400 / 50 = 8 segments
```

At nominal PVT, 8 segments provide exactly 50 ohms per side.

### Step 2: Calculate impedance at PVT corners

**Slow corner (SS, 0.72V, 125C):**

```
R_unit_slow = 400 * 1.40 = 560 ohms
R_total_slow (8 segments) = 560 / 8 = 70 ohms
```

This is 40% above the 50-ohm target, causing severe impedance mismatch.

**Fast corner (FF, 0.88V, -40C):**

```
R_unit_fast = 400 * 0.70 = 280 ohms
R_total_fast (8 segments) = 280 / 8 = 35 ohms
```

This is 30% below the 50-ohm target.

### Step 3: Calculate calibration range

With 32 available segments and calibration resolution of 1 segment:

**Minimum impedance (all 32 segments enabled):**

At nominal: R_min = 400/32 = 12.5 ohms
At slow: R_min_slow = 560/32 = 17.5 ohms
At fast: R_min_fast = 280/32 = 8.75 ohms

**Maximum impedance (1 segment enabled):**

At nominal: R_max = 400/1 = 400 ohms
At slow: R_max_slow = 560/1 = 560 ohms
At fast: R_max_fast = 280/1 = 280 ohms

### Step 4: Find the calibration code for each corner

**Nominal (400 ohms/unit):**

```
N_nom = 400 / 50 = 8 segments
```

**Slow corner (560 ohms/unit):**

```
N_slow = 560 / 50 = 11.2 segments -> round to 11
R_actual_slow = 560 / 11 = 50.9 ohms (1.8% error)
```

Or with 12 segments: R = 560/12 = 46.7 ohms (6.7% error). Choose N = 11 for smaller error.

**Fast corner (280 ohms/unit):**

```
N_fast = 280 / 50 = 5.6 segments -> round to 6
R_actual_fast = 280 / 6 = 46.7 ohms (6.7% error)
```

Or with 5 segments: R = 280/5 = 56.0 ohms (12% error). Choose N = 6 for smaller error.

### Step 5: Check calibration range coverage

Required segment range: 6 (fast) to 11 (slow) out of 32 available.

```
Minimum segments needed: 6
Maximum segments needed: 11
Available range: 1 to 32
```

The calibration range easily covers the PVT variation. The unused segments (12-32) provide margin for even more extreme conditions or for aging effects.

### Step 6: Analyze impedance accuracy

| Corner | Segments | Impedance | Error |
|--------|----------|-----------|-------|
| Nominal | 8 | 50.0 ohms | 0.0% |
| Slow | 11 | 50.9 ohms | +1.8% |
| Fast | 6 | 46.7 ohms | -6.7% |

The fast corner has the largest error (6.7%) because the unit resistance at this corner (280 ohms) does not divide evenly by values that yield 50 ohms. The specification typically allows plus or minus 10-15%, so 6.7% is within spec but uses a significant portion of the budget.

### Step 7: Calculate reflection coefficient

The reflection coefficient for each corner:

```
rho = (Z_driver - Z_line) / (Z_driver + Z_line)
```

| Corner | Z_driver | rho | Return Loss |
|--------|----------|-----|-------------|
| Nominal | 50.0 | 0.000 | infinity |
| Slow | 50.9 | 0.009 | 41 dB |
| Fast | 46.7 | -0.034 | 29 dB |

All corners achieve better than 20 dB return loss, which is well within specification. The fast corner is the worst case with 29 dB return loss.

### Step 8: Consider PAM4 impedance variation

For PAM4, the driver impedance varies with the output level because different numbers of pull-up and pull-down segments are active for each level. For level 3 (maximum output), all segments pull up and the impedance is dominated by the PMOS (or the pull-up NMOS in a full-swing design). For level 0, all segments pull down.

The intermediate levels (1 and 2) have a different effective impedance because some segments pull up and others pull down, and the parallel combination depends on both the pull-up and pull-down on-resistances.

Assuming the pull-up and pull-down unit cells have the same on-resistance (ideally matched):

Level 3: N_up = 8, N_down = 0 -> Z = 400/8 = 50 ohms (from pull-ups only)
Level 2: N_up = 5.33, N_down = 2.67 -> effective Z depends on the parallel combination
Level 1: N_up = 2.67, N_down = 5.33 -> symmetric to level 2
Level 0: N_up = 0, N_down = 8 -> Z = 400/8 = 50 ohms (from pull-downs only)

The intermediate levels have a lower effective impedance (because both pull-up and pull-down contribute in parallel), which causes a level-dependent impedance variation. This is mitigated by adding impedance compensation segments that are always on (connected to the common-mode voltage) to maintain constant output impedance across all levels.

### Result

The calibration system with 32 segments can cover the full PVT range with worst-case impedance error of 6.7% (at the fast corner). The return loss exceeds 29 dB across all corners. For production use, a finer calibration resolution (such as using 64 segments or a combination of coarse and fine adjustment) would reduce the fast-corner error.

### Key Takeaways

1. PVT variation can cause 40% or more change in transistor on-resistance.
2. Calibration with 32 segments provides more than adequate range for this PVT spread.
3. The quantization error from integer segment counts limits the achievable impedance accuracy.
4. PAM4 level-dependent impedance variation requires additional compensation.
5. Fast corner (low resistance) is typically the hardest to calibrate because fewer segments are used, giving coarser resolution.
