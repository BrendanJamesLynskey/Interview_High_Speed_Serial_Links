# Quiz: System Integration and Signal Integrity

Test your knowledge of PCB design, signal integrity, testing, and system-level considerations.

---

**1.** What is the primary purpose of via back-drilling?
- A) To reduce via impedance
- B) To remove the unused via stub that causes resonance
- C) To improve via conductivity
- D) To reduce manufacturing cost

**2.** What does FEXT stand for?
- A) Forward electromagnetic cross-termination
- B) Far-end crosstalk
- C) Frequency-extended transmission
- D) Fast error cross-testing

**3.** Which PCB material has the lowest loss tangent?
- A) Standard FR4
- B) Megtron 4
- C) Megtron 6
- D) Megtron 7

**4.** What is the GSSG routing pattern?
- A) Ground-signal-signal-ground, providing shielding between pairs
- B) A grounding strategy for power planes
- C) A serialization protocol
- D) A test pattern for compliance

**5.** What causes ground bounce in SerDes circuits?
- A) Excessive trace length
- B) Transient current through parasitic inductance of the ground path
- C) High-frequency clock noise
- D) Material defects in the PCB

**6.** What instrument measures the impedance profile along a transmission line?
- A) VNA
- B) Oscilloscope
- C) TDR
- D) Spectrum analyzer

**7.** What is the typical differential impedance tolerance for high-speed serial links?
- A) +/- 1%
- B) +/- 5%
- C) +/- 10%
- D) +/- 20%

**8.** What is a bathtub curve?
- A) A plot of BER vs sampling position across the eye
- B) A plot of impedance vs frequency
- C) A plot of insertion loss vs trace length
- D) A plot of temperature vs time

**9.** What is the purpose of the anti-pad around a via?
- A) To increase via capacitance
- B) To provide electrical clearance between the via and the plane
- C) To reduce crosstalk
- D) To improve thermal conductivity

**10.** What is TDECQ?
- A) Time-domain equalization quality
- B) Transmitter and dispersion eye closure quaternary
- C) Total deterministic error count quality
- D) Thermal-dependent equalization coefficient

**11.** What type of voltage regulator provides the cleanest supply for PLL circuits?
- A) Buck converter
- B) Boost converter
- C) LDO (Low Drop-Out) regulator
- D) Charge pump

**12.** When is a retimer preferred over a redriver?
- A) When the channel is short and low-loss
- B) When the channel loss exceeds the SerDes equalization capability
- C) When minimum latency is required
- D) When minimum power is required

**13.** What is lane margining used for?
- A) Increasing the lane width
- B) Measuring signal quality margins without interrupting data traffic
- C) Adjusting the equalization settings
- D) Testing the FEC codec

**14.** What is the fiber weave effect in PCB design?
- A) Signal attenuation from copper roughness
- B) Periodic impedance variations from glass weave in the laminate
- C) Crosstalk from parallel routing
- D) Insertion loss from via transitions

**15.** What is the typical power consumption of a 112G PAM4 SerDes lane?
- A) 10-50 mW
- B) 100-350 mW
- C) 1-3 W
- D) 5-10 W

**16.** What is COM (Channel Operating Margin)?
- A) The maximum channel length
- B) A figure of merit predicting link margin from S-parameters and equalization models
- C) The common-mode voltage specification
- D) The connector operating manual

---

## Answer Key

1. **B** - Back-drilling removes the stub that creates quarter-wave resonance
2. **B** - Far-end crosstalk: coupling appearing at the far end of the victim
3. **D** - Megtron 7 has Df approximately 0.002, the lowest of the options
4. **A** - GSSG places ground traces between signal pairs for shielding
5. **B** - Ground bounce = L * dI/dt through the ground path inductance
6. **C** - TDR measures impedance vs position along the line
7. **C** - +/- 10% is the standard tolerance (90-110 ohms for 100-ohm target)
8. **A** - Bathtub curve plots BER vs horizontal sampling position
9. **B** - Anti-pads provide clearance to prevent shorting via to plane
10. **B** - TDECQ measures transmitter impairments for PAM4 compliance
11. **C** - LDO provides highest PSRR and cleanest output for sensitive analog circuits
12. **B** - Retimers are needed when channel loss exceeds equalization capability
13. **B** - Lane margining measures margins in-service for proactive monitoring
14. **B** - Fiber weave creates periodic Dk variations affecting impedance
15. **B** - 100-350 mW per lane is typical for 112G PAM4 SerDes
16. **B** - COM predicts link margin from S-parameters and equalization models
