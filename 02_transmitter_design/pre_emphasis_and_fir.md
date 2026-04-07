# Pre-emphasis and FIR Filters

## Overview

Transmitter pre-emphasis using FIR (Finite Impulse Response) filters is a critical equalization technique that compensates for channel loss by boosting high-frequency signal content before transmission. This section covers FIR filter design, tap coefficient optimization, de-emphasis strategies, and the interaction between TX equalization and the rest of the link equalization chain.

---

### Q1. What is transmitter pre-emphasis and why is it needed?

**Answer:**

Transmitter pre-emphasis is a technique that modifies the transmitted signal to pre-compensate for the frequency-dependent loss of the channel. Since the channel attenuates high-frequency components more than low-frequency components, pre-emphasis boosts the high-frequency content of the signal before it enters the channel. After passing through the channel, the boosted high frequencies are attenuated back to a level closer to the low frequencies, resulting in a more uniform frequency response at the receiver and reduced ISI.

Pre-emphasis is implemented as a FIR (Finite Impulse Response) filter in the transmitter's data path. The filter operates on the data sequence at the symbol rate, combining the current symbol with weighted and delayed versions of adjacent symbols to produce the output. The filter coefficients (taps) are chosen to approximate the inverse of the channel's frequency response, within the constraints of the filter length and coefficient resolution.

Pre-emphasis is needed because the channel loss at modern data rates is too severe to be compensated by receiver-side equalization alone. A typical 112G channel may have 25 dB of loss at Nyquist, and the receiver's CTLE and DFE might compensate for 15-20 dB. The transmitter FIR provides the remaining 5-10 dB. Moreover, the TX FIR has a fundamental advantage: it boosts the signal before the channel, so the boosted high-frequency content competes only with the channel's thermal noise floor, not with the amplified noise that results from receiver-side equalization. This is why TX pre-emphasis is always preferred over adding more CTLE gain when the TX has sufficient headroom.

The cost of pre-emphasis is a reduction in the average transmitted signal amplitude, because the energy allocated to boosting transitions is taken from the flat portions of the signal. This tradeoff between peak (pre-emphasized) and average signal power is a key design consideration.

---

### Q2. Explain the structure of a multi-tap FIR filter used for transmitter pre-emphasis.

**Answer:**

A multi-tap FIR filter produces an output that is a weighted sum of the current input sample and several delayed versions of the input. For a serial link transmitter, the FIR filter operates at the symbol rate and has the form:

```
y[n] = c[-k]*x[n+k] + ... + c[-1]*x[n+1] + c[0]*x[n] + c[1]*x[n-1] + ... + c[m]*x[n-m]
```

Where x[n] is the input data sequence, c[i] are the tap coefficients, c[0] is the main cursor tap, c[-1] through c[-k] are pre-cursor taps (operating on future data), and c[1] through c[m] are post-cursor taps (operating on past data). A typical 5-tap FIR has 1 pre-cursor tap, 1 main cursor tap, and 3 post-cursor taps.

In a segmented SST driver implementation, the taps are realized by dividing the output driver into groups of segments. Each group is driven by the data delayed by the appropriate number of UI. The pre-cursor group is driven by the "future" data bit, which is available because the serializer provides a small lookahead buffer. The main cursor group is driven by the current data. The post-cursor groups are driven by data from 1, 2, and 3 UI ago, stored in shift registers.

The tap coefficients are normalized so that the sum of the absolute values of all taps equals 1.0 (or close to it), ensuring the output voltage does not exceed the driver's supply voltage. A typical coefficient set might be: c[-1] = -0.05, c[0] = 0.70, c[1] = -0.15, c[2] = -0.07, c[3] = -0.03. The main cursor tap c[0] is always positive and has the largest magnitude, while the pre-cursor and post-cursor taps are typically negative (providing de-emphasis of adjacent symbols).

The frequency response of the FIR filter can be computed as the discrete-time Fourier transform of the coefficient sequence. The filter provides a high-pass characteristic that partially inverts the channel's low-pass response, with the peaking frequency and gain controlled by the tap coefficients.

---

### Q3. What is the difference between pre-emphasis and de-emphasis?

**Answer:**

Pre-emphasis and de-emphasis are two ways to describe the same FIR filter operation, depending on the reference point. Pre-emphasis refers to boosting the signal amplitude at transitions (where the data changes value) relative to the steady-state level. De-emphasis refers to reducing the signal amplitude during sustained runs (where the data remains at the same value) relative to the transition level.

Mathematically, they produce the same output waveform. The distinction is in how the output swing is interpreted. In a pre-emphasis interpretation, the maximum output swing occurs at transitions and equals the full driver capability (Vpp_max). The average output power is reduced because non-transition bits have lower amplitude. In a de-emphasis interpretation, the non-transition bits are defined as the "normal" swing, and the transition bits are boosted above this level.

The practical difference lies in compliance testing and power analysis. Standards may specify the transmitter output swing in terms of the de-emphasized (flat) level rather than the peak (pre-emphasized) level, which affects how the signal amplitude is measured for compliance. For power analysis, the de-emphasis interpretation is more intuitive: the average power consumption is dominated by the de-emphasized level (since most bits are not at transitions in a random data pattern), and the transition boost adds only a modest power overhead.

For a specific example, consider a 3-tap FIR with c[-1] = 0, c[0] = 0.8, c[1] = -0.2. The first bit after a transition has amplitude proportional to c[0] = 0.8 of the full swing. The second and subsequent identical bits have amplitude proportional to c[0] + c[1] = 0.6 of the full swing (because the post-cursor tap subtracts from the main cursor when the previous bit is the same). The ratio of transition amplitude to steady-state amplitude is 0.8/0.6 = 1.33, or 2.5 dB of pre-emphasis/de-emphasis.

---

### Q4. How are FIR tap coefficients optimized for a specific channel?

**Answer:**

FIR tap coefficient optimization is performed during the link training phase, where the transmitter and receiver negotiate the best equalization settings through a series of training patterns and feedback exchanges. The goal is to find the tap coefficients that maximize the eye opening (or equivalently, minimize the BER) at the receiver.

The optimization can be framed as a minimum mean-squared error (MMSE) problem. Given the channel pulse response h[n], the FIR coefficients c[n] are chosen to minimize the sum of squared residual ISI at the receiver sampling point. The optimal MMSE solution can be computed analytically using the Wiener-Hopf equation if the channel response is known, but in practice, the channel is not known a priori and the optimization must be done adaptively.

During link training, the transmitter sends known training patterns (such as PRBS sequences) while the receiver monitors the eye quality (eye height, eye width, or BER estimate). The transmitter adjusts its FIR coefficients and the receiver reports the resulting eye quality through a back-channel (a low-speed sideband communication path). The optimization algorithm iterates through coefficient values, using gradient descent or exhaustive search within the allowed coefficient ranges, until the best setting is found.

Modern SerDes use a two-phase approach: first, a coarse sweep identifies the approximate optimal region; then, a fine-tuning phase refines the coefficients within this region. The coefficient resolution is typically 6 bits per tap (64 steps), with the main cursor tap ranging from approximately 0.5 to 1.0 and each auxiliary tap ranging from approximately -0.3 to +0.1.

Constraints on the optimization include: the sum of tap magnitudes must not exceed 1.0 (to prevent clipping), the main cursor tap must be positive and the dominant coefficient, and the total equalization must be shared with the CTLE and DFE at the receiver. The TX FIR optimization is therefore typically performed jointly with the CTLE and DFE settings in an iterative outer loop.

---

### Q5. Why is the pre-cursor tap particularly important and how does it differ from post-cursor taps?

**Answer:**

The pre-cursor tap compensates for pre-cursor ISI, which is the interference from the symbol that arrives after the current symbol. Pre-cursor ISI arises from the channel's group delay variation and from the interaction of the equalization chain with the channel. It appears as energy in the pulse response that precedes the main cursor.

The pre-cursor tap is particularly important because pre-cursor ISI cannot be canceled by the DFE at the receiver. DFE operates on past decisions: it knows the values of previously decided symbols and subtracts their ISI contribution from the current sample. But pre-cursor ISI depends on future symbols that have not yet been decided, so DFE is fundamentally unable to address it. Only the TX pre-cursor FIR tap and the receiver's CTLE (which is a linear equalizer that affects both pre-cursor and post-cursor ISI) can compensate for pre-cursor ISI.

In a typical 112G channel, the pre-cursor ISI is smaller than the post-cursor ISI (because the channel is predominantly causal), but it is not negligible. A pre-cursor ISI of 5-10% of the main cursor amplitude, if left uncompensated, directly reduces the eye opening by 5-10%. For a PAM4 eye with only 50-80 mV of height, this represents a significant margin degradation.

Post-cursor taps, by contrast, compensate for post-cursor ISI, which can also be partially addressed by the DFE. The division of post-cursor equalization between TX FIR and RX DFE is an optimization variable: TX FIR cancellation occurs before the channel and therefore does not suffer from noise enhancement, but it also reduces the transmitted energy that the receiver could use. DFE cancellation is noise-free (the feedback is based on hard decisions, not noisy signals) but is limited to post-cursor taps and suffers from error propagation.

The optimal partition typically allocates the first 1-2 post-cursor taps to DFE (because these have the largest magnitude and DFE handles them efficiently) and uses TX FIR for the pre-cursor and for longer-tail post-cursor ISI that exceeds the DFE's tap count.

---

### Q6. How does FIR pre-emphasis interact with PAM4 signaling?

**Answer:**

FIR pre-emphasis in a PAM4 transmitter is more complex than in NRZ because each symbol has four possible levels instead of two. The FIR filter operates on the multi-level symbols, and the output is a weighted sum of current and adjacent symbol levels. This means the FIR output can take on a large number of possible levels (for a 3-tap FIR operating on PAM4 data, the output can theoretically take 4^3 = 64 distinct values, though many may be degenerate).

The practical implication is that the DAC resolution and dynamic range of the driver must accommodate the expanded set of output levels. If the FIR coefficients are chosen such that the output levels cluster densely, the driver must resolve small voltage differences between adjacent levels, requiring higher DAC resolution. If the coefficients are too aggressive, the output levels may exceed the driver's voltage range, causing clipping distortion.

The constraint on the tap coefficients for PAM4 is more complex than for NRZ. For NRZ, the constraint is simply that the tap magnitudes sum to 1.0 or less. For PAM4, the constraint must ensure that all possible output levels (for all combinations of current and adjacent symbol values) fall within the DAC's output range and have sufficient spacing for the slicer to resolve. This typically means the auxiliary tap magnitudes must be smaller relative to the main tap for PAM4 than for NRZ.

Another interaction is that PAM4 level non-linearity in the driver interacts with the FIR filter to create non-linear distortion products. If the driver has different effective impedance at different output levels (due to transistor non-linearity), the FIR de-emphasis at each level is slightly different, creating data-dependent distortion. This is addressed by calibrating the FIR segments for each output level separately or by applying non-linear pre-distortion in the digital domain before the FIR filter.

---

### Q7. What is the effect of FIR filter length on equalization performance and complexity?

**Answer:**

The FIR filter length (number of taps) determines the frequency resolution of the equalization and the range of ISI that can be compensated. A longer FIR filter can more closely approximate the inverse channel response and cancel ISI from more distant symbol positions, but it also increases implementation complexity, power consumption, and the time required for coefficient optimization.

A 1-tap post-cursor FIR (2 taps total: main + 1 post) provides basic de-emphasis and can compensate for the dominant first post-cursor ISI component. This is sufficient for short, low-loss channels where the pulse response decays quickly. The frequency response of a 2-tap FIR is a single sinusoidal shape with limited control over the peaking frequency and shape.

A 3-tap FIR (1 pre + 1 main + 1 post) adds pre-cursor compensation and provides more flexibility in shaping the frequency response. This is the minimum useful configuration for 112G channels and is adequate for many medium-reach applications.

A 5-tap FIR (1 pre + 1 main + 3 post) provides significantly better frequency response shaping and can cancel ISI from 3 post-cursor positions. This is typical for long-reach 112G designs and provides enough flexibility to compensate for complex channel responses with multiple reflections.

Beyond 5 taps, the incremental benefit of each additional tap diminishes because the post-cursor ISI magnitude typically decays exponentially with tap position. However, channels with strong reflections (from connectors, vias, or impedance mismatches) can have significant ISI at tap positions beyond 5, in which case the DFE is generally more efficient than extending the TX FIR.

The implementation complexity scales linearly with the number of taps: each tap requires a separate group of driver segments, a shift register stage for the data delay, and routing for the delayed data to the driver array. The power overhead per additional tap is approximately 5-10% of the main driver power.

---

### Q8. How is the FIR filter's frequency response related to its tap coefficients?

**Answer:**

The frequency response of a FIR filter is the discrete-time Fourier transform (DTFT) of its tap coefficient sequence. For a FIR filter with coefficients c[n] at tap positions n (where n is in units of UI), the frequency response is:

```
H(f) = sum over n of c[n] * exp(-j*2*pi*f*n*T)
```

Where T is the unit interval (1/baud_rate). The frequency response is periodic with period 1/T (the baud rate) and is completely determined by the tap coefficients.

For the most common 3-tap configuration with c[-1] (pre-cursor), c[0] (main), and c[1] (post-cursor), the magnitude response is:

```
|H(f)|^2 = c[0]^2 + c[-1]^2 + c[1]^2 + 2*c[0]*c[-1]*cos(2*pi*f*T)
           + 2*c[0]*c[1]*cos(2*pi*f*T) + 2*c[-1]*c[1]*cos(4*pi*f*T)
```

At DC (f = 0), the response is |c[-1] + c[0] + c[1]|. At Nyquist (f = 1/(2T)), the response depends on the signs: if c[-1] and c[1] are negative (de-emphasis configuration), the Nyquist response is |c[0] - c[-1] - c[1]|, which is larger than the DC response, creating the desired high-frequency boost.

The boost at Nyquist relative to DC, in dB, is:

```
Boost = 20*log10(|c[0] - c[-1] - c[1]| / |c[0] + c[-1] + c[1]|)
```

For example, with c[-1] = -0.05, c[0] = 0.80, c[1] = -0.15:

```
DC gain = |0.80 - 0.05 - 0.15| = 0.60
Nyquist gain = |0.80 + 0.05 + 0.15| = 1.00
Boost = 20*log10(1.00/0.60) = 4.4 dB
```

This 4.4 dB of pre-emphasis partially compensates for the channel's frequency rolloff. The shape of the frequency response between DC and Nyquist can be further controlled by adding more taps, which allows for a more precise match to the inverse channel response.

---

### Q9. What are the limitations of transmitter-side equalization compared to receiver-side equalization?

**Answer:**

Transmitter-side FIR equalization has several limitations that must be understood for optimal link design. The most fundamental limitation is that FIR is a linear equalizer, meaning it can only compensate for linear channel impairments. Non-linear effects such as dielectric dispersion non-linearity or connector non-linearity cannot be addressed by linear FIR pre-emphasis.

The second limitation is the peak power constraint. The FIR filter redistributes the transmitter's finite output energy: boosting high frequencies necessarily reduces the average signal amplitude. For aggressive pre-emphasis (high auxiliary tap values), the average transmitted power can be reduced significantly. For example, a FIR with c[0] = 0.6 and the remaining taps summing to -0.4 delivers only 60% of the driver's available swing for non-transition bits, reducing the receiver's signal amplitude by 4.4 dB for long runs of identical bits. This "average power penalty" must be weighed against the equalization benefit.

The third limitation is the finite coefficient resolution. In a segmented SST driver with 64 segments, each tap coefficient can only take values that are multiples of 1/64, limiting the precision of the frequency response shaping. Fine-tuning within a 1/64 step is not possible, which can leave residual ISI that must be addressed by the receiver.

The fourth limitation is the lack of adaptivity during data transmission. Unlike the receiver's adaptive DFE and CTLE, the TX FIR coefficients are typically set during link training and remain fixed during normal operation. If the channel characteristics drift (due to temperature changes, connector aging, or vibration), the TX FIR cannot adapt in real time. Some advanced SerDes implement dynamic TX FIR adaptation using a back-channel from the receiver, but the adaptation bandwidth is limited by the back-channel latency.

Finally, TX equalization occurs before the noise is added by the channel and receiver, so it does not amplify noise (unlike CTLE). This is its primary advantage over CTLE. The optimal equalization strategy uses TX FIR to compensate as much loss as possible within the peak power constraint, then uses CTLE for additional compensation (accepting the noise penalty), and finally uses DFE for remaining post-cursor ISI.

---

### Q10. How does link training determine the optimal TX FIR settings?

**Answer:**

Link training is a multi-phase initialization process where the transmitter and receiver negotiate equalization settings. The specific protocol varies by standard, but the general flow for PCIe Gen5/Gen6 is representative.

In Phase 1 (Receiver Detection), the transmitter sends a low-frequency pattern to allow the receiver to detect the presence of a link partner and establish basic communication. In Phase 2 (Preset Evaluation), the transmitter steps through a set of predefined equalization presets (combinations of FIR tap settings). For PCIe, there are 11 presets defined by the specification, ranging from no equalization to maximum de-emphasis. The receiver evaluates the eye quality for each preset and reports the results through the back-channel.

In Phase 3 (Coefficient Adjustment), the transmitter fine-tunes the FIR coefficients around the best preset. The receiver sends requests for the transmitter to increase or decrease specific tap coefficients (using codes like "increment post-cursor" or "decrement pre-cursor"), and the transmitter adjusts accordingly. The receiver continues to monitor eye quality and sends further adjustment requests until the optimal setting is reached or a maximum number of iterations is exhausted.

The receiver-side evaluation metrics used during training typically include: the CDR's lock status and jitter estimate, the eye monitor's measured eye height and width, the DFE tap values (which indicate residual ISI), and sometimes a direct BER estimate using a built-in error counter and known training patterns.

For multi-lane links, each lane is trained independently because the channels may have different characteristics (due to different trace lengths, routing, or connector pin assignments). The training of all lanes can proceed in parallel if the back-channel supports per-lane communication.

The total training time is typically 10-100 milliseconds, which is acceptable for system boot but may be problematic for hot-plug scenarios where the link must be re-established quickly. Some standards define abbreviated training sequences for reconnection that start from the previously known good settings.

---

### Q11. What is the impact of FIR pre-emphasis on jitter performance?

**Answer:**

FIR pre-emphasis introduces additional deterministic jitter (DJ) into the transmitted signal while simultaneously reducing the ISI-induced jitter at the receiver. The net effect on link BER depends on the balance between these two contributions.

The additional DJ arises because the pre-emphasized waveform has data-dependent transition timing. When a bit transition occurs, the pre-emphasis changes the signal amplitude for the transitioning bit, which shifts the zero-crossing time of the transition. For a rising transition preceded by multiple low bits (which are de-emphasized), the signal starts from a lower voltage and takes longer to cross the threshold than a rising transition preceded by a high bit (which starts from the pre-emphasized level). This pattern-dependent variation in transition timing is a form of data-dependent jitter (DDJ).

The DDJ contribution of a 3-tap FIR can be estimated as:

```
DDJ = T * (|c[1]| / (slew_rate * T)) approximately proportional to |c[1]|/c[0]
```

For a typical FIR with c[0] = 0.7 and c[1] = -0.15, the DDJ might be 0.02-0.04 UI, which is within the typical transmitter jitter budget but not negligible.

At the receiver, the pre-emphasis reduces ISI, which reduces ISI-induced jitter (a major component of deterministic jitter in the received signal). The reduction in ISI jitter is typically much larger than the DDJ added by the pre-emphasis, so the net effect is a significant improvement in total jitter at the receiver.

However, excessive pre-emphasis (with large auxiliary tap values) can create situations where the added DDJ outweighs the ISI reduction benefit, particularly on short channels where the ISI is already small. This is why the FIR optimization must consider jitter as well as amplitude margin.

The PLL phase noise of the transmitter clock is another jitter source that is not affected by FIR pre-emphasis. The pre-emphasis modifies the signal amplitude but not the timing of the clock that drives the serializer, so PLL jitter passes through the FIR filter unchanged.

---

### Q12. How do standards specify the constraints on transmitter FIR coefficients?

**Answer:**

Standards define the allowable range and resolution of transmitter FIR tap coefficients to ensure interoperability between transmitters and receivers from different vendors. The specifications balance flexibility (allowing enough equalization range to cover diverse channels) with predictability (ensuring the receiver can handle the transmitter's output waveform).

PCIe Gen5 defines 11 transmitter presets (P0 through P10) that specify combinations of pre-cursor (pre-shoot) and post-cursor (de-emphasis) levels. Each preset defines the ratio of the de-emphasized voltage to the peak voltage, expressed in dB. The range covers from 0 dB de-emphasis (no pre-emphasis, preset P0) to approximately 6 dB of post-cursor de-emphasis and 3.5 dB of pre-cursor pre-shoot. The main cursor is implicitly defined as 1 minus the sum of auxiliary tap magnitudes.

IEEE 802.3ck for 100G Ethernet defines the transmitter FIR constraints in terms of normalized tap coefficients. The main cursor tap c[0] must be between 0.6 and 1.0. The pre-cursor tap c[-1] must be between -0.1 and 0.05. The post-cursor taps c[1] and c[2] must each be between -0.25 and 0.05. The sum |c[-1]| + c[0] + |c[1]| + |c[2]| must equal 1.0 (full-scale normalization).

OIF CEI-112G-LR (the industry interoperability specification for 112G long-reach links) specifies a 4-tap FIR with similar constraints but adds a requirement for the minimum main cursor amplitude to ensure sufficient signal strength: the main cursor must be at least 0.5 of the full-scale output.

The coefficient resolution is typically 1/64 (6-bit resolution) for each tap, meaning each coefficient can take 64 equally spaced values within its allowed range. During link training, the coefficients are adjusted in steps of 1/64. The training algorithm must converge to the optimal setting within a specified number of iterations (typically 24-48 steps per coefficient for PCIe).

These standardized constraints ensure that a receiver designed for a given standard can handle any transmitter pre-emphasis setting within the specified range, simplifying interoperability testing and reducing the risk of incompatible equalization configurations in multi-vendor systems.
