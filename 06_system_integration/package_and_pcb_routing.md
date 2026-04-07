# Package and PCB Routing

## Overview

The physical realization of high-speed serial links requires careful package and PCB design to maintain signal integrity at 56 GBd and beyond. This section covers controlled impedance design, BGA breakout routing, PCB stackup, material selection, via optimization, and length matching for AI system interconnects.

---

### Q1. What are the key considerations for controlled impedance trace design in high-speed serial links?

**Answer:**

Controlled impedance design ensures that the characteristic impedance of the differential pair remains at the target value (typically 85 or 100 ohms) throughout the signal path. The impedance is determined by the trace width, spacing, dielectric height, dielectric constant, and copper thickness, and all of these parameters must be controlled within tight tolerances.

For stripline differential pairs (the preferred geometry for high-speed serial links), the impedance is calculated from the geometry using field-solver tools (such as Polar SI9000, Ansys 2D Extractor, or Cadence PowerSI). The key geometric parameters are: trace width (W, typically 3-5 mils for 100-ohm differential), trace spacing within the pair (S, typically 5-8 mils), dielectric height above and below the trace (H, typically 3-5 mils), copper thickness (T, typically 0.5-1.0 mil for signal layers), and dielectric constant (Dk, typically 3.2-3.8 for low-loss materials).

The impedance tolerance for high-speed serial links is typically plus or minus 10% of the target. For a 100-ohm target, this means the impedance must stay between 90 and 110 ohms. The tolerance is consumed by manufacturing variations in trace width (plus or minus 0.5-1.0 mil), dielectric height (plus or minus 0.5-1.0 mil), and dielectric constant (plus or minus 5-10% across the board).

For AI server boards with dozens of high-speed lanes, impedance uniformity across the board is critical. The dielectric constant can vary by 3-5% across a large panel (24x18 inches), creating systematic impedance variations. Designers use impedance simulation at the panel level to identify regions where the impedance may be out of specification and adjust the routing or trace geometry accordingly.

---

### Q2. Describe the BGA breakout routing strategy for high-speed SerDes signals.

**Answer:**

BGA (Ball Grid Array) breakout routing is the process of connecting the BGA ball pads to the traces on the PCB signal layers. For high-speed SerDes signals, the breakout is critical because the transition from the BGA pad to the trace introduces impedance discontinuities, and the dense pin field creates crosstalk between adjacent signals.

The breakout strategy depends on the BGA pitch and the number of signal layers available. For 0.8-1.0 mm pitch BGAs (typical for AI accelerators), the outer rows of balls can be escaped on the top signal layers (directly routing from the pad to the trace without a via), while inner rows require vias to deeper signal layers. The via transition creates a significant impedance discontinuity that must be managed through anti-pad optimization and back-drilling.

Dog-bone breakout is the most common pattern: a short trace (dog-bone) connects the BGA pad to an offset via, which transitions to the signal layer. The dog-bone length and width must be controlled to minimize the impedance perturbation. The via anti-pad diameter on each ground plane must be optimized to maintain the target impedance through the via transition.

For differential pairs, the two signals of the pair should be routed together through the breakout, maintaining matched lengths and consistent spacing. The pair should transition through their vias at the same location (side-by-side vias) to minimize intra-pair skew. The via spacing between the two vias of a differential pair affects the differential impedance through the via, and must be designed to match the trace impedance.

The breakout region is typically the most challenging part of the PCB for signal integrity because the routing constraints are tightest (high density, limited space for spacing and shielding) and the signal transitions (pad to trace, via layer change) are concentrated. 3D electromagnetic simulation of the breakout region (using tools like Ansys HFSS or CST) is standard practice for 56 GBd and above designs.

---

### Q3. How does PCB stackup design affect high-speed serial link performance?

**Answer:**

The PCB stackup defines the layer sequence, dielectric thicknesses, copper weights, and material selection that determine the electrical performance of all traces on the board. For AI server boards with high-speed serial links, the stackup is one of the most impactful design decisions.

A typical 24-layer stackup for an AI server board might include 4 high-speed signal layers (for NVLink, PCIe, and Ethernet traces), 4 low-speed signal layers (for DDR, SPI, I2C, and other interfaces), 8 ground reference planes (providing return current paths and shielding for the signal layers), 4 power planes (for different voltage domains), and 4 additional signal or power layers.

The high-speed signal layers are placed in stripline configuration, sandwiched between two ground planes. The dielectric thickness between the signal layer and each ground plane (typically 3-5 mils) determines the trace impedance and the coupling to the ground planes. Thinner dielectric provides tighter field confinement (better shielding, less crosstalk) but may require narrower traces for the same impedance, increasing manufacturing sensitivity.

The ground reference plane integrity is essential: no routing should cross the ground plane under a high-speed signal trace, and the ground planes should be as continuous as possible. Any gap, slot, or via clearance in the ground plane disrupts the return current path and creates a radiation point that increases EMI and crosstalk.

Material selection for the high-speed layers uses low-loss laminates (Megtron 6 or 7 class) while the low-speed and power layers can use standard materials (mid-loss or even standard FR4). Mixed-material stackups reduce cost compared to using premium material throughout, but the interface between different materials must be managed (different CTE can cause delamination under thermal cycling).

---

### Q4. What is anti-pad optimization and why is it critical for via transitions?

**Answer:**

The anti-pad is the clearance hole in a ground or power plane around a through-hole via. The anti-pad provides electrical clearance to prevent shorting the via to the plane, but it also creates a discontinuity in the plane that affects the impedance and signal integrity of the via transition.

Anti-pad optimization adjusts the anti-pad diameter on each plane to achieve the target impedance through the via. A larger anti-pad reduces the capacitance between the via and the plane (raising the impedance), while a smaller anti-pad increases the capacitance (lowering the impedance). The via barrel and pad add inductance, and the net impedance depends on the balance of all these elements.

For a typical differential via pair in a 24-layer PCB, the impedance through the via transition varies with the anti-pad design. With uniform anti-pads on all layers (a common default), the impedance may dip significantly below the target at the layers where the signal connects (due to the pad capacitance) and may vary on the non-functional layers (where the via passes through without connecting).

Optimized anti-pads use different diameters on different layers: larger anti-pads on the layers where the signal connects (to compensate for the pad capacitance), and tuned anti-pads on the non-functional layers (to maintain a consistent impedance). The anti-pad on each layer is sized using 3D electromagnetic simulation, targeting an impedance variation of less than plus or minus 5 ohms through the entire via transition.

Non-functional pads (NFPs, the pads on layers where the via does not connect) should be removed entirely if manufacturing rules allow, because they add unnecessary capacitance. Removing NFPs can improve the via impedance match by 5-10 ohms and improve the via return loss by 3-5 dB.

---

### Q5. Why is via back-drilling necessary for high-speed serial links and what are its limitations?

**Answer:**

Via back-drilling removes the unused portion of a through-hole via that extends beyond the deepest signal layer connection. The stub portion acts as an open-circuited transmission line that creates a resonance at a quarter-wavelength frequency, causing a deep notch in the insertion loss at that frequency.

For a stub of length L_stub in a dielectric with epsilon_r, the first resonance occurs at f_res = c / (4 * L_stub * sqrt(epsilon_r)). For a 40-mil stub in FR4 (epsilon_r approximately 4), f_res approximately 18.75 GHz, which falls squarely in the passband of 56 GBd signals (28 GHz Nyquist). This resonance can cause 10-20 dB of additional insertion loss at the resonance frequency, completely destroying the signal.

Back-drilling uses a mechanical drill to remove the stub after the via is plated. The drill depth is controlled to leave a small remaining stub (typically 4-8 mils, limited by the drill depth accuracy). The remaining stub pushes the resonance to a higher frequency: a 6-mil stub has a resonance at approximately 125 GHz, well above the band of interest for 56 GBd signals.

Limitations include drill depth accuracy (typically plus or minus 3-4 mils for mechanical drilling), which can leave stubs of 8-12 mils in the worst case; increased manufacturing cost (each back-drilled via requires an additional drilling operation); mechanical weakening of the via (the back-drill reduces the plated barrel length, potentially affecting reliability); and difficulty with very deep boards (for boards thicker than 120 mils, the drill accuracy may be insufficient to reliably back-drill to the target depth).

For 112 GBd (224G) signals, even back-drilled stubs of 4-6 mils may be problematic (resonance at 62-94 GHz), driving the industry toward blind and micro-via technologies that eliminate stubs entirely.

---

### Q6. How are length matching and skew management implemented for multi-lane serial links?

**Answer:**

Length matching ensures that the propagation delays of all lanes in a multi-lane link (such as a PCIe x16 or NVLink interface) are within the specification tolerance. Intra-pair skew (delay difference between D+ and D- within a differential pair) must be less than approximately 1-3 ps (0.05-0.15 UI at 56 GBd). Inter-lane skew (delay difference between lanes) must be less than the de-skew capability of the receiver (typically 8-32 UI).

Intra-pair skew is managed by routing the two traces of a differential pair symmetrically. Any asymmetry (different trace lengths at bends, different via structures, or different coupling environments) creates intra-pair skew. Techniques include using arc bends instead of right-angle bends (arcs are inherently symmetric for a differential pair), ensuring the pair maintains consistent spacing through all routing features, and adding compensating length (serpentine) to the shorter trace if asymmetry is unavoidable.

Inter-lane skew is managed by length-matching serpentine patterns added to shorter lanes to equalize the total path length. The serpentine pattern must be designed to avoid creating additional signal integrity problems: the serpentine amplitude should be less than 3-4 times the trace width to avoid self-coupling, the serpentine should not be adjacent to other signal traces (to avoid creating crosstalk), and the serpentine segments should be spaced apart to avoid coupling between adjacent segments.

The propagation velocity depends on the effective dielectric constant of the surrounding material, which can vary between layers (if the dielectric height or material differs) and between stripline and microstrip geometries. Length matching must account for these velocity differences: two traces with the same physical length but on different layers may have different electrical delays.

---

### Q7. What PCB materials are used for AI server boards at different data rates?

**Answer:**

PCB material selection is driven by the loss tangent (Df) at the Nyquist frequency, as dielectric loss often dominates total channel loss for traces longer than a few inches. The industry uses a hierarchy of materials matched to the data rate requirements:

| Data Rate | Nyquist | Material Class | Df (10 GHz) | Examples | Cost Relative |
|-----------|---------|---------------|-------------|----------|---------------|
| 25G NRZ | 12.5 GHz | Mid-loss | 0.008-0.012 | Megtron 4, IS680 | 1.0x |
| 56G PAM4 | 28 GHz | Low-loss | 0.003-0.005 | Megtron 6, EM-891K | 2.0x |
| 112G PAM4 | 28 GHz | Low-loss | 0.003-0.005 | Megtron 6 | 2.0x |
| 112G NRZ | 56 GHz | Ultra-low-loss | 0.001-0.003 | Megtron 7, Tachyon | 3.5x |
| 224G PAM4 | 56 GHz | Ultra-low-loss | 0.001-0.003 | Megtron 7+, new materials | 4.0x |

The dielectric constant (Dk) also matters for impedance control and propagation velocity. Lower Dk reduces the propagation delay and shifts via stub resonances to higher frequencies (beneficial). However, the Dk variation across the board must be tightly controlled (plus or minus 3% or better) to maintain impedance within specification.

For AI server boards, mixed-material stackups are common: the high-speed signal layers (2-4 layers carrying NVLink, PCIe, and Ethernet traces) use Megtron 6 or 7, while the remaining layers use standard or mid-loss material. The cost savings from this approach can be 30-50% compared to using premium material throughout, which is significant for boards that cost $200-500 each.

The glass weave pattern in the laminate affects signal integrity through fiber weave effect: the dielectric constant differs between the glass fiber regions and the resin regions, creating periodic impedance variations along the trace. For high-speed traces, the routing should be at an angle to the glass weave (typically 5-15 degrees) to average out the fiber weave effect, or spread-glass laminates (with a more uniform glass distribution) should be used.

---

### Q8. What is the GSSG routing pattern and when is it used?

**Answer:**

GSSG (ground-signal-signal-ground) is a routing pattern where differential signal pairs are flanked by grounded conductors. Each differential pair consists of two signal traces (SS), and ground traces (GG) are placed on either side of the pair. The ground traces provide electromagnetic shielding between adjacent differential pairs, reducing crosstalk.

The GSSG pattern is an alternative to simply routing differential pairs with spacing between them (the SS-gap-SS pattern). GSSG provides approximately 10-15 dB better FEXT isolation than the same center-to-center spacing without ground traces, because the ground traces intercept the electromagnetic field lines that would otherwise couple between pairs.

GSSG is used for high-speed serial links operating at 56 GBd and above, where the crosstalk budget is tight and every dB of isolation matters. It is particularly useful in BGA breakout regions where the routing density is highest and the spacing between pairs is limited by the BGA pitch.

The ground traces in GSSG must be connected to the ground plane through vias (ground stitching) at regular intervals to be effective. The stitching interval should be less than one-quarter wavelength at the highest frequency of interest. For 56 GBd (28 GHz Nyquist), this means stitching every 1-2 mm, which requires many ground vias and consumes PCB area.

The GSSG pattern has a cost in routing density: the ground traces consume space that could be used for additional signal pairs. For a 5-mil trace width with 5-mil spacing, the GSSG pitch is approximately 30 mils per differential pair (G-5-S-5-S-5-G), compared to approximately 20 mils for a minimal SS-gap-SS pattern. The 50% density penalty must be weighed against the improved crosstalk performance.

---

### Q9. How does package design affect SerDes signal integrity in AI accelerators?

**Answer:**

The package is a critical part of the signal path between the die and the PCB, and its design directly impacts the channel loss, impedance matching, and bandwidth for high-speed serial links. AI accelerators use advanced packaging technologies (flip-chip BGA, CoWoS, EMIB) that must be co-designed with the SerDes.

The package contributes several impairments: bump/pillar transition (the connection from the die pad to the package substrate introduces parasitic inductance and capacitance), package trace routing (the redistribution layer or substrate traces add insertion loss and potential impedance discontinuities), via transitions within the package (for multi-layer substrates, the signal may change layers, with each transition adding loss and reflections), and BGA ball transition (the solder ball connecting the package to the PCB adds inductance).

For standard organic packages (with 2-6 substrate layers), the total package insertion loss for a SerDes signal is typically 1-3 dB at 28 GHz, depending on the trace length and the number of via transitions. The package return loss must be better than 10-15 dB across the frequency band.

For advanced packages (CoWoS, EMIB), the silicon interposer provides superior electrical performance: the interposer traces have lower loss than organic substrate traces (due to the low-loss silicon dioxide dielectric), the micro-bumps have lower inductance than standard flip-chip bumps, and the shorter die-to-die distances reduce the total path length. However, the interposer adds an additional layer of complexity (interposer fabrication, assembly, testing) and cost.

The package and SerDes are co-designed as an integrated system: the SerDes driver impedance and equalization are optimized for the specific package model (extracted from 3D electromagnetic simulation), and the package geometry is optimized for the SerDes electrical requirements. Changes to either the package or the SerDes design require re-optimization of both.

---

### Q10. What are the thermal management considerations for PCB routing of high-speed serial links?

**Answer:**

Thermal management affects PCB routing of high-speed serial links through two mechanisms: temperature-dependent material properties and thermal expansion-induced mechanical stress.

The dielectric constant and loss tangent of PCB materials change with temperature. For Megtron 6, the Dk increases by approximately 0.5-1.0% per 10 degrees Celsius, which shifts the trace impedance by approximately 0.25-0.5%. This shift is within the typical 10% impedance tolerance but can be significant for marginal designs. The Df also increases with temperature (approximately 5-10% per 20 degrees Celsius), increasing the channel loss by a corresponding amount.

The copper resistivity increases with temperature (approximately 0.4% per degree Celsius), increasing the conductor loss. For a 10-inch trace at 28 GHz, a 40-degree temperature rise increases the conductor loss by approximately 0.7 dB.

In AI servers, the PCB temperature varies significantly across the board: the region under the GPU (which dissipates 300-700W) can be 20-40 degrees Celsius hotter than the edge of the board. The SerDes traces routed near the GPU experience higher temperatures and therefore higher loss than traces routed in cooler regions. This temperature-dependent loss variation must be accounted for in the link budget analysis.

Thermal expansion mismatch between the die (silicon CTE approximately 2.5 ppm/C), the package substrate (organic CTE approximately 15-17 ppm/C), and the PCB (FR4 CTE approximately 14-16 ppm/C in-plane) creates mechanical stress at the solder joints (both BGA balls and die bumps). Over many thermal cycles (power on/off cycles), this stress can cause fatigue failure of the solder joints, degrading the SerDes signal integrity. The solder joint reliability is a key consideration for the 5-7 year operational lifetime expected of AI server boards.

See also: [Retimer and Redriver Design](retimer_and_redriver_design.md) for extending link reach beyond PCB limitations.

---

### Q11. How is PCB manufacturing variation managed for high-speed serial link designs?

**Answer:**

PCB manufacturing introduces variations in trace geometry, dielectric properties, and via structures that affect the impedance, loss, and signal integrity of high-speed serial links. Managing these variations requires design-for-manufacturing (DFM) practices and statistical analysis.

Trace width variation is typically plus or minus 0.5-1.0 mil for standard PCB processes and plus or minus 0.3-0.5 mil for premium processes. For a 4-mil trace, a plus or minus 1.0-mil variation is plus or minus 25%, which directly affects the impedance by approximately 10-12%. This is a major contributor to impedance variation and can be the limiting factor for designs that are marginal on impedance budget.

Dielectric height variation is typically plus or minus 0.5-1.0 mil for standard processes. This affects both the impedance (approximately 5% per mil of height variation for typical geometries) and the propagation velocity (affecting length matching accuracy).

Copper roughness affects conductor loss at high frequencies. The RMS roughness of copper foil (typically 0.3-2.0 um) creates additional surface area that increases the effective skin-effect resistance. At 28 GHz, the skin depth in copper is approximately 0.39 um, which is comparable to the surface roughness. The roughness-induced loss increase can be 20-50% above the smooth-copper prediction, depending on the foil type. Low-roughness copper foil (such as VLP or HVLP grades) reduces this penalty but is more expensive.

Via drill accuracy affects back-drill depth (as discussed in Q5) and via placement (important for differential pair via impedance matching). The drill position accuracy is typically plus or minus 2-3 mils, which can cause the two vias of a differential pair to be offset from their ideal positions, creating asymmetric coupling and intra-pair skew.

Statistical simulation (Monte Carlo analysis) is used to predict the yield of high-speed channels given the manufacturing variations. The simulation varies all geometric and material parameters within their manufacturing tolerances and computes the resulting COM for each realization. The design must achieve COM greater than 3 dB for at least 99% (or 99.7% for 3-sigma) of the Monte Carlo realizations.

---

### Q12. What are the emerging PCB technologies for 224G serial link routing?

**Answer:**

Several PCB technologies are emerging to support 224G (112 GBd PAM4, 56 GHz Nyquist) serial link routing. Ultra-low-loss materials with Df below 0.002 at 10 GHz are being developed by Panasonic (Megtron 7+), Isola (Astra MT77), Rogers, and others. These materials reduce dielectric loss at 56 GHz to levels that make 6-8 inch traces feasible for 224G, compared to the 3-4 inch maximum practical length with current materials.

HDI (High Density Interconnect) PCB technology uses multiple sequential lamination cycles to create micro-vias and buried vias that eliminate through-hole via stubs entirely. By using only blind or buried vias for high-speed signals, the stub resonance problem is eliminated, improving the via transition quality by 5-10 dB compared to back-drilled through-hole vias.

PTFE-based substrates (such as Rogers materials) provide extremely low loss (Df below 0.001) but are difficult to process with standard PCB manufacturing equipment. Hybrid stackups that combine PTFE-based core materials (for the high-speed layers) with standard materials (for the remaining layers) are being developed to achieve the best of both worlds.

Embedded trace technology places the copper traces within the dielectric (rather than on the surface), improving the impedance uniformity and reducing the copper roughness impact. Embedded traces can achieve tighter impedance tolerance (plus or minus 5%) than conventional traces.

These technologies increase the PCB cost by 2-5 times compared to current Megtron 6-based designs, but the cost is justified for AI server boards where the SerDes performance directly impacts the system's compute bandwidth and training efficiency.
