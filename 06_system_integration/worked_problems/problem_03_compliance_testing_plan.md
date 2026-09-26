# Worked Problem: Compliance Testing Plan

## Problem Statement

The retimed PCIe Gen5 x16 link from Problem 02 (Option A: CPU → 15.3 in → retimer → GPU riser, each segment 22.0 dB at 16 GHz) is going into production. Write the validation and compliance test plan for the motherboard. For each test, state the instrument, the pass limit and its source, and the expected result. Then work out how long the BER testing will take. The lab can test 9 environmental corners: 3 supply voltages × 3 temperatures.

---

## Worked Solution

### Step 1: Collect the pass limits

| Parameter | Limit | Source |
|-----------|-------|--------|
| Channel insertion loss, per segment | ≤ 36 dB bump-to-bump at 16 GHz | PCIe 5.0 Base Spec |
| Bit error ratio | ≤ 10⁻¹² | PCIe 5.0 Base Spec |
| Post-equalisation eye at the reference receiver | EH ≥ 15 mV, EW ≥ 0.3 UI at BER ≤ 10⁻¹² | PCIe 5.0 Base Spec |
| Reference clock jitter | ≤ 150 fs RMS after each of the 16 specified 32 GT/s filter functions | PCIe 5.0 Base Spec |
| Retimer-added latency | ≤ 64 ns (non-SRIS) | PCIe Base Spec |
| Slot electrical compliance | PCI-SIG CEM 5.0 tests, run through the Compliance Load Board (CLB) | PCI-SIG CEM 5.0 |

At 32 GT/s, 1 UI = 1 / 32 GHz = **31.25 ps**, so the eye-width limit is 0.3 × 31.25 = **9.375 ps**.

### Step 2: Pre-silicon and bare-board checks

1. **Channel simulation.** Simulate each segment separately (CPU → retimer, retimer → GPU) with IBIS-AMI models, the way retimed links are analysed. Each segment has a 22.0 dB budget, which leaves 14.0 dB of margin before hardware exists.
2. **VNA on the bare board.** Measure SDD21 on each segment's coupon or probe points and de-embed the fixture. If the laminate measures 0.80 dB/inch instead of 0.75, the segments become 22.8 dB and 22.5 dB. Both still pass with more than 13 dB to spare.
3. **TDR.** Check the differential impedance stays within 100 Ω ± 10 % (90–110 Ω) through the breakout vias, the retimer BGA and the riser connector.

### Step 3: Reference clock

Measure the 100 MHz reference clock at the retimer and at the CEM slot with a phase-noise analyser, and apply all 16 specified filter functions. If the worst filtered result is, say, 110 fs RMS (illustrative), the margin is (150 − 110) / 150 = **27 %**. A retimer regenerates the data, not the reference clock, so a noisy clock buffer feeding both segments hurts both of them.

### Step 4: Eye margin at the receiver (dual-Dirac)

Suppose the post-equalisation receiver eye in simulation (or from the scope with the reference equaliser applied) shows DJ = 4.0 ps and RJ = 0.30 ps RMS (illustrative). At BER = 10⁻¹², Q = 7.034, and:

TJ = DJ + 2Q · σ_RJ = 4.0 + 14.07 × 0.30 = **8.22 ps**

EW = 31.25 − 8.22 = **23.0 ps = 0.74 UI**, against the 0.3 UI limit. The eye height is checked against 15 mV in the same way.

### Step 5: BER test time

To show BER ≤ 10⁻¹² at 95 % confidence with zero errors, you need N = −ln(1 − 0.95) / 10⁻¹² = **3.0 × 10¹²** bits per lane.

- At 32 Gb/s per lane that takes 3.0 × 10¹² / 32 × 10⁹ = **94 s**. All 16 lanes, and both segments, can run at once using the retimer's and endpoint's pattern checkers, so one corner takes about 94 s.
- 9 corners × 94 s = 842 s ≈ **14 minutes per board**.
- Showing 10⁻¹⁵ the same way would take 1000× longer: 26 hours per corner, or 234 hours (9.8 days) for the matrix. Deep BER is therefore not measured directly. It is inferred from margin.

### Step 6: Margin, latency and system tests

1. **Lane Margining at the Receiver** (mandatory for Gen4 and later) on every lane of both segments, at every corner. It reports how far the sampler can move in time and voltage before errors appear. Record the worst lane per corner, and compare boards so that a marginal assembly shows up before it fails in the field.
2. **Retimer latency.** Measure with the retimer vendor's test mode or a protocol analyser, and check against 64 ns. About 10 ns is expected in low-latency mode.
3. **CEM compliance.** Plug the PCI-SIG Compliance Load Board into the slot and capture the Tx signal from the retimer. Then take the board to a PCI-SIG compliance workshop for the Integrators List.
4. **System soak.** Run GPU training traffic for 72–168 hours, logging corrected-error counts (AER) and link retrains. Add power and thermal cycling to check that the link comes back up after resets.

### Result

| Test | Instrument | Limit | Expected | Margin |
|------|------------|-------|----------|--------|
| Segment IL | VNA | ≤ 36 dB | 22.0 dB (22.8 dB worst) | ≥ 13.2 dB |
| Impedance | TDR | 90–110 Ω | — | — |
| RefClk jitter | Phase-noise analyser | ≤ 150 fs RMS | 110 fs (illustrative) | 27 % |
| Eye width | Scope / simulation | ≥ 0.3 UI | 0.74 UI | 0.44 UI |
| BER, 9 corners | On-chip checkers | ≤ 10⁻¹² | 0 errors in 3.0 × 10¹² bits | 95 % confidence |
| Retimer latency | Protocol analyser | ≤ 64 ns | ~10 ns | ~54 ns |
| Soak | Workload + AER logs | No uncorrected errors | — | — |

Illustrative equipment cost for an in-house Gen5 lab: $0.5–1M for a ≥ 50 GHz real-time scope, a 32 GT/s BERT and a 4-port VNA. The alternative is renting time at a test house, typically +$5–15k per board spin (also illustrative).

### Key Takeaways

1. Start from a limits table with a source for every row. A plan without cited limits cannot pass or fail anything.
2. With a retimer, test each segment against its own budget. The CPU → retimer and retimer → GPU segments are separate compliance problems.
3. The zero-error BER bound (N = −ln(1 − CL) / BER) turns a BER target into test time: 94 s per corner at 10⁻¹², but days at 10⁻¹⁵.
4. Deep BER is inferred from lane margining, eye extrapolation and long soak tests, not measured directly.
5. The reference clock is shared by both segments. It is a common-mode risk that retiming does not remove.
