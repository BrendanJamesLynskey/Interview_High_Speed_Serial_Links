# Power Supply Noise Coupling

## Overview

Power supply noise is a critical impairment in high-speed serial links, particularly in AI accelerators where hundreds of amps of switching current create significant voltage fluctuations on the power delivery network (PDN). This section covers simultaneous switching noise, ground bounce, jitter from supply noise, and PDN design for SerDes.

---

### Q1. How does power supply noise couple into SerDes circuits and affect signal quality?

**Answer:**

Power supply noise couples into SerDes circuits through multiple paths. The transmitter driver's output voltage is directly modulated by supply noise because the driver transistors use VDD as their voltage source. For a voltage-mode (SST) driver, approximately 50% of the supply noise appears on the output (as discussed in the driver architectures section). The receiver's CTLE and slicer are also affected: supply noise modulates the bias points of the amplifier transistors, shifting the gain and threshold voltage. The PLL/CDR VCO frequency is sensitive to supply voltage through the varactor capacitance (for LC-PLLs) or the inverter delay (for ring oscillators), causing supply-induced jitter.

The coupling mechanisms include direct modulation (the supply voltage appears as part of the signal), common-mode to differential conversion (supply noise that couples asymmetrically to the D+ and D- paths appears as differential noise), and substrate coupling (supply noise propagates through the silicon substrate to sensitive analog circuits, bypassing the intended power distribution).

The magnitude of supply noise in a typical AI accelerator is 10-50 mV peak-to-peak on the core supply (0.7-0.9V) and 5-20 mV on the I/O supply, depending on the PDN impedance and the switching activity. This noise has spectral content from DC to several hundred MHz (from the switching regulator ripple) and from hundreds of MHz to several GHz (from the digital core's switching activity).

For a PAM4 link with 50 mV inner eye height, even 5 mV of supply-induced noise at the sampling point represents a 10% eye height reduction, which can be the difference between meeting and failing the COM specification. Supply noise management is therefore a critical aspect of SerDes design in AI systems.

---

### Q2. What is simultaneous switching noise (SSN) and how does it affect SerDes in AI accelerators?

**Answer:**

Simultaneous switching noise (SSN) occurs when many digital circuits switch simultaneously, drawing large transient currents from the power supply. The transient current (dI/dt) flowing through the parasitic inductance (L) of the power delivery network creates a voltage drop: V_noise = L * dI/dt. This voltage drop reduces the effective supply voltage for the switching circuits and creates noise that couples to neighboring circuits, including the SerDes.

In an AI accelerator, SSN is particularly severe because the compute core (GPU/TPU) draws hundreds of amps and switches at high frequencies. A typical GPU core with 100A of switching current and a package inductance of 10 pH creates: V_SSN = 10e-12 * 100 / 1e-9 = 1V of instantaneous noise for a 1 ns transition. In practice, the on-die decoupling capacitance limits the dI/dt, reducing the peak SSN to 10-50 mV, but this is still significant.

The SSN spectrum depends on the switching activity pattern. Workload-dependent activity (such as matrix multiplication kernels that activate many compute units simultaneously) creates low-frequency SSN (10-100 MHz) that corresponds to the workload's activity cycle. Clock-tree switching creates SSN at the core clock frequency and its harmonics (1-5 GHz). Power state transitions (such as entering or exiting power-saving modes) create wideband SSN as the current demand changes abruptly.

SSN affects SerDes through several paths. The shared power delivery network allows SSN from the digital core to propagate to the I/O supply, particularly if the power domains are not well isolated. Substrate coupling transmits SSN through the silicon substrate. Electromagnetic coupling from the power planes to the signal traces creates common-mode noise on the SerDes signal paths.

Mitigation techniques include separate power domains for the digital core and I/O (with independent voltage regulators and physically separated power planes), on-die decoupling capacitance near the SerDes circuits (to provide local charge storage), and PDN impedance engineering (maintaining low impedance across the frequency band of interest).

---

### Q3. What is ground bounce and how does it differ from supply droop in SerDes context?

**Answer:**

Ground bounce is the transient elevation of the local ground potential relative to the reference ground, caused by current flowing through the parasitic inductance of the ground path. Supply droop is the transient reduction of the supply voltage caused by current flowing through the parasitic inductance and resistance of the supply path. Both effects reduce the effective supply voltage available to the circuits and create noise.

The key difference is in the coupling mechanism to the SerDes signal. Ground bounce shifts the entire circuit's reference point (since all voltages are measured relative to ground), effectively adding a common-mode offset to both D+ and D-. If the ground bounce is identical at the transmitter and receiver (which it is not, since they are on different dies), it would be perfectly rejected by the differential receiver. In practice, the ground bounce is different at the TX and RX, creating a differential noise component.

Supply droop directly modulates the transmitter output swing (for voltage-mode drivers) and the receiver's threshold accuracy. A 20 mV supply droop on a transmitter with 800 mVpp output swing reduces the swing by 2.5%, which for PAM4 reduces the inner eye height by approximately 6.7 mV (from 266.7 mV to 260.0 mV in the ideal case, before channel loss).

In AI accelerators, ground bounce is often the more problematic effect because the ground plane is shared between the digital core and the I/O circuits, and the massive switching currents of the compute core create ground bounce that couples to the SerDes. The ground bounce frequency spectrum extends from the core clock frequency (where the switching current is highest) down to the workload activity frequency.

Mitigation includes dedicated ground planes for the I/O circuits (physically separated from the core ground and connected only at a single star point), extensive ground via arrays around the SerDes I/O pad ring (to provide low-impedance local grounding), and common-mode feedback circuits in the SerDes that sense and compensate for ground bounce.

---

### Q4. How does PDN impedance affect SerDes jitter and what impedance targets are used?

**Answer:**

The PDN (Power Delivery Network) impedance determines how much supply voltage fluctuation results from a given current transient. The relationship is: V_noise(f) = Z_PDN(f) * I_load(f), where Z_PDN is the frequency-dependent PDN impedance and I_load is the frequency-dependent load current. Lower PDN impedance means less supply noise for the same current demand.

For SerDes circuits, the PDN impedance must be low across the frequency band from the switching regulator frequency (typically 0.5-3 MHz) up to the SerDes operating frequency (56 GBd). The target impedance is typically computed from the allowable supply voltage variation:

```
Z_target = V_ripple_max / I_transient_max
```

For a SerDes I/O supply with V_ripple_max = 10 mV and I_transient_max = 100 mA per lane: Z_target = 10 mV / 100 mA = 100 milliohms. This target must be met from DC to at least 10 GHz.

The PDN impedance is dominated by different components at different frequencies. At DC to about 10 kHz, the voltage regulator module (VRM) output impedance dominates. From 10 kHz to 10 MHz, the bulk decoupling capacitors (10-100 uF, placed on the PCB near the BGA) dominate. From 10 MHz to 1 GHz, the high-frequency decoupling capacitors (100 pF to 10 nF, placed close to the BGA or on the package) dominate. Above 1 GHz, the on-die decoupling capacitance (MOS capacitors integrated on the die) dominates.

The anti-resonance between capacitor stages (where the inductance of one stage resonates with the capacitance of the next) can create impedance peaks that violate the target impedance. These peaks are managed by overlapping the frequency ranges of the decoupling stages and adding damping resistance.

For AI accelerators, the PDN design must accommodate the extreme current demands (100+ amps total) while maintaining milliohm-level impedance up to several GHz. This requires a multi-layer PCB stackup with dedicated power planes, hundreds of decoupling capacitors in a hierarchical arrangement, and on-die decoupling integrated into the SerDes floor plan.

---

### Q5. How does power supply noise convert to jitter in the PLL and CDR?

**Answer:**

Power supply noise converts to jitter through the supply sensitivity of the VCO or CDR oscillator. The VCO frequency depends on the supply voltage because the transistor parameters (threshold voltage, transconductance, capacitance) that determine the oscillation frequency are all supply-dependent.

For an LC-VCO, the supply noise modulates the varactor bias voltage and the active device transconductance, changing the oscillation frequency. The supply-to-frequency conversion gain (K_supply) is typically 1-10 MHz/mV for an on-chip LC-VCO. A 10 mV supply fluctuation at 100 MHz creates a frequency modulation of 10-100 MHz at 100 MHz rate, which translates to a phase modulation (jitter) of:

```
Jitter_pk = K_supply * V_noise / (2*pi*f_noise)
= 10e6 * 10e-3 / (2*pi*100e6)
= 159 fs peak
```

This 159 fs of supply-induced jitter is a significant fraction of the 100-150 fs RMS jitter budget for a 112G PLL.

For a ring oscillator-based CDR, the supply sensitivity is much higher (50-200 MHz/mV) because the oscillation frequency is directly proportional to the inverter delay, which is strongly supply-dependent. This is one reason ring oscillator PLLs are rarely used for the main clock generation in high-speed SerDes.

The PLL loop filter provides some attenuation of supply-induced jitter: supply noise at frequencies above the PLL bandwidth is transferred to the output with the VCO's supply sensitivity, while noise at frequencies within the PLL bandwidth is partially suppressed by the loop's frequency correction. However, supply noise at frequencies near the PLL bandwidth can be amplified if the loop has peaking.

Mitigation techniques include dedicated, low-noise supply regulators for the VCO and PLL circuits, on-die LDO (Low Drop-Out) regulators that provide additional supply filtering, supply-rejection circuit techniques in the VCO (such as current-mode biasing with high-impedance current sources), and layout techniques that isolate the PLL supply from the noisy digital supply.

---

### Q6. What is the power supply rejection ratio (PSRR) of a SerDes transceiver and how is it specified?

**Answer:**

The power supply rejection ratio (PSRR) quantifies the SerDes circuit's ability to reject power supply noise. It is defined as the ratio of the supply noise to the resulting signal impairment (output jitter, eye closure, or BER degradation), expressed in dB. Higher PSRR means better noise rejection.

For the transmitter, PSRR is defined as: PSRR_TX = 20*log10(V_supply_noise / V_output_noise), where V_output_noise is the noise that appears on the differential output due to V_supply_noise on the supply. For a voltage-mode SST driver with basic design, PSRR_TX is approximately 6 dB (50% of supply noise appears on the output). With supply-noise compensation circuits, PSRR_TX can be improved to 20-30 dB.

For the PLL, PSRR is defined in terms of jitter: PSRR_PLL(f) = V_supply_noise(f) / (Jitter_output(f) * 2*pi*f_carrier), which converts the frequency sensitivity to a voltage rejection ratio. A well-designed LC-PLL with dedicated LDO achieves PSRR of 30-40 dB across the frequency band of interest.

For the receiver (CTLE and slicer), PSRR determines how much supply noise appears as threshold shift or gain variation. PSRR_RX of 20-30 dB is typical, meaning a 10 mV supply fluctuation causes less than 1 mV of threshold shift.

The PSRR is frequency-dependent and typically degrades at higher frequencies (where the parasitic capacitances bypass the rejection circuits). The specification must cover the frequency range from the switching regulator frequency (approximately 1 MHz) to the CDR bandwidth (approximately 10 MHz) and beyond, up to the Nyquist frequency of the data.

---

### Q7. How is on-die decoupling designed for SerDes circuits in AI accelerators?

**Answer:**

On-die decoupling capacitance provides local charge storage that supplies the transient current demands of the SerDes circuits without drawing current through the package inductance. The on-die capacitance must be sufficient to maintain the supply voltage within specification during the fastest switching events (which occur at the baud rate).

The required on-die capacitance for a SerDes lane can be estimated from the transient current and the allowable voltage droop:

```
C_decap = I_transient * dt / dV_max
```

For a driver that draws 10 mA of transient current over one UI (17.9 ps at 56 GBd) with 5 mV allowable droop: C_decap = 10e-3 * 17.9e-12 / 5e-3 = 35.8 pF per lane.

On-die decoupling is implemented using MOS capacitors (MOSCAP), which use the gate oxide of MOSFET transistors as the dielectric. The capacitance density depends on the process node: 5nm FinFET provides approximately 10-15 fF/um^2 of MOSCAP capacitance, so 35.8 pF requires approximately 2400-3600 um^2 (approximately 50x70 um) of die area per lane. This is a meaningful but manageable area allocation.

The placement of on-die decoupling is critical: it must be physically close to the circuits it serves (within approximately 100 um) to minimize the interconnect inductance between the capacitor and the circuit. This means the decoupling is typically placed within the SerDes macro, interleaved with the analog and digital circuits.

For a multi-lane SerDes with 50+ lanes, the total on-die decoupling area is approximately 120000-180000 um^2 (approximately 400x400 um), which is significant but small compared to the total die area (approximately 400-800 mm^2 for an AI accelerator).

---

### Q8. What role do voltage regulators play in SerDes power delivery?

**Answer:**

Voltage regulators provide the stable supply voltage needed by SerDes circuits, filtering the noise from the upstream power distribution and regulating the voltage against load variations. Modern AI accelerators use a hierarchy of voltage regulators to provide clean power to the SerDes.

Board-level switching regulators (buck converters) convert the 12V or 48V board input to the die voltage (0.7-0.9V for core, 0.8-1.1V for I/O). These regulators operate at 0.5-3 MHz switching frequency and provide high efficiency (90-95%) but have limited output noise performance (10-30 mV ripple at the switching frequency). They are placed on the PCB near the die and connected through the package power pins.

Package-level voltage regulators (FIVR, Fully Integrated Voltage Regulators) are integrated on the package substrate or interposer. They operate at higher switching frequencies (50-300 MHz) and provide cleaner output (1-5 mV ripple) but at lower efficiency (80-90%). Intel's server processors use FIVR for die-level voltage regulation.

On-die LDO (Low Drop-Out) regulators are integrated on the die itself and provide the cleanest supply to the most noise-sensitive circuits (PLL, VCO). LDOs operate as linear regulators with very high bandwidth (100 MHz to 1 GHz) and excellent noise rejection (40-60 dB PSRR). The cost is efficiency: LDOs dissipate power proportional to (V_input - V_output) * I_load, which is 10-30% of the output power for typical voltage margins.

For SerDes in AI accelerators, the typical power delivery chain is: board-level buck converter provides bulk current at the nominal voltage, with on-die LDOs providing filtered supply to the PLL/VCO and the most sensitive analog circuits. The transmitter and receiver analog circuits may use the unregulated supply (filtered by on-die decoupling) or a separate LDO, depending on the PSRR requirements.

---

### Q9. How does the PCB stackup design affect power supply noise coupling to SerDes signals?

**Answer:**

The PCB stackup determines the impedance and isolation between the power delivery network and the signal traces. A well-designed stackup minimizes the coupling between power plane noise and signal trace quality.

The key stackup design principles for SerDes power integrity are: place signal traces on layers that are sandwiched between ground planes (stripline configuration), which provides maximum shielding from power plane noise; place power planes adjacent to ground planes (with thin dielectric, approximately 2-3 mils) to create low-impedance power-ground plane pairs that provide distributed decoupling capacitance; avoid placing signal layers adjacent to power planes, which creates direct capacitive coupling between supply noise and signal traces.

The distributed capacitance of a power-ground plane pair is approximately 30-50 pF per square inch for typical dielectric thickness and material. For a full server motherboard (approximately 300 square inches), the total distributed capacitance is approximately 9-15 nF, which provides significant high-frequency decoupling.

The PCB layer count for AI server boards is typically 20-30 layers, with the following allocation: 4-6 signal layers for high-speed SerDes, 4-6 signal layers for low-speed and control signals, 6-10 power/ground plane pairs, and 2-4 ground reference planes dedicated to the high-speed signal layers.

The ground plane integrity is critical: any cuts, gaps, or splits in the ground plane under a signal trace create return current discontinuities that generate common-mode noise and increase crosstalk. All ground planes under high-speed traces must be continuous and unbroken, with no routing of other signals across them.

---

### Q10. What is the impact of simultaneous switching output (SSO) noise on multi-lane SerDes transmitters?

**Answer:**

SSO noise occurs when multiple SerDes transmitter lanes switch simultaneously, drawing transient current from the shared supply and ground. For a 16-lane PCIe interface where all lanes transition at the same instant (worst case), the total transient current is 16 times the per-lane switching current, creating a correspondingly large supply droop and ground bounce.

The worst-case SSO event for NRZ signaling occurs when all lanes transition in the same direction (all from low to high, or all from high to low). For PAM4, the worst case is when all lanes make the maximum transition (from level 0 to level 3 or vice versa) simultaneously. The probability of the worst case depends on the data encoding: 128b/130b encoding (PCIe Gen5) does not prevent all-lanes-same-direction transitions, but the probability is low for random data.

The SSO noise magnitude depends on the number of switching lanes, the per-lane switching current (typically 5-15 mA for a 50-ohm driver with 800 mV swing), the package inductance (typically 50-200 pH per power/ground pin), and the number of power/ground pins (which determines the effective inductance).

For 16 lanes switching simultaneously with 10 mA per lane and 100 pH effective inductance:

```
dI/dt = 16 * 10 mA / 0.5 UI = 16 * 10e-3 / 8.9e-12 = 18 A/ns
V_SSO = L_eff * dI/dt = 100e-12 * 18e9 = 1.8 mV
```

This is relatively small because the effective inductance is low (many parallel power/ground pins). However, if the package has fewer pins or higher inductance, the SSO can be significant: with 500 pH effective inductance, V_SSO = 9 mV, which is 18% of a PAM4 inner eye.

Mitigation includes maximizing the number of power and ground pins in the BGA (reducing effective inductance), distributing the power/ground pins uniformly among the signal pins (rather than clustering them), using staggered driver enable timing (intentionally delaying some lanes by a fraction of UI to spread the switching current), and on-die current-mode regulation that limits the dI/dt of each driver.

---

### Q11. How is power integrity simulation performed for SerDes in AI accelerator designs?

**Answer:**

Power integrity (PI) simulation for SerDes involves modeling the complete power delivery network from the voltage regulator to the on-die circuit, and computing the supply voltage waveforms at each circuit node for representative workload conditions.

The PI simulation flow typically includes VRM modeling (the voltage regulator is modeled as a voltage source with output impedance and transient response), PCB PDN extraction (the power and ground planes are modeled using 2D electromagnetic simulation or distributed RLC networks), package PDN extraction (the package power delivery is extracted from 3D electromagnetic simulation of the BGA, redistribution layer, and bump structures), and on-die PDN modeling (the on-die power grid and decoupling capacitance are modeled using parasitic extraction from the physical layout).

The simulation is performed in both frequency domain (impedance analysis, to verify that the Z_target is met across the frequency band) and time domain (transient analysis, to verify that the voltage droops and bounces are within specification for representative current waveforms).

For SerDes-specific PI analysis, the simulation must capture the data-dependent switching current of the transmitter drivers (which depends on the data pattern and the FIR pre-emphasis setting), the supply sensitivity of the PLL/VCO (to predict supply-induced jitter), and the supply sensitivity of the receiver circuits (to predict supply-induced threshold shifts and gain variations).

The simulation tools used include Ansys SIwave and HFSS for PCB and package PI extraction, Cadence Sigrity for system-level PI analysis, and SPICE-based circuit simulators for the combined electrical-circuit simulation. The accuracy of PI simulation is typically within 20-30% of measurement for the supply voltage magnitude and within 2-3 dB for the impedance profile.

---

### Q12. What are the emerging challenges for power integrity in 224G SerDes designs?

**Answer:**

At 224G (112 GBd), power integrity faces several new challenges. The SerDes power consumption increases to approximately 7-15 pJ/bit per lane (up from 3-5 pJ/bit at 112G), driven by the more aggressive equalization, higher-resolution DAC/ADC, and faster clock circuits. For a multi-lane design with 64 lanes, the total SerDes power could reach 50-100W, creating substantial current demands on the PDN.

The frequency content of the switching noise extends to higher frequencies. At 112 GBd, the driver switching events occur every 8.9 ps, creating spectral content up to 56 GHz. The on-die decoupling must provide low impedance up to these frequencies, which requires very close placement of the decoupling (within 10-20 um of the driver) and advanced MOS capacitor structures with low series resistance.

The PLL/VCO supply sensitivity becomes more critical as the jitter budget shrinks. At 112 GBd, the total PLL jitter budget is approximately 50-80 fs RMS, and supply-induced jitter must be limited to less than 20-30 fs RMS. This requires PSRR improvement of approximately 6 dB compared to current 56 GBd designs, achievable through better on-die LDO design or embedded supply-noise cancellation circuits in the VCO.

The package power delivery for 224G SerDes must maintain lower impedance at higher frequencies. Advanced packaging technologies (silicon interposers, embedded bridge die) provide shorter power delivery paths with lower inductance, which is one of the drivers for their adoption in AI accelerators beyond the signal integrity benefits.

Co-packaged optics (CPO) may partially address the power integrity challenge for 224G by eliminating the need for long copper channels and the associated high-power equalization. The optical transceivers operate at lower power (approximately 5 pJ/bit for the electrical-to-optical conversion) and are placed at the package edge, where the power delivery is more straightforward.
