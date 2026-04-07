# Retimer and Redriver Design

## Overview

Retimers and redrivers extend the reach of high-speed serial links beyond what a single channel can support. This section covers the architectures, tradeoffs, and applications of these active signal conditioning devices in AI system interconnects.

---

### Q1. What is the difference between a retimer and a redriver?

**Answer:**

A retimer is an active device that fully recovers the data from the incoming serial stream using its own CDR (clock and data recovery), makes a digital decision on each symbol, and re-transmits the data with a clean clock and full signal swing. The retimer effectively creates two independent link segments: the upstream channel (from the original transmitter to the retimer's receiver) and the downstream channel (from the retimer's transmitter to the final receiver). Each segment must independently meet the link budget requirements, but neither needs to meet the full end-to-end budget.

A redriver (also called a signal conditioner or linear repeater) is a simpler device that amplifies and equalizes the analog signal without making data decisions. The redriver applies CTLE-like equalization and gain to the received signal, compensating for channel loss, and retransmits the amplified signal. The redriver does not recover the clock or make data decisions, so noise, jitter, and ISI from the upstream channel pass through (partially equalized but not eliminated) to the downstream channel.

The key tradeoffs are signal quality (retimers provide a completely clean signal with no accumulated jitter or noise, while redrivers pass through residual impairments), latency (retimers add approximately 4-8 ns per device for the CDR and serialization pipeline, while redrivers add only approximately 0.1-0.5 ns for the amplifier propagation delay), power (retimers consume approximately 0.5-2W per lane for the full transceiver, while redrivers consume approximately 0.1-0.5W per lane for the amplifier), and protocol awareness (retimers must implement the full physical layer protocol including link training and equalization negotiation, while redrivers are protocol-agnostic).

---

### Q2. When should a retimer be used versus a redriver in an AI system?

**Answer:**

The decision between retimer and redriver depends on the channel loss, jitter budget, protocol requirements, and latency sensitivity of the specific link.

Retimers are preferred when the total channel insertion loss exceeds the SerDes equalization capability (typically above 25-30 dB at Nyquist for 112G). The retimer divides the channel into two shorter segments, each of which can be independently equalized. Retimers are also required when the jitter has accumulated to the point where the CDR cannot track it (typically when the total jitter exceeds 0.3 UI peak-to-peak). In PCIe Gen5/Gen6 systems, retimers are commonly used for riser cards, cable connections, and any channel that exceeds approximately 20 inches of total trace length.

Redrivers are appropriate for shorter channels where the loss is moderate (10-20 dB at Nyquist) and the signal quality degradation is primarily due to amplitude loss rather than jitter accumulation. Redrivers are used in PCIe Gen3/Gen4 systems for short channel extensions (such as from the CPU to a PCIe switch on the motherboard). However, at Gen5 and above data rates, the channel impairments are often too severe for a redriver to adequately address, making retimers the preferred choice.

For AI systems specifically, the latency tradeoff is important. NVLink connections between GPUs are typically short enough (on the same motherboard) that neither retimers nor redrivers are needed. PCIe connections that traverse risers, cables, or switch hops may benefit from retimers. CXL connections for memory expansion may prefer redrivers for their lower latency (200 ns matters for memory access latency), but the signal integrity requirements of Gen5/Gen6 speeds often mandate retimers.

---

### Q3. What are the major retimer vendors and their product offerings for AI systems?

**Answer:**

Several vendors offer retimers for high-speed serial links used in AI systems. Intel (formerly Pericom/Diodes) offers PCIe Gen5 retimers in the Laguna series, supporting up to x16 lane configurations. These are widely used in server platforms for connecting CPUs to PCIe switches and downstream devices. Marvell (formerly Inphi/Aquantia) offers 112G PAM4 retimers for Ethernet and PCIe applications, including the Alaska series for 100G/400G Ethernet and PCIe Gen5/Gen6. Broadcom offers PCIe Gen5 retimers and is developing Gen6 retimers for AI server platforms. Texas Instruments offers PCIe Gen4/Gen5 retimers and redrivers for cost-sensitive applications.

The retimer product specifications for AI systems include per-lane data rate (32 GT/s for Gen5, 64 GT/s for Gen6), lane count (x4 to x16 per device), insertion loss compensation (typically 15-25 dB per segment at Nyquist), jitter generation (less than 1.5 ps RMS for the retimer's own contribution), power consumption (200-500 mW per lane, depending on lane count and features), and package size (8x8 mm to 15x15 mm BGA).

The total retimer power for a x16 Gen5 link is approximately 3-8W, which adds to the system power budget and requires thermal management. The retimer is placed on the motherboard between the host device and the downstream device, requiring PCB area and routing resources. For AI servers with 4-8 GPUs, each with a x16 PCIe link and potentially 1-2 retimers per link, the total retimer power can reach 25-60W, a non-trivial contribution to the system power.

---

### Q4. How does a retimer handle link training in a PCIe system?

**Answer:**

A PCIe retimer participates in the link training process as a transparent device that creates two independent link segments. The retimer implements the full PCIe LTSSM (Link Training and Status State Machine) on both its upstream and downstream ports, and it coordinates the training of both segments.

During equalization training, the retimer trains its upstream receiver (connected to the host transmitter) and its downstream transmitter (connected to the device receiver) independently. The retimer's upstream receiver performs Phase 1-3 equalization with the host transmitter, optimizing the CTLE, DFE, and requesting TX FIR adjustments through the training ordered sets. Simultaneously (or sequentially), the retimer's downstream port trains with the downstream device.

The retimer's own transmitter must also be equalized for the downstream channel. The downstream device's receiver evaluates the retimer's transmitter output and requests FIR coefficient adjustments, just as it would for a direct connection to the host.

PCIe Gen5/Gen6 supports up to 2 retimers per link, creating up to 3 independent link segments. The specification defines the maximum total retimer latency (to ensure the protocol-level timeouts are not exceeded) and the retimer's jitter transfer and jitter generation specifications (to ensure the cumulative jitter budget across multiple retimers remains within the receiver's tolerance).

---

### Q5. What are the power and latency tradeoffs of using retimers in AI interconnects?

**Answer:**

Power and latency are the two primary costs of using retimers, and they must be weighed against the signal integrity benefit.

The power tradeoff is straightforward: each retimer lane consumes 200-500 mW (depending on the data rate and equalization complexity). For a x16 link, this adds 3-8W to the system power. In an AI server with 8 GPUs and 8 retimed PCIe links, the total retimer power is approximately 25-60W, which is 1-3% of the total server power (2-4 kW) but represents a meaningful contributor to the thermal budget, particularly because retimers are distributed across the motherboard (not concentrated at the heatsink-cooled GPU).

The latency tradeoff is more nuanced. A retimer adds approximately 4-8 ns per device for the CDR and serializer/deserializer pipeline. For PCIe data transfers (which are typically large-block DMA operations with latency requirements on the order of microseconds), the retimer latency is negligible (less than 1% of the total latency). For CXL memory access (with latency targets of 200-300 ns), a retimer adds approximately 2-4% latency overhead, which is noticeable but generally acceptable.

For NVLink GPU-to-GPU communication, retimers are not used because the on-board channel is short enough and the protocol does not support retimers. For future optical NVLink interconnects that span rack-scale distances, optical-electrical-optical retimers (or equivalently, optical amplifiers) may be needed, with similar power and latency considerations.

---

### Q6. What is the architecture of a modern PCIe Gen5 retimer?

**Answer:**

A modern PCIe Gen5 retimer integrates two complete SerDes transceivers (upstream and downstream) with a digital data path between them. The architecture includes an upstream receiver (with CTLE, DFE, and CDR operating at 32 GBd NRZ), a digital retransmit FIFO (storing a few symbols of data to absorb the clock domain crossing between the upstream recovered clock and the downstream transmit clock), a downstream transmitter (with FIR pre-emphasis and driver), the symmetric reverse path (downstream receiver and upstream transmitter for the other direction of the full-duplex link), a PLL (generating the transmit clock from the reference clock), LTSSM logic (implementing the PCIe link training protocol for both ports), and a sideband interface (for configuration, diagnostics, and firmware updates).

The retimer die is typically fabricated in an advanced CMOS process (7nm or 5nm) to achieve the required SerDes bandwidth and power efficiency. The die area for a x16 retimer is approximately 30-60 mm2, containing 32 full SerDes transceivers (16 lanes x 2 directions).

The retimer package is a BGA with 300-500 balls, including signal pins for the upstream and downstream differential pairs (64 pairs for x16 bidirectional), power and ground pins (providing the current for the SerDes and digital logic), reference clock inputs (100 MHz for PCIe), and sideband pins (for I2C or SMBus configuration).

---

### Q7. How do redrivers compare to retimers for PCIe Gen4 and Gen5 applications?

**Answer:**

At PCIe Gen4 (16 GT/s NRZ, 8 GHz Nyquist), redrivers are often adequate because the channel loss at 8 GHz is typically manageable (10-15 dB for medium-length channels). A Gen4 redriver provides 10-15 dB of CTLE-like equalization, compensating for most of the channel loss without the latency and power overhead of a retimer. The redriver passes through any jitter and residual ISI from the upstream channel, but at 16 GT/s the timing margins are generous enough (62.5 ps UI) to accommodate this.

At PCIe Gen5 (32 GT/s NRZ, 16 GHz Nyquist), redrivers are marginal. The channel loss at 16 GHz is typically 15-25 dB, and the timing margins shrink to 31.25 ps UI. A redriver can compensate for 12-18 dB of loss, but the residual ISI and jitter from the upstream channel consume a significant portion of the downstream receiver's budget. In practice, Gen5 redrivers are used only for short channel extensions (adding 3-5 inches of additional reach), while retimers are needed for longer channels.

At PCIe Gen6 (64 GT/s PAM4, 16 GHz Nyquist), redrivers face the additional challenge of PAM4 signaling. The redriver must maintain PAM4 level linearity while providing equalization, which requires a linear amplifier with wider dynamic range than an NRZ redriver. The PAM4 noise margins (one-third of the NRZ margins) leave very little room for the noise and ISI passed through by a redriver, making retimers the strongly preferred choice for Gen6.

---

### Q8. What is the impact of retimers on SerDes jitter budgets?

**Answer:**

Retimers affect the jitter budget by breaking the end-to-end link into independent segments and by adding their own jitter contribution. The total jitter at the final receiver is determined by the retimer's jitter generation (not by the accumulated jitter from the upstream channel, because the retimer fully regenerates the signal).

The retimer's jitter generation specification defines the jitter added by the retimer to the signal as it passes through. For PCIe Gen5 retimers, the specification limits the retimer-added jitter to approximately 1.5 ps RMS (total jitter) and 0.5 ps RMS (random jitter). This jitter is added to the downstream channel's jitter budget: the downstream receiver must tolerate the sum of the retimer's jitter generation and the downstream channel's jitter.

The retimer's jitter transfer specification defines how much jitter from the upstream channel passes through to the downstream output. For a retimer, the jitter transfer should be essentially zero at frequencies above the CDR bandwidth (because the CDR regenerates the clock). The specification limits the jitter peaking (jitter amplification near the CDR bandwidth) to less than 0.1 dB, preventing jitter accumulation in systems with multiple retimers.

For a PCIe Gen5 link with 2 retimers, the total jitter at the final receiver is: retimer 1 jitter generation + downstream channel 1 jitter + retimer 2 jitter generation + downstream channel 2 jitter. Each segment is budgeted independently, with the retimer jitter generation contributing approximately 1.5 ps to each segment. The total is typically 3-5 ps RMS, which is within the receiver's tolerance of approximately 6-8 ps (0.2 UI at 32 GBd).

---

### Q9. How are retimers tested and qualified for AI server platforms?

**Answer:**

Retimer testing covers three levels: silicon-level characterization, board-level validation, and system-level qualification. Silicon-level testing verifies the retimer's SerDes performance against the specification using a compliance test board with calibrated channels. The tests include transmitter output compliance (eye height, eye width, jitter, level linearity), receiver tolerance (jitter tolerance mask, stressed receiver sensitivity), retimer latency measurement, power consumption at different operating modes, and lane margining capability.

Board-level validation tests the retimer in the actual AI server motherboard environment. The tests include link training success rate with various endpoint devices, BER measurement on the retimed link (both upstream and downstream segments), lane margining of both segments (to verify adequate signal quality margin), thermal testing (verifying retimer performance at the maximum operating temperature, which can reach 85-100 degrees Celsius in a server), and power integrity (measuring the retimer's supply noise rejection and its own contribution to supply noise).

System-level qualification runs the AI workloads on the server with the retimers in the data path, verifying that the retimer does not cause data corruption, link instability, or performance degradation. The tests include extended stress testing (running GPU training workloads for 72-168 hours continuously), power cycling and thermal cycling (to verify link re-establishment after reset), and error rate monitoring (counting any corrected or uncorrected errors over extended operation).

---

### Q10. What emerging retimer technologies are being developed for 224G serial links?

**Answer:**

For 224G (112 GBd PAM4) serial links, retimer technology must evolve to handle the extreme channel loss at 56 GHz Nyquist and the tight jitter budget at 8.9 ps UI. Several technology directions are being pursued.

DSP-based retimers use high-speed ADCs and digital signal processing to perform equalization in the digital domain, providing more powerful and flexible equalization than analog-only approaches. The ADC digitizes the received signal at 112 GBd with 5-6 bits of resolution, and the DSP applies FFE (feed-forward equalizer), DFE, and non-linear equalization algorithms. This enables the retimer to handle channels with 35+ dB of loss at 56 GHz Nyquist, which would be beyond the capability of analog-only equalization.

Low-power retimers in advanced process nodes (3nm, 2nm) reduce the per-lane power consumption from the current 300-500 mW to approximately 200-350 mW for 224G, which is critical for maintaining acceptable system power as the lane count and data rate increase.

Optical retimers (or more precisely, optical-electrical-optical regenerators) convert the electrical signal to optical at one end, transmit over fiber, and convert back to electrical at the other end. Each conversion includes a retiming function, and the optical transmission eliminates the bandwidth-distance tradeoff of copper channels. Co-packaged optics with integrated retiming is being developed for AI datacenter interconnects.

Multi-protocol retimers that support PCIe Gen6 (64 GT/s PAM4), CXL 3.0, and potentially NVLink on the same silicon are being developed to reduce the number of different retimer devices needed in AI servers. The multi-protocol capability is implemented through programmable SerDes configurations and protocol-layer firmware.
