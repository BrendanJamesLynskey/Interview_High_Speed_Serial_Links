# Worked Problem: Compliance Testing Plan

## Problem Statement

An AI server motherboard must route a PCIe Gen5 x16 link from the CPU to a GPU across 12 inches of PCB trace, through one mid-board connector. The COM analysis of the initial design shows COM = 2.1 dB, which is below the 3 dB requirement. The channel breakdown is: TX package 1.5 dB, PCB trace 9.0 dB (12 inches at 0.75 dB/inch at 16 GHz on Megtron 6), connector 2.0 dB, RX package 1.5 dB, total 14.0 dB at Nyquist. Determine what design changes can restore COM to >= 3 dB.

---

## Worked Solution

### Step 1: Identify the dominant loss contributor

The PCB trace accounts for 9.0 / 14.0 = 64% of the total loss. This is the largest single contributor and the most amenable to optimization.

### Step 2: Evaluate Option A - Reduce trace length

Reducing the trace length from 12 to 8 inches saves 4 * 0.75 = 3.0 dB of insertion loss. This would reduce the total channel loss to 11.0 dB. Based on the COM sensitivity of approximately 1 dB COM per 2 dB of insertion loss improvement, this should improve COM by approximately 1.5 dB, from 2.1 to approximately 3.6 dB. This exceeds the 3 dB requirement.

However, reducing trace length by 4 inches may require component placement changes that affect the overall board layout.

### Step 3: Evaluate Option B - Upgrade PCB material

Switching from Megtron 6 (0.75 dB/inch at 16 GHz) to Megtron 7 (0.55 dB/inch at 16 GHz) saves 12 * (0.75 - 0.55) = 2.4 dB. New total: 11.6 dB. COM improvement approximately 1.2 dB, from 2.1 to approximately 3.3 dB. This barely meets the 3 dB requirement with minimal margin.

### Step 4: Evaluate Option C - Add a retimer

A retimer divides the channel into two segments. If placed at the midpoint: segment 1 = 7.0 dB (TX pkg + 6 inches trace), segment 2 = 7.0 dB (6 inches trace + connector + RX pkg). Each segment easily achieves COM >= 5 dB.

Cost: retimer power (~4W for x16), latency (+5 ns), BOM cost (+5-25).

### Step 5: Evaluate Option D - Improve connector

Replacing the 2.0 dB connector with a 1.0 dB connector saves 1.0 dB. COM improvement approximately 0.5 dB, from 2.1 to approximately 2.6 dB. This alone is insufficient.

### Step 6: Recommend combined approach

Best approach for minimal disruption: combine B (material upgrade) + D (connector upgrade):
- Material upgrade: -2.4 dB
- Connector upgrade: -1.0 dB  
- Total improvement: -3.4 dB
- New total loss: 10.6 dB
- Expected COM: approximately 3.8 dB (margin of 0.8 dB above requirement)

### Result

| Option | Loss Reduction | New COM | Meets 3 dB? | Cost Impact |
|--------|---------------|---------|-------------|-------------|
| A: Shorter trace | 3.0 dB | ~3.6 dB | Yes | Layout change |
| B: Better material | 2.4 dB | ~3.3 dB | Barely | +30% PCB cost |
| C: Add retimer | N/A (two segments) | ~5+ dB each | Yes | +4W, +0, +5ns |
| D: Better connector | 1.0 dB | ~2.6 dB | No | + |
| B+D: Material + connector | 3.4 dB | ~3.8 dB | Yes | +35% PCB, + |

### Key Takeaways

1. PCB trace loss is the dominant contributor and the primary optimization target.
2. Material upgrade provides the most improvement without layout changes.
3. Retimers are the most reliable solution but add power, cost, and latency.
4. Combining multiple small improvements can achieve the target without any single large change.
5. COM margin of 0.8 dB above the requirement provides adequate manufacturing tolerance.
