![High Speed Serial Links](https://img.shields.io/badge/topic-high%20speed%20serial%20links-blue)

# Interview Preparation: High Speed Serial Links for AI

A comprehensive interview preparation repository covering high-speed serial link design for AI systems. From SerDes fundamentals at 56G/112G/224G per lane to protocol-specific knowledge for PCIe Gen5/Gen6, UCIe, and NVLink, this resource provides detailed Q&A, worked problems, and quizzes for hardware engineering interviews.

---

## Table of Contents

### 01 - Foundations
- [Serial Link Fundamentals](01_foundations/serial_link_fundamentals.md) - SerDes architecture, baud rate vs data rate, unit interval, link budget basics
- [Channel Loss and Modeling](01_foundations/channel_loss_and_modeling.md) - S-parameters, insertion loss, return loss, impedance discontinuities, COM methodology
- [Signaling and Modulation](01_foundations/signaling_and_modulation.md) - NRZ, PAM4, PAM6, differential signaling, Gray coding, FEC interaction
- Worked Problems:
  - [Problem 01: Link Budget Analysis](01_foundations/worked_problems/problem_01_link_budget_analysis.md)
  - [Problem 02: Channel Loss Estimation](01_foundations/worked_problems/problem_02_channel_loss_estimation.md)
  - [Problem 03: NRZ vs PAM4 Comparison](01_foundations/worked_problems/problem_03_nrz_vs_pam4_comparison.md)

### 02 - Transmitter Design
- [Driver Architectures](02_transmitter_design/driver_architectures.md) - Voltage-mode vs current-mode, SST vs CML, impedance matching, PAM4 DAC
- [Pre-emphasis and FIR](02_transmitter_design/pre_emphasis_and_fir.md) - FIR filter design, tap optimization, de-emphasis, coefficient constraints
- [Clock Generation and PLL](02_transmitter_design/clock_generation_and_pll.md) - LC-PLL, ring oscillator, fractional-N, jitter types, phase noise
- Worked Problems:
  - [Problem 01: TX FIR Tap Optimization](02_transmitter_design/worked_problems/problem_01_tx_fir_tap_optimization.md)
  - [Problem 02: Driver Impedance Matching](02_transmitter_design/worked_problems/problem_02_driver_impedance_matching.md)
  - [Problem 03: PLL Jitter Budgeting](02_transmitter_design/worked_problems/problem_03_pll_jitter_budgeting.md)

### 03 - Receiver Design
- [CTLE and Equalization](03_receiver_design/ctle_and_equalization.md) - CTLE topology, peaking design, noise enhancement, adaptation
- [DFE and Adaptive Equalization](03_receiver_design/dfe_and_adaptive_equalization.md) - Unrolled DFE, LMS adaptation, error propagation, floating taps
- [CDR Architectures](03_receiver_design/cdr_architectures.md) - Bang-bang, Mueller-Muller, loop bandwidth, jitter tolerance/transfer
- Worked Problems:
  - [Problem 01: CTLE Peaking Design](03_receiver_design/worked_problems/problem_01_ctle_peaking_design.md)
  - [Problem 02: DFE Tap Analysis](03_receiver_design/worked_problems/problem_02_dfe_tap_analysis.md)
  - [Problem 03: CDR Bandwidth Selection](03_receiver_design/worked_problems/problem_03_cdr_bandwidth_selection.md)

### 04 - Protocols for AI
- [PCIe Gen5 and Gen6](04_protocols_for_ai/pcie_gen5_gen6.md) - FLIT mode, equalization training, lane margining, CXL comparison
- [UCIe and Die-to-Die](04_protocols_for_ai/ucie_and_die_to_die.md) - Advanced/standard package modules, chiplet integration, link training
- [NVLink and Custom Interconnects](04_protocols_for_ai/nvlink_and_custom_interconnects.md) - NVSwitch, Google ICI, AMD Infinity Fabric, Intel UPI
- Worked Problems:
  - [Problem 01: PCIe Bandwidth Calculation](04_protocols_for_ai/worked_problems/problem_01_pcie_bandwidth_calculation.md)
  - [Problem 02: UCIe Link Design](04_protocols_for_ai/worked_problems/problem_02_ucie_link_design.md)
  - [Problem 03: Multi-GPU Interconnect Topology](04_protocols_for_ai/worked_problems/problem_03_multi_gpu_interconnect_topology.md)

### 05 - Signal Integrity
- [Eye Diagram Analysis](05_signal_integrity/eye_diagram_analysis.md) - Eye height/width, bathtub curves, BER contours, COM methodology
- [Crosstalk and Interference](05_signal_integrity/crosstalk_and_interference.md) - NEXT, FEXT, guard traces, GSSG routing, EMI management
- [Power Supply Noise Coupling](05_signal_integrity/power_supply_noise_coupling.md) - SSN, ground bounce, PDN design, PLL supply sensitivity
- Worked Problems:
  - [Problem 01: Eye Margin Budgeting](05_signal_integrity/worked_problems/problem_01_eye_margin_budgeting.md)
  - [Problem 02: Crosstalk Mitigation](05_signal_integrity/worked_problems/problem_02_crosstalk_mitigation.md)
  - [Problem 03: PDN-to-SerDes Coupling](05_signal_integrity/worked_problems/problem_03_pdn_to_serdes_coupling.md)

### 06 - System Integration
- [Package and PCB Routing](06_system_integration/package_and_pcb_routing.md) - Controlled impedance, BGA breakout, stackup design, material selection
- [PCB Power Delivery](06_system_integration/pcb_power_delivery.md) - Target impedance, decoupling hierarchy, embedded capacitance, PMIC placement, transient analysis, PDN verification
- [PCB Thermal Management](06_system_integration/pcb_thermal_management.md) - SerDes power dissipation, thermal-electrical coupling, thermal vias, cooling strategies, verification
- [Retimer and Redriver Design](06_system_integration/retimer_and_redriver_design.md) - When to use each, architecture, vendors, power/latency tradeoffs
- [Testing and Compliance](06_system_integration/testing_and_compliance.md) - BERT, oscilloscope, TDR, VNA, stressed receiver testing, COM
- Worked Problems:
  - [Problem 01: Channel Routing Optimization](06_system_integration/worked_problems/problem_01_channel_routing_optimization.md)
  - [Problem 02: Retimer Placement](06_system_integration/worked_problems/problem_02_retimer_placement.md)
  - [Problem 03: Compliance Testing Plan](06_system_integration/worked_problems/problem_03_compliance_testing_plan.md)

### 07 - Quizzes
- [Quiz: Foundations](07_quizzes/quiz_foundations.md) - 16 questions on serial link fundamentals
- [Quiz: TX/RX Design](07_quizzes/quiz_tx_rx.md) - 16 questions on transmitter and receiver design
- [Quiz: Protocols](07_quizzes/quiz_protocols.md) - 16 questions on PCIe, UCIe, NVLink, and AI interconnects
- [Quiz: System Integration](07_quizzes/quiz_system.md) - 16 questions on PCB design, SI, and testing

---

## How to Use

1. **Sequential study:** Work through sections 01-06 in order for a comprehensive review of high-speed serial link design for AI systems.
2. **Targeted review:** Jump to specific sections based on the role you are interviewing for (analog design, signal integrity, system architecture).
3. **Practice problems:** Attempt the worked problems before reading the solutions to test your problem-solving approach.
4. **Self-assessment:** Take the quizzes to identify knowledge gaps, then review the corresponding concept files.
5. **Interview simulation:** Use the Q&A format to practice explaining concepts clearly and concisely.

## Contributing

Contributions are welcome. Please open an issue or submit a pull request with corrections, additional questions, or new worked problems.

## Related Repositories

- Additional interview preparation materials may be found in related repositories covering analog/mixed-signal design, digital VLSI, and FPGA topics.

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

Last updated: 2026-04-11
