# Driver Architectures

## Overview

The transmitter driver is the final analog stage that converts digital data into an analog signal on the differential transmission line. Driver architecture choices directly impact signal quality, power consumption, impedance matching, and scalability to higher data rates. This section covers the key driver topologies used in modern AI system SerDes.

---

### Q1. Compare voltage-mode and current-mode transmitter driver architectures.

**Answer:**

Voltage-mode drivers operate by switching impedance-matched output elements between supply voltage and ground, creating a voltage divider with the load termination. In the simplest form, a voltage-mode driver consists of PMOS pull-up and NMOS pull-down transistors sized to present an output impedance equal to the characteristic impedance of the transmission line (typically 50 ohms single-ended or 100 ohms differential). The output voltage swing is determined by the supply voltage divided between the driver output impedance and the load impedance. For a perfectly matched driver (Zout = Zload = 50 ohms), the output voltage at the driver pad is Vdd/2, and the voltage delivered to the far end of the line (assuming no loss) is also Vdd/2 due to the matched termination.

Current-mode logic (CML) drivers use a constant tail current source that is steered between two differential output branches by the input data. The output voltage swing is determined by the tail current multiplied by the load resistance: Vswing = I_tail * R_load. CML drivers inherently produce a constant current draw from the supply, which reduces simultaneous switching noise (SSN) because dI/dt is minimal during data transitions.

The key tradeoffs are power efficiency and supply voltage scalability. Voltage-mode drivers are more power-efficient because they dissipate power proportional to Vdd^2/R, and as Vdd scales with process nodes, power decreases quadratically. CML drivers dissipate power proportional to I_tail * Vdd, with the tail current set by the required output swing, making CML power less dependent on Vdd scaling and generally higher than voltage-mode at advanced nodes. However, CML provides better high-frequency performance due to constant current operation and lower sensitivity to supply noise. Modern 112G SerDes predominantly use voltage-mode drivers for power efficiency, with careful design to mitigate the SSN disadvantage.

---

### Q2. What is a Source-Series Terminated (SST) driver and why is it preferred for high-speed SerDes?

**Answer:**

A Source-Series Terminated (SST) driver is a voltage-mode driver where the output impedance of the driver itself provides the source termination for the transmission line. The driver consists of transistor segments whose combined on-resistance is designed to equal the characteristic impedance of the line (50 ohms per side for a 100-ohm differential pair). No additional termination resistor is needed at the transmitter, which saves area and power.

The SST topology is preferred for several reasons. When the driver launches a signal onto the line, the initial voltage step at the driver pad is Vdd/2 (for a matched 50-ohm driver into a 50-ohm line). This signal propagates to the receiver, where it is absorbed by the far-end termination. If the far-end termination is also matched, no reflection returns to the driver, and the signal is clean. Even if there is a small reflection from the far end, the SST driver's output impedance absorbs the returning reflection, preventing re-reflection.

Power efficiency is a key advantage: the SST driver delivers maximum power to the line because the output impedance matches the line impedance. The current drawn from the supply is Vdd / (Zout + Zline) = Vdd / (2 * 50) = Vdd / 100 per side. For Vdd = 0.9V, this is 9 mA per side, resulting in approximately 8 mW per differential driver at the output stage.

In practice, SST drivers are implemented as segmented arrays of unit cells, where each cell contributes a fraction of the total output impedance. The segmentation enables FIR pre-emphasis (by applying different data to different segments, as discussed in [Pre-emphasis and FIR](pre_emphasis_and_fir.md)), impedance calibration (by enabling or disabling segments to adjust the total impedance), and DAC functionality for PAM4 (by controlling which segments drive high or low to produce four output levels).

The main challenge of SST drivers is that the output impedance is sensitive to process, voltage, and temperature (PVT) variations. Transistor on-resistance varies with Vgs, Vds, temperature, and process corner, requiring calibration circuits that periodically adjust the number of active segments to maintain impedance matching.

---

### Q3. How does impedance calibration work in a modern SerDes transmitter?

**Answer:**

Impedance calibration adjusts the driver's output impedance to match the target characteristic impedance of the transmission line, compensating for PVT variations that would otherwise cause impedance mismatch and signal reflections.

The most common approach uses a replica-based calibration scheme. A replica driver cell (identical to the main driver unit cells but not connected to the signal path) is connected to an external precision resistor (typically 100 ohms for differential calibration). A feedback loop compares the voltage at the junction of the replica driver and the reference resistor against a reference voltage (typically Vdd/2). If the replica driver impedance is too low, the current through it is too high, and the junction voltage is higher than the reference; the calibration logic then reduces the driver strength by disabling some segments. Conversely, if the impedance is too high, more segments are enabled.

The calibration output is a digital code (typically 5-7 bits) that controls the number of active driver segments or adjusts the gate bias voltage of the driver transistors. This code is applied to all the main driver cells in the SerDes, under the assumption that all cells track the replica cell (which is valid because they are fabricated on the same die with the same process conditions).

Calibration is performed at power-up and periodically during operation (every few milliseconds to track temperature changes). Some designs use continuous background calibration that does not interrupt normal data transmission.

For PAM4 drivers, impedance calibration is even more critical because the four output levels are determined by the output impedance divider ratio. An impedance error shifts all four levels, reducing the already-tight spacing between adjacent levels and degrading the eye height. Independent calibration of the MSB and LSB segments may be required to maintain level linearity.

---

### Q4. What is the difference between CML and SST drivers in terms of supply noise sensitivity?

**Answer:**

CML (Current-Mode Logic) drivers draw a constant current from the supply regardless of the data pattern. The tail current source maintains I_tail whether the data is driving the positive or negative output, and only the distribution of current between the two branches changes during transitions. This means the total supply current has minimal data-dependent variation, and the dI/dt that causes supply noise coupling through package inductance is very small. The supply noise rejection ratio (SNRR) of CML drivers is typically 20-30 dB.

SST (Source-Series Terminated) voltage-mode drivers, by contrast, draw different amounts of current depending on the data pattern. When both the D+ and D- drivers are at the same level (common-mode transitions or during PAM4 middle levels), the supply current is different from when they are at opposite extremes. This data-dependent current creates switching noise on the supply that can couple back into the signal through the driver's finite power supply rejection.

The supply noise impact is quantified as follows. For a CML driver, the output voltage is V_out = I_tail * R_load, where I_tail is set by a current source with high output impedance. Supply noise Vn_dd modulates V_out only through the finite output resistance of the current source (typically 1-10 kohms), giving a rejection of Vn_out/Vn_dd = R_load / R_tail_source, which is approximately -40 to -60 dB.

For an SST driver, the output voltage is determined by the Vdd-to-ground voltage divided by the impedance ratio. Supply noise directly modulates the output: Vn_out/Vn_dd = Zline / (Zout + Zline) = approximately 0.5 (or -6 dB). This means 50% of the supply noise appears on the output signal.

Modern SST drivers mitigate this through several techniques: regulated supply for the output stage, differential operation (which cancels common-mode supply noise), careful power distribution network design with on-die decoupling, and supply noise compensation circuits that sense the supply variation and apply a corrective signal. The combination of these techniques improves the effective PSRR to approximately 20-30 dB, approaching CML performance but at lower power consumption.

---

### Q5. How are PAM4 transmitter levels generated in an SST driver architecture?

**Answer:**

PAM4 requires four equally spaced voltage levels at the driver output. In an SST driver, these levels are generated by controlling the fraction of driver segments that pull high versus low. The driver array is divided into segments weighted by their contribution to the output voltage.

The most common approach uses a thermometer-coded or segmented DAC architecture. Consider a driver with N unit segments, each contributing Vdd/(2N) to the output (when driving into a matched load). For the four PAM4 levels, the number of segments driving high is:

- Level 3 (highest): N segments high, 0 low
- Level 2: 2N/3 segments high, N/3 low
- Level 1: N/3 segments high, 2N/3 low
- Level 0 (lowest): 0 segments high, N segments low

The resulting output voltages (into a matched load) are approximately +Vdd/4, +Vdd/12, -Vdd/12, and -Vdd/4, giving three equally spaced eye openings of Vdd/6 each.

An alternative approach splits the driver into an MSB (most significant bit) sub-DAC and an LSB (least significant bit) sub-DAC, each controlling half the segments. The MSB selects between the upper and lower halves of the voltage range, and the LSB selects within each half. This binary-weighted approach is simpler to control but more susceptible to DNL (differential non-linearity) errors from MSB/LSB mismatch.

Practical challenges include maintaining level linearity across PVT corners. The spacing between levels must be uniform to within approximately 5% (as specified by RLM requirements) to avoid degrading the worst-case eye. Calibration circuits measure the actual output levels (using an on-die voltage comparator or an off-chip measurement during production testing) and adjust segment weights to correct for non-linearity. Timing alignment between MSB and LSB paths is also critical: if the MSB transitions at a different instant than the LSB, the output momentarily passes through an incorrect level, creating data-dependent jitter and eye distortion.

---

### Q6. What are the key specifications for a transmitter driver in a 112G PAM4 SerDes?

**Answer:**

A 112G PAM4 transmitter driver operating at 56 GBd must meet stringent specifications across multiple performance dimensions. The output swing is typically specified as 400-1000 mVpp differential, with most designs targeting 600-800 mVpp for an optimal balance between signal amplitude and power consumption. Higher swing improves the link budget but increases driver power and may violate electromagnetic compatibility limits.

The output impedance must be 50 ohms (plus or minus 10-15%) per side across the frequency band from DC to at least 40 GHz (1.5 times the Nyquist frequency). Impedance variation with frequency (due to parasitic capacitance and package effects) must be minimized through co-design of the driver and package.

Jitter specifications are critical: the total transmitter jitter must not exceed approximately 0.15-0.20 UI peak-to-peak, which corresponds to 2.7-3.6 ps at 56 GBd. The random jitter (RJ) component must be less than approximately 0.02 UI RMS (360 fs), and the deterministic jitter (DJ) must be less than approximately 0.10 UI (1.8 ps). These jitter limits are necessary to preserve the timing margin at the receiver CDR.

For PAM4, the transmitter linearity is specified by the Ratio Level Mismatch (RLM), which must exceed approximately 0.92 (meaning each eye height is at least 92% of the ideal value). The transmitter signal-to-noise-and-distortion ratio (SNDR) at the output must be at least 25-28 dB at 56 GBd.

The FIR filter typically provides 3-5 taps with at least 6 bits of coefficient resolution per tap. The main cursor tap has a coefficient of approximately 0.7-0.9, with pre-cursor and post-cursor taps providing up to 0.3 of the main cursor amplitude. Rise and fall times must be symmetric to within 10% to minimize duty cycle distortion, and the common-mode output voltage must remain stable within approximately 10 mV during transitions.

---

### Q7. How does driver segmentation enable FIR pre-emphasis in an SST architecture?

**Answer:**

Driver segmentation is the technique of dividing the output driver into multiple groups of unit cells, each driven by a different delayed version of the data. This allows the driver to implement a multi-tap FIR (finite impulse response) filter directly in the analog output stage, without a separate digital-to-analog conversion step.

In a 3-tap SST FIR driver, the segments are divided into three groups: the pre-cursor group is driven by the data that is one UI ahead of the current bit (D[n+1]), the main cursor group is driven by the current data (D[n]), and the post-cursor group is driven by the data from one UI earlier (D[n-1]). The number of segments in each group determines the tap coefficients. For example, if the total driver has 64 segments and the allocation is 8 segments for pre-cursor, 48 for main, and 8 for post-cursor, the tap coefficients are approximately c[-1] = -8/64 = -0.125, c[0] = 48/64 = 0.75, and c[1] = -8/64 = -0.125.

The sign of each tap is controlled by whether the data drives the segments in the normal or inverted direction. For de-emphasis taps (which reduce the amplitude of bits adjacent to transitions), the pre-cursor and post-cursor groups are driven with inverted polarity relative to the main cursor. This creates the characteristic pre-emphasis waveform where the first bit after a transition has full amplitude and subsequent same-valued bits have reduced amplitude.

The key advantage of this approach is that the FIR filtering happens at the point of signal generation, avoiding the need for a separate high-speed DAC and the associated power and bandwidth limitations. The disadvantage is that coefficient resolution is limited by the number of segments (64 segments provide approximately 6-bit resolution), and the impedance of the driver changes slightly as the tap allocation changes (because each group's effective impedance depends on how many segments are driving high versus low). Advanced designs use additional impedance compensation segments that activate to maintain constant output impedance regardless of the FIR setting.

See also: [Pre-emphasis and FIR](pre_emphasis_and_fir.md) for detailed FIR design and optimization.

---

### Q8. What are the bandwidth requirements for a 56 GBd transmitter driver and how are they achieved?

**Answer:**

A 56 GBd transmitter driver must provide adequate signal amplitude and timing accuracy up to at least the Nyquist frequency of 28 GHz, and preferably up to 1.5-2 times Nyquist (42-56 GHz) to preserve the higher-frequency content that contributes to sharp transitions and eye quality. The 3-dB bandwidth of the driver output stage should be at least 35-40 GHz.

Achieving this bandwidth in CMOS is challenging because the driver transistors have significant parasitic capacitance (gate-drain, gate-source, and drain-source capacitances) that limit the output bandwidth. For a 5nm FinFET process, the transit frequency (fT) of NMOS transistors is approximately 300-400 GHz, which is well above the required bandwidth. However, the actual driver bandwidth is limited not by fT but by the RC time constant of the output node, which includes the driver's output resistance, the driver parasitic capacitance, the ESD protection capacitance, and the package parasitic inductance and capacitance.

Design techniques to achieve the required bandwidth include minimizing the ESD protection capacitance (using low-capacitance ESD structures with total capacitance below 100 fF per pin), co-designing the driver with the package to absorb package parasitics into a broadband matching network, using inductive peaking (series inductors in the signal path that resonate with the parasitic capacitance to extend the bandwidth), and employing cascode or stacked transistor topologies that reduce the Miller effect and improve the output impedance bandwidth.

For PAM4 at 56 GBd, the bandwidth requirement is more nuanced: the driver must maintain level linearity across the bandwidth, not just amplitude. A driver that has 40 GHz bandwidth for NRZ may exhibit level-dependent bandwidth compression for PAM4 because the transistors operate at different bias points for each level, resulting in different parasitic capacitances and therefore different bandwidths. This level-dependent bandwidth variation is one of the sources of transmitter non-linearity that degrades SNDR.

---

### Q9. How does the transmitter output stage interface with the package, and what parasitic effects must be managed?

**Answer:**

The transmitter output stage interfaces with the package through a chain of structures: the on-die output pad (typically a large metal structure with significant capacitance, approximately 50-100 fF), the bump or wire bond (which adds inductance, approximately 50-200 pH for flip-chip bumps or 1-3 nH for wire bonds), the package redistribution layer (RDL) or substrate trace, the package via transitions, and the BGA ball that connects to the PCB.

Each of these structures introduces parasitic elements that degrade the signal. The pad capacitance and bump inductance form a low-pass filter that rolls off the high-frequency content. The package trace introduces additional insertion loss and potential impedance discontinuities. The BGA ball adds inductance (approximately 100-300 pH) and the PCB pad adds capacitance.

To manage these parasitics, the driver and package are co-designed as an integrated system. The pad capacitance and bump inductance can be absorbed into a pi-network matching structure by adding intentional capacitance or inductance to form a broadband impedance transformer. The package trace impedance is designed to match the target differential impedance (85 or 100 ohms), with careful routing to maintain impedance continuity.

For advanced package technologies used in AI accelerators (such as TSMC CoWoS or Intel EMIB), the silicon interposer or bridge provides much lower parasitic interconnects than traditional organic substrates. A CoWoS micro-bump has only approximately 10-30 pH of inductance, compared to 50-200 pH for a standard flip-chip bump, which significantly relaxes the driver bandwidth requirements for die-to-die interfaces.

The package co-design process typically involves iterative electromagnetic simulation of the package model (using tools like Ansys HFSS or Cadence Clarity) to extract accurate S-parameters, followed by combined simulation of the driver circuit with the package model to verify that the overall transmitter output meets the impedance and bandwidth specifications.

---

### Q10. What is duty cycle distortion (DCD) in a transmitter and how is it controlled?

**Answer:**

Duty cycle distortion (DCD) is a deviation of the data pulse width from the ideal 50% duty cycle. In an NRZ transmitter, a positive DCD means the high pulse is wider than the low pulse (or vice versa). DCD is measured as the difference between the actual and ideal crossing point of the differential signal, expressed in UI or picoseconds. For a 56 GBd transmitter, the DCD specification is typically less than 0.02-0.03 UI (360-540 fs).

DCD arises from asymmetry between the pull-up and pull-down paths in the driver. In a CMOS SST driver, the PMOS pull-up has different switching characteristics than the NMOS pull-down due to mobility differences (PMOS has approximately 2-3x lower mobility than NMOS in advanced nodes), threshold voltage mismatch, and different parasitic capacitances. This asymmetry causes the rising and falling edges to have different delays, shifting the zero-crossing point and creating DCD.

DCD is harmful because it introduces a systematic jitter component that cannot be tracked by the receiver's CDR (which attempts to find the average crossing point). The CDR positions its sampling clock between the average rising and falling edges, but if the duty cycle is distorted, the sampling point is not optimal for either transition type, reducing the effective eye width.

Control techniques include: static calibration, where a duty cycle correction (DCC) circuit measures the average DC level of the output (which deviates from Vdd/2 when DCD is present) and adjusts the pull-up/pull-down balance by changing the relative driver strengths or supply voltages; dynamic calibration, where a feedback loop continuously adjusts the DCC during normal data transmission; and balanced circuit design, where the driver topology is designed for inherent symmetry (such as using complementary driver pairs with matched load capacitances).

For PAM4 transmitters, DCD manifests differently: the asymmetry affects transitions between different level pairs, creating pattern-dependent DCD that is more complex than the simple NRZ case. PAM4 DCD calibration must separately adjust the timing for each transition type.

---

### Q11. What are the thermal considerations for high-speed transmitter drivers in AI accelerator packages?

**Answer:**

Thermal management of transmitter drivers is a critical concern in AI accelerators because the SerDes I/O consumes a significant fraction of the total chip power, and the driver transistors are concentrated in a small area at the chip periphery. An NVIDIA H100 GPU, for example, has over 100 SerDes lanes, each dissipating approximately 100-200 mW in the output driver alone, for a total driver power of 10-20W concentrated along the chip edges.

Temperature affects driver performance in several ways. The on-resistance of MOSFET transistors increases with temperature (approximately 0.3-0.5% per degree Celsius for advanced FinFET devices), which shifts the driver output impedance upward. If the impedance calibration loop has insufficient bandwidth or is calibrated infrequently, the impedance mismatch can increase during thermal transients, degrading return loss.

Carrier mobility decreases with temperature, reducing the transistor's transconductance and bandwidth. At elevated temperatures (100-125 degrees Celsius junction temperature, which is typical in AI accelerators), the driver bandwidth may decrease by 5-10% compared to room temperature, reducing the eye opening at high frequencies.

Threshold voltage decreases with temperature, which affects the bias points of the driver transistors and can shift the PAM4 output levels. The temperature coefficient of Vth is approximately -0.5 to -1.0 mV per degree Celsius, which for a 150 mV PAM4 eye height represents a potential 1-2% shift per 30-degree temperature change.

Power dissipation increases in a positive feedback loop: higher temperature increases on-resistance, which for voltage-mode drivers can increase the current draw (if the supply voltage is fixed and the impedance calibration overcompensates), further increasing temperature. This thermal runaway risk is managed by thermal throttling circuits that reduce the data rate or output swing when the junction temperature exceeds a safe threshold.

Design mitigations include distributing the driver power across multiple power supply domains to avoid local hotspots, using metal fill structures to improve lateral heat spreading, placing temperature sensors near the driver array for real-time monitoring, and designing the impedance calibration loop with sufficient bandwidth to track thermal transients.

---

### Q12. How do emerging driver architectures address the challenges of 224G per lane signaling?

**Answer:**

At 224G per lane (112 GBd PAM4), the driver design challenges push beyond the capabilities of conventional SST architectures. The required bandwidth (56+ GHz 3-dB bandwidth at the driver output) approaches the limits of even 3nm FinFET processes when accounting for package parasitics and ESD protection.

Several emerging architectural approaches address these challenges. DSP-DAC transmitters replace the traditional analog FIR driver with a high-speed digital signal processor feeding a multi-bit DAC. The DSP computes the pre-equalized signal (including FIR, non-linear pre-distortion, and Tomlinson-Harashima pre-coding) in the digital domain at the full baud rate, and the DAC converts this to the analog output. This approach enables more complex equalization (longer FIR filters, non-linear compensation) but requires a DAC with 6-8 bits of effective resolution at 112 GBd, which is a significant analog design challenge.

Time-interleaved architectures use multiple parallel driver sub-arrays that operate at a fraction of the full baud rate, with their outputs combined using precise interleaving. For example, two driver sub-arrays operating at 56 GBd each can be interleaved to produce a 112 GBd output. This relaxes the bandwidth requirement of each sub-array but introduces timing skew between the interleaved paths that must be calibrated to sub-picosecond accuracy.

Active equalization at the transmitter package boundary uses on-package or in-package active circuits to compensate for package parasitics. Instead of trying to minimize package effects (which becomes increasingly difficult), the active circuits measure and cancel the package impairments, effectively extending the driver bandwidth.

Photonic integration, where the electrical driver is co-packaged with a silicon photonic modulator (co-packaged optics or CPO), eliminates the need for long electrical channels entirely. The driver in this case only needs to drive the modulator, which is located within a few millimeters, drastically reducing the bandwidth and power requirements. CPO is being actively developed for AI datacenter interconnects by Intel, Broadcom, and others.
