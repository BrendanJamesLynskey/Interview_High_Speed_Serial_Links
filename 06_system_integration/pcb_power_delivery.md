# PCB Power Delivery for High-Speed Serial Links

## Overview

Power delivery network (PDN) design at the PCB level is the foundation that lets a SerDes meet its jitter and BER specifications. While the on-die decoupling and package PDN handle the highest frequencies and tightest loops, the PCB PDN must deliver clean supply current to the package BGA from DC up to hundreds of MHz — the frequency range where switching regulator output, plane inductance, and discrete capacitor ESL interact. A poor PCB PDN manifests as supply-induced jitter, random BER excursions, and failures that correlate with workload rather than channel quality. This section covers target impedance, decoupling hierarchy, component selection, layout rules, and PDN verification for AI serial links.

---

### Q1. How is the target impedance calculated for a SerDes PCB PDN?

**Answer:**

The target impedance Z_target is the maximum allowable PDN impedance such that the worst-case transient current does not cause a supply voltage excursion exceeding the noise budget allocated to the PDN. The formula is Z_target = V_ripple / I_transient, where V_ripple is the allowable noise and I_transient is the worst-case current transient.

For a 112G PAM4 SerDes on a 0.85V core supply, the PDN noise budget is typically 2-3% of the supply voltage (17-26 mV peak-to-peak), allocated against the total jitter budget. The transient current depends on the number of active lanes, the lane power, and the correlation of switching across lanes. A 16-lane SerDes transmitter running at 112 GBd can draw 4-8 A average; transient current (peak-to-peak delta during workload transitions) is typically 30-50% of average, so 1.2-4 A.

For V_ripple = 20 mV and I_transient = 2 A, Z_target = 10 mohm. The PCB PDN must present less than 10 mohm across the relevant frequency range, which for a 16-lane 112G link extends from about 100 kHz (workload-activity frequency) to 1-2 GHz (above this, package and die decoupling dominate).

Hitting 10 mohm from 100 kHz to 2 GHz requires a well-designed decoupling hierarchy, low-inductance via patterns, and tight coupling to ground planes. It is significantly tighter than typical digital PDN targets (50-100 mohm) and is one of the main reasons AI accelerator boards are expensive.

A useful rule of thumb: for each doubling of SerDes power, halve Z_target. Modern AI accelerators have pushed the core supply from 1.0V to 0.7V while raising current from 100 A to 400 A, tightening Z_target from roughly 5 mohm to under 1 mohm for the core rail — a challenge that requires on-package inductors, embedded capacitance, and new decoupling architectures.

---

### Q2. What supply rails does a SerDes require, and what are their characteristics?

**Answer:**

A modern SerDes has several distinct power rails, each with different noise sensitivity, current level, and regulation requirements. The PCB PDN must deliver all of them with appropriate isolation between domains.

| Rail | Typical voltage | Typical current per lane | Noise sensitivity | Source |
|------|-----------------|--------------------------|-------------------|--------|
| VDD_ANA (analog core) | 0.8-1.0 V | 20-50 mA | Very high (affects PLL, CTLE, slicer) | LDO from a higher rail |
| VDD_IO (driver supply) | 0.8-1.2 V | 50-150 mA | High (directly modulates output) | Buck converter, often dedicated |
| VDD_DIG (digital logic) | 0.7-0.9 V | 30-100 mA | Low (digital tolerates noise) | Shared with core compute supply |
| VDD_PLL (dedicated PLL) | 0.8-1.0 V | 5-20 mA | Extreme (PLL jitter) | LDO with RC filter |
| VDD_TERM (termination) | 0.6-1.0 V | 10-30 mA | Medium | Often derived from VDD_IO |

For a 16-lane 112G SerDes, total power is 2-4 W per lane including transmitter, receiver, and clocking. The 16-lane subsystem draws 30-65 W, most of it on VDD_DIG and VDD_IO. The PDN must handle this current with low impedance from DC to beyond the core clock frequency.

VDD_ANA is the most demanding rail. It feeds the analog front-end (CTLE, VGA, DFE summer, slicer) where every millivolt of supply noise modulates the signal path and is counted against the inner-eye budget. LDOs feeding VDD_ANA typically have PSRR better than 40 dB up to 100 MHz; the PCB PDN delivering to the LDO input must be clean enough for the LDO to do its job.

VDD_PLL is even more sensitive because PLL jitter contribution scales directly with supply noise in the VCO. The PLL supply often uses a dedicated LDO with an RC filter or a ferrite bead filter to give 60 dB+ PSRR across the VCO's sensitivity band (1-100 MHz). The PCB routing to the PLL supply is typically a single narrow trace with extensive decoupling.

VDD_IO is the highest-current rail. It feeds the output drivers, and its PDN impedance is what sets the SSN on the transmitter output swing. Managing VDD_IO PDN is the primary focus of PCB PI for SerDes.

---

### Q3. How is the decoupling capacitor hierarchy structured for a SerDes PCB?

**Answer:**

The decoupling hierarchy provides low PDN impedance across a wide frequency range by combining capacitors of different values, each effective in its own band. Each capacitor acts as an ideal cap up to its self-resonant frequency (SRF = 1/(2*pi*sqrt(LC))), then becomes inductive beyond SRF. Combining many caps with staggered SRFs produces a composite low-impedance curve.

Typical SerDes decoupling hierarchy for VDD_IO or VDD_DIG:

| Cap value | Package | SRF | Effective range | Count per rail | Placement |
|-----------|---------|-----|-----------------|----------------|-----------|
| 100-470 uF | tantalum or polymer | 50-200 kHz | 1 kHz - 1 MHz | 2-4 | Near PMIC output |
| 10-22 uF | ceramic 0805 X5R | 1-3 MHz | 100 kHz - 5 MHz | 4-8 | Near package, perimeter |
| 1-4.7 uF | ceramic 0402 X5R | 5-10 MHz | 1-20 MHz | 10-20 | Near BGA, close pattern |
| 100 nF-1 uF | ceramic 0201 X5R | 15-50 MHz | 10-100 MHz | 20-50 | Under and around BGA |
| 1-22 nF | ceramic 01005 X5R | 60-200 MHz | 50-500 MHz | 30-80 | Under BGA via-in-pad |
| Embedded MLCC or thin prepreg | distributed | 500 MHz+ | 0.5-5 GHz | Distributed | Under package |

The 01005 caps are the current state of the art for near-package decoupling. They are 0.4x0.2 mm and can be placed under a BGA with via-in-pad construction at a density of 2-3 caps per mm^2. Their small size gives them low ESL (0.3-0.5 nH) which pushes their effective range to 500 MHz, bridging to the package decoupling.

Placement rules: every decoupling cap must have a short, low-inductance path to both the target rail and ground. A cap placed far from the BGA with long vias and narrow traces has high loop inductance and is ineffective at high frequencies. The cap's own ESL is often smaller than the loop inductance of the placement, so placement dominates.

Capacitor effectiveness at high frequency scales with 1/(N*L_loop) where N is the count and L_loop is the per-cap loop inductance. Ten caps at 1 nH each give effective inductance of 0.1 nH; halving to 0.5 nH gives 50 pH. At 1 GHz, 50 pH corresponds to 0.3 ohm impedance, which is sufficient for most SerDes rails.

For a 112G SerDes, the decoupling hierarchy typically includes 4-8 bulk caps, 20-40 mid-frequency caps, and 40-80 high-frequency caps per VDD rail. On a 16-lane board with VDD_IO, VDD_ANA, VDD_DIG, and VDD_PLL, the cap count approaches 500-1000 discretes dedicated to SerDes alone.

---

### Q4. How is PDN impedance simulated, and what anti-resonances typically appear?

**Answer:**

PDN impedance simulation uses frequency-domain solvers such as Cadence Sigrity PowerSI, Ansys SIwave, or Siemens HyperLynx PI to compute Z(f) looking into the PDN from the SerDes supply pin. The solver imports the PCB layout, assigns capacitor models (including ESL and DC bias derating), defines source and sink ports, and computes the transfer impedance across frequency.

The output is a Z(f) plot from DC to 1-10 GHz. The designer overlays Z_target and checks where the impedance exceeds target. Anti-resonance peaks — where cap-ESL resonates with plane inductance or with another cap's capacitance — are the most common problems.

Common anti-resonance locations:

- **Cap bank to plane inductance** at 100-500 MHz. The high-frequency caps (nF range) have ESL that resonates with the plane inductance seen looking into the PCB, producing a peak that can exceed Z_target by 2-5x. Mitigation: add more caps, reduce loop inductance, or use caps with different ESL to damp the peak.
- **Bulk cap to plane cap resonance** at 1-10 MHz. The bulk cap capacitance resonates with the board plane inductance against the mid-frequency cap. Flattens with intermediate-value caps.
- **Plane cavity resonance** at 500 MHz - 2 GHz. The power plane and ground plane form a parallel-plate cavity whose modes resonate at f = (c/2*sqrt(Dk)) * sqrt((m/L)^2 + (n/W)^2) for a L x W plane. A 100 x 100 mm plane pair has first cavity mode near 750 MHz. Mitigation: add damping with lossy material, avoid exciting the modes with decoupling placement, or use embedded capacitance to suppress the cavity.
- **PMIC-to-board resonance** at 10-100 kHz. The PMIC output inductor (if any) resonates with the bulk cap. Usually flat after the PMIC feedback loop is tuned.

Detailed simulation also models the AMI receiver's supply sensitivity to convert PDN noise into jitter contribution. The result is a jitter budget line item for "supply-induced jitter" that must be less than the total jitter budget allocated to this source, typically 0.1-0.2 UI RMS.

PDN simulation must include DC bias derating of ceramic caps. A 10 uF 6.3V X5R cap used at 3.3V bias has effective capacitance of 4-5 uF, not 10 uF. If the simulation uses nameplate values, it reports an optimistic PDN that will fail in production. Vendor-provided S-parameter files that include bias derating are preferred.

---

### Q5. How is the PMIC placement and VRM topology chosen for SerDes power delivery?

**Answer:**

The voltage regulator module (VRM) for a SerDes must meet the DC current requirement, regulate to the required accuracy, recover from load transients within the PDN budget, and have a stable control loop. Its placement on the PCB affects the DC IR drop, the AC impedance, and the thermal coupling to nearby components.

Topology choices:

- **Multi-phase buck converter** is the standard for high-current rails (10-100 A). Four to eight phases are typical for an AI accelerator core supply. Phases are interleaved to reduce output ripple and improve transient response. Switching frequency is 500 kHz - 2 MHz per phase. Output current shares across phases.
- **Single-phase buck** for moderate rails (1-10 A). Used for VDD_IO or auxiliary rails. Simpler and cheaper but less load transient capability.
- **LDO** for low-noise rails (VDD_ANA, VDD_PLL). The LDO input is usually fed from a buck converter output 100-300 mV above the target voltage. The LDO burns the headroom as heat in exchange for PSRR of 40-70 dB up to 100 MHz.
- **Switched-cap or integrated power stage** for ultra-high-density applications. Provides voltage conversion inside the package or on a vertical power module directly under the SerDes BGA. Used in 400 A AI accelerator packages.

Placement rules:

- Place the VRM as close to the SerDes power pins as possible. Every mm of distance adds inductance to the PDN and increases DC IR drop. Typical distance is 10-25 mm for board VRMs, less than 5 mm for on-package VRMs.
- Orient the VRM output toward the SerDes BGA. The output filter caps and inductor should be on the SerDes-facing side of the VRM.
- Provide a solid power plane from VRM output to SerDes BGA with minimal cuts. Each via and cut adds inductance.
- Separate the VRM switching node from sensitive analog traces. The switching node is a large dV/dt source and radiates to nearby circuits.
- Thermal isolation: the VRM dissipates 1-5 W typically and heats the nearby copper. Avoid placing the VRM directly adjacent to temperature-sensitive components (reference clocks, sensitive LDOs) unless a thermal break is present.

For AI accelerators at 400 A on a 0.85V supply, traditional board-level VRMs run out of headroom. The industry has moved to vertical power modules (a stack of switching phases mounted directly above the accelerator package) and in-package VRMs that eliminate the PCB power delivery path entirely. This shifts the PI design problem from the PCB to the package, but the PCB must still deliver the higher-voltage input to the in-package VRM through the BGA.

---

### Q6. What is the role of embedded capacitance in PCB PDN for SerDes?

**Answer:**

Embedded capacitance is a distributed parallel-plate capacitor formed by laminating a thin, high-Dk dielectric between a power plane and a ground plane. Typical construction uses 25-50 um of filled epoxy (Dk 4-40) between the planes, producing a sheet capacitance of 0.3-3 nF per cm^2. For a SerDes package footprint of 30x30 mm, this gives 3-27 nF of distributed capacitance directly under the BGA.

The advantage of embedded capacitance is near-zero loop inductance. Because the capacitance is located directly under the current-sink via field with no discrete component body and no via paths, the high-frequency impedance is flat and low. Discrete caps, even 01005 with via-in-pad, have 200-500 pH of loop inductance; embedded capacitance has effective inductance of 10-50 pH determined only by the thin dielectric thickness.

This translates to effective PDN impedance suppression from 500 MHz to 5 GHz, where discrete caps have all gone inductive. For 112G SerDes, this band corresponds directly to the Nyquist region of workload-induced noise, which is exactly where the discrete decoupling hierarchy runs out of capability.

Drawbacks:

- **Cost**: embedded capacitance laminate adds 20-50% to PCB cost versus standard FR-4. The material itself is more expensive and the lamination process is more complex.
- **Dk variation**: the high Dk of the filled material affects any traces routed on adjacent layers. Impedance must be recalculated and trace widths adjusted.
- **Voltage rating**: thin dielectrics have lower breakdown voltage. Most embedded capacitance materials are rated 10-50V, suitable for core supplies but not for high-voltage applications.
- **Reliability**: thinner dielectrics are more vulnerable to humidity ingress and mechanical cracking during thermal cycling. Not all embedded materials have automotive-grade qualification.

For AI server boards running 112G and 224G SerDes, embedded capacitance is increasingly standard despite the cost. It is the only way to hit Z_target in the GHz band without massively over-populating discrete caps. For 56G boards, it is an option that some designs use and others avoid; for 25G boards, it is rarely justified.

The decision to use embedded capacitance is made at stackup design time and cannot be retrofitted. If the design budget can afford it and the target rate is 56G or higher, specify it early.

---

### Q7. How are VRM-to-load transient effects analysed for SerDes workloads?

**Answer:**

Transient analysis complements frequency-domain PDN analysis by modelling the time-domain response of the PDN to a realistic load current profile. It captures dynamic effects that the frequency-domain view averages out: droop during workload ramps, ringing after workload stops, and recovery times.

The input is a current profile I(t) representing the worst-case load transient. For a SerDes, this might be: all lanes powered off, then all lanes simultaneously enabled and running maximum switching activity. The transition time is set by the software/firmware enable sequence, typically 1-100 us.

Simulation tools such as Cadence PowerSI Transient, Ansys SIwave Transient, or SPICE with the extracted PDN model apply I(t) to the PDN and compute V(t) at each load pin. The output is a voltage waveform showing droop, settling, and steady state.

Key metrics:

- **Initial droop**: the peak voltage deviation during the fastest edge of the transient. Limited by the highest-frequency decoupling (on-die caps, package decoupling, embedded capacitance). Typical target: less than 50% of the static V_ripple budget.
- **Settling time**: the time for the voltage to return within 1-5% of the steady state. Limited by the VRM control loop bandwidth, typically 50-500 us for a buck converter.
- **Ringing**: oscillation during settling. Caused by under-damped resonances in the PDN or VRM loop. Indicates insufficient damping.
- **Long-term droop**: a slow voltage sag during a sustained high-current workload, caused by VRM regulation error or thermal effects on the regulator.

Transient analysis exposes issues that frequency-domain analysis misses. A PDN with low Z(f) across the band may still have a specific resonance that rings for a long time after a workload change, creating bit errors during the ringing period. Frequency-domain analysis would show the resonance but not its time-domain consequence.

For modern AI workloads, the load profile is highly non-stationary: matrix multiplication bursts, all-to-all communication, training step synchronisation. Each creates a distinctive current signature. A robust PDN must handle all of these without exceeding the noise budget, not just a simplified step transient.

Transient analysis is typically run at a few representative operating points: cold start, workload ramp-up, workload steady state, workload ramp-down, and thermal steady state. Each reveals different aspects of the PDN response.

---

### Q8. How does PDN design interact with retimer and redriver placement?

**Answer:**

Retimers and redrivers are active components that amplify and reshape SerDes signals. They require their own power delivery and contribute their own supply sensitivity to the channel's noise budget. The PCB PDN must handle them as first-class loads.

A typical retimer consumes 0.5-2 W per lane (higher than a typical SerDes transmitter because the retimer has both a receiver and a transmitter). A 16-lane retimer dissipates 8-30 W, which is a significant fraction of the total board power. The retimer usually requires 2-4 separate supply rails (core, IO, PLL, reference) and has PSRR requirements similar to a SerDes.

PDN design considerations:

- **Dedicated decoupling**: each retimer needs its own decoupling hierarchy near the package. Sharing decoupling with the main SerDes is tempting (reduces part count) but couples noise between the two components. Dedicated decoupling is worth the extra cost.
- **Ground stitching**: the retimer's ground pins must be tightly coupled to the system ground plane via many stitching vias. Ground bounce at the retimer translates directly to output jitter on the forwarded signal.
- **Reference clock quality**: retimers use a reference clock (often 100-200 MHz) that feeds the internal PLL. The reference clock supply must be clean — typically a dedicated LDO. Noise on the reference propagates into output jitter with roughly unity gain.
- **Thermal coupling**: retimers run hot (case temperature 60-90C in still air). Place them where airflow or heat spreading is available. Thermal coupling to the SerDes ASIC can push both components into derating.
- **Power sequencing**: retimers have specific power-up sequencing requirements. VDD must rise before VDD_IO before reference clock is applied, or the retimer latches up. PMIC sequencing logic or an external sequencer handles this.

Retimer placement is typically at the mid-point of a long channel — for example, 8-12 inches of backplane routing split into two 4-6 inch segments with a retimer between. The retimer restores a clean signal at its output, resetting the channel loss budget. The PDN at the retimer's location must be as clean as at the SerDes transmitter, or the retimer's own output quality is degraded.

For AI server systems with multiple accelerators connected by NVLink or PCIe Gen6, the number of retimers can be 20-50 per chassis. The aggregate retimer power is large, and the PDN design must allocate VRMs and decoupling to each retimer group with care.

See also [Retimer and Redriver Design](retimer_and_redriver_design.md) for retimer architecture details.

---

### Q9. What EMC implications does the PCB PDN have for SerDes systems?

**Answer:**

The PDN is both a source and a victim for EMI. As a source, switching currents in the PDN create radiated fields from the power planes and decoupling structures. As a victim, external fields couple into the PDN and modulate the SerDes supplies, creating supply-induced jitter.

Radiation mechanisms:

- **Plane-cavity modes**: the gap between a power plane and a ground plane forms a resonant cavity. Energy from switching circuits excites cavity modes that radiate from the board edge. The radiation is broadband, with peaks at cavity resonance frequencies. Mitigation: absorb the modes with damping (lossy materials or resistive terminations at plane edges), or suppress excitation with dense decoupling across the plane.
- **Edge fringing**: fields from the power plane fringe past the board edge and radiate. Mitigation: add a ground guard ring on the edge layer, or pull the power plane in from the edge and surround it with ground plane (20-to-1 rule: keep power plane 20x dielectric-thickness inside the ground edge).
- **Via radiation**: vias carrying switching current radiate like small monopoles. Ground stitching vias near every signal via reduce the loop area and therefore the radiation.

Susceptibility:

- **External fields couple into the power plane through apertures** (connector openings, card slots, cable entries). The coupled noise appears on the PDN and modulates the SerDes supplies.
- **Common-mode current** flowing on cables attached to the board can drive the power plane directly if the cable ground is not properly bonded to the chassis.

EMC compliance tests (CISPR 32, FCC Part 15) measure radiated emissions from 30 MHz to 6 GHz or higher. SerDes-related emissions typically show up at the reference clock frequency, the data rate fundamentals, and their harmonics. PDN-related emissions show up at the switching regulator frequency and harmonics (tens of kHz to a few MHz), and as broadband noise from plane cavity modes.

Design techniques for EMC:

- Use spread-spectrum clocking (SSC) on the reference clock to spread the emission spectrum, reducing peak emission at any single frequency. Trade-off: SSC adds jitter to the SerDes output, which must be within the SerDes jitter budget.
- Keep the power plane pulled in from the board edge.
- Add absorptive material (ferrite tiles, RF absorber) at known resonance locations if radiation targets are missed.
- Enclose the board in a metal chassis with good seam continuity. A chassis provides 20-40 dB of isolation at GHz frequencies.

For AI accelerators in compute chassis, the chassis enclosure typically handles EMC. The PCB-level EMC focus is on ensuring that aggressive PDN design (large plane area, dense decoupling) does not create unintended emitters that compromise the chassis envelope.

---

### Q10. How is PDN quality verified after PCB fabrication and during bringup?

**Answer:**

Post-fabrication PDN verification combines DC measurement, small-signal AC measurement, transient measurement, and system-level correlation with the SerDes eye quality.

**DC measurement**: load the board with a known current profile using a DC electronic load or the actual silicon, measure VDD_IO and VDD_ANA at the package BGA with a 4-wire Kelvin probe. Compare against the target voltage. Voltage drops exceeding the IR budget indicate a PCB problem (missing via, thin plane, wrong trace width). Typical tools: Keithley 2400 SMU, Tektronix PA-1000 power analyser.

**AC PDN impedance measurement**: inject a small-signal current into the PDN with a signal generator and measure the voltage response with a VNA or a shunt-through measurement (e.g., Keysight E5061B with PDN option). The measurement produces Z(f) from kHz to GHz, which is then compared against simulation. Correlation within 20-30% is acceptable; larger discrepancies indicate model errors (missing caps, wrong plane thickness, unmodelled parasitics). This measurement typically requires a dedicated PDN measurement coupon or a test point array near the BGA, because probing the actual BGA directly is not feasible.

**Transient measurement**: apply a known load transient (fast-switching load, typically a FET switching between two load currents at known frequency) and measure the voltage ringing with a scope. The droop magnitude and settling time are compared against transient simulation. This is the most direct measurement of dynamic PDN performance under realistic conditions.

**SerDes eye correlation**: run the SerDes at full rate with a stress pattern and measure the eye margin through the link's built-in margining features (on-die scope, BER margin test). Correlate eye quality with measured PDN noise. A design that shows bad eye and good PDN has a signal integrity problem; a design with good eye and bad PDN may still have hidden marginality.

**Thermal correlation**: measure temperature across the board under load and correlate with the PDN measurements at those temperatures. PDN quality changes with temperature (cap derating, plane resistance) and should be verified at the hot operating point.

Common issues found in PDN bringup:

- **Missing decoupling caps**: the DFM placement missed a group, or the pick-and-place failed to populate them. Shows as high impedance at the intended decoupling band.
- **Wrong cap values**: 10 nF populated where 100 nF was intended (or vice versa). Shows as resonance shift from predicted.
- **Damaged vias**: cracked or broken vias under the BGA from thermal stress. Shows as high local impedance and sometimes as DC open.
- **Inadequate via count**: the layout only placed enough vias for static current, not enough for AC loop inductance. Shows as high impedance above 100 MHz.
- **Plane cut**: a routing channel created an unintended plane cut, increasing local inductance. Shows as impedance peak at the cut frequency.

Once identified, PDN issues are fixed by component rework (add or change caps), via rework (drill and fill), or PCB respin (plane or layer changes).

---

See also:
- [Power Supply Noise Coupling](../05_signal_integrity/power_supply_noise_coupling.md)
- [Package and PCB Routing](package_and_pcb_routing.md)
- [PCB Thermal Management](pcb_thermal_management.md)
- [Testing and Compliance](testing_and_compliance.md)
