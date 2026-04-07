# Channel Loss and Modeling

## Overview

Accurate channel modeling is the foundation of high-speed serial link design. This section covers insertion loss mechanisms, S-parameter analysis, impedance discontinuities, and the modeling techniques used to predict whether a channel can support 56G, 112G, or 224G data rates.

---

### Q1. What are the primary loss mechanisms in a high-speed serial link channel?

**Answer:**

There are three primary loss mechanisms in copper-based serial link channels. Conductor loss (also called skin-effect loss or resistive loss) arises because AC current flows only within a thin layer near the conductor surface, with the skin depth decreasing as the inverse square root of frequency. This concentrates current into a smaller cross-sectional area at higher frequencies, increasing resistance. Conductor loss scales approximately with the square root of frequency, so it contributes an attenuation that increases as sqrt(f). For a typical PCB stripline trace, conductor loss might contribute 0.3-0.5 dB/inch at 28 GHz.

Dielectric loss results from the polarization of the PCB substrate material in the alternating electric field. Energy is dissipated as the dielectric dipoles oscillate. Dielectric loss is proportional to frequency and to the material's loss tangent (tan(delta) or Df). Low-loss materials like Megtron 6 (Df approximately 0.004) or Megtron 7 (Df approximately 0.002) are selected specifically to minimize this component. At 28 GHz, dielectric loss may contribute 0.4-0.8 dB/inch depending on the substrate.

Radiation loss occurs when the trace acts as an antenna and radiates energy into the surrounding environment. This is generally a smaller contributor than conductor or dielectric loss for well-designed stripline geometries but can become significant for microstrip traces at frequencies above 20 GHz.

In total, a typical PCB trace might exhibit 0.8-1.2 dB/inch of insertion loss at 28 GHz (56 GBd Nyquist). At 56 GHz (112 GBd Nyquist), this can increase to 1.4-2.0 dB/inch, which is why material selection and trace length minimization are critical for 224G designs.

---

### Q2. How are S-parameters used to build a complete channel model for simulation?

**Answer:**

A complete channel model for serial link simulation is assembled by cascading the S-parameter representations of each physical segment in the signal path. A typical end-to-end channel includes: the transmitter package (die to ball/bump), a PCB breakout from the BGA pad to the first via transition, the main PCB trace (potentially with multiple layer transitions), a connector or socket if present, the receiving PCB trace, and the receiver package.

Each segment is characterized by its own S-parameter file, typically a .s4p (4-port mixed-mode) Touchstone file. To cascade these segments, each S-parameter matrix is first converted to a T-parameter (transfer matrix) representation, the T-matrices are multiplied in order, and the result is converted back to S-parameters. Alternatively, ABCD matrices can be used for the same purpose.

The cascaded channel S-parameters are then used to derive the channel pulse response: the frequency-domain transfer function SDD21(f) is windowed, converted to a causal impulse response via inverse FFT, and then integrated to obtain the pulse response. The pulse response directly reveals the main cursor amplitude and the ISI structure (pre-cursors and post-cursors), which feed into the equalization model.

Modern link simulation tools such as the IEEE COM (Channel Operating Margin) methodology and proprietary tools from Keysight, Cadence, and Synopsys use this approach. The simulation applies models for TX FIR, CTLE, and DFE to the pulse response, then adds noise contributions (thermal noise, crosstalk, jitter) to compute the final eye opening and BER estimate. The accuracy of the simulation depends critically on the quality and bandwidth of the S-parameter data: data must extend to at least 1.5 times the Nyquist frequency, be causal and passive, and have sufficient frequency resolution (typically 10 MHz or finer).

---

### Q3. What is insertion loss and how is it specified for different channel reach categories?

**Answer:**

Insertion loss is the ratio of signal power delivered to the receiver compared to the signal power available from the transmitter, expressed in decibels as a function of frequency. Mathematically, IL(f) = -20*log10(|SDD21(f)|) for the differential mode. Insertion loss is always a positive number when expressed as a loss (with the negative sign convention), and it increases with frequency for all practical channels.

Standards organizations define channel reach categories based on the allowable insertion loss at the Nyquist frequency. For IEEE 802.3ck (100G per lane Ethernet at 53.125 GBd PAM4), the categories are approximately: chip-to-chip (C2C) with less than 5 dB at Nyquist, very short reach (VSR) with up to 10 dB, short reach (SR) with up to 15 dB, medium reach (MR) with up to 23 dB, and long reach (LR) with up to 33 dB.

For PCIe Gen5 (32 GT/s NRZ at 16 GHz Nyquist), the specification defines channel insertion loss limits per CEM (card electromechanical) slot type, ranging from approximately 18 dB for a standard add-in card channel to 30+ dB for an extended reach channel through a riser or cable.

The equalization complexity required scales with the insertion loss. A C2C channel may need only minimal CTLE and no DFE, while an LR channel requires aggressive TX FIR (3+ taps), high-gain CTLE, and multi-tap DFE. The SerDes power consumption also scales with equalization complexity, which is why AI system designers carefully optimize channel lengths and material choices to stay within the minimum necessary reach category.

---

### Q4. Explain return loss and its impact on signal integrity in serial links.

**Answer:**

Return loss measures the fraction of signal power reflected back toward the transmitter at any point in the channel, expressed as RL(f) = -20*log10(|SDD11(f)|). A higher return loss value (in dB) indicates less reflection and better impedance matching. Typical specifications require at least 8-12 dB of differential return loss across the frequency band of interest.

Reflections occur at any impedance discontinuity in the channel: BGA pad transitions, via structures, connector interfaces, trace width changes, or impedance mismatches in the PCB stackup. Each reflection creates a delayed copy of the transmitted signal that adds to the ISI at the receiver. Unlike the smooth frequency rolloff caused by insertion loss, reflections create resonances and ripples in the channel transfer function, which can be particularly damaging because they introduce frequency-selective notches where the signal is severely attenuated.

In a long channel with multiple discontinuities, the reflections can interact through multiple bounces, creating a complex pattern of ISI that is difficult to equalize. The DFE can cancel post-cursor ISI from reflections, but pre-cursor reflections (where a reflection from a downstream discontinuity arrives at the receiver before the main signal) must be handled by CTLE or TX pre-cursor taps.

At 112 GBd, even small impedance discontinuities can be significant. A via stub of 4 mils (0.1 mm) creates a capacitive discontinuity that degrades return loss at frequencies above approximately 30 GHz. This is why back-drilling (removing unused portions of through-hole vias) is mandatory for 56G+ designs, and why controlled-depth laser drilling or HDI (High Density Interconnect) via structures are increasingly used for 112G and 224G channels.

See also: [Eye Diagram Analysis](../05_signal_integrity/eye_diagram_analysis.md) for how return loss impacts eye quality.

---

### Q5. What is impedance discontinuity and what are the most common sources in a serial link channel?

**Answer:**

An impedance discontinuity is any point in the transmission line where the characteristic impedance deviates from the target value (typically 85 or 100 ohms differential). At the discontinuity, a portion of the incident signal is reflected according to the reflection coefficient: rho = (Z2 - Z1) / (Z2 + Z1), where Z1 is the impedance before and Z2 is the impedance after the discontinuity. The reflected energy creates ISI at the receiver and degrades the eye diagram.

The most common sources of impedance discontinuities are via transitions, where the signal changes layers in the PCB or package. A via introduces excess capacitance from the pad and anti-pad structure, and excess inductance from the via barrel. The resulting impedance dip (from capacitance) or rise (from inductance) depends on the via geometry. For a typical PTH (plated through-hole) via in a 120-mil PCB, the via stub (unused portion of the barrel extending beyond the signal layer) creates a resonance at f = c / (4 * stub_length * sqrt(epsilon_r)), which can fall squarely in the band of interest for 56G+ signaling.

BGA pad transitions are another major source: the large pads required for BGA attachment (typically 300-400 um diameter for 0.8 mm pitch) create capacitive loading that drops the local impedance. Connector interfaces introduce discontinuities from the pin-to-PCB transition and from the mating region geometry. Trace width changes, either intentional (for routing density) or unintentional (from manufacturing tolerances), also create discontinuities.

Mitigation techniques include via back-drilling to remove stubs, anti-pad optimization to control via capacitance, ground via placement to reduce loop inductance, pad size minimization, and the use of blind/buried vias. For 224G designs, the allowed impedance variation budget is extremely tight, often requiring full-wave 3D electromagnetic simulation of every transition.

---

### Q6. How are via stubs problematic and what techniques are used to mitigate them?

**Answer:**

A via stub is the unused portion of a plated through-hole (PTH) via that extends beyond the signal layer. For example, if a signal transitions from layer 3 to layer 10 in a 20-layer PCB, the via barrel continues from layer 10 to layer 20, creating a 10-layer stub. This stub acts as a short transmission line terminated in an open circuit, creating a resonance where the stub length equals a quarter wavelength. At this resonant frequency, the stub presents a short circuit to the signal path, causing a deep notch in the insertion loss and catastrophic signal degradation.

The resonant frequency is given by: f_resonance = c / (4 * L_stub * sqrt(epsilon_eff)), where L_stub is the stub length and epsilon_eff is the effective dielectric constant. For a 60-mil stub in FR4 (epsilon_r approximately 4), the first resonance occurs at approximately 12.5 GHz, which is below the Nyquist frequency of all modern high-speed standards. Even a 20-mil stub creates a resonance around 37.5 GHz, which impacts 112G signaling.

Mitigation techniques include back-drilling, where the unused portion of the via barrel is mechanically drilled out after PCB fabrication. Back-drilling can reduce the stub to approximately 4-8 mils, pushing the resonance above 50 GHz. However, back-drilling adds manufacturing cost and has depth tolerance limitations. Blind vias, drilled from one surface to an internal layer, eliminate stubs entirely but are limited in the layer span they can achieve and add process complexity. Buried vias connect two internal layers without extending to the surface. Sequential lamination with multiple via types (micro-via, buried, through) provides the most flexibility but at the highest cost. For 224G designs, the industry is moving toward HDI (High Density Interconnect) PCB technology with multiple sequential lamination cycles to eliminate stubs entirely from the critical signal path.

---

### Q7. What is the Channel Operating Margin (COM) methodology and how is it used in serial link design?

**Answer:**

Channel Operating Margin (COM) is a figure of merit defined by IEEE 802.3 that predicts the performance of a high-speed serial link by combining channel S-parameter data with standardized models for the transmitter, receiver, noise, and equalization. COM is expressed in decibels, and a positive COM indicates that the link has sufficient margin to operate at the target BER. Typical specifications require COM greater than or equal to 3 dB.

The COM calculation proceeds as follows. First, the channel pulse response is computed from the S-parameter data. Then, an optimization algorithm selects the best TX FIR tap coefficients (within specified limits), CTLE setting (from a discrete set of peaking values), and DFE tap coefficients (within specified limits) to maximize the signal-to-noise ratio at the sampler. The signal component is the equalized main cursor amplitude. The noise components include residual ISI (from imperfect equalization), crosstalk (from aggressor channels, modeled using their S-parameters), random jitter (converted to voltage noise at the sampler), transmitter noise (bounded jitter and distortion), and receiver noise (thermal noise of the analog front end).

COM is computed as: COM = 20 * log10(A_signal / A_noise), where A_signal is the peak signal amplitude at the sampler and A_noise is the total RMS noise amplitude scaled to the target BER using a crest factor (typically 3.5 to 4.0 sigma for BER 1e-6 to 1e-4).

COM is valuable because it provides a single number that captures the combined effect of channel quality, equalization capability, and noise environment. System designers use COM analysis during the board design phase to evaluate routing options, material choices, and connector selections. If a proposed channel fails COM (less than 3 dB), the designer must either improve the channel (shorter trace, better material, fewer connectors) or relax the channel requirements by using a higher-capability SerDes with more equalization headroom.

---

### Q8. How does PCB material selection impact channel loss at 56 GBd and above?

**Answer:**

PCB material selection is one of the most impactful decisions in high-speed serial link design because the dielectric loss tangent (Df or tan(delta)) directly determines the frequency-dependent portion of channel loss that scales linearly with frequency. At frequencies above approximately 10 GHz, dielectric loss often dominates conductor loss for trace lengths typical of server and AI system motherboards (4-12 inches).

Standard FR4 has a Df of approximately 0.020 at 10 GHz, which is unacceptable for any link operating at 28 GBd or above. The industry has converged on a hierarchy of low-loss materials: mid-loss materials like Panasonic Megtron 4 (Df approximately 0.008) are suitable for 25G signaling; low-loss materials like Megtron 6 (Df approximately 0.004) are the workhorse for 56G and some 112G designs; ultra-low-loss materials like Megtron 7 (Df approximately 0.002) are used for long-reach 112G channels; and emerging very-low-loss materials from manufacturers like Tachyon and Astra MT77 target 224G applications.

To quantify the impact, consider a 10-inch differential stripline trace at 28 GHz (56 GBd Nyquist). With Megtron 6 (Df = 0.004), the dielectric loss component is approximately 0.5 dB/inch, contributing 5 dB of loss over 10 inches. With standard FR4 (Df = 0.020), the same trace would contribute approximately 2.5 dB/inch or 25 dB of dielectric loss alone, making the channel inoperable. With Megtron 7 (Df = 0.002), the contribution drops to approximately 0.25 dB/inch or 2.5 dB, providing 2.5 dB of margin improvement.

The dielectric constant (Dk or epsilon_r) also matters: lower Dk reduces signal propagation delay and slightly reduces the effective wavelength, which affects via stub resonance frequencies. However, Dk variation across the board area and across manufacturing lots must be tightly controlled to maintain impedance uniformity, as a 10% Dk variation directly translates to a 5% impedance variation.

Material cost increases dramatically with lower Df: Megtron 7 is roughly 3-4 times more expensive than Megtron 6 per panel. AI system designers must balance channel performance requirements against bill-of-materials cost, often using mixed-material stackups where only the high-speed signal layers use ultra-low-loss material while power and low-speed signal layers use standard material.

---

### Q9. What is time-domain reflectometry (TDR) and how is it used to characterize serial link channels?

**Answer:**

Time-domain reflectometry (TDR) is a measurement technique that characterizes the impedance profile of a transmission line as a function of position along the line. A TDR instrument launches a fast-rising step edge into the channel and records the reflected waveform as a function of time. Since the propagation velocity in the medium is known (approximately 6 inches per nanosecond for FR4), the time axis maps directly to physical position along the trace.

At each impedance discontinuity, a portion of the step edge is reflected back to the instrument. A capacitive discontinuity (impedance dip) produces a downward reflection, while an inductive discontinuity (impedance rise) produces an upward reflection. By analyzing the reflected waveform, the impedance at every point along the channel can be reconstructed, providing a spatial impedance profile.

TDR is invaluable for diagnosing problems in serial link channels. It can identify via transitions (which appear as impedance dips from pad capacitance followed by impedance rises from barrel inductance), connector interfaces (which show characteristic impedance patterns), trace impedance variations (which appear as gradual impedance trends), and solder joint quality (which can create localized impedance anomalies).

For differential serial links, differential TDR (DTDR) measures the differential impedance by launching complementary step edges on both conductors of the pair. The target differential impedance is typically 85 or 100 ohms, and the specification usually requires the impedance to remain within plus or minus 10% of the target throughout the channel.

The spatial resolution of TDR is limited by the rise time of the step edge: a 20 ps rise time provides a resolution of approximately 1.2 mm in FR4. Modern TDR instruments and sampling oscilloscopes achieve rise times of 15-25 ps, which is adequate for identifying most features in 56G and 112G channels. For 224G designs, the features of interest (such as via anti-pad dimensions and micro-via geometry) may be smaller than the TDR resolution, requiring supplementation with full-wave 3D electromagnetic simulation.

---

### Q10. How do crosstalk S-parameters (NEXT and FEXT) affect channel modeling?

**Answer:**

Crosstalk is the unintended coupling of signal energy from an aggressor lane to a victim lane, and it is characterized by off-diagonal terms in the multi-port S-parameter matrix. Near-end crosstalk (NEXT) is coupling between ports on the same end of the channel, measured as the ratio of coupled signal at the near end to the incident signal. Far-end crosstalk (FEXT) is coupling measured at the opposite end, arriving at the victim receiver along with the desired signal.

In a full channel model with N differential pairs, the S-parameter matrix is 2N x 2N (or 4N x 4N in single-ended representation). The SDD21 terms on the diagonal represent insertion loss for each lane, while the off-diagonal SDD terms represent differential crosstalk between lanes. SCD and SDC terms capture mode conversion, where differential signals couple into common-mode signals and vice versa.

FEXT is typically the dominant crosstalk mechanism in serial link channels because the coupled signal arrives at the receiver at the same time as the desired signal and cannot be distinguished by timing alone. In a PCB with parallel routed lanes, FEXT accumulates over the coupled length and can be significant for traces routed in parallel for several inches. The FEXT coupling coefficient scales with the coupled length and inversely with the square of the spacing between pairs.

In the COM methodology, crosstalk is modeled by applying the aggressor signals (with worst-case amplitude and timing) through the crosstalk transfer functions to compute the integrated crosstalk noise (ICN) at the victim receiver. This noise is combined with residual ISI and other noise sources to determine the total noise, which then determines the COM figure of merit.

Mitigation strategies include increased lane-to-lane spacing (the most effective approach), guard traces (grounded traces between adjacent pairs), staggered via fields (to reduce via-to-via coupling), and GSSG (ground-signal-signal-ground) routing patterns that provide inherent shielding between pairs.

See also: [Crosstalk and Interference](../05_signal_integrity/crosstalk_and_interference.md) for detailed crosstalk mitigation techniques.

---

### Q11. What is the role of causality and passivity enforcement in S-parameter data quality?

**Answer:**

S-parameter data used for serial link simulation must satisfy two fundamental physical constraints: causality and passivity. Causality requires that the system's impulse response is zero for negative time, meaning no output appears before the input arrives. Passivity requires that the network does not generate energy, meaning the total power scattered from all ports must not exceed the incident power at any frequency.

In practice, measured S-parameter data often violates one or both constraints due to measurement noise, calibration errors, time-gating artifacts, and finite frequency resolution. Non-causal data produces impulse responses with precursors that are artifacts rather than true physical effects, leading to incorrect ISI modeling. Non-passive data can cause time-domain simulations to become unstable, with artificial gain in certain frequency bands producing oscillations or divergent results.

Causality can be checked by verifying that the real and imaginary parts of each S-parameter satisfy the Kramers-Kronig relations, which are a mathematical consequence of causality. Passivity is verified by checking that the eigenvalues of (I - S^H * S) are non-negative at all frequencies, where S^H is the conjugate transpose of the S-parameter matrix and I is the identity matrix.

When violations are detected, enforcement algorithms adjust the S-parameter data to satisfy the constraints while minimizing changes to the original data. Common approaches include least-squares fitting to rational function models (which are inherently causal) and iterative eigenvalue adjustment for passivity. Tools such as Keysight ADS, Ansys HFSS, and Cadence Sigrity include automated enforcement routines.

For high-confidence link simulation, the S-parameter data quality should be validated before use. Key checks include verifying passivity and causality, checking reciprocity (S21 should equal S12 for a passive network), confirming that the data extends to sufficient bandwidth, and examining the impulse response for artifacts. Poor-quality S-parameter data is one of the most common causes of disagreement between simulation and measurement in serial link design.

---

### Q12. How does frequency-dependent loss impact the pulse response and eye diagram of a high-speed serial link?

**Answer:**

Frequency-dependent loss transforms a sharp, rectangular transmitted pulse into a smeared, rounded received pulse. The mechanism is straightforward: the channel attenuates high-frequency components of the pulse more than low-frequency components, and these high-frequency components are precisely what create the sharp transitions and flat tops of the pulse. When they are removed, the pulse edges become gradual and the pulse spreads in time.

Quantitatively, the pulse response is the inverse Fourier transform of the channel transfer function H(f) = SDD21(f). For a channel with 30 dB of insertion loss at Nyquist, the alternating bit pattern (whose fundamental is at Nyquist) is attenuated by a factor of approximately 31.6 (10^(30/20)), while DC and low-frequency components pass with minimal attenuation. This dramatic variation in attenuation across the signal bandwidth is what creates ISI.

The pulse response directly determines the eye diagram shape. Each bit in the data stream contributes a time-shifted copy of the pulse response to the received waveform, with polarity determined by the data bit value. The superposition of all these contributions, evaluated at the sampling instant, determines the eye opening. The main cursor of the pulse response contributes the desired signal, while the pre-cursor and post-cursor tails contribute ISI that partially closes the eye.

For a 112G PAM4 link, the situation is more challenging because the eye diagram has three vertically stacked eyes, each one-third the height of an NRZ eye. The frequency-dependent loss affects all three eyes equally, but the reduced vertical opening means that the same amount of ISI-induced eye closure represents a much larger fractional reduction in margin. This is why PAM4 links require both more aggressive equalization and FEC to achieve acceptable BER.

The relationship between channel loss and eye closure is nonlinear because equalization can recover a significant portion of the ISI, but only up to a point. Beyond the equalization capability of the SerDes (limited by the number and resolution of FIR taps, CTLE gain range, and DFE tap count), additional channel loss directly translates to eye closure and increased BER.
