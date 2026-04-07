# UCIe and Die-to-Die Interfaces

## Overview

Universal Chiplet Interconnect Express (UCIe) is an open industry standard for die-to-die communication in advanced packages, enabling heterogeneous chiplet integration critical for next-generation AI accelerators. This section covers UCIe architecture, electrical specifications, and the role of die-to-die links in AI systems.

---

### Q1. What is UCIe and why is it important for AI accelerator design?

**Answer:**

UCIe (Universal Chiplet Interconnect Express) is an open standard published by the UCIe Consortium that defines a die-to-die interconnect for chiplets within a multi-die package. UCIe specifies the physical layer (electrical signaling, equalization), the die-to-die adapter layer (link training, retry), and the protocol layer (supporting PCIe, CXL, and streaming protocols).

UCIe is important for AI accelerators because the scaling of monolithic dies is reaching practical limits. A single large die (reticle-limited at approximately 800-850 mm2) cannot accommodate the growing compute, memory bandwidth, and I/O requirements of next-generation AI accelerators. Chiplet architectures decompose the design into smaller dies (chiplets) that are interconnected within a package, allowing each chiplet to use the optimal process node (such as 3nm for compute, 5nm for I/O, and specialized processes for analog/RF).

For AI specifically, UCIe enables larger effective die sizes (by combining multiple compute chiplets), heterogeneous integration (combining CPU, GPU, HBM, and I/O chiplets in a single package), improved yield (smaller chiplets have higher manufacturing yield than large monolithic dies), and mix-and-match flexibility (different chiplets from different vendors can be combined using the standard interface).

The bandwidth density of UCIe is a key advantage: the advanced package module specification achieves approximately 1.3 Tbps/mm of edge bandwidth using 2D bump pitches as fine as 25 um, which is 10-20 times higher than the bandwidth density of organic package-based I/O (such as PCIe or Ethernet). This enables the terabytes-per-second of bandwidth needed between compute chiplets and HBM stacks in AI accelerators.

---

### Q2. Describe the two UCIe module types and their electrical specifications.

**Answer:**

UCIe defines two module types that address different packaging technologies and channel characteristics.

The Advanced Package Module is designed for silicon interposer or silicon bridge-based packaging (such as TSMC CoWoS, Intel EMIB, or similar 2.5D/3D technologies). It uses micro-bumps with pitches as fine as 25 um for the shortest die-to-die distances (less than 2 mm typical). The electrical specifications include: single-ended signaling (not differential, to maximize bandwidth density), NRZ at up to 32 Gbps per lane, supply voltage of 0.5-0.9V, bump pitch of 25-55 um, and no equalization required for the shortest channels (sub-1 mm). The bandwidth per module is up to 1.3 Tbps/mm of edge bandwidth.

The Standard Package Module is designed for organic substrate-based packaging where the die-to-die distance is longer (2-25 mm) and the channel loss is higher. It uses standard C4 bumps with pitches of 100-130 um. The electrical specifications include: differential signaling (similar to external SerDes), NRZ or PAM4 signaling at up to 32 Gbps per lane, supply voltage of 0.8-1.1V, and equalization (CTLE and potentially DFE) required for the lossier channels. The bandwidth density is lower than the advanced module but still significantly higher than external I/O.

| Parameter | Advanced Package | Standard Package |
|-----------|-----------------|-----------------|
| Signaling | Single-ended | Differential |
| Max data rate | 32 Gbps/lane | 32 Gbps/lane |
| Modulation | NRZ | NRZ or PAM4 |
| Bump pitch | 25-55 um | 100-130 um |
| Channel length | < 2 mm | 2-25 mm |
| Equalization | Minimal/none | CTLE, optional DFE |
| BW density | ~1.3 Tbps/mm | ~0.2 Tbps/mm |
| FEC | Optional | Optional |

The choice between module types depends on the packaging technology used for the AI accelerator. High-end GPUs (such as NVIDIA's next-generation designs) are expected to use the advanced package module on silicon interposers, while lower-cost or larger-form-factor designs may use the standard package module.

---

### Q3. How does UCIe handle link training and equalization for die-to-die links?

**Answer:**

UCIe link training is simpler than PCIe link training because the die-to-die channel is shorter, more predictable, and does not change after manufacturing (unlike a pluggable PCIe connector). The training protocol includes lane detection, lane repair, equalization, and pattern verification.

Lane detection: the transmitter sends a known pattern on all lanes, and the receiver detects which lanes are functional. UCIe supports lane repair, where a spare lane can replace a defective lane, improving manufacturing yield.

Lane repair: if one or more lanes are detected as defective (due to bump defects, alignment errors, or die defects), the redundant lanes can be activated to replace them. This is a unique feature of UCIe not found in PCIe, and it is essential for manufacturing yield because the fine-pitch bumps in advanced packages have a non-zero defect rate.

Equalization: for the advanced package module with sub-2 mm channels, the channel loss is typically less than 3 dB at Nyquist, and no equalization may be needed. For the standard package module with longer channels, the equalization training follows a similar approach to PCIe: the transmitter sweeps through preset settings, the receiver evaluates eye quality, and the optimal setting is selected. The equalization options are typically limited to a few CTLE settings, since the short channels do not require aggressive DFE.

Pattern verification: after equalization, both endpoints exchange known patterns and verify error-free reception. The verification uses CRC checking over a specified pattern length. If verification fails, the link may retry with different equalization settings or report a link failure.

The total training time for UCIe is typically 1-10 milliseconds, which is faster than PCIe because the simpler channel and equalization require fewer training iterations. Once trained, the settings are fixed for the lifetime of the package (there is no re-training during operation, unlike PCIe which can re-train in response to detected errors).

---

### Q4. What protocols does UCIe support and how are they mapped to the physical layer?

**Answer:**

UCIe supports three protocol types through its die-to-die adapter layer: PCIe (for standard I/O device semantics), CXL (for cache-coherent memory and device access), and a streaming protocol (for raw data transfer with minimal protocol overhead).

The protocol mapping works through the UCIe die-to-die (D2D) adapter, which sits between the protocol layer (PCIe/CXL/streaming) and the physical layer. The adapter performs FLIT packaging (assembling protocol data units into fixed-size FLITs for transmission), CRC generation and checking, link-level retry for error recovery, and flow control.

The FLIT format is protocol-dependent: PCIe FLITs carry TLP data with PCIe-specific headers, CXL FLITs carry CXL.cache or CXL.mem transactions, and streaming FLITs carry raw data with minimal overhead. The physical layer transports FLITs without knowledge of their protocol content, providing a clean separation between physical and protocol layers.

For AI accelerators, the streaming protocol is particularly relevant because it allows high-bandwidth, low-latency data transfer between compute chiplets without the overhead of PCIe or CXL protocol processing. For example, the partial sum data exchanged between tensor processing elements during matrix multiplication can use the streaming protocol, achieving nearly 100% utilization of the physical layer bandwidth.

CXL over UCIe enables cache-coherent access to memory attached to another chiplet, which is important for AI accelerators that combine compute chiplets with memory controller chiplets. The CXL.mem protocol allows any compute chiplet to access any HBM stack in the package with load/store semantics, creating a unified memory space across chiplets.

---

### Q5. How does UCIe achieve its high bandwidth density and what are the physical layer design challenges?

**Answer:**

UCIe achieves high bandwidth density through several techniques. Fine-pitch bumps (25-55 um for advanced package) allow more I/O connections per unit of die edge length. Single-ended signaling (for advanced package) doubles the number of data lanes compared to differential signaling for the same number of bumps, because each bump carries one signal instead of two bumps carrying one differential signal.

The bandwidth density calculation for the advanced package at 25 um pitch: each bump carries 32 Gbps, bumps are spaced 25 um apart, so the bandwidth per unit edge length is 32 Gbps / 25 um = 1.28 Tbps/mm. Accounting for ground bumps, clock bumps, and spare lanes, the effective bandwidth density is approximately 1.0-1.3 Tbps/mm.

The physical layer design challenges at fine pitch include signal integrity in single-ended mode (without the common-mode rejection advantage of differential signaling, single-ended links are more susceptible to supply noise and crosstalk), crosstalk between adjacent lanes (at 25 um pitch, the coupling between neighboring bumps is significant, requiring careful shielding with ground bumps and ground planes), clock distribution (a forwarded clock is used rather than CDR, because the short channel makes clock forwarding practical and avoids the complexity and latency of per-lane CDR), and power delivery (the fine-pitch bumps must carry both signals and power, and the power bumps must be distributed among the signal bumps to provide local decoupling).

The transmitter and receiver circuits for UCIe advanced package are significantly simpler than external SerDes: no pre-emphasis is needed for sub-2 mm channels, no DFE is needed (the ISI is negligible), and the driver can operate at lower voltage (0.5-0.9V) because the channel loss is minimal. This simplicity translates to much lower power per bit: UCIe advanced package achieves approximately 0.5-1.0 pJ/bit, compared to 3-7 pJ/bit for external 112G SerDes.

---

### Q6. What is the role of UCIe retimers in die-to-die communication?

**Answer:**

UCIe supports a retimer-less mode for the advanced package module, where the die-to-die channel is short enough that no clock recovery is needed. The transmitter forwards a clock along with the data, and the receiver samples the data using the forwarded clock. This eliminates the latency, power, and area of CDR circuits.

For the standard package module with longer channels, UCIe can optionally use retimers to extend the reach. A UCIe retimer functions similarly to a PCIe retimer: it fully recovers the data, re-clocks it, and re-transmits with full signal quality. The retimer can be an active silicon bridge (such as Intel EMIB) or a separate die in the package.

The retimer-less operation of UCIe advanced package is a key advantage for AI accelerators because it minimizes latency. The end-to-end latency of a UCIe advanced package link is dominated by the flight time across the silicon interposer (approximately 5-10 ps/mm, so 10-20 ps for a 2 mm channel) plus the transmitter and receiver pipeline latency (approximately 1-3 ns for serialization, transmission, and deserialization). The total latency is approximately 2-5 ns, compared to approximately 10-20 ns for a retimed link.

For die-to-die links in AI accelerators, the low latency is critical because the chiplets exchange data frequently during tensor operations, and every nanosecond of latency reduces the effective compute utilization. A matrix multiplication that requires 100 data exchanges per operation would accumulate 200-500 ns of communication latency with UCIe (retimer-less), compared to 1-2 us with retimed links, a 4-5x difference.

---

### Q7. How does UCIe compare to proprietary die-to-die interfaces used in current AI accelerators?

**Answer:**

Before UCIe, AI accelerator vendors used proprietary die-to-die interfaces optimized for their specific packaging technologies. NVIDIA's proprietary interface (used in the A100 and H100 for connecting the GPU die to HBM memory controllers) and AMD's Infinity Fabric On-Package (IFOP, used in EPYC processors for connecting CCD chiplets to the IOD) are examples of such interfaces.

These proprietary interfaces typically achieve higher bandwidth density and lower power than the UCIe specification because they are optimized for a specific process node, package technology, and channel. For example, AMD's IFOP achieves approximately 36 Gbps per lane with very low power (approximately 0.5 pJ/bit) using a custom PHY tuned for the specific TSMC CoWoS-S platform. UCIe, as a multi-vendor standard, must accommodate a range of process nodes and package technologies, which requires more conservative specifications.

The advantage of UCIe is interoperability. A UCIe-compliant chiplet from vendor A can be connected to a UCIe-compliant chiplet from vendor B within the same package, enabling a heterogeneous chiplet ecosystem. For AI systems, this could enable combining a compute chiplet from one vendor with a memory controller chiplet from another vendor and a networking chiplet from a third vendor, optimizing each component independently.

The bandwidth comparison is approximate:

| Interface | BW Density | Power/bit | Latency | Interoperable? |
|-----------|-----------|-----------|---------|----------------|
| UCIe Advanced | ~1.3 Tbps/mm | 0.5-1.0 pJ/bit | 2-5 ns | Yes |
| UCIe Standard | ~0.2 Tbps/mm | 2-4 pJ/bit | 5-10 ns | Yes |
| AMD IFOP | ~1.5 Tbps/mm | ~0.5 pJ/bit | ~2 ns | No |
| NVIDIA proprietary | ~1.0 Tbps/mm | ~0.5 pJ/bit | ~2 ns | No |

The industry trend is toward adopting UCIe while maintaining proprietary interfaces for the highest-performance internal links. AI accelerator vendors may use UCIe for standard chiplet interfaces (such as I/O and memory controller chiplets) while continuing to use proprietary interfaces for the most bandwidth-critical links (such as compute-to-compute chiplet connections).

---

### Q8. What are the testing and debug challenges for UCIe die-to-die links?

**Answer:**

Testing UCIe die-to-die links presents unique challenges compared to testing external SerDes links because the signals are buried within the package and cannot be directly probed. The die-to-die bumps are not accessible after package assembly, so traditional oscilloscope-based eye diagram measurement and TDR characterization are not possible.

Built-in self-test (BIST) is essential for UCIe testing. The UCIe specification includes a mandatory loopback test mode where the transmitter sends known patterns and the receiver verifies them, measuring BER. The BIST can operate in near-end loopback (the transmitted data is routed back to the receiver on the same die, testing the analog circuits without the channel), far-end loopback (the data traverses the full channel to the remote die, which loops it back, testing the complete link), and pattern-based BER testing (transmitting PRBS patterns and counting errors over a specified period).

Eye diagram measurement uses the on-die eye monitor (phase interpolator plus voltage offset comparator) to scan the received eye, providing the same information as an oscilloscope but without external access. The eye monitor data is read out through a JTAG or sideband interface and can be used for debug and characterization.

Channel characterization at the die-to-die level relies on simulation rather than measurement. The channel S-parameters are computed from electromagnetic simulation of the package model (using tools like Ansys HFSS or Cadence Clarity), and the link performance is predicted using COM or equivalent methodology. The correlation between simulation and silicon measurement (via the BIST BER test) validates the model accuracy.

Production testing uses a combination of BIST (running link-level tests on every unit to verify functionality) and parametric testing (measuring DC characteristics like driver impedance and receiver threshold through dedicated test modes). The test time per unit is a critical consideration for manufacturing cost, and the test suite must be optimized to catch defects efficiently.

For debug of failing units, the sideband registers provide diagnostic information including DFE tap values (indicating channel quality), adaptation convergence status, error count per lane, and lane repair map. These diagnostics, combined with the eye monitor data, usually provide sufficient information to diagnose the root cause without physical probing.

---

### Q9. How does UCIe address reliability for die-to-die links in AI accelerators?

**Answer:**

UCIe addresses reliability through several mechanisms at the physical, link, and system levels. Lane redundancy and repair is the primary physical-level reliability feature: each UCIe module includes spare lanes that can be activated during manufacturing test (if a lane has a defective bump) or during operation (if a lane degrades during the product lifetime). The typical redundancy ratio is 5-10% (for example, 64 data lanes plus 4 spare lanes).

Link-level reliability uses CRC-protected FLITs with retry. Each FLIT includes a CRC field, and if the receiver detects a CRC error, it requests retransmission. The retry mechanism handles transient errors (caused by noise, cosmic rays, or marginal bump contacts) without system-level intervention. The expected retry rate for a healthy link is less than 1 per billion FLITs.

Error monitoring provides continuous visibility into link health. The UCIe adapter maintains error counters (corrected errors, uncorrected errors, retry events) that the system software can read to detect degradation trends. A link that shows an increasing error rate may be approaching a wear-out failure and can be flagged for proactive maintenance.

For AI accelerators, reliability is especially important because a single defective die-to-die link can disable an entire GPU or reduce its effective memory bandwidth. The combination of lane repair, retry, and error monitoring ensures that the die-to-die links maintain functionality over the product lifetime (typically 5-7 years for server-class AI accelerators) despite the aging mechanisms that affect fine-pitch solder bumps (electromigration, thermal fatigue, intermetallic growth).

---

### Q10. What is the bandwidth and latency of UCIe compared to HBM for AI accelerator memory access?

**Answer:**

UCIe and HBM serve complementary roles in AI accelerator memory systems. HBM (High Bandwidth Memory) provides the primary memory bandwidth for GPU compute, while UCIe connects chiplets within the package.

HBM3 provides approximately 819 GB/s per stack (with 16 channels of approximately 51 GB/s each). An NVIDIA H100 uses 5 HBM3 stacks for a total of approximately 3.35 TB/s of memory bandwidth. The HBM interface uses a wide parallel bus (1024 data pins per stack) at relatively low per-pin rates (approximately 9.6 Gbps for HBM3, 12.8 Gbps for HBM3E).

UCIe, when used to connect a compute chiplet to a memory controller chiplet (which then connects to HBM stacks), must provide at least as much bandwidth as the HBM interface it serves. For a single HBM3E stack at approximately 1.2 TB/s, the UCIe link must provide at least 1.2 TB/s of bandwidth, which at 32 Gbps per lane requires approximately 37,500 lanes (for single-ended signaling) or approximately 4.7 mm of die edge (at 1.0 Tbps/mm bandwidth density).

The latency comparison is informative: HBM3 access latency is approximately 15-20 ns (from the GPU memory controller to the HBM die and back). If UCIe is inserted in the path (between the GPU compute chiplet and a separate memory controller chiplet), it adds approximately 2-5 ns each way, increasing the total memory access latency to approximately 19-30 ns, a 20-50% increase.

For AI workloads dominated by large matrix multiplications (which are bandwidth-bound, not latency-bound), this latency increase has minimal impact on performance. For AI inference workloads with small batch sizes (which can be latency-sensitive), the additional UCIe latency may reduce throughput by 5-10%. This tradeoff is acceptable because the chiplet architecture provides compensating benefits in die yield, cost, and scalability.

---

### Q11. How does power delivery work for UCIe links in advanced packages?

**Answer:**

Power delivery for UCIe links in advanced packages is interleaved with the signal bumps. In the advanced package module, the bump array includes signal bumps (carrying data and clock), ground bumps (providing the return current path and shielding), and power bumps (delivering VDD to the transmitter and receiver circuits).

The power bump placement must satisfy two competing requirements: sufficient power delivery capacity to supply the I/O circuits, and sufficient ground bump density to maintain signal integrity (by providing low-impedance return paths and shielding between signal lanes).

A typical UCIe bump map allocates approximately 30-40% of the total bumps for signals, 40-50% for ground, and 10-20% for power. For a module with 200 total bumps in a 10x20 array at 45 um pitch, this might be 64 signal bumps (data + clock), 100 ground bumps, and 36 power bumps.

The power consumption of UCIe advanced package I/O is approximately 0.5-1.0 pJ/bit, which at 32 Gbps per lane and 64 lanes gives a total power of approximately 1.0-2.0 W per module. The 36 power bumps must deliver this power at the module's supply voltage (0.5-0.9V), corresponding to approximately 1.1-4.0 A of total current. Each power bump carries approximately 30-110 mA, which is within the electromigration limits for micro-bumps at typical temperatures (less than 100 degrees Celsius).

The on-die decoupling capacitance near the UCIe I/O circuits is critical for maintaining supply voltage stability during switching. The simultaneous switching of dozens of lanes creates transient current demands (dI/dt) that can cause supply voltage droops if the decoupling is insufficient. Typical designs place 50-200 pF of MOS-capacitor decoupling within 100 um of each I/O cell, supplemented by the interposer's embedded decoupling (if available).

---

### Q12. What is the UCIe roadmap and how will it evolve for future AI accelerators?

**Answer:**

UCIe 1.0 (published in 2022) defines the baseline specification with data rates up to 32 Gbps per lane for both advanced and standard package modules. The UCIe Consortium has indicated plans for future revisions that will increase data rates, reduce power, and add features relevant to AI accelerators.

UCIe 1.1 and 2.0 (expected 2024-2026) are anticipated to include higher per-lane data rates (64 Gbps or above, using PAM4 for the standard module and potentially higher NRZ rates for the advanced module), improved latency specifications (tighter bounds on the adapter layer processing time), enhanced streaming protocol (with features like multicast, quality of service, and credit-based flow control optimized for AI workloads), and finer bump pitches (below 25 um for advanced packaging using hybrid bonding).

Hybrid bonding (copper-to-copper direct bonding at pitches below 10 um) is a transformative technology for future UCIe. It eliminates the solder bump entirely, replacing it with a direct metal bond between the copper pads on two dies. Hybrid bonding achieves bump pitches of 1-10 um, enabling bandwidth densities of 10+ Tbps/mm (10 times higher than current UCIe advanced packaging). TSMC's SoIC and Intel's Foveros Direct technologies are examples of hybrid bonding platforms that future UCIe revisions may target.

For AI accelerators, the UCIe roadmap enables several architectural trends. Disaggregated GPU architectures can place compute, memory controller, and I/O functions on separate chiplets, each at the optimal process node. Wafer-scale integration (extending beyond reticle limits by interconnecting multiple reticle-sized chiplets) becomes practical with high-density UCIe links. Heterogeneous AI processors can combine GPU, CPU, NPU, and analog processing chiplets in a single package, with UCIe providing the standard interface between them.

The competitive landscape includes proprietary alternatives (such as NVIDIA's and AMD's internal interfaces) that may continue to offer higher performance than UCIe for the highest-end applications, with UCIe serving as the standard interface for the broader chiplet ecosystem.
