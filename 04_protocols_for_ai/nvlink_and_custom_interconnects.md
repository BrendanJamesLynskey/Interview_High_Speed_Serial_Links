# NVLink and Custom Interconnects

## Overview

Proprietary high-speed interconnects are the lifeblood of AI accelerator clusters, providing the massive bandwidth needed for distributed training of large models. This section covers NVIDIA NVLink, NVSwitch, and other vendor-specific interconnects including Google ICI, AMD Infinity Fabric, and Intel UPI.

---

### Q1. What is NVLink and how does it differ from PCIe for GPU-to-GPU communication?

**Answer:**

NVLink is NVIDIA's proprietary high-speed point-to-point interconnect designed specifically for GPU-to-GPU communication. Unlike PCIe, which is a general-purpose host-to-device interface, NVLink is optimized for the bandwidth-intensive, latency-sensitive data exchange patterns of distributed AI training.

NVLink 4.0 (used in the H100 GPU) operates at 112 Gbps PAM4 per lane (56 GBd), with each link consisting of 2 differential pairs (one in each direction for full-duplex operation). Each link provides 50 GB/s of bidirectional bandwidth (25 GB/s each direction). The H100 has 18 NVLink 4.0 links, providing a total bidirectional bandwidth of 900 GB/s.

The key differences from PCIe include bandwidth (NVLink 4.0 provides 900 GB/s total per GPU, compared to approximately 128 GB/s for PCIe Gen5 x16, a 7x advantage), topology (NVLink supports direct GPU-to-GPU connections in various topologies including rings, meshes, and full connectivity via NVSwitch, while PCIe is hierarchical with a root complex), memory model (NVLink supports GPU-to-GPU memory access with cache coherency through NVLink's NVLM protocol, while PCIe requires DMA transfers through the CPU), and latency (NVLink achieves approximately 5-10 microseconds for a GPU-to-GPU memory read, compared to approximately 15-30 microseconds for a PCIe round-trip through the CPU).

From a SerDes perspective, NVLink uses the same fundamental 112G PAM4 signaling as PCIe Gen6 and 100G Ethernet, with similar equalization requirements. However, NVLink can optimize certain parameters because both endpoints are NVIDIA devices: the link training protocol can be simplified, the FEC code can be chosen without multi-vendor interoperability constraints, and the channel design can be co-optimized with the GPU package and motherboard.

---

### Q2. How does NVSwitch enable full-bandwidth all-to-all GPU communication?

**Answer:**

NVSwitch is NVIDIA's dedicated NVLink switch ASIC that enables any-to-any GPU communication within a system at full NVLink bandwidth. Without NVSwitch, GPUs can only communicate directly with their immediate NVLink-connected neighbors, limiting the topology to rings or small meshes. NVSwitch provides a non-blocking crossbar that connects all GPUs in the system with full bisection bandwidth.

The third-generation NVSwitch (used in the DGX H100 system) has 64 NVLink 4.0 ports operating at 112 Gbps per lane, providing a total of 3.6 Tbps of aggregate switching bandwidth. The DGX H100 uses 4 NVSwitch ASICs to interconnect 8 H100 GPUs, creating a topology where every GPU can communicate with every other GPU at the full 900 GB/s NVLink bandwidth simultaneously.

The NVSwitch SerDes design is particularly demanding because the switch ASIC must support 64 high-speed links on a single die, each requiring a full 112G PAM4 transceiver. The total SerDes power for a 64-port NVSwitch is approximately 20-40W (at 200-350 mW per lane, with 128 lanes total), which is a significant fraction of the NVSwitch's total power budget (approximately 100-150W).

The latency through an NVSwitch is approximately 5-10 ns (switch fabric latency) plus the SerDes latency (approximately 10-20 ns per link traversal), for a total of approximately 20-40 ns per NVSwitch hop. For a GPU-to-GPU transfer through one NVSwitch, the total latency is approximately 40-80 ns (GPU TX SerDes + channel + NVSwitch RX/fabric/TX SerDes + channel + GPU RX SerDes).

For multi-node AI training clusters, NVIDIA extends NVLink beyond a single server using NVLink Network (the NVLink-based InfiniBand replacement) with NVSwitch at the rack level, providing 400 Gbps per GPU of inter-node bandwidth.

---

### Q3. What topologies are used for NVLink-connected GPU systems and how do they affect training performance?

**Answer:**

NVLink GPU systems use several topologies depending on the number of GPUs and the available NVLink links per GPU.

Ring topology: in a ring, each GPU is connected to two neighbors (left and right). Data transfers use ring-based collective operations (ring all-reduce) where each GPU sends data to its neighbor in one direction while receiving from the other. The ring bandwidth is limited by the bandwidth of a single NVLink link multiplied by the number of links in the ring direction. A ring of 8 GPUs with 4 NVLink links per direction achieves approximately 200 GB/s of effective all-reduce bandwidth.

Fully-connected (all-to-all) topology: with NVSwitch, every GPU can communicate directly with every other GPU at full bandwidth. This eliminates the multi-hop latency of ring topology and enables collective operations to complete in one step rather than N-1 steps (for N GPUs). The DGX H100's 8-GPU fully-connected topology achieves approximately 900 GB/s of effective all-reduce bandwidth, approximately 4.5 times better than a ring of the same GPUs.

Hierarchical topology: for multi-node systems (such as NVIDIA SuperPODs), the topology is hierarchical: within each node (8 GPUs), the GPUs are fully connected via NVSwitch at 900 GB/s per GPU; between nodes, the GPUs communicate via NVLink Network at 400 Gbps per GPU or via InfiniBand at 400 Gbps. This creates a bandwidth hierarchy: intra-node bandwidth (900 GB/s) is much higher than inter-node bandwidth (400 Gbps = 50 GB/s), which must be considered in the parallelism strategy.

The topology directly affects AI training performance through the all-reduce operation, which is the dominant communication pattern in data-parallel training. The time for all-reduce on a ring of N GPUs is approximately 2*(N-1)/N * data_size / bandwidth, while for a fully-connected topology it is approximately data_size / bandwidth (independent of N). For large GPU counts, the fully-connected topology provides a significant speedup, which is why NVSwitch is essential for large-scale AI training.

---

### Q4. Describe Google's Inter-Chip Interconnect (ICI) used in TPU systems.

**Answer:**

Google's Inter-Chip Interconnect (ICI) is the proprietary high-speed link used to connect TPU (Tensor Processing Unit) chips in Google's AI training clusters. ICI enables direct chip-to-chip communication within TPU pods, providing the bandwidth needed for distributed training of Google's large language models and other AI workloads.

TPU v4 uses ICI links operating at approximately 4.8 Tbps total per chip, connecting each TPU to its neighbors in a 3D torus topology. Each TPU v4 chip connects to 6 neighbors (2 in each of the 3 dimensions), and the torus topology enables efficient all-reduce operations across the pod without requiring a switch ASIC (unlike NVLink's use of NVSwitch).

The ICI physical layer uses high-speed SerDes operating at rates comparable to industry-standard 112G per lane, with Google-proprietary equalization and protocol. The SerDes design leverages Google's custom ASIC expertise (developed through multiple generations of TPU chips) to optimize the power and performance for the specific channel characteristics of TPU boards and backplanes.

The 3D torus topology has interesting bandwidth properties. The bisection bandwidth (the minimum bandwidth across a cut that divides the network in half) scales as O(N^(2/3)) for a 3D torus of N nodes, compared to O(N) for a fully-connected topology. However, the torus has the advantage of scalability: it can be built with a fixed number of links per chip (6 for 3D torus) regardless of the total number of chips, while a fully-connected topology requires the link count to scale with N.

Google has published that TPU v4 pods scale to 4096 chips, interconnected by ICI in a 16x16x16 3D torus. The aggregate ICI bandwidth within the pod is enormous (approximately 4.8 Tbps * 4096 / 2 = approximately 9.8 Pbps of bisection bandwidth), enabling the efficient training of models with hundreds of billions of parameters.

---

### Q5. How does AMD Infinity Fabric provide inter-chip connectivity in AMD AI accelerators?

**Answer:**

AMD Infinity Fabric is a scalable interconnect architecture used across AMD's product lines, from EPYC server CPUs to Instinct MI-series AI accelerators. Infinity Fabric provides both on-package (die-to-die) and off-package (chip-to-chip) connectivity.

For AMD Instinct MI300X (AMD's flagship AI accelerator), Infinity Fabric connects the XCD (accelerator compute die) chiplets to the IOD (I/O die) within the package. The MI300X uses a 2.5D package with 8 XCD chiplets and 4 IOD chiplets, interconnected by Infinity Fabric on-package links operating at high speeds over the silicon interposer.

For multi-GPU communication, AMD uses Infinity Fabric over the external links (xGMI, extended Global Memory Interconnect), which operates at 112 Gbps per lane using PAM4 signaling (similar to NVLink). The MI300X provides approximately 896 GB/s of bidirectional bandwidth between GPUs, comparable to NVLink 4.0's 900 GB/s.

The Infinity Fabric architecture differs from NVLink in its coherency model. Infinity Fabric provides full cache coherency across all connected devices using an extension of AMD's coherent protocol, which enables a unified memory model where any GPU can access any other GPU's memory with hardware-maintained coherency. NVLink also supports coherency through NVIDIA's protocol, but the specific implementation details differ.

The SerDes design for Infinity Fabric uses standard 112G PAM4 transceivers, similar to those used for PCIe and Ethernet. AMD leverages its SerDes IP across multiple products (EPYC, Instinct, Ryzen) to amortize the design cost. The equalization and link training protocols are AMD-proprietary, allowing optimization for the specific channel characteristics of AMD platforms.

---

### Q6. What is Intel Ultra Path Interconnect (UPI) and how is it used in Intel-based AI systems?

**Answer:**

Intel Ultra Path Interconnect (UPI) is Intel's proprietary coherent interconnect used for processor-to-processor communication in multi-socket Xeon server systems. UPI replaces the earlier QPI (QuickPath Interconnect) and provides the high-bandwidth, low-latency connection needed for cache-coherent multi-processor computing.

UPI in Sapphire Rapids Xeon processors operates at 16 GT/s with up to 4 links per socket, each providing approximately 64 GB/s of bidirectional bandwidth. The total inter-socket bandwidth is approximately 256 GB/s for a dual-socket system. UPI uses NRZ signaling with pre-emphasis and is designed for the relatively short channels between CPU sockets on a server motherboard (typically 10-20 cm).

For Intel-based AI systems, UPI is relevant because it provides the backbone for multi-socket CPU systems that host Intel Gaudi AI accelerators or discrete GPUs via PCIe. The UPI bandwidth determines how efficiently the CPUs can share memory and coordinate the AI accelerators. In a dual-socket system with 4 UPI links per direction, the inter-socket bandwidth of 256 GB/s is sufficient for most CPU-based AI workloads (inference, data preprocessing) but may become a bottleneck for workloads that require frequent data sharing between the two CPU sockets.

Intel is developing CXL-based alternatives to UPI for future platforms, which would use the standard CXL protocol over PCIe physical layers (potentially Gen6 PAM4) instead of the proprietary UPI protocol. This transition would leverage the broader industry investment in CXL while providing similar bandwidth and latency characteristics.

---

### Q7. How do custom AI interconnects compare in terms of bandwidth, latency, and power efficiency?

**Answer:**

The major custom AI interconnects can be compared across several dimensions critical to distributed AI training performance.

| Interconnect | BW/GPU (bidir.) | Latency (hop) | Power/bit | Topology |
|-------------|-----------------|---------------|-----------|----------|
| NVLink 4.0 | 900 GB/s | ~40-80 ns | ~5 pJ/bit | Full mesh (via NVSwitch) |
| AMD xGMI | 896 GB/s | ~50-100 ns | ~5 pJ/bit | Ring/mesh |
| Google ICI v4 | ~600 GB/s | ~100-200 ns | ~3-5 pJ/bit | 3D torus |
| Intel UPI | 256 GB/s | ~100-150 ns | ~5-7 pJ/bit | Point-to-point |

The bandwidth per GPU is the most important metric for data-parallel AI training because it determines the all-reduce throughput. NVLink 4.0 and AMD xGMI lead with approximately 900 GB/s per GPU, enabled by their large number of high-speed lanes and the switch ASICs (NVSwitch for NVLink). Google ICI provides lower per-chip bandwidth but excels in scalability (supporting thousands of chips without a switch ASIC).

Latency per hop affects the performance of latency-sensitive collective operations (such as all-reduce with small data sizes, which occurs during gradient synchronization of small model parameters). NVLink's lower latency per hop (enabled by the short on-board channels and the low-latency NVSwitch fabric) gives it an advantage for small-message collectives.

Power efficiency (pJ/bit) determines the total interconnect power, which can be a significant fraction of the system power for bandwidth-intensive topologies. Google ICI's 3D torus topology is more power-efficient per GPU than NVLink's full-mesh topology because each chip has only 6 links (versus 18+ for NVLink), but the reduced connectivity increases the average hop count and therefore the aggregate link bandwidth needed for all-to-all communication.

The choice of interconnect architecture is deeply tied to the AI training software stack: each vendor's interconnect is co-designed with the collective communication library (NCCL for NVIDIA, RCCL for AMD, XLA for Google) to optimize the communication patterns of distributed training.

---

### Q8. What are the SerDes design challenges specific to multi-GPU interconnects in AI systems?

**Answer:**

Multi-GPU interconnects in AI systems present several SerDes design challenges beyond standard high-speed serial link design. The lane count per device is extremely high: an NVIDIA H100 has 36 differential pairs for NVLink alone (18 links with 2 pairs each), plus additional lanes for PCIe (16 differential pairs), for a total of 52+ high-speed SerDes lanes per GPU. Each lane requires a full transceiver (TX driver, CTLE, DFE, CDR, PLL), and the aggregate power consumption is significant (approximately 50-100 mW per lane for the analog portion, plus digital overhead).

The package pin count and routing density for 52+ differential pairs (104+ signal balls) is a major constraint on the GPU package design. The lanes must be routed from the die edge through the package redistribution layers to the BGA balls, with controlled impedance and matched length within each pair. For advanced packages with silicon interposers (like the H100's CoWoS package), the interposer provides an additional routing layer but also adds loss and impedance discontinuities.

Signal integrity in dense lane arrays is challenging because the close proximity of many lanes creates significant crosstalk. In a server motherboard with 36 NVLink pairs and 16 PCIe pairs routed in parallel, the far-end crosstalk (FEXT) from adjacent aggressors can degrade the victim lane's eye opening by 10-20%. The SerDes must include sufficient margin for crosstalk, and the board design must use techniques like GSSG (ground-signal-signal-ground) routing and via fencing to minimize coupling.

Thermal management of the SerDes I/O ring is challenging because the transceivers are concentrated along the chip edges and dissipate significant power density. The junction temperature of the I/O transistors can be 10-20 degrees Celsius higher than the core due to localized heating, which affects the SerDes performance (increased jitter, reduced bandwidth, shifted impedance) and must be accounted for in the design.

---

### Q9. How do NVLink and PCIe coexist on the same GPU, and what are the SerDes sharing implications?

**Answer:**

Modern NVIDIA GPUs support both NVLink and PCIe on the same die, with some SerDes lanes being configurable for either protocol. The H100, for example, has dedicated NVLink lanes and dedicated PCIe lanes, but some configurations allow trading NVLink bandwidth for additional PCIe bandwidth (or vice versa) through lane remapping.

From a SerDes perspective, the NVLink and PCIe transceivers may share the same analog front-end (driver, CTLE, DFE, CDR) with different protocol-layer configurations. The electrical signaling for both NVLink (112G PAM4) and PCIe Gen5 (32G NRZ) uses the same fundamental SerDes building blocks, although the operating modes differ (PAM4 vs NRZ, different baud rates, different equalization settings).

A multi-mode SerDes that supports both NVLink and PCIe must be designed for the more demanding specification: NVLink's 56 GBd PAM4 requires wider bandwidth, finer clock resolution, and more DFE taps than PCIe Gen5's 32 GBd NRZ. When operating in PCIe mode, the excess capability is unused (or can be power-gated to save energy).

The PLL design must support multiple frequencies: 28 GHz for NVLink (56 GBd half-rate) and 16 GHz for PCIe Gen5 (32 GBd half-rate). This requires either a fractional-N PLL that can lock to both frequencies, or separate PLLs for each mode. The reference clock may also differ: PCIe uses a 100 MHz reference, while NVLink may use a proprietary reference frequency.

The lane remapping between NVLink and PCIe must maintain signal integrity for both protocols. PCIe lanes may need AC coupling capacitors (required by the PCIe specification for hot-plug support and DC voltage isolation), while NVLink lanes may use DC coupling for better signal quality. If the same physical lanes are shared, the AC coupling capacitors must be designed to not degrade NVLink performance when present.

---

### Q10. What are the emerging trends in custom AI interconnects for next-generation systems?

**Answer:**

Several emerging trends are shaping the next generation of custom AI interconnects. The move to 224G per lane (112 GBd PAM4) will double the per-lane bandwidth, enabling each GPU to support more bandwidth without increasing the pin count. NVIDIA's next-generation NVLink (expected in the Blackwell architecture and beyond) is anticipated to use 224G signaling, potentially doubling the per-GPU bandwidth to 1.8+ TB/s.

Co-packaged optics (CPO) replaces the electrical copper channels between GPUs with optical links, eliminating the bandwidth-distance tradeoff of copper channels. With CPO, each GPU package includes silicon photonic transceivers that convert electrical signals to optical signals for transmission over fiber. The bandwidth density of optical links is much higher than copper (because optical signals do not suffer from skin-effect or dielectric loss), and the reach can extend to tens of meters (enabling rack-scale all-to-all connectivity). NVIDIA, Broadcom, and Intel are all developing CPO solutions for AI datacenter interconnects.

Chiplet-based switch ASICs use UCIe or proprietary die-to-die links to build larger switch fabrics than are possible with a single monolithic die. A future NVSwitch could combine multiple switch chiplets via UCIe to support 128+ ports, enabling direct all-to-all connectivity for 16+ GPUs without the need for hierarchical topologies.

In-network computing integrates simple processing elements (such as reduction engines for gradient summation) into the switch fabric, enabling collective operations to be performed in the network rather than at the endpoints. This reduces the number of data traversals across the links and can improve the effective bandwidth utilization by 2-3 times for all-reduce operations.

Photonic interconnect fabrics that use optical switching (rather than electrical switching with optical I/O) are being researched as a long-term solution for AI supercomputer interconnects. Optical switches can provide non-blocking all-to-all connectivity at the speed of light with minimal energy per bit, but the technology is still in the research stage.

---

### Q11. How does the choice of interconnect topology affect AI training efficiency?

**Answer:**

The interconnect topology determines three critical parameters for AI training efficiency: the bisection bandwidth (which limits the total data throughput of collective operations), the average hop count (which affects latency and energy consumption), and the scalability (how the performance scales as more GPUs are added).

For data-parallel training, the dominant communication pattern is all-reduce, which aggregates gradients across all GPUs. The all-reduce time is approximately: T_allreduce = alpha + beta * data_size, where alpha is the latency (proportional to the number of hops and the per-hop latency) and beta is the inverse bandwidth (determined by the bisection bandwidth and the all-reduce algorithm).

A fully-connected topology (like NVSwitch-based systems) provides the best performance for all-reduce because every GPU can communicate with every other GPU simultaneously, achieving T_allreduce = alpha_1hop + data_size / BW_per_link. The latency is one hop (minimum), and the bandwidth is the full per-GPU link bandwidth.

A ring topology requires N-1 steps for N GPUs, giving T_allreduce = (N-1) * (alpha_1hop + data_size/(N*BW_ring)). The latency scales with N, and the bandwidth utilization is only 1/N of the total link bandwidth per step, but the aggregate bandwidth utilization across all steps is nearly 100% for large data sizes.

A 3D torus (like Google's TPU topology) is between these extremes: the hop count is O(N^(1/3)) for a cubic torus of N nodes, and efficient all-reduce algorithms (such as the recursive halving-doubling algorithm) can achieve bandwidth utilization close to the bisection bandwidth.

For large-scale AI training (1000+ GPUs), hierarchical topologies are used: fully-connected within each node (8-16 GPUs via NVSwitch), fat-tree or Clos network between nodes (via InfiniBand or Ethernet). The training software must be aware of the topology hierarchy to optimize data placement and communication patterns, typically using a combination of data parallelism (across nodes) and tensor/pipeline parallelism (within nodes).

---

### Q12. What bandwidth will be required for interconnects in future AI systems with 10x larger models?

**Answer:**

Projecting interconnect requirements for AI systems 10 times larger than current systems requires understanding the scaling relationships between model size, compute, and communication.

Current state of the art (2025-2026): models with approximately 1 trillion parameters, trained on clusters of approximately 4000-8000 GPUs (like NVIDIA H100), with per-GPU interconnect bandwidth of approximately 900 GB/s (NVLink) and inter-node bandwidth of approximately 400 Gbps per GPU.

For 10x larger models (approximately 10 trillion parameters) expected in 2027-2029: the compute requirement scales super-linearly (approximately 10-30x for 10x more parameters), requiring approximately 40000-100000 GPUs. The per-GPU interconnect bandwidth must scale proportionally to the compute-to-communication ratio of the parallelism strategy.

For data parallelism, the all-reduce data volume scales linearly with model size (the gradient tensor is the same size as the model). For 10x larger models, the all-reduce volume is 10x larger, requiring either 10x more bandwidth or 10x more time. Since the compute time per iteration also increases (roughly proportionally to model size for fixed batch size), the bandwidth requirement scales with the compute-to-communication ratio, which is approximately constant for well-balanced systems. This suggests per-GPU bandwidth of approximately 900 GB/s to 1.8 TB/s (1-2x current NVLink) would suffice for data parallelism.

For tensor/pipeline parallelism (which is used within each node), the communication volume scales with the activation tensor size, which can be O(model_size^0.5) to O(model_size) depending on the parallelism strategy. This suggests intra-node bandwidth requirements of approximately 2-5 TB/s per GPU for 10x larger models.

The SerDes technology needed to support 2-5 TB/s per GPU includes 224G per lane (doubling the per-lane rate), more lanes per GPU (50-100+ lanes), and potentially co-packaged optics (for inter-node links that need to span rack-scale distances). The total SerDes power per GPU would be approximately 30-60W at 224G (assuming 5-7 pJ/bit), which is 10-20% of the GPU's total power budget (approximately 300-500W for next-generation GPUs).
