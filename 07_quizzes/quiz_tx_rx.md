# Quiz: Transmitter and Receiver Design

Test your knowledge of driver architectures, equalization, PLL, CTLE, DFE, and CDR.

---

**1.** What is the primary advantage of SST drivers over CML drivers?
- A) Better high-frequency performance
- B) Higher power efficiency
- C) Constant supply current
- D) Simpler design

**2.** How many parallel slicer paths does a 1-tap unrolled DFE require for PAM4?
- A) 2
- B) 4
- C) 8
- D) 16

**3.** What does CTLE noise enhancement refer to?
- A) Amplification of the signal at high frequencies
- B) Amplification of noise along with the signal due to frequency-dependent gain
- C) Reduction of noise by the equalizer
- D) Introduction of quantization noise

**4.** Which PLL type provides better phase noise for high-speed SerDes?
- A) Ring oscillator PLL
- B) LC-PLL
- C) Relaxation oscillator PLL
- D) Crystal oscillator

**5.** What is the purpose of the DFE in a receiver?
- A) To recover the clock
- B) To cancel post-cursor ISI using past decisions
- C) To boost high-frequency signal content
- D) To detect the data pattern

**6.** Which phase detector type does NOT require an edge sample?
- A) Alexander (bang-bang)
- B) Mueller-Muller
- C) Hogge
- D) Phase-frequency detector

**7.** What is the typical number of DFE taps for a 112G PAM4 SerDes?
- A) 1-2
- B) 4-6
- C) 8-15
- D) 20-30

**8.** What determines the pre-emphasis frequency response of a TX FIR filter?
- A) The channel length
- B) The tap coefficients and their positions
- C) The receiver sensitivity
- D) The PLL bandwidth

**9.** Why is the sign-sign LMS algorithm preferred for DFE adaptation?
- A) It converges faster than full LMS
- B) It eliminates the need for multipliers, simplifying hardware
- C) It provides more accurate tap values
- D) It works only for NRZ

**10.** What is duty cycle distortion (DCD) in a transmitter?
- A) Variation in output amplitude
- B) Difference between rising-edge and falling-edge delays
- C) Frequency offset of the PLL
- D) Crosstalk from adjacent lanes

**11.** What is the CDR loop bandwidth tradeoff?
- A) Wider bandwidth improves jitter transfer but degrades jitter tolerance
- B) Wider bandwidth improves jitter tolerance at high frequencies but degrades jitter transfer
- C) Bandwidth has no effect on jitter
- D) Narrower bandwidth always improves performance

**12.** What is the typical PLL jitter budget for a 112G SerDes?
- A) 1-10 ps RMS
- B) 100-150 fs RMS
- C) 500-1000 fs RMS
- D) 10-50 ns RMS

**13.** What is error propagation in DFE?
- A) Errors in the PLL that propagate to the CDR
- B) An incorrect slicer decision causing wrong DFE feedback for subsequent symbols
- C) ISI spreading beyond the DFE tap range
- D) Clock jitter affecting multiple lanes

**14.** How does impedance calibration work in an SST driver?
- A) By adjusting the supply voltage
- B) By comparing a replica driver against a reference resistor and adjusting segment count
- C) By changing the clock frequency
- D) By modifying the FIR tap coefficients

**15.** What is the function of the proportional path in a CDR loop?
- A) Frequency acquisition
- B) Immediate phase correction proportional to the phase error
- C) Jitter filtering
- D) Data decision

**16.** Why is a pre-cursor FIR tap particularly important?
- A) It provides the most equalization gain
- B) It cancels pre-cursor ISI that DFE cannot address
- C) It reduces power consumption
- D) It improves the PLL phase noise

---

## Answer Key

1. **B** - SST drivers are more power-efficient due to quadratic Vdd scaling
2. **B** - PAM4 has 4 possible values for d[n-1], requiring 4 parallel paths
3. **B** - CTLE amplifies noise at high frequencies along with the signal
4. **B** - LC-PLLs achieve 60-100 fs RMS jitter, much better than ring oscillators
5. **B** - DFE cancels post-cursor ISI by subtracting known ISI from past decisions
6. **B** - Mueller-Muller uses data samples only, no edge sampling needed
7. **C** - 8-15 taps are typical for medium to long-reach 112G channels
8. **B** - The DTFT of the tap coefficient sequence determines the frequency response
9. **B** - SS-LMS replaces multipliers with sign comparisons (XOR gates)
10. **B** - DCD is the asymmetry between rising and falling edge delays
11. **B** - Wider CDR BW tracks more jitter (better tolerance) but passes more to output (worse transfer)
12. **B** - 100-150 fs RMS is typical for the PLL contribution in 112G SerDes
13. **B** - Wrong decisions feed incorrect ISI values, potentially causing subsequent errors
14. **B** - A replica driver is compared against a precision resistor to calibrate segment count
15. **B** - The proportional path provides immediate phase correction
16. **B** - Pre-cursor ISI cannot be canceled by DFE, only by TX pre-cursor tap or CTLE
