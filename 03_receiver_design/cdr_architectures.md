# CDR Architectures

## Overview

Clock and Data Recovery (CDR) extracts the timing information embedded in the serial data stream to position the receiver's sampling clock at the optimal point within each unit interval. This section covers CDR phase detector types, loop design, and the tradeoffs between jitter tolerance and jitter transfer.

---

### Q1. What is clock and data recovery (CDR) and why is it needed in serial links?

**Answer:**

Clock and data recovery (CDR) is the process of extracting a clock signal from a serial data stream that carries no explicit clock signal. In high-speed serial links, the transmitter embeds timing information in the data by using the transition edges between symbols as implicit clock references. The CDR circuit at the receiver uses these transitions to reconstruct a clock that is aligned with the data, enabling the receiver to sample the data at the optimal point within each unit interval.

CDR is needed because it is impractical to distribute a separate high-speed clock from the transmitter to the receiver. A forwarded clock at 56 GHz would suffer from the same channel loss and jitter as the data signal, and the skew between the clock and data paths would be difficult to control. Instead, embedding the clock in the data ensures that any delay or jitter that affects the data also affects the implicit timing references, maintaining the alignment between clock and data at the receiver.

The CDR circuit consists of three main components: a phase detector that compares the phase of the data transitions with the sampling clock, a loop filter that processes the phase error signal, and a clock source (either a VCO or a phase interpolator) that adjusts the clock phase or frequency based on the filtered error signal. The loop operates as a feedback system that continuously adjusts the sampling clock to track the data transitions, maintaining the sampling point near the center of the data eye.

The CDR must handle two operating modes: frequency acquisition (when the receiver first establishes a link and the clock frequency may be significantly different from the data rate) and phase tracking (during normal operation, when the clock frequency is locked and only phase adjustments are needed). Frequency acquisition requires a wider capture range and may use a separate frequency-locked loop or a reference clock to bring the VCO close to the correct frequency before the CDR takes over for phase tracking.

---

### Q2. Compare the Alexander (bang-bang) phase detector with the Mueller-Muller phase detector.

**Answer:**

The Alexander (bang-bang) phase detector and the Mueller-Muller phase detector are the two most common phase detector types in modern SerDes CDR circuits. They differ in how they extract phase information from the data stream.

The Alexander phase detector (also called a bang-bang or binary phase detector) takes three samples per UI: a data sample (D) at the center of the eye, and an edge sample (E) at the transition between consecutive UI. By comparing the data samples of adjacent UI with the edge sample, the phase detector determines whether the sampling clock is early or late. If D[n-1] differs from D[n] (indicating a transition) and E matches D[n-1], the clock is late; if E matches D[n], the clock is early. The output is a binary signal: early or late, with no magnitude information.

The Mueller-Muller phase detector does not require an edge sample. Instead, it uses the data samples at two consecutive UI and correlates them with the signal amplitude. The principle is that if the clock is early, the sample is taken on the rising/falling edge of a transition, and the correlation between consecutive samples has a specific sign. The Mueller-Muller phase error is computed as:

```
PE[n] = d[n-1] * x[n] - d[n] * x[n-1]
```

Where d[n] is the decided data value and x[n] is the analog sample value. This provides a phase error with both sign and magnitude information.

| Feature | Alexander (Bang-Bang) | Mueller-Muller |
|---------|----------------------|----------------|
| Samples per UI | 2 (data + edge) | 1 (data only) |
| Output type | Binary (early/late) | Linear (proportional to error) |
| Hardware complexity | Higher (edge sampler) | Lower (no edge sampler) |
| CDR loop behavior | Non-linear (limit cycle) | Linear (smooth tracking) |
| Jitter tolerance | Generally better | Good |
| Sensitivity to ISI | Less sensitive | More sensitive to residual ISI |
| Use case | Most common for NRZ/PAM4 | Gaining adoption for PAM4 |

The Alexander detector is more widely used because it is robust against ISI (the edge sample provides a direct measurement of the transition timing). The Mueller-Muller detector is gaining popularity for PAM4 because it avoids the need for an edge sampler at the PAM4 transitions (which are hard to sample accurately due to the multiple transition types) and because its linear output provides smoother CDR loop behavior.

---

### Q3. How does the CDR loop bandwidth affect jitter tolerance and jitter transfer?

**Answer:**

The CDR loop bandwidth is the most critical design parameter, determining the tradeoff between jitter tolerance (JTOL) and jitter transfer (JTRAN). These are the two key CDR specifications, and they are fundamentally linked through the loop bandwidth.

Jitter tolerance (JTOL) is the maximum sinusoidal jitter amplitude that can be applied to the data without causing the BER to exceed the target. JTOL is a function of frequency: at low frequencies (well below the CDR bandwidth), the CDR tracks the jitter and the tolerance is limited by the CDR's tracking range (which is very large, typically hundreds of UI). At high frequencies (well above the CDR bandwidth), the CDR cannot track the jitter and the tolerance is limited by the eye opening (typically 0.1-0.3 UI). At frequencies near the CDR bandwidth, there is a transition between these two regimes.

A wider CDR bandwidth provides better jitter tolerance at high frequencies (because the CDR can track faster jitter) but worse jitter transfer (because the CDR passes more of the input jitter to the recovered clock). Conversely, a narrower bandwidth provides better jitter filtering (lower jitter transfer) but worse jitter tolerance at high frequencies.

Jitter transfer (JTRAN) is the ratio of output jitter to input jitter as a function of frequency. For a well-designed CDR, the jitter transfer is approximately 0 dB at low frequencies (the CDR tracks the input jitter faithfully), peaks slightly near the CDR bandwidth (due to loop peaking, typically less than 0.1-0.2 dB), and rolls off at 20 dB/decade above the bandwidth.

Standards specify minimum JTOL (the CDR must tolerate at least a certain amount of jitter at each frequency) and maximum JTRAN (the CDR must not amplify jitter, with peaking limited to 0.1-0.2 dB). These dual specifications constrain the CDR bandwidth to a specific range, typically 4-10 MHz for 112G links.

The CDR bandwidth must also be wide enough to track spread-spectrum clocking (SSC) modulation at 33 kHz with 5000+ ppm frequency deviation, and to maintain lock during data stream interruptions (such as SKP ordered sets in PCIe that create periodic gaps in the data stream).

---

### Q4. Explain the proportional and integral paths in a CDR loop filter.

**Answer:**

The CDR loop filter processes the phase detector output to generate the control signal for the VCO or phase interpolator. A typical CDR loop filter has two paths: a proportional path and an integral path, analogous to a PI controller.

The proportional path provides immediate phase correction proportional to the phase error. When the phase detector indicates that the sampling clock is early or late, the proportional path adjusts the clock phase by a fixed step in the corrective direction. The proportional gain (Kp) determines the step size and therefore the speed of phase tracking. A larger Kp provides faster tracking (better high-frequency jitter tolerance) but causes more jitter in steady state (because the proportional path responds to noise in the phase detector output, creating a limit cycle in bang-bang CDRs).

The integral path provides frequency correction by accumulating the phase errors over time. If the data rate and the local clock frequency are slightly different (due to frequency offset between the transmitter and receiver reference clocks), the phase error accumulates in one direction. The integral path detects this trend and adjusts the clock frequency to eliminate the offset. The integral gain (Ki) determines the speed of frequency acquisition and the steady-state frequency tracking accuracy. A larger Ki provides faster frequency acquisition but also amplifies low-frequency phase noise.

The combined transfer function of the PI loop filter creates a second-order CDR loop with the following characteristics. The CDR bandwidth is primarily set by the proportional gain Kp: BW approximately equal to Kp * K_VCO / (2*pi), where K_VCO is the VCO gain (Hz/V) or the phase interpolator step size (UI/code). The damping ratio is set by the ratio of Kp to Ki: zeta = (Kp / (2*Ki)) * sqrt(Ki * K_VCO). A damping ratio of 0.7-1.0 is typical for critically damped to slightly overdamped response, which provides fast settling without excessive ringing.

For digital CDR implementations (where the loop filter is a digital accumulator), the proportional gain is determined by the phase interpolator step size per phase detector update, and the integral gain is determined by the accumulator's gain (how many phase detector updates are averaged before adjusting the frequency).

---

### Q5. What is the jitter peaking specification and why is it critical for cascaded links?

**Answer:**

Jitter peaking is the maximum amplification of input jitter by the CDR, occurring at frequencies near the CDR loop bandwidth. It is expressed in dB and is specified to be less than 0.1-0.2 dB for most standards. Excessive jitter peaking means the CDR amplifies jitter at certain frequencies, which can cause jitter accumulation in cascaded links.

In AI systems, data often passes through multiple SerDes links in series: for example, a GPU transmits data to a switch, which re-serializes and transmits to another GPU. Each link's CDR recovers the clock and re-transmits with its own jitter characteristics. If each CDR has jitter peaking of P dB at a specific frequency, the total jitter amplification after N links is N*P dB at that frequency. For a system with 4 cascaded links and 0.2 dB peaking per link, the total amplification is 0.8 dB, which is significant.

If the peaking exceeds 0 dB at any frequency, jitter accumulates exponentially with the number of links, eventually causing link failure. This is why the jitter peaking specification is so stringent. PCIe specifies less than 0.1 dB peaking for up to 6 cascaded links (including retimers). Ethernet specifies less than 0.1 dB for multi-hop networks.

Jitter peaking in the CDR arises from underdamped loop response (insufficient damping ratio) or from peaking in the VCO/PI transfer function. Reducing peaking requires careful loop filter design: increasing the damping ratio (by increasing the proportional-to-integral gain ratio) reduces peaking but also narrows the CDR bandwidth, which degrades jitter tolerance. The design must balance peaking, bandwidth, and tolerance simultaneously.

For retimer-less links (such as UCIe die-to-die interfaces), jitter peaking is not a concern because there is no clock recovery between the two endpoints. For links that traverse retimers or switches with CDR, jitter peaking management is essential.

---

### Q6. How does CDR design differ for PAM4 compared to NRZ signaling?

**Answer:**

CDR design for PAM4 presents several unique challenges compared to NRZ. The most fundamental difference is that PAM4 has three distinct eye openings, and not all symbol transitions carry the same timing information. In NRZ, every transition crosses the single decision threshold and provides a timing reference. In PAM4, transitions between adjacent levels (0-1, 1-2, 2-3) have different amplitudes than transitions between non-adjacent levels (0-2, 1-3, 0-3), and each transition crosses the zero-crossing at a different point.

For Alexander-type bang-bang CDR, the edge sample must be positioned at the optimal timing point. For NRZ, this is the zero-crossing of the differential signal. For PAM4, there are three distinct threshold crossings (between levels 0 and 1, between 1 and 2, and between 2 and 3), and the edge sample must work correctly for all three. One approach uses the middle threshold (between levels 1 and 2) as the edge sampling point, since all transitions cross this threshold. However, small-amplitude transitions (between adjacent levels) have noisy crossings, degrading the phase detector accuracy.

Mueller-Muller CDR is attractive for PAM4 because it operates on data samples only and does not require edge sampling. The phase error computation uses the correlation between decided data values and analog sample amplitudes, which works naturally for multi-level signals. The Mueller-Muller phase error for PAM4 is:

```
PE[n] = d[n-1] * x[n] - d[n] * x[n-1]
```

Where d[n] takes one of four PAM4 level values. This computation inherently weights the phase error by the transition amplitude, giving more weight to large transitions (which have better timing information) and less weight to small transitions.

The CDR bandwidth for PAM4 is typically set slightly narrower than for NRZ at the same baud rate, because the noisier phase detector output (due to the reduced PAM4 eye margins) creates more jitter in the recovered clock. A narrower bandwidth filters this additional jitter but reduces the jitter tolerance, requiring careful optimization.

PAM4 CDR must also handle the increased deterministic jitter from PAM4-specific effects such as level-dependent transition timing (where the transition speed depends on the starting and ending levels, creating data-dependent jitter) and RLM-induced timing asymmetry.

---

### Q7. What is the CDR frequency acquisition process and how is it handled in modern SerDes?

**Answer:**

Frequency acquisition is the initial phase where the CDR must lock to the correct frequency before it can track the phase. If the local clock frequency is significantly different from the incoming data rate (due to different reference clocks, frequency tolerance, or spread-spectrum clocking), the CDR's phase detector sees a continuously rotating phase that it cannot track with phase adjustments alone. The integral path of the CDR loop eventually adjusts the frequency, but the frequency acquisition range of the CDR loop is limited (typically plus or minus 100-1000 ppm of the nominal frequency).

Modern SerDes handle frequency acquisition through several approaches. Reference-based acquisition uses a local PLL locked to a known reference clock to set the initial VCO or PI frequency close to the expected data rate. If the reference is accurate to within 100 ppm, the remaining frequency offset is within the CDR loop's tracking range, and no separate frequency acquisition is needed.

Frequency detector-assisted acquisition adds a dedicated frequency detector (separate from the phase detector) that measures the frequency offset and drives the VCO toward the correct frequency. Common frequency detector implementations include the rotational frequency detector, which counts the number of phase rotations per unit time, and the quadricorrelator, which cross-correlates early/late phase detector outputs to estimate the frequency offset. The frequency detector is active during initialization and disabled after frequency lock is achieved, with the CDR phase detector taking over for phase tracking.

Sweep-based acquisition slowly varies the VCO frequency across its tuning range while monitoring the phase detector output for lock indication. When the frequency comes within the CDR's pull-in range, the CDR locks and the sweep stops. This approach is simple but slow (potentially taking milliseconds for a wide sweep range).

For AI system links that use a common reference clock distribution (such as PCIe with a shared 100 MHz reference), the frequency offset between transmitter and receiver is typically less than 300 ppm, which is well within the CDR's pull-in range without a separate frequency detector. For links with independent clocks (such as Ethernet over long cables), a frequency detector may be needed.

---

### Q8. How does the CDR maintain lock during data stream interruptions (electrical idle, SKP ordered sets)?

**Answer:**

Modern serial link protocols periodically interrupt the data stream for protocol-level functions such as SKP ordered sets (PCIe), idle characters (Ethernet), or low-power states (electrical idle). During these interruptions, the CDR may not receive valid data transitions, and the clock must maintain its frequency and phase until data resumes.

SKP ordered sets in PCIe are inserted periodically (approximately every 1180 symbols at Gen5) to allow the receiver to compensate for frequency differences between the transmitter and receiver reference clocks. During SKP insertion, the data stream is briefly interrupted by a known pattern. The CDR must continue tracking through the SKP ordered set, using the transitions in the SKP pattern (if any) or coasting on its current frequency and phase.

The CDR maintains lock during interruptions through several mechanisms. The VCO or phase interpolator has inherent frequency memory: the control voltage (for VCO-based CDR) or the frequency accumulator (for PI-based CDR) holds its value during the interruption, and the clock continues running at the last tracked frequency. For interruptions shorter than the CDR's frequency drift time (determined by how quickly the VCO drifts due to leakage or noise), the CDR maintains adequate frequency accuracy.

Some CDR implementations use a "hold" mode during known interruptions: the protocol layer signals the CDR that an interruption is coming, and the CDR freezes its loop filter (disabling both proportional and integral updates) for the duration. This prevents the CDR from responding to non-data content that could corrupt the phase/frequency estimate.

For longer interruptions (such as electrical idle, which can last milliseconds to seconds), the CDR must re-acquire lock when data resumes. The re-acquisition time depends on how far the clock has drifted during idle: if the frequency drift is within the CDR's pull-in range, re-acquisition takes only a few microseconds; if it exceeds the pull-in range, a full frequency acquisition may be needed.

---

### Q9. What is the CDR's role in equalization and how does it interact with DFE adaptation?

**Answer:**

The CDR and DFE are tightly coupled because both optimize the sampling of the received signal, but they adjust different parameters. The CDR adjusts the sampling clock timing (horizontal positioning within the eye), while the DFE adjusts the decision threshold and ISI cancellation (vertical positioning and eye opening). The interaction between these two loops must be carefully managed to avoid instability or suboptimal convergence.

The coupling arises because the DFE tap values depend on the sampling phase. If the CDR shifts the sampling point by a fraction of a UI, the pulse response at the new sampling point has different ISI values, requiring the DFE taps to adapt to the new values. Conversely, the DFE affects the CDR's phase detector: by canceling ISI, the DFE changes the apparent transition timing, which shifts the CDR's preferred sampling point.

In practice, the CDR loop bandwidth is much wider than the DFE adaptation bandwidth. The CDR tracks phase changes on a symbol-by-symbol basis (bandwidth of several MHz), while DFE adaptation converges over thousands of symbols (bandwidth of 10-100 kHz). This separation of bandwidths ensures stability: the CDR settles to a new phase before the DFE has time to react, and the DFE then adapts to the settled phase. If the bandwidths were similar, the two loops could interact and create oscillations.

For Mueller-Muller CDR, the interaction with DFE is particularly important because the Mueller-Muller phase detector's output depends on the signal amplitude, which is affected by the DFE's ISI cancellation. If the DFE taps change, the phase detector's characteristic changes, which can shift the CDR's lock point. This is managed by ensuring that the DFE adaptation converges before the CDR is allowed to track, or by designing the Mueller-Muller detector to use the decided data values (which are discrete and unaffected by the DFE) rather than the analog samples.

Some advanced CDR architectures jointly optimize the CDR phase and DFE taps using a unified cost function, such as minimizing the mean squared error across both phase and ISI dimensions simultaneously. This joint optimization avoids the suboptimality of sequential adaptation but requires more complex hardware.

---

### Q10. What are the key CDR specifications for 112G PAM4 SerDes?

**Answer:**

The CDR for a 112G PAM4 SerDes operating at 56 GBd has demanding specifications that reflect the tight timing margins at this data rate. The CDR bandwidth is typically 4-10 MHz, chosen to balance jitter tolerance and jitter transfer. IEEE 802.3ck specifies a minimum bandwidth of 4 MHz for jitter tolerance compliance and a maximum bandwidth for jitter transfer compliance (implicitly constrained by the 0.1 dB peaking limit).

Jitter tolerance must meet the standard's JTOL mask, which specifies the minimum tolerable sinusoidal jitter amplitude as a function of frequency. At low frequencies (below 100 kHz), the tolerance must be at least 250-500 UI (enormous, because the CDR tracks low-frequency jitter). At the CDR bandwidth (4-10 MHz), the tolerance transitions to a constant value of approximately 0.15-0.25 UI. At high frequencies (above 80 MHz), the tolerance is limited by the eye opening and is typically 0.05-0.1 UI.

Jitter transfer peaking must be less than 0.1 dB to prevent jitter accumulation in cascaded links. The 3-dB jitter transfer bandwidth must be within the specified range (typically 4-12 MHz).

The CDR lock time must be less than the link training time budget, typically 1-10 milliseconds from initial frequency acquisition to phase lock. This includes the time for frequency acquisition (if needed), phase lock, and initial equalization convergence.

The phase interpolator resolution should be 7-8 bits (128-256 steps per UI), providing a phase adjustment step of approximately 0.004-0.008 UI (70-140 fs at 56 GBd). This resolution determines the CDR's quantization jitter, which contributes to the recovered clock jitter. The PI linearity (integral non-linearity, INL) should be less than 1-2 LSB to avoid systematic phase errors that create periodic jitter in the recovered clock.

For PAM4, the CDR must also handle the increased pattern-dependent jitter (from PAM4 level-dependent transition timing) and the noisier phase detector output (from the reduced PAM4 eye margins), which typically requires 2-3 dB more SNR margin in the CDR design compared to NRZ at the same baud rate.

---

### Q11. How do retimers use CDR to extend serial link reach?

**Answer:**

A retimer is an active device placed in the middle of a serial link channel that fully recovers the data using its receiver CDR, re-serializes the data with a clean transmitter clock, and re-transmits with full signal quality. The retimer effectively divides a long channel into two shorter segments, each of which independently meets the link budget. The CDR in the retimer is the critical component that enables this function.

The retimer CDR must meet stringent jitter transfer and jitter generation specifications because it sits in the middle of the link and any jitter it adds accumulates with the end-to-end jitter budget. PCIe Gen5/Gen6 specifies retimer jitter transfer peaking of less than 0.1 dB and retimer-added jitter of less than 1.5 ps RMS at 32 GT/s.

The retimer CDR architecture is typically a PI-based (phase interpolator) design rather than a VCO-based design. The PI uses the retimer's local PLL (locked to the same reference clock as the endpoint devices) as the clock source and adjusts only the phase to track the incoming data. This ensures that the retimer's output frequency is determined by the local reference (which is accurate and low-jitter), not by the recovered clock (which may have accumulated jitter from the upstream link).

The latency added by a retimer includes the CDR lock time (during initialization only), the serializer-to-deserializer pipeline latency (typically 4-8 UI for the parallel data path through the retimer), and the re-serialization latency. The total retimer latency is typically 16-32 UI (approximately 0.3-0.6 ns at 56 GBd), which is added to the end-to-end link latency. For AI training workloads, where latency impacts the efficiency of gradient synchronization, this added latency must be carefully considered.

PCIe Gen5/Gen6 supports up to 2 retimers per link, and the equalization training protocol is extended to handle the multi-segment channel: each segment is trained independently, with the retimer acting as both the receiver for the upstream segment and the transmitter for the downstream segment.

---

### Q12. What is the impact of CDR bandwidth on the recovered clock jitter and data decision quality?

**Answer:**

The CDR bandwidth directly determines the spectral content of the recovered clock jitter, which in turn affects the data decision quality at the slicer. The recovered clock jitter consists of two components: jitter tracked from the incoming data (which is transferred through the CDR with the jitter transfer function) and jitter generated by the CDR circuit itself (phase detector noise, VCO/PI noise, loop filter noise).

At frequencies below the CDR bandwidth, the CDR tracks the incoming data jitter, so the recovered clock jitter follows the data jitter. This is beneficial because the clock and data are correlated: any low-frequency jitter on the data is also present on the clock, and the two cancel at the sampling instant. The net timing uncertainty at the slicer for low-frequency jitter is approximately zero (perfect correlated jitter cancellation).

At frequencies above the CDR bandwidth, the CDR does not track the data jitter. The data jitter at these frequencies appears as timing uncertainty at the slicer because the clock does not follow. Additionally, the CDR's own noise sources contribute jitter at high frequencies (where the loop does not suppress them). The sum of the untracked data jitter and the CDR-generated jitter determines the timing margin reduction at the slicer.

A wider CDR bandwidth tracks more of the data jitter, reducing the untracked component and improving the data decision quality for jitter that is within the wider bandwidth. However, the wider bandwidth also allows the CDR to respond to amplitude noise and ISI residuals as if they were phase errors (through the phase detector's conversion of amplitude noise to phase noise), creating jitter on the recovered clock that would not exist with a narrower bandwidth.

The optimal CDR bandwidth minimizes the total timing uncertainty at the slicer, balancing the untracked jitter (which decreases with wider bandwidth) against the noise-induced jitter (which increases with wider bandwidth). For 112G PAM4 links with significant residual ISI and noise, the optimal bandwidth is typically 4-8 MHz. This is narrower than the optimal bandwidth for NRZ at the same baud rate (which would be 6-12 MHz) because the PAM4 phase detector is noisier and benefits from more filtering.
