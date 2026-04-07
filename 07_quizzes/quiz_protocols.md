# Quiz: Protocols for AI

Test your knowledge of PCIe, UCIe, NVLink, and custom AI interconnects.

---

**1.** What signaling does PCIe Gen6 use?
- A) NRZ at 64 GBd
- B) PAM4 at 32 GBd
- C) PAM4 at 64 GBd
- D) NRZ at 32 GBd

**2.** What is FLIT mode in PCIe Gen6?
- A) A low-power mode
- B) Fixed-size flow control units replacing 128b/130b encoding
- C) A new equalization algorithm
- D) A cable specification

**3.** How many NVLink 4.0 links does the NVIDIA H100 GPU have?
- A) 8
- B) 12
- C) 18
- D) 36

**4.** What is the total bidirectional NVLink bandwidth of the H100?
- A) 450 GB/s
- B) 600 GB/s
- C) 900 GB/s
- D) 1800 GB/s

**5.** What is the UCIe advanced package bump pitch?
- A) 100-130 um
- B) 45-55 um
- C) 25-55 um
- D) 10-15 um

**6.** What topology does Google use for TPU v4 interconnects?
- A) Ring
- B) Fully connected mesh
- C) 3D torus
- D) Fat tree

**7.** What is the function of NVSwitch?
- A) Converting NVLink to PCIe
- B) Providing non-blocking crossbar for all-to-all GPU communication
- C) Managing GPU power states
- D) Encrypting GPU-to-GPU data

**8.** What per-lane rate does PCIe Gen5 operate at?
- A) 16 GT/s
- B) 25 GT/s
- C) 32 GT/s
- D) 64 GT/s

**9.** Which UCIe module type uses single-ended signaling?
- A) Standard Package Module
- B) Advanced Package Module
- C) Both modules
- D) Neither module

**10.** What is CXL primarily used for in AI systems?
- A) GPU-to-GPU communication
- B) Memory expansion and cache-coherent device access
- C) Storage connectivity
- D) Display output

**11.** What is the approximate bandwidth density of UCIe advanced package?
- A) 0.1 Tbps/mm
- B) 0.5 Tbps/mm
- C) 1.3 Tbps/mm
- D) 10 Tbps/mm

**12.** How many retimers does PCIe Gen5/Gen6 support per link?
- A) 0
- B) 1
- C) 2
- D) Unlimited

**13.** What is AMD's proprietary interconnect called?
- A) NVLink
- B) UPI
- C) Infinity Fabric
- D) ICI

**14.** What encoding overhead does PCIe Gen5 128b/130b have?
- A) 0%
- B) 1.54%
- C) 3.125%
- D) 20%

**15.** What is the primary advantage of a 3D torus topology for large-scale AI clusters?
- A) Lowest latency
- B) Highest bisection bandwidth
- C) Fixed link count per node regardless of cluster size
- D) Simplest routing

**16.** What is the approximate latency of a UCIe advanced package link?
- A) 100 ns
- B) 20-50 ns
- C) 2-5 ns
- D) 0.1 ns

---

## Answer Key

1. **B** - Gen6 uses PAM4 at 32 GBd (same baud rate as Gen5 NRZ)
2. **B** - FLITs are fixed 256-byte units that replace the variable-length TLP encoding
3. **C** - H100 has 18 NVLink 4.0 links
4. **C** - 18 links * 50 GB/s per link = 900 GB/s bidirectional
5. **C** - Advanced package supports 25-55 um bump pitch
6. **C** - TPU v4 uses a 3D torus (16x16x16 for 4096 chips)
7. **B** - NVSwitch provides non-blocking any-to-any switching for GPUs
8. **C** - PCIe Gen5 operates at 32 GT/s NRZ
9. **B** - Advanced Package Module uses single-ended signaling for maximum density
10. **B** - CXL provides memory expansion and cache-coherent access for AI systems
11. **C** - Approximately 1.3 Tbps/mm for the advanced package module
12. **C** - PCIe supports up to 2 retimers per link
13. **C** - AMD uses Infinity Fabric for inter-chip connectivity
14. **B** - 128b/130b has 2/130 = 1.54% overhead
15. **C** - 3D torus requires only 6 links per node regardless of cluster size
16. **C** - UCIe advanced package achieves 2-5 ns one-way latency
