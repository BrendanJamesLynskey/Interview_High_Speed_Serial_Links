# Eye Diagram Analysis

## Overview

The eye diagram is the primary visual and quantitative tool for assessing the quality of a high-speed serial link. This section covers eye diagram construction, measurement, bathtub curves, BER contours, and the Channel Operating Margin (COM) methodology used in AI system SerDes design.

---

### Q1. What is an eye diagram and how is it constructed?

**Answer:**

An eye diagram is a superposition of all possible bit transitions in a serial data stream, overlaid on a common time axis spanning one or two unit intervals. It is constructed by triggering an oscilloscope on the recovered clock (or a reference clock synchronized to the data) and overlaying successive unit intervals of the waveform. As thousands or millions of UI are accumulated, the overlay reveals the "eye" pattern: a diamond-shaped opening in the center where the signal can be sampled with low error probability, surrounded by the traces of all possible transition paths.

For NRZ signaling, the eye diagram has a single opening. For PAM4, there are three vertically stacked eye openings (between levels 0-1, 1-2, and 2-3), each typically one-third the height of the full signal swing. The eye height (vertical opening) and eye width (horizontal opening) at a specified BER are the primary quality metrics.

The eye diagram encodes all the signal quality information in a single image: the vertical opening reflects the signal amplitude minus the effects of noise, ISI, and crosstalk; the horizontal opening reflects the timing margin minus jitter; the thickness of the transition traces reflects the amount of deterministic jitter; the distribution of crossing points reveals duty cycle distortion; and the slope of the transitions at the crossing points indicates the signal bandwidth.

Modern eye diagram analysis uses statistical methods rather than simple accumulation. The vertical histogram at the sampling instant (center of the eye) gives the signal and noise distributions, from which the BER can be calculated. The horizontal histogram at the decision threshold gives the jitter distribution. These histograms can be decomposed into Gaussian (random) and bounded (deterministic) components using tail-fitting techniques.

---

### Q2. Define eye height, eye width, and how they are measured at a specific BER.

**Answer:**

Eye height is the vertical distance between the upper and lower bounds of the eye at the sampling instant, measured at a specified BER. For NRZ, it is the distance between the lowest high-level sample and the highest low-level sample. For PAM4, the inner eye height is the distance between adjacent level distributions within the smallest of the three eyes.

Eye height at a specific BER (such as 1e-6 or 1e-12) is extrapolated from the voltage histogram at the sampling instant. The histogram shows the distribution of sampled voltages, which is the superposition of signal, ISI, noise, and crosstalk. Each level's distribution can be modeled as a convolution of a bounded (deterministic ISI) component and a Gaussian (random noise) component. The eye height at BER = Q is found by determining the voltage at which the tail of the level distribution exceeds the threshold at the target BER probability. For a Gaussian distribution, the eye height at BER = 1e-12 is approximately the eye height at BER = 0.5 minus 2 * 7.03 * sigma_noise (where 7.03 is the quantile for BER = 1e-12).

Eye width is the horizontal distance (in UI or picoseconds) between the left and right edges of the eye at the decision threshold voltage, measured at a specified BER. The eye width represents the timing margin available for the CDR to position the sampling clock. Eye width at a specific BER is determined from the jitter histogram at the threshold crossing, using similar tail-fitting techniques.

Both eye height and eye width decrease as the target BER decreases (tighter requirements). A link that has 50 mV eye height at BER = 1e-4 might have only 20 mV at BER = 1e-12, because the tails of the noise distribution extend further at lower BER.

---

### Q3. What is a bathtub curve and how is it derived from the eye diagram?

**Answer:**

A bathtub curve is a plot of BER as a function of sampling position (either horizontal or vertical) across the eye. The horizontal bathtub curve plots BER versus sampling phase (in UI) at a fixed voltage threshold (typically the nominal decision threshold). The vertical bathtub curve plots BER versus sampling voltage at a fixed phase (typically the optimal sampling phase).

The horizontal bathtub curve has the shape of a bathtub (hence the name): the BER is extremely high (approaching 0.5) at the edges of the eye (where the sampling point is at the transition), decreases rapidly as the sampling point moves toward the center of the eye, reaches a minimum at the optimal sampling phase, and then increases again toward the other edge. The flat bottom of the bathtub represents the range of sampling phases where the BER is below the target, and the width of this flat region (at the target BER) equals the eye width.

The bathtub curve is derived from the jitter histogram at the eye crossing. The jitter histogram is the probability density function of the crossing times, which can be decomposed into a deterministic component (bounded, from DDJ, DCD, etc.) and a random component (Gaussian, from RJ). The bathtub curve is the cumulative distribution function (CDF) of the jitter distribution, computed from both edges of the eye. Mathematically:

```
BER(phase) = 0.5 * erfc((phase - mu_left) / (sqrt(2) * sigma_RJ)) 
           + 0.5 * erfc((mu_right - phase) / (sqrt(2) * sigma_RJ))
```

Where mu_left and mu_right are the mean crossing positions (shifted by the DJ distribution) and sigma_RJ is the RMS random jitter.

Bathtub curves are essential for determining the margin at any target BER. For compliance testing, the bathtub curve must show that the BER is below the target (e.g., 1e-6 for PAM4 before FEC) across a specified minimum eye width (e.g., 0.3 UI).

---

### Q4. What are BER contours and how do they provide a complete picture of link margin?

**Answer:**

BER contours are curves of constant BER plotted on the two-dimensional eye diagram (with time on the horizontal axis and voltage on the vertical axis). Each contour represents the locus of points where the BER equals a specific value, and a family of contours at different BER levels (such as 1e-3, 1e-6, 1e-9, 1e-12) reveals the full margin structure of the eye.

The BER contour at a given level forms a closed curve within the eye opening. The innermost contour (highest BER) is the largest, and the contours shrink as the BER level decreases (lower BER requires more margin). The area enclosed by the contour at the target BER represents the "safe" region where the sampling point can be placed while meeting the BER requirement. The center of this region is the optimal sampling point.

BER contours provide more information than eye height and eye width alone, because they reveal the shape of the margin in two dimensions simultaneously. An eye might have adequate eye height and adequate eye width when measured independently, but the BER contour might show that the height and width cannot be achieved simultaneously (because the corners of the eye are more restricted than the center). This situation arises when there is correlation between vertical and horizontal noise components (such as when jitter and amplitude noise are both caused by the same power supply disturbance).

BER contours are measured using the eye monitor circuit: the phase interpolator and voltage offset are scanned across the eye, and the BER is measured at each (phase, voltage) point using an error counter. The measurement time is proportional to 1/BER_target * number_of_points, which can be very long for low BER targets (measuring BER = 1e-12 at 1000 points would take approximately 10^15 / 56e9 / 1000 = 17.8 hours at 56 GBd). In practice, statistical extrapolation from shorter measurements (at higher BER levels) is used to estimate the contours at low BER.

---

### Q5. Explain the Channel Operating Margin (COM) methodology and its components.

**Answer:**

COM (Channel Operating Margin) is the IEEE 802.3-defined figure of merit that predicts the link margin of a high-speed serial link by combining channel characterization (S-parameters), transmitter specifications, receiver specifications, and equalization models. COM is expressed in dB, and a positive value indicates that the link has sufficient margin. The minimum acceptable COM is typically 3 dB.

The COM computation involves optimizing the equalization settings (TX FIR taps, CTLE peaking, DFE taps) to maximize the signal-to-noise ratio at the receiver sampling point. The signal component is the equalized main cursor amplitude. The noise components include residual ISI (the sum of all ISI contributions not canceled by equalization), crosstalk noise (from aggressor channels, computed from their S-parameters), transmitter noise (jitter converted to amplitude noise at the sampling instant), and receiver thermal noise.

The key steps in COM computation are: compute the channel pulse response from the SDD21 S-parameters via IFFT; optimize the TX FIR coefficients to maximize the equalized cursor while satisfying the coefficient constraints; for each CTLE setting in the discrete set, compute the CTLE-equalized pulse response; optimize the DFE taps to cancel the post-cursor ISI; compute the residual ISI from uncanceled taps; compute the crosstalk noise from the aggressor S-parameters; sum all noise components and compute COM = 20*log10(signal/noise).

COM has become the industry-standard metric for channel qualification in 100G and above serial link design, replacing the simpler insertion loss-based metrics used at lower data rates. It is computed using MATLAB scripts (available from IEEE) or commercial tools, and it is used during PCB design, connector selection, and system-level signal integrity analysis.

See also: [Channel Loss and Modeling](../01_foundations/channel_loss_and_modeling.md) for S-parameter inputs to COM.

---

### Q6. How do you interpret a PAM4 eye diagram differently from an NRZ eye diagram?

**Answer:**

A PAM4 eye diagram has three vertically stacked eyes instead of one, corresponding to the three transition zones between the four voltage levels. The interpretation differs from NRZ in several important ways.

Eye height analysis must consider all three eyes. The link BER is dominated by the worst-case (smallest) eye, so the minimum eye height across all three eyes determines the performance. Level non-linearity in the transmitter or receiver can cause the three eyes to have unequal heights, and the smallest eye may not be the middle eye (which would be expected for an ideal transmitter).

The voltage thresholds for a PAM4 eye diagram are three decision thresholds (between levels 0 and 1, between 1 and 2, and between 2 and 3), compared to one threshold for NRZ. Each threshold must be set accurately to minimize the BER for its respective eye. Threshold optimization is performed independently for each eye during receiver adaptation.

Timing analysis for PAM4 is more complex because different transition types have different timing characteristics. A transition from level 0 to level 3 (a large transition) may have different edge timing than a transition from level 1 to level 2 (a small transition), creating data-dependent jitter that is PAM4-specific. The eye width must be measured for the worst-case transition type.

The gray-coded bit mapping means that the upper bit (MSB) and lower bit (LSB) have different error characteristics. The MSB has one decision threshold (between levels 1 and 2), while the LSB has two decision thresholds (between levels 0 and 1, and between levels 2 and 3). The LSB BER is typically higher than the MSB BER because it depends on two eyes instead of one.

---

### Q7. What is the relationship between eye margin and BER in a serial link?

**Answer:**

Eye margin and BER are directly related through the probability distributions of signal and noise at the sampling point. The eye margin (in both voltage and timing) quantifies how far the nominal sampling point is from the BER threshold, expressed either in physical units (mV, ps) or in statistical units (sigma).

For a Gaussian noise model, the BER is related to the voltage margin by:

```
BER = erfc(V_margin / (sqrt(2) * sigma_noise)) / 2
```

Where V_margin is the distance from the decision threshold to the nearest level distribution mean, and sigma_noise is the RMS noise. For BER = 1e-12, the required V_margin is approximately 7.03 * sigma_noise. For BER = 1e-6 (typical pre-FEC target for PAM4), the required V_margin is approximately 4.75 * sigma_noise.

The margin budget is consumed by several factors. Static ISI reduces the mean distance between levels (closing the eye deterministically). Random noise (thermal, shot, flicker) creates a distribution around each level mean that encroaches on the decision boundary. Jitter creates a distribution of sampling times that samples away from the eye center, where the voltage margin is reduced. Crosstalk adds data-dependent noise from adjacent channels.

For a well-designed 112G PAM4 link with COM = 3 dB, the voltage margin is approximately 1.4 times the noise (10^(3/20) = 1.41), which corresponds to a BER of approximately 1e-1 to 1e-2 at the sampling point. This is well within the FEC correction capability (RS-544-514 corrects from approximately 2.4e-4), providing multiple layers of margin: the equalization provides most of the BER reduction, the COM margin provides a guard band against variation, and the FEC handles residual errors.

---

### Q8. How are eye diagrams measured in production SerDes using built-in eye monitors?

**Answer:**

Production SerDes include on-die eye monitors that measure the eye diagram without external test equipment. The eye monitor consists of a phase interpolator (PI) and a voltage comparator that can be offset from the nominal sampling point. By systematically varying the PI phase and the comparator threshold, the eye monitor scans across the (phase, voltage) space and measures the BER at each point.

The measurement process works as follows. The PI is set to a specific phase offset (relative to the CDR's optimal sampling phase). The comparator threshold is set to a specific voltage offset (relative to the nominal decision threshold). A PRBS pattern is transmitted, and the monitor comparator's output is compared with the main slicer's output. Any disagreement between the monitor and the main slicer indicates that the monitor's offset sampling point is outside the eye, and the disagreement rate equals the BER at that point.

The scan resolution is determined by the PI resolution (typically 7-8 bits, giving 128-256 phase steps per UI) and the comparator offset resolution (typically 6-8 bits, giving 64-256 voltage steps across the eye height). A full 2D scan with 128 x 128 = 16384 points, measuring each point for 10^6 bits (for BER = 1e-4 sensitivity), takes approximately 16384 * 10^6 / 56e9 = 293 ms at 56 GBd. For deeper BER sensitivity (1e-8), the measurement takes proportionally longer.

The eye monitor data is read out through a sideband interface (JTAG, SPI, or a register interface) and can be displayed as a 2D heat map (BER vs. phase and voltage) that represents the eye diagram at the receiver's sampling point. This includes all the effects of the channel, equalization, and receiver impairments, providing a true end-to-end picture of the link quality.

For production testing, a quick scan (measuring only a few points near the eye boundary) can verify that the eye meets the minimum specification in 10-50 ms per lane, which is fast enough for high-volume manufacturing.

---

### Q9. What eye diagram metrics are used for compliance testing of 112G serial links?

**Answer:**

Compliance testing for 112G serial links (IEEE 802.3ck, OIF CEI-112G) defines specific eye diagram metrics measured at the compliance test points (CTPs). The key metrics include:

Transmitter eye height (TEH): the minimum vertical opening of the transmitter output eye, measured at the compliance test point after a reference channel. For PAM4, TEH is specified for the worst-case (smallest) inner eye. Typical requirement: greater than 15-25 mV at BER = 1e-6.

Transmitter eye width (TEW): the minimum horizontal opening of the transmitter output eye at the decision threshold voltage. Typical requirement: greater than 0.25-0.35 UI at BER = 1e-6.

Transmitter SNDR (signal-to-noise-and-distortion ratio): a frequency-domain metric that quantifies the transmitter signal quality by measuring the signal power relative to noise and distortion at each frequency. SNDR greater than 25-28 dB is typical.

Receiver eye height and width: measured at the receiver's internal sampling point using the eye monitor, after all equalization (CTLE, DFE). These metrics verify that the receiver's equalization is sufficient for the reference channel.

Jitter measurements: total jitter (TJ), random jitter (RJ), and deterministic jitter (DJ) are measured from the horizontal eye closure and the jitter histograms. Typical limits: TJ less than 0.28 UI at BER = 1e-6, RJ less than 0.02 UI RMS, DJ less than 0.10 UI.

Level linearity (RLM): for PAM4, the Ratio Level Mismatch measures the uniformity of the four level spacings. RLM must be greater than 0.92 (each eye is at least 92% of the ideal height).

These metrics are measured using a high-bandwidth real-time or sampling oscilloscope, a BERT (bit error rate tester), and specialized compliance test fixtures that provide the reference channel and calibrated measurement environment.

---

### Q10. How does the eye diagram change with different equalization settings?

**Answer:**

The eye diagram transforms dramatically as different equalization stages are applied, and understanding these transformations is essential for optimizing the link.

Without any equalization (raw channel output), the eye is typically completely closed for a 112G PAM4 link on a medium-to-long channel. The waveform at the receiver is a smeared, ISI-dominated signal where individual symbols cannot be distinguished. The eye height is zero or negative (the ISI exceeds the signal).

After TX FIR pre-emphasis, the eye begins to open. The pre-emphasis boosts transitions relative to sustained levels, creating a distinctive waveform where the first bit after a transition has larger amplitude than subsequent identical bits. The eye diagram shows a partially open eye with characteristic "rabbit ears" at the transitions (where the pre-emphasized bits have higher amplitude). The improvement is typically 3-8 dB of eye height restoration.

After CTLE at the receiver, the eye opens further. The CTLE boosts high-frequency content, sharpening the transitions and reducing the ISI. However, the noise floor also rises (noise enhancement), and the eye may appear noisier even though the opening is larger. The eye height improvement is typically 5-12 dB, partially offset by 3-5 dB of noise enhancement.

After DFE, the post-cursor ISI is canceled, and the eye reaches its final quality. The DFE effectively removes the tails of the pulse response that extend into subsequent UI, sharpening the eye boundaries and maximizing the opening. The improvement from DFE is typically 4-10 dB of additional eye height. The final eye after all equalization should show a clear opening with measurable eye height and width at the target BER.

For PAM4, the three eyes respond differently to equalization because the ISI, noise, and non-linearity affect each level pair differently. The equalization is optimized for the worst-case eye, which means the other eyes may have excess margin.

---

### Q11. What is effective number of bits (ENOB) and how does it relate to PAM4 eye quality?

**Answer:**

The effective number of bits (ENOB) quantifies the resolution of a signal by expressing its signal-to-noise-and-distortion ratio in terms of the equivalent ideal ADC resolution. ENOB is defined as:

```
ENOB = (SNDR - 1.76) / 6.02
```

Where SNDR is in dB and 6.02 dB corresponds to one bit of resolution (a factor of 2 in voltage). This metric is particularly relevant for PAM4 transmitters and ADC-based receivers.

For a PAM4 transmitter, the four output levels represent a 2-bit DAC operation. An ideal 2-bit DAC has SNDR of 2 * 6.02 + 1.76 = 13.8 dB, corresponding to ENOB = 2.0. In practice, the transmitter's noise, non-linearity, and bandwidth limitations reduce the SNDR, giving ENOB less than 2.0. A transmitter with ENOB of 1.8 (SNDR = 12.6 dB) has its signal quality degraded by the equivalent of 0.2 bits of resolution loss, which translates to approximately 1.2 dB of penalty in the link budget.

For a PAM4 receiver with an ADC, the ENOB of the ADC directly limits the receiver's ability to distinguish between the four PAM4 levels. A 6-bit ADC (ideal ENOB = 6.0) provides approximately 4 bits of resolution beyond the minimum 2 bits needed for PAM4, giving approximately 24 dB of quantization-limited SNR. In practice, ADC ENOB at 56 GBd is typically 4.5-5.5, giving 15-20 dB of quantization-limited SNR.

The total link ENOB is limited by the weakest link in the chain: if the transmitter has ENOB of 5.5, the channel adds 20 dB of loss (reducing ENOB by approximately 3.3 bits), and the equalization recovers 15 dB (approximately 2.5 bits), the effective ENOB at the receiver input is approximately 5.5 - 3.3 + 2.5 = 4.7 bits, which must exceed the minimum of approximately 2.5-3.0 bits for reliable PAM4 detection at the target BER.

---

### Q12. How will eye diagram analysis evolve for 224G signaling?

**Answer:**

At 224G (112 GBd PAM4), eye diagram analysis faces new challenges that will drive evolution in both measurement techniques and analysis methodologies. The unit interval shrinks to approximately 8.9 ps, and the eye dimensions shrink proportionally, requiring higher measurement resolution and bandwidth.

Measurement bandwidth requirements increase significantly. To accurately capture the eye at 112 GBd, the oscilloscope must have at least 70-80 GHz of analog bandwidth (1.5 times Nyquist). Current state-of-the-art real-time oscilloscopes achieve approximately 70 GHz, which is marginally sufficient. Sampling oscilloscopes can achieve higher effective bandwidth but require repetitive signals and longer measurement times.

The eye opening at 224G is expected to be extremely small: inner eye heights of 5-15 mV and eye widths of 0.15-0.25 UI at BER = 1e-6. These tiny dimensions make the measurement more susceptible to noise and calibration errors in the test equipment itself, requiring careful de-embedding of the test fixture and probe contributions.

Statistical analysis becomes more important as direct BER measurement at 1e-12 becomes impractical (requiring 10^12 / 112e9 = 8.9 seconds of data, which is feasible but slow for 2D scanning). Extrapolation from higher-BER measurements using the dual-Dirac model or Gaussian mixture models will be the primary method for estimating eye margins at low BER.

Machine learning-based analysis may supplement or replace traditional eye diagram metrics. Neural networks trained on eye diagram images can predict the BER and margin with higher accuracy than analytical models, by capturing non-linear relationships between the eye features and the link performance that traditional models approximate with simplified assumptions.

For compliance testing at 224G, new metrics beyond COM may be needed to capture the additional impairments (non-linear ISI, level-dependent bandwidth, PAM4-specific distortion) that become dominant at 112 GBd. The industry is actively developing these metrics through the IEEE 802.3dj task force.
