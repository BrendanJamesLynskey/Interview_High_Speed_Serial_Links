# Signaling and Modulation

## Overview

The choice of signaling scheme fundamentally determines the tradeoffs between data rate, noise margin, equalization complexity, and power consumption in high-speed serial links. This section covers NRZ, PAM4, proposed PAM6, differential signaling, and common-mode rejection as applied to AI interconnects.

---

### Q1. Compare NRZ and PAM4 signaling schemes in terms of spectral efficiency, noise margin, and application in AI interconnects.

**Answer:**

NRZ (Non-Return-to-Zero) is a two-level signaling scheme where each symbol encodes one bit. The symbol takes one of two voltage levels (high or low), and the spectral efficiency is 1 bit per symbol. PAM4 (Pulse Amplitude Modulation with 4 levels) encodes two bits per symbol using four equally spaced voltage levels (typically labeled 0, 1, 2, 3 or mapped to Gray-coded bit pairs 00, 01, 11, 10). The spectral efficiency is 2 bits per symbol.

| Parameter | NRZ | PAM4 |
|-----------|-----|------|
| Bits per symbol | 1 | 2 |
| Levels | 2 | 4 |
| Eye height (relative) | 1.0 | 1/3 |
| SNR penalty | Reference | +9.54 dB |
| Nyquist freq (for 112 Gbps) | 56 GHz | 28 GHz |
| FEC requirement | Optional at lower rates | Mandatory |
| Equalization complexity | Lower | Higher (tighter margins) |

For AI interconnects operating at 112 Gbps per lane, PAM4 at 56 GBd has become the universal choice over NRZ at 112 GBd. The key reason is that channel loss increases super-linearly with frequency. At 56 GHz (NRZ Nyquist for 112 Gbps), a typical PCB channel might exhibit 40+ dB of insertion loss, which is beyond the equalization capability of practical SerDes designs. At 28 GHz (PAM4 Nyquist for 112 Gbps), the same channel exhibits approximately 25 dB of loss, which, combined with the 9.54 dB PAM4 penalty, yields an effective total of approximately 34.5 dB, still better than the 40+ dB NRZ scenario.

At lower rates (56 Gbps per lane and below), NRZ remains preferred when channel loss permits, because the simpler two-level signaling provides better noise margin and requires less complex equalization. PCIe Gen5, for example, uses NRZ at 32 GT/s (16 GHz Nyquist), taking advantage of the lower Nyquist frequency to avoid PAM4 complexity.

---

### Q2. What is PAM6 and why has it been proposed for 224G serial links?

**Answer:**

PAM6 is a six-level pulse amplitude modulation scheme that encodes log2(6) = approximately 2.585 bits per symbol. Since this is not an integer, PAM6 requires a fractional coding scheme where groups of symbols are mapped to groups of bits. For example, 5 PAM6 symbols can encode 5 * log2(6) = 12.92 bits, so a practical mapping might encode 12 or 13 bits per 5 symbols.

PAM6 has been proposed as an alternative to PAM4 for 224 Gbps serial links. The rationale is that PAM4 at 112 GBd (required for 224 Gbps) pushes the Nyquist frequency to 56 GHz, where channel loss is extremely high even with ultra-low-loss PCB materials. PAM6 would require a baud rate of only 224 / 2.585 = approximately 86.7 GBd for 224 Gbps, reducing the Nyquist frequency to approximately 43.3 GHz, which significantly reduces channel loss compared to the 56 GHz required by PAM4.

However, PAM6 introduces its own penalty. The distance between adjacent levels is reduced by a factor of 5 compared to NRZ (five inter-level gaps for the same peak-to-peak swing), resulting in a 20*log10(5) = 14 dB SNR penalty relative to NRZ. Compared to PAM4's 9.54 dB penalty, PAM6 adds an incremental 4.46 dB of SNR penalty. The question is whether the channel loss savings from the reduced Nyquist frequency more than compensate for this additional modulation penalty.

Analysis shows that PAM6 is advantageous for long-reach channels where the loss difference between 43 GHz and 56 GHz is large (greater than 4.46 dB), but disadvantageous for short-reach channels where the channel loss is already manageable. The IEEE 802.3dj task force has evaluated PAM6 alongside PAM4 for the 200G per lane Ethernet standard, with PAM4 at 112 GBd ultimately being favored for most use cases due to simpler implementation and existing PAM4 ecosystem advantages.

---

### Q3. Explain differential signaling and why it is universally used for high-speed serial links.

**Answer:**

Differential signaling transmits information as the voltage difference between two complementary conductors, typically called D+ and D- (or Tx+/Tx- and Rx+/Rx-). When D+ transitions high, D- transitions low by the same amount, and vice versa. The receiver measures only the differential voltage Vdiff = V(D+) - V(D-), rejecting any voltage common to both conductors.

The universal adoption of differential signaling for high-speed serial links is driven by several fundamental advantages. Common-mode noise rejection is the most critical: in an AI system with hundreds of amps of current switching on the power delivery network, ground bounce and supply noise can easily reach tens of millivolts. This noise couples approximately equally to both conductors of a differential pair (since they are physically close together and have similar coupling to the noise source), and is therefore canceled when the receiver takes the difference. The common-mode rejection ratio (CMRR) of a well-designed differential receiver is typically 30-40 dB.

The effective signal swing is doubled: for a given single-ended voltage swing of Vpp, the differential swing is 2*Vpp. This provides a 6 dB advantage in signal-to-noise ratio compared to a single-ended signal operating at the same per-pin voltage. The simultaneous switching noise (SSN) impact is also reduced: when one conductor transitions high and the other transitions low, the net current drawn from the supply is approximately constant (it redistributes between the two drivers), reducing the dI/dt that causes supply noise.

Electromagnetic interference (EMI) is reduced because the equal and opposite currents in the two conductors create fields that largely cancel at distances greater than the pair spacing. This reduces both emissions from the transmitter and susceptibility to external interference at the receiver.

The return current path is well-defined: the return current for each conductor flows primarily through the other conductor (at frequencies well above the pair's common-mode cutoff), reducing dependence on the ground plane return path and making the signal integrity more predictable.

---

### Q4. What is common-mode rejection and what factors degrade it in a real serial link?

**Answer:**

Common-mode rejection is the ability of a differential receiver to reject signals that appear equally on both conductors of a differential pair. Ideally, any common-mode voltage Vcm = (V(D+) + V(D-))/2 should have zero effect on the receiver's decision. In practice, common-mode rejection is finite and is quantified by the common-mode rejection ratio (CMRR), typically expressed in dB.

Several factors degrade common-mode rejection in real serial links. Intra-pair skew, the difference in electrical length between the D+ and D- conductors, converts common-mode noise into differential-mode interference. If the two conductors have a length mismatch of delta_L, a common-mode signal at frequency f is converted to a differential signal with amplitude proportional to sin(2*pi*f*delta_L/(2*v_p)), where v_p is the propagation velocity. At 28 GHz, even 1 ps of intra-pair skew (corresponding to approximately 0.15 mm of length mismatch) creates measurable mode conversion.

Asymmetric impedance discontinuities, such as non-symmetric via structures or asymmetric routing around obstacles, create differential-to-common-mode and common-to-differential-mode conversion. This mode conversion is captured by the SCD21 and SDC21 terms in the mixed-mode S-parameter matrix.

Imbalanced coupling to aggressor signals degrades common-mode rejection because crosstalk that couples unequally to the two conductors appears as a differential signal to the receiver. This is particularly problematic in dense BGA breakout regions where routing constraints may force asymmetric trace paths.

Receiver circuit imbalances, including input offset voltage, gain mismatch between the D+ and D- input paths, and sampling clock skew between the two paths, reduce the effective CMRR of the analog front end. Modern SerDes receivers employ calibration circuits to minimize these imbalances, achieving CMRR of 30-40 dB across the signal bandwidth.

---

### Q5. What is Gray coding and why is it used in PAM4 signaling?

**Answer:**

Gray coding is a binary encoding scheme in which adjacent symbols differ by exactly one bit. For PAM4, the four voltage levels are assigned two-bit labels such that each level differs from its nearest neighbors by only one bit. A typical Gray code mapping is: level 0 = "00", level 1 = "01", level 2 = "11", level 3 = "10". Note that adjacent levels (0 and 1, 1 and 2, 2 and 3) always differ in exactly one bit position, while non-adjacent levels (0 and 2, 1 and 3, 0 and 3) differ in two bit positions.

The importance of Gray coding becomes clear when considering the error mechanisms in PAM4. The most probable error is a single-level amplitude error, where the receiver incorrectly decides on an adjacent level (for example, receiving level 2 instead of level 1). With Gray coding, this single-level error produces exactly one bit error out of the two bits carried by the symbol. Without Gray coding (using natural binary: 00, 01, 10, 11), a single-level error between levels 1 ("01") and 2 ("10") would produce two bit errors, doubling the BER.

The quantitative impact is significant: Gray coding reduces the average BER for single-level errors by nearly a factor of 2 compared to natural binary coding. Since single-level errors dominate in a well-equalized PAM4 link (the probability of a two-level or three-level error is much smaller), Gray coding provides a nearly 3 dB improvement in effective SNR at the BER level of interest.

All modern PAM4 SerDes implementations use Gray coding. The encoding is applied in the digital domain before the DAC in the transmitter, and the corresponding decoding is applied after the ADC or slicer in the receiver. The coding is transparent to the equalization chain, which operates on symbol levels regardless of the bit mapping.

---

### Q6. How does the choice of signaling scheme affect the transmitter DAC resolution and linearity requirements?

**Answer:**

The transmitter for an NRZ link is fundamentally a two-state switch (when combined with FIR pre-emphasis, it becomes a multi-tap FIR filter with binary inputs). The output takes one of two voltage levels per UI, and linearity is not a concern because there are only two levels. The primary requirement is fast switching and accurate impedance matching.

For a PAM4 transmitter, the output must produce four equally spaced voltage levels, which typically requires a DAC (digital-to-analog converter) with at least 2 bits of resolution per symbol. In practice, the DAC must be significantly more precise than 2 bits because the linearity of the level spacing directly affects the eye diagram quality. The ratio of level spacing (RLM, Ratio Level Mismatch) must be maintained within tight bounds, typically within 5-10% as specified by standards like OIF CEI-112G.

Level non-linearity in the PAM4 transmitter creates two distinct problems. Static non-linearity (unequal spacing between the four levels) directly reduces the eye height of the worst-case inner eye. If the three eyes have unequal heights, the smallest eye determines the BER, and the margin is wasted in the larger eyes. Dynamic non-linearity (level-dependent settling behavior or data-dependent jitter) creates pattern-dependent effects that are difficult to equalize.

For a proposed PAM6 transmitter, the requirements become even more demanding: six levels must be produced with uniform spacing, requiring an effective DAC resolution of at least 3 bits with linearity better than the minimum eye height. The DAC power consumption and complexity scale with the number of levels and the required linearity, which is one of the practical challenges that has slowed PAM6 adoption.

Modern PAM4 transmitters use segmented DAC architectures with calibration to achieve the required linearity. The DAC segments are trimmed during manufacturing or through run-time calibration to correct for process variations that would otherwise cause level mismatch.

---

### Q7. Explain the relationship between modulation scheme and the required FEC coding gain.

**Answer:**

The required FEC coding gain is directly determined by the gap between the raw BER achievable by the analog front end (transmitter, channel, and receiver equalization) and the target post-FEC BER required by the system. Higher-order modulation schemes like PAM4 produce higher raw BER for the same channel and equalization, necessitating more powerful (higher coding gain) FEC.

For an NRZ link at 56 Gbps, a well-equalized receiver on a moderate channel can achieve a raw BER of 1e-9 or better. If the target post-FEC BER is 1e-15, the required FEC coding gain is approximately 6 orders of magnitude of BER improvement, which a relatively simple FEC code (such as a firecode or basic RS code) can provide. Many 56G NRZ links operate without FEC entirely.

For a PAM4 link at 112 Gbps on the same channel, the 9.54 dB SNR penalty means the raw BER after equalization is typically in the range of 1e-4 to 1e-6, depending on channel loss and equalization capability. Achieving a post-FEC BER of 1e-15 now requires a coding gain of 9-11 orders of magnitude, which demands a powerful code such as RS(544,514) (which can bridge from approximately 2.4e-4 to 1e-15) or concatenated codes.

The selection of FEC code involves tradeoffs in addition to coding gain. Latency increases with code strength: RS(544,514) adds approximately 100 ns of latency, which is acceptable for Ethernet but may be problematic for latency-sensitive AI interconnects. Power consumption for the FEC encoder/decoder logic scales with code complexity and data rate. Bandwidth overhead (the parity bits that reduce effective throughput) ranges from 2.7% for simple codes to 5.8% for RS(544,514).

For 224G links using PAM4 at 112 GBd, the raw BER is expected to be even higher (closer to 1e-3 to 1e-4) due to the extreme channel loss at 56 GHz Nyquist, which may require concatenated FEC schemes (such as an inner code plus an outer code) to achieve the necessary coding gain.

---

### Q8. What is the role of pre-coding in PAM4 and higher-order signaling?

**Answer:**

Pre-coding (specifically Tomlinson-Harashima pre-coding or its variants) is a technique that moves some or all of the DFE (decision feedback equalizer) functionality from the receiver to the transmitter. In a standard PAM4 receiver, the DFE subtracts the estimated ISI contribution of previously decided symbols from the current sample before the slicer makes its decision. Pre-coding performs this subtraction at the transmitter, modifying the transmitted symbol levels to pre-compensate for the channel ISI.

The primary motivation for pre-coding in PAM4 is to mitigate the error propagation problem of DFE. In a conventional DFE, if the slicer makes an incorrect decision, the DFE feedback for subsequent symbols is also incorrect, causing a burst of errors. For PAM4, where the noise margins are already tight, error propagation can significantly degrade the effective BER beyond what single-symbol error statistics would predict.

With Tomlinson-Harashima pre-coding, the transmitter has knowledge of the channel response (obtained during link training) and applies modulo arithmetic to the transmitted symbols to cancel the post-cursor ISI at the receiver. The receiver then only needs to perform a modulo operation to recover the original data, with no feedback loop and therefore no error propagation. The modulo operation wraps the signal levels to keep them within the transmitter's output range.

The challenges of pre-coding include the requirement for the transmitter to know the channel response (requiring a back-channel for communicating the channel estimate from receiver to transmitter), the power and area overhead of implementing the ISI cancellation at the transmitter, and the increased peak-to-average power ratio of the pre-coded signal. Some pre-coding schemes also require higher DAC resolution at the transmitter to accommodate the modified symbol levels.

Pre-coding is used in some UCIe die-to-die links and in certain proprietary AI interconnects where the short channel length limits the number of significant ISI taps and makes pre-coding practical.

---

### Q9. How does the transition to 224G per lane affect signaling scheme selection?

**Answer:**

The transition to 224 Gbps per lane represents the most challenging signaling design point in the history of copper-based serial links. The two primary candidates are PAM4 at 112 GBd (56 GHz Nyquist) and PAM4 at lower baud rates with additional coding or higher-order modulation. The industry is converging on PAM4 at 112 GBd as the baseline, but the challenges are formidable.

At 56 GHz Nyquist, channel loss on even ultra-low-loss PCB materials (Megtron 7 class) reaches 1.5-2.0 dB/inch for stripline traces. A typical 8-inch channel with two connector transitions might exhibit 20-25 dB of insertion loss at Nyquist, which, combined with the 9.54 dB PAM4 penalty, requires extremely aggressive equalization. The transmitter FIR filter needs 4-5 taps with high resolution, the CTLE must provide 15+ dB of peaking, and the DFE needs 12+ taps with precise adaptation.

The analog circuit design challenges at 112 GBd are severe. The sampling clock must have sub-50 fs RMS jitter to maintain adequate timing margin in the approximately 8.9 ps UI. The CTLE and DFE circuits must operate at bandwidths approaching 70-80 GHz, pushing the limits of even advanced FinFET processes (5nm and 3nm). The power consumption of a single 224G SerDes lane is estimated at 7-15 pJ/bit, compared to 3-5 pJ/bit for current 112G designs.

Alternative approaches being explored include: coherent signaling (used in optical communications but largely impractical for electrical links due to complexity), duobinary signaling (which trades bandwidth for ISI in a controlled manner), and PAM6 (discussed in Q2). DSP-intensive receivers that digitize the received signal with a high-speed ADC and perform equalization in the digital domain are gaining traction, as the digital processing can implement more complex equalization algorithms than analog-only approaches. Several leading SerDes IP vendors have demonstrated 224G PAM4 prototypes in 3nm and 4nm process nodes, with production deployment expected in the 2026-2028 timeframe.

---

### Q10. What is the significance of the signal-to-noise-and-distortion ratio (SNDR) for PAM4 transmitters?

**Answer:**

The signal-to-noise-and-distortion ratio (SNDR) captures the combined effect of all impairments in the PAM4 transmitter, including thermal noise, quantization noise, level non-linearity, timing jitter, and ISI from the package. SNDR is defined as the ratio of the signal power to the total noise-plus-distortion power, and it sets a fundamental limit on the achievable raw BER even with a perfect channel and perfect receiver.

For a PAM4 transmitter, the minimum SNDR required for a target BER can be computed from the statistical distance between adjacent levels relative to the total noise-plus-distortion. At a raw BER of 1e-6 (a typical target before FEC), the required SNDR is approximately 23-25 dB, depending on the noise distribution and coding scheme. At a raw BER of 1e-4 (relaxed target with stronger FEC), the required SNDR drops to approximately 19-21 dB.

The dominant distortion components in a practical PAM4 transmitter are level non-linearity (characterized by RLM), timing skew between the MSB and LSB DAC paths, duty cycle distortion (DCD), and data-dependent jitter (DDJ) from bandwidth limitations. Each of these components reduces the SNDR from its ideal value.

For 112G PAM4 transmitters in advanced CMOS processes, achieving a SNDR of 25+ dB is challenging due to device mismatch, power supply sensitivity, and parasitic effects at 56 GBd operation. Calibration techniques are essential: background calibration loops continuously adjust DAC segment weights, timing offsets, and bias voltages to maintain SNDR over process, voltage, and temperature (PVT) variations. The SNDR is typically measured using the transmitter and distortion eye analysis defined in standards such as OIF CEI-112G-LR/MR.

---

### Q11. How does encoding overhead differ between NRZ-based and PAM4-based serial link standards?

**Answer:**

Encoding overhead varies significantly between NRZ and PAM4 standards, reflecting the different design philosophies and error correction needs of each signaling approach. NRZ-based standards traditionally used 8b/10b encoding (25% overhead) or 64b/66b encoding (3.125% overhead) for DC balance and word alignment. PAM4-based standards have moved toward different encoding strategies that combine FEC with lightweight framing.

For NRZ standards: PCIe Gen1/Gen2 used 8b/10b encoding with 20% overhead. PCIe Gen3/Gen4/Gen5 uses 128b/130b encoding with 1.54% overhead plus a separate CRC. Ethernet 10G/25G uses 64b/66b encoding with 3.125% overhead. These encodings provide DC balance (ensuring the signal has no DC component, which is important for AC-coupled links), word boundary detection, and basic error detection.

For PAM4 standards: IEEE 802.3 100G Ethernet uses RS(544,514) FEC with 5.84% overhead, applied after 256b/257b transcoding. PCIe Gen6 uses FLIT-mode with integrated CRC and retry at the link layer, operating at 64 GT/s PAM4 with approximately 4.7% overhead for the FLIT header and CRC. The encoding overhead for PAM4 links is dominated by FEC parity rather than line coding, because PAM4 signaling typically does not require the same DC balance constraints as NRZ (many PAM4 links use pre-coding schemes that inherently manage spectral content).

The effective data rate for each standard is: raw bit rate * (1 - encoding overhead). For example, PCIe Gen6 at 64 GT/s with PAM4 delivers approximately 60.6 Gbps of effective throughput per lane after FLIT overhead. A 100G Ethernet lane at 106.25 Gbps raw rate with RS-FEC overhead delivers approximately 100 Gbps of effective payload data.

---

### Q12. What are the considerations for AC coupling versus DC coupling in high-speed serial links?

**Answer:**

AC coupling uses series capacitors in the signal path to block the DC component, while DC coupling provides a direct galvanic connection between transmitter and receiver. The choice between AC and DC coupling has significant implications for signaling, equalization, and system design.

AC coupling is used in most standard-based serial links (PCIe, Ethernet, USB) for several reasons. It allows the transmitter and receiver to operate at different DC bias voltages, which is essential when they are in different power domains or on different ICs with independent voltage regulators. It eliminates DC offset that could shift the signal away from the receiver's optimal common-mode input range. It enables hot-plug capability, since the AC coupling capacitors isolate the two sides. The capacitor value is chosen to provide a low-frequency cutoff well below the lowest signal frequency of interest (typically 100 nF to 1 uF, giving a cutoff below 100 kHz).

However, AC coupling creates challenges. The low-frequency cutoff causes baseline wander: long runs of identical bits cause the DC level to drift toward the bias point, reducing the effective signal swing. This is mitigated by encoding schemes that limit run length (such as 64b/66b, which guarantees a transition every 66 bits) or by receiver baseline wander correction circuits. The coupling capacitor itself introduces an impedance discontinuity (ESL, ESR, and capacitance variation with frequency), which degrades return loss, particularly at low frequencies.

DC coupling is used in some die-to-die interfaces (UCIe standard packaging module) and chip-to-chip links where both endpoints share a common substrate or package. DC coupling eliminates the capacitor discontinuity, removes the baseline wander problem, and allows transmission of DC-balanced and non-DC-balanced signals. However, it requires careful design of the common-mode voltage between transmitter and receiver, and it does not provide hot-plug isolation.

For AI systems, the choice depends on the link type: NVLink chip-to-chip links within a module may use DC coupling for optimal signal integrity, while PCIe and Ethernet links to external devices use AC coupling per the standard requirements.
