# Worked Problem: Retimer Placement

## Problem Statement

An AI server connects a CPU to a GPU with a PCIe Gen5 x16 link (32 GT/s NRZ, 16 GHz Nyquist). The GPU sits in a CEM slot on a riser card. The path, with insertion loss at 16 GHz:

- CPU (root complex) package: 9.0 dB
- Motherboard trace: 24 inches at 0.75 dB/inch = 18.0 dB (Megtron 6, as in Problem 01)
- Motherboard-to-riser connector: 1.5 dB
- Riser trace: 2 inches at 0.75 dB/inch = 1.5 dB
- CEM connector: 1.5 dB
- GPU add-in card, including its package: 9.5 dB

The PCIe 5.0 Base Specification allows 36 dB bump-to-bump at 16 GHz, and a retimer resets that budget on each side of it. Determine whether a retimer is needed, where to place it, and what it costs. Assume the retimer's own package adds 1.5 dB on each side (illustrative).

---

## Worked Solution

### Step 1: Check the end-to-end budget

Total loss = 9.0 + 18.0 + 1.5 + 1.5 + 1.5 + 9.5 = **41.0 dB**, which is **5.0 dB over** the 36 dB budget. Designers normally also keep 10–20 % of the budget (3.6–7.2 dB) as margin for temperature, humidity and manufacturing spread. That makes the practical target 28.8–32.4 dB. The link cannot be fixed with material or connector upgrades alone: Problem 01's best combination recovered 3.4 dB, and this link needs at least 5.0 dB before any margin. A retimer is required.

The CPU package (9.0 dB), CEM connector (1.5 dB) and add-in card (9.5 dB) figures are the standard CEM 5.0 allocations. They leave 16.0 dB for the system board, and this board spends 22.5 dB (18.0 + 1.5 + 1.5 + 1.5).

### Step 2: Write each segment's loss as a function of placement

Put the retimer on the motherboard, x inches from the CPU (0 ≤ x ≤ 24):

- Segment 1 (CPU → retimer) = 9.0 + 0.75x + 1.5 = **10.5 + 0.75x dB**
- Segment 2 (retimer → GPU) = 1.5 + 0.75(24 − x) + 1.5 + 1.5 + 1.5 + 9.5 = **33.5 − 0.75x dB**

Each segment must be ≤ 36 dB. With a 20 % margin (≤ 28.8 dB):

- Segment 1: x ≤ (28.8 − 10.5) / 0.75 = 24.4 in, so all of the motherboard qualifies.
- Segment 2: x ≥ (33.5 − 28.8) / 0.75 = 6.3 in.

Any position from **6.3 to 24 inches** from the CPU keeps both segments at or below 28.8 dB.

### Step 3: Evaluate candidate positions

**Option A — balanced, on the motherboard.** Setting 10.5 + 0.75x = 33.5 − 0.75x gives x = 23.0 / 1.5 = **15.3 inches**. Both segments are then **22.0 dB**, leaving 14.0 dB of margin on each side. This maximises the worst-case margin.

**Option B — next to the CPU (x = 2 in).** Segment 1 = 12.0 dB and segment 2 = 32.0 dB. The worst margin is 4.0 dB (11 %), within spec but at the bottom of the 10–20 % guideline. Almost all the loss is on the GPU side, where the add-in card's contribution varies from card to card.

**Option C — on the riser card, next to the riser connector.** Segment 1 = 9.0 + 18.0 + 1.5 + 1.5 = 30.0 dB. Segment 2 = 1.5 + 1.5 + 1.5 + 9.5 = 14.0 dB. The worst margin is 6.0 dB (17 %). This passes, and it puts the retimer only on systems that use the riser, but the motherboard side is left with little headroom.

**Sensitivity check (Option A).** If the laminate's loss rises from 0.75 to 0.80 dB/inch (hot and humid), segment 1 becomes 9.0 + 0.80 × 15.3 + 1.5 = 22.8 dB and segment 2 becomes 22.5 dB. Both are still more than 13 dB under 36 dB.

### Step 4: Check latency, power and cost

- **Latency:** The Base Specification caps retimer-added latency at 64 ns (non-SRIS, 5–32 GT/s). Retimers in their low-latency path achieve about 10 ns. A read crosses the retimer twice (request and completion), so it adds about 20 ns. That is small compared with a DMA transfer measured in microseconds.
- **Power:** About 4 W for an x16 retimer (illustrative, as in Problem 01). Place it where it gets airflow and not in the GPU's exhaust, because its SerDes margin is characterised only up to its maximum junction temperature.
- **Cost:** About +$15–25 per link in BOM (illustrative), plus board area for a large BGA and a 100 MHz reference clock.

### Step 5: Rule out the alternatives

- **Redriver:** A redriver is linear, so it does not reset the budget: the 41.0 dB end-to-end channel still has to be equalised by the GPU's receiver. The Base Specification defines retimers but not redrivers, so interoperability depends on vendor tuning. The chapter notes treat Gen5 redrivers as marginal.
- **Second retimer:** The specification allows up to two retimers per link, but one retimer supports up to 36 dB on each side (72 dB in total). A 41 dB channel does not need a second one.

### Result

| Placement | Segment 1 | Segment 2 | Worst Margin to 36 dB | Meets 20 % Guideline? | Notes |
|-----------|-----------|-----------|----------------------|------------------------|-------|
| No retimer | 41.0 dB (end-to-end) | — | −5.0 dB | No | Fails |
| A: Motherboard, 15.3 in | 22.0 dB | 22.0 dB | 14.0 dB | Yes | Recommended |
| B: Motherboard, 2 in | 12.0 dB | 32.0 dB | 4.0 dB | No | Margin concentrated on the GPU side |
| C: Riser card | 30.0 dB | 14.0 dB | 6.0 dB | No | Modular; tight on the CPU side |

All the retimer options add about 4 W, about +$15–25 and 10–64 ns of latency per crossing.

### Key Takeaways

1. A retimer splits one budget into two independent ones. Placement decides how the loss is shared between them.
2. Write each segment's loss as a function of position. The feasible window and the balanced point then follow directly.
3. Include the retimer's own package loss in both segments.
4. Balancing the segments maximises worst-case margin. Riser placement trades margin for modularity.
5. Check latency against the 64 ns specification limit, and plan the retimer's power, cooling and reference clock at the same time as its position.
