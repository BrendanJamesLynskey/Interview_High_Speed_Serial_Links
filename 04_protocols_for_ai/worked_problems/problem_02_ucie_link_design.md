# Worked Problem 02: UCIe Link Design

## Problem Statement

An AI accelerator chiplet architecture requires a UCIe advanced package module link between a compute chiplet and a memory controller chiplet on a silicon interposer. The requirements are: 800 GB/s bidirectional bandwidth, less than 3 ns one-way latency, bump pitch of 45 um, and power budget of 2W for the link.

Determine the number of lanes, the required die edge allocation, and verify the power and latency budgets.

---

## Worked Solution

### Step 1: Calculate the number of lanes needed

Bidirectional bandwidth: 800 GB/s = 400 GB/s each direction.

```
Per-lane rate: 32 Gbps = 4 GB/s
Lanes per direction: 400 GB/s / 4 GB/s = 100 lanes
Total lanes (both directions): 200 data lanes
```

### Step 2: Calculate bump count including overhead

UCIe advanced package uses single-ended signaling, so each lane requires 1 signal bump. Additional bumps needed for ground (approximately 1 ground per 2 signals), clock (forwarded clock, 2 clock bumps per group of ~16 data lanes), spare lanes (5-10% redundancy), and power bumps.

```
Data bumps: 200
Ground bumps: ~100 (1 per 2 data bumps)
Clock bumps: ~26 (200/16 * 2 for differential clock pairs)
Spare bumps: ~20 (10% of data)
Power bumps: ~40
Total bumps: ~386
```

### Step 3: Calculate die edge allocation

At 45 um bump pitch, assuming bumps arranged in rows along the die edge:

For a 2-row bump array (typical for UCIe):
```
Bumps per mm of edge (2 rows): 2 / 0.045 mm = 44.4 bumps/mm
Edge length needed: 386 / 44.4 = 8.7 mm
```

For a 4-row bump array (for higher density):
```
Bumps per mm of edge (4 rows): 4 / 0.045 mm = 88.9 bumps/mm
Edge length needed: 386 / 88.9 = 4.3 mm
```

### Step 4: Verify bandwidth density

```
BW density (2-row): 800 GB/s / 8.7 mm = 91.9 GB/s/mm = 0.735 Tbps/mm
BW density (4-row): 800 GB/s / 4.3 mm = 186 GB/s/mm = 1.49 Tbps/mm
```

The 4-row configuration exceeds the UCIe advanced package target of 1.3 Tbps/mm, confirming the design is feasible but aggressive. The 2-row configuration at 0.735 Tbps/mm is more conservative and easier to implement.

### Step 5: Verify power budget

UCIe advanced package power: 0.5-1.0 pJ/bit (typical for short-channel, low-swing links).

```
Total data rate: 800 GB/s = 6.4 Tbps
Power at 0.5 pJ/bit: 6.4e12 * 0.5e-12 = 3.2 W
Power at 1.0 pJ/bit: 6.4e12 * 1.0e-12 = 6.4 W
Power at 0.75 pJ/bit (typical): 6.4e12 * 0.75e-12 = 4.8 W
```

The 2W power budget is insufficient for 800 GB/s at typical UCIe power efficiency. The design must either: reduce the target bandwidth to approximately 400 GB/s (which fits within 2W at 0.5 pJ/bit), use more aggressive low-power design (targeting 0.3 pJ/bit), or increase the power budget to 4-5W.

### Step 6: Verify latency budget

UCIe advanced package latency components:
```
Serialization (32:1 MUX): ~0.5 ns
TX driver + channel propagation (2 mm at 6 ps/mm): ~12 ps
RX sampling + deserialization: ~0.5 ns
D2D adapter (FLIT processing): ~1.0 ns
Total one-way: ~2.0 ns
```

This is within the 3 ns budget with 1 ns margin.

### Result

| Parameter | Value | Budget | Status |
|-----------|-------|--------|--------|
| Lanes per direction | 100 | - | - |
| Total bumps | ~386 | - | - |
| Die edge (2-row) | 8.7 mm | - | Feasible |
| Die edge (4-row) | 4.3 mm | - | Aggressive |
| Power | 4.8 W (typical) | 2 W | Exceeds budget |
| Latency | 2.0 ns | 3 ns | Within budget |

### Key Takeaways

1. 800 GB/s bidirectional UCIe requires approximately 200 data lanes at 32 Gbps each.
2. The die edge allocation (4.3-8.7 mm) is a significant fraction of a typical chiplet edge.
3. Power is the binding constraint: 800 GB/s at typical UCIe efficiency requires 4-5W, exceeding the 2W budget.
4. Latency (2 ns) comfortably meets the 3 ns requirement.
5. Reducing power to meet budget requires either lower bandwidth or advanced low-power I/O design.
