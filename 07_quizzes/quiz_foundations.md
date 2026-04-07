# Quiz: Foundations of High-Speed Serial Links

Test your knowledge of serial link fundamentals, channel modeling, and signaling schemes.

---

**1.** What is the Nyquist frequency of a 56 GBd PAM4 serial link?
- A) 56 GHz
- B) 28 GHz
- C) 14 GHz
- D) 112 GHz

**2.** How many bits does each PAM4 symbol encode?
- A) 1
- B) 2
- C) 4
- D) 6

**3.** What is the SNR penalty of PAM4 relative to NRZ?
- A) 3.01 dB
- B) 6.02 dB
- C) 9.54 dB
- D) 12.04 dB

**4.** Which loss mechanism scales linearly with frequency?
- A) Conductor (skin effect) loss
- B) Dielectric loss
- C) Radiation loss
- D) Reflection loss

**5.** What does the S-parameter SDD21 represent?
- A) Differential return loss
- B) Differential insertion loss
- C) Common-mode rejection
- D) Mode conversion

**6.** What is the unit interval (UI) at 112 GBd?
- A) 17.86 ps
- B) 8.93 ps
- C) 35.71 ps
- D) 4.46 ps

**7.** Why is FEC mandatory for PAM4 links at 112 Gbps?
- A) To provide DC balance
- B) Because PAM4 requires more bandwidth
- C) Because the reduced eye margins make raw BER of 1e-12 impractical
- D) To reduce latency

**8.** What is the typical COM threshold for a passing channel?
- A) 0 dB
- B) 1 dB
- C) 3 dB
- D) 6 dB

**9.** Which type of ISI cannot be corrected by DFE?
- A) Post-cursor ISI
- B) Pre-cursor ISI
- C) Second post-cursor ISI
- D) All ISI can be corrected by DFE

**10.** What does Gray coding achieve for PAM4?
- A) Reduces bandwidth
- B) Minimizes bit errors from single-level decision errors
- C) Provides DC balance
- D) Increases data rate

**11.** What is the typical differential impedance target for serial links?
- A) 50 ohms
- B) 75 ohms
- C) 100 ohms
- D) 150 ohms

**12.** What is a via stub resonance caused by?
- A) Excessive trace length
- B) Unused via barrel acting as an open-circuit transmission line
- C) Crosstalk between adjacent vias
- D) Impedance mismatch at the connector

**13.** Which encoding scheme does PCIe Gen5 use?
- A) 8b/10b
- B) 64b/66b
- C) 128b/130b
- D) PAM4 with FLIT mode

**14.** What is the primary advantage of differential signaling?
- A) Higher bandwidth
- B) Common-mode noise rejection
- C) Lower power
- D) Simpler routing

**15.** At what channel loss (at Nyquist) does PAM4 become advantageous over NRZ for the same bit rate?
- A) When the loss difference between NRZ and PAM4 Nyquist exceeds 3 dB
- B) When the loss difference exceeds 6 dB
- C) When the loss difference exceeds 9.54 dB
- D) PAM4 is always advantageous

**16.** What FEC code is commonly used for 100G Ethernet?
- A) Reed-Solomon RS(255,239)
- B) Reed-Solomon RS(544,514)
- C) LDPC
- D) Hamming code

---

## Answer Key

1. **B** - Nyquist = baud rate / 2 = 56/2 = 28 GHz
2. **B** - PAM4 encodes log2(4) = 2 bits per symbol
3. **C** - 20*log10(3) = 9.54 dB (three times smaller eye)
4. **B** - Dielectric loss scales linearly with frequency; conductor loss scales with sqrt(f)
5. **B** - SDD21 is the differential-mode forward transmission (insertion loss)
6. **B** - UI = 1/112e9 = 8.93 ps
7. **C** - PAM4's 9.54 dB penalty makes achieving raw BER < 1e-12 impractical with equalization alone
8. **C** - COM >= 3 dB is the standard threshold for IEEE 802.3
9. **B** - Pre-cursor ISI depends on future symbols, which DFE cannot access
10. **B** - Gray coding ensures adjacent levels differ by only 1 bit, halving BER for single-level errors
11. **C** - 100 ohms differential (50 ohms per side) is standard
12. **B** - The unused via stub creates a quarter-wave resonance
13. **C** - PCIe Gen5 uses 128b/130b encoding with 1.54% overhead
14. **B** - Common-mode noise rejection is the primary advantage
15. **C** - PAM4 wins when channel loss difference exceeds the 9.54 dB PAM4 penalty
16. **B** - RS(544,514) is the standard FEC for 100G and above Ethernet
