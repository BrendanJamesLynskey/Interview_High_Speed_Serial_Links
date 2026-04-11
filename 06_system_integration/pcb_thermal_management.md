# PCB Thermal Management for High-Speed Serial Links

## Overview

Thermal management is a first-class concern for high-speed serial link PCBs because modern SerDes are power-hungry (1-4 W per lane for 112G and 224G) and because temperature directly affects signal integrity, timing, and long-term reliability. An AI accelerator running 128 lanes of 112G SerDes can dissipate 200-400 W just in the IO subsystem, and the PCB is a significant part of the heat conduction path from die to ambient. This section covers thermal modelling, copper thermal design, thermal-electrical coupling, and thermal verification for SerDes PCBs.

---

### Q1. How much power does a high-speed SerDes dissipate, and how does it scale with data rate?

**Answer:**

SerDes power dissipation is driven by the combination of signalling speed, equalisation complexity, and modulation format. Power per lane scales roughly as the data rate multiplied by an efficiency factor that has been decreasing with each generation.

| Generation | Data rate | Power per lane (TX+RX) | Total for 16-lane |
|------------|-----------|------------------------|-------------------|
| 25G NRZ | 25 Gbps | 150-250 mW | 2.4-4.0 W |
| 56G PAM4 | 56 Gbps | 400-700 mW | 6.4-11.2 W |
| 112G PAM4 | 112 Gbps | 1.5-2.5 W | 24-40 W |
| 224G PAM4 | 224 Gbps | 3-5 W (est.) | 48-80 W |

The breakdown within a 112G lane is roughly: transmitter driver and FFE 30-40%, receiver CTLE and VGA 10-15%, DFE 15-25% (more taps means more power), CDR and slicers 10-15%, PLL and clock distribution 15-20%, and auxiliary digital (training logic, FEC, encoding) 5-10%. The receiver equalisation dominates at high rates because DFE tap count grows to handle channel loss.

For AI accelerators, total SerDes power is dominated by the IO count rather than the per-lane power. A GPU with 18 NVLink ports at 112 GBd and 56 PCIe Gen6 lanes uses approximately 250-400 W for the SerDes subsystem alone. The logic compute core adds another 300-500 W, giving total device power of 600-900 W — with corresponding heat removal requirements.

Power density is the more relevant metric for PCB thermal design. A 50x50 mm SerDes BGA dissipating 100 W has a power density of 4 W/cm^2 averaged over the package. Under the localised IO regions (corners of the BGA where the SerDes macros are concentrated) the local density can reach 10-20 W/cm^2. This is comparable to CPU core hotspots and requires similar thermal solutions (heat spreaders, vapour chambers, liquid cooling).

Idle power is much lower than peak: a lane in idle with L0s or L0p power state dissipates 10-50 mW. However, AI workloads rarely leave SerDes idle for long, so thermal design must plan for sustained peak power.

---

### Q2. How does temperature affect SerDes signal integrity and jitter?

**Answer:**

Temperature affects SerDes performance through multiple coupled mechanisms. Each effect is small individually but they accumulate to meaningful margin loss at elevated temperature.

**PCB dielectric loss**: the loss tangent Df of PCB laminates increases with temperature, typically 5-15% per 50C for Megtron 6 or similar materials. A 10-inch 112G trace with 15 dB loss at 25C may have 16-17 dB loss at 85C, consuming 1-2 dB of margin. This is the dominant temperature-dependent channel impairment.

**Copper resistance**: copper resistivity rises 0.4% per degree Celsius. A 50C rise increases conductor loss by 20%. For a 10-inch trace with 2 dB of conductor loss, this is 0.4 dB added at hot operation. Less significant than dielectric loss but accumulates.

**Transistor mobility and threshold voltage**: on-die devices run slower at high temperature (electron mobility drops approximately 0.5% per degree C) and have shifted threshold voltages. The SerDes driver edge rate decreases, increasing rise/fall time and eroding eye width. CTLE gain decreases. DFE tap values may need to adapt. PLL jitter typically increases 10-30% at hot corner versus cold corner.

**Package substrate expansion**: the package substrate and die have different CTE, causing mechanical warping at elevated temperature. The warping creates stress on the BGA joints and on internal substrate vias, adding insertion loss and reflections that degrade SI.

**PDN resistance**: copper plane resistance rises with temperature, worsening IR drop on PDN and increasing supply-induced jitter.

**Package and PCB cavity resonances**: the Dk of the laminate shifts slightly with temperature, shifting resonance frequencies by 1-2% per 50C. Resonances that are above the Nyquist at cold may shift down into the band at hot.

Cumulative effect: a SerDes that has 5 dB of margin at 25C may have only 2-3 dB at 85C. At the thermal extreme (105C for extended-range parts), margin may vanish. This is why SerDes datasheets usually specify performance at a single junction temperature (often 85C) rather than across the full range, and rely on the PCB thermal design to keep operating temperature within the specified envelope.

For 224G SerDes where margin is inherently tight, the temperature coefficient is even more important: a design that closes at 25C may not close at 85C, and the thermal budget directly gates the achievable data rate.

---

### Q3. What is the heat conduction path from a SerDes die to ambient?

**Answer:**

Heat generated at the SerDes die must travel through a series of thermal resistances to reach the ambient environment. The path depends on package construction, PCB design, and cooling strategy.

For a lidded BGA SerDes ASIC (AI accelerator, network switch) with a top-side heat sink:

1. **Die to lid** via thermal interface material (TIM1). Junction-to-case top (theta-JC-top) typically 0.1-0.3 C/W for a large die with good TIM. This is the dominant path — 70-90% of the heat takes this route.
2. **Lid to heat sink** via TIM2 (thermal grease or phase-change material). Typical resistance 0.1-0.3 C/W depending on TIM quality and mounting pressure.
3. **Heat sink to ambient** via forced air or liquid. Forced-air heat sinks: 0.05-0.3 C/W depending on size and airflow. Liquid cold plates: 0.02-0.1 C/W.

Minor path through the PCB:

1. **Die to substrate** through flip-chip bumps. Approximately 0.5-1 C/W for a dense bump pattern.
2. **Substrate to PCB** through BGA balls. Each ball contributes 300-500 K/W; 1000-3000 ground/power balls in parallel give 0.1-0.5 C/W.
3. **PCB to ambient** through copper planes, thermal vias, and eventually chassis contact or air. Highly design-dependent, typically 2-10 C/W.

Because the top path dominates in a lidded ASIC, the PCB thermal design is a secondary heat path but still matters: it carries 10-30% of the heat and influences the die-to-board temperature gradient. For components without top-side cooling (low-cost designs, compact modules), the PCB becomes the primary heat path.

For bare-die (unlidded) packages such as CoWoS or EMIB-based accelerators, the die surface is exposed and TIM1 is applied directly to silicon. These packages have even better theta-JC but require more careful mounting force to avoid cracking the die.

A typical AI accelerator thermal budget: 300 W dissipated, 65C ambient (server inlet), 100C maximum junction. The total thermal resistance budget is (100 - 65) / 300 = 0.117 C/W. This is only achievable with a liquid cold plate (0.02 C/W) plus TIM1 (0.05 C/W) plus die-to-lid (0.02 C/W) plus package and interface overhead (0.03 C/W). Air cooling alone cannot meet this budget; modern AI accelerators are liquid cooled.

---

### Q4. How are thermal vias used under a SerDes BGA?

**Answer:**

Thermal vias are vertical copper connections under the BGA that carry heat from the top surface (solder balls and adjacent copper) into the PCB's internal copper layers and eventually to a bottom-side copper pour or chassis contact. They are distinct from signal vias in that their purpose is heat conduction, not electrical signalling.

Design guidelines:

- **Via size**: 0.3-0.5 mm drill diameter with 0.5-0.7 mm pad. Larger vias have lower per-via thermal resistance but take more PCB area.
- **Pitch and pattern**: 0.8-1.2 mm pitch under the BGA footprint, in a regular grid. A 40x40 mm BGA can accommodate 1000-2500 thermal vias.
- **Via-in-pad**: thermal vias placed in the BGA pad itself (via-in-pad) provide the shortest heat path but require the via to be filled and planarised to prevent solder wicking. Via-in-pad adds PCB cost (20-40%) but is standard for high-power BGAs.
- **Via fill**: filled vias (conductive epoxy or copper fill) have 50-100% lower thermal resistance than unfilled. Fully copper-filled via-in-pad is the best option. Standard filled epoxy is cheaper but thermally worse.
- **Connected layers**: thermal vias should connect as many copper layers as possible. A via that only connects the top pad to a single ground plane is much less effective than one that connects all 8 or 16 layers of ground or power plane.
- **Tie to power or ground**: thermal vias can double as PDN vias if tied to ground or power, providing both thermal and electrical function. Tie them to ground rather than power where possible (ground planes are larger and provide better heat spreading).

Thermal performance: a dense thermal via array (2000 vias, 0.4 mm drill, filled, connecting all layers) under a 40x40 mm BGA gives junction-to-board thermal resistance of approximately 1-3 C/W for a modern 16-layer board. Without thermal vias (only signal and power BGA via fanout) the same BGA has theta-JB of 5-10 C/W, three to five times worse.

Thermal vias also reduce the PCB temperature gradient under the BGA. Without them, the top layer of the PCB directly under the BGA can be 10-30C hotter than the surrounding area because heat has no path downward. Thermal vias distribute the heat across the full PCB thickness, reducing peak temperature on any one layer.

For SerDes packages that are wire-bonded or have limited top-side cooling, thermal vias are essential and should be planned at stackup time. For lidded packages with good top-side cooling, thermal vias are supplementary but still recommended.

---

### Q5. How does copper pour and plane area affect thermal performance?

**Answer:**

Copper pour and plane area function as heat spreaders, conducting heat laterally from concentrated sources (component footprints) to larger radiating areas. Copper's thermal conductivity is approximately 400 W/m.K, far higher than PCB dielectric (0.3-0.5 W/m.K), so copper is the dominant lateral heat conductor.

Thermal spreading resistance depends on the source size, the spreading area, and the copper thickness:

- **Small source, large spreader**: spreading resistance approaches zero as area grows. A 1 cm^2 source on a 100 cm^2 copper pour has approximately 2-5 C/W spreading resistance for 1 oz copper; doubling the pour area to 200 cm^2 gives approximately 1.5-3 C/W.
- **Copper thickness**: doubling copper thickness (1 oz to 2 oz) roughly halves the spreading resistance. Inner ground planes are typically 1 oz; outer planes can be 2 oz for thermal purposes.
- **Multiple parallel layers**: four 1 oz ground planes stacked gives spreading equivalent to one 4 oz plane. For a typical 16-layer board with 8 ground planes, the combined spreading is excellent.

Design guidelines:

- Under every high-power component (SerDes, PMIC, retimer), provide solid ground planes on all internal layers with no cuts or slits.
- Extend copper pour 10-20 mm beyond the component footprint to provide a large-area spreader.
- Avoid breaking ground planes with routing channels under high-power components. Routing channels create thermal bottlenecks and hot spots.
- Use 2 oz copper on inner ground planes for boards with total dissipation above 50 W, especially if air cooling is the only heat removal method.
- Connect adjacent ground planes with stitching vias at 2-4 mm pitch to equalise temperature across the stack.
- Fill unused signal-layer areas with ground pour (not isolated) to maximise lateral heat conduction on every layer.

For AI accelerator boards, the thermal design also uses dedicated thermal layers: one or two inner layers of 2-4 oz copper whose sole purpose is heat spreading. These layers are tied only to ground and carry no signals. They add 0.1-0.3 mm of thickness and cost but reduce die junction temperature by 5-15C in passive cooling scenarios.

The limit of copper-based spreading is the ambient heat transfer coefficient: once the heat is spread over a large area, it still has to leave the board. In still air (natural convection), the limit is approximately 10 W/m^2.K, giving maximum heat flux of roughly 1 W/cm^2 at a 100C board-to-ambient delta. Forced air (fan cooling) raises this 5-20x, and liquid cooling raises it 100-500x. Copper spreading only helps up to the point where the local heat flux is less than the ambient's capability.

---

### Q6. How are thermal simulations performed for SerDes PCB designs?

**Answer:**

Thermal simulation uses computational fluid dynamics (CFD) and thermal finite-element solvers to predict temperature distribution across the PCB and components. For AI server boards, thermal simulation is a standard part of the design flow because the thermal margin is tight and lab verification is expensive.

Tools: ANSYS Icepak, Siemens Flotherm, 6SigmaET, COMSOL Multiphysics, Cadence Celsius. Each has different strengths — Icepak for detailed CFD around components, Flotherm for system-level airflow, 6SigmaET for rapid early-stage analysis.

Inputs:

- **3D geometry**: the PCB, component packages (as thermal blocks or detailed models), heat sinks, and enclosure. Imported from the mechanical CAD or constructed in the thermal tool.
- **PCB stackup**: layer-by-layer copper density. Dense copper layers conduct heat well; sparse layers are insulators. The thermal tool computes an effective thermal conductivity per layer from the copper fraction.
- **Component power dissipation**: worst-case power for each component, usually from the vendor datasheet or simulation. Time-varying power profiles can be used for transient simulation.
- **Package thermal models**: two-resistor models (theta-JC and theta-JB) for simple components, or detailed compact thermal models (DELPHI, CTM) for complex packages. The model accuracy directly affects the simulation accuracy.
- **Boundary conditions**: ambient temperature, airflow rate and direction, chassis coupling (convective or conductive), radiation to external surfaces.

Solution: the solver computes steady-state temperature everywhere in the model. For transient simulation, it computes temperature versus time given a time-varying load profile.

Outputs:

- **Temperature maps**: colour-coded per component and per PCB region, showing hot spots.
- **Junction temperatures**: predicted T_j for each component. Compared against the datasheet T_j_max with margin.
- **Component temperature gradients**: hotspot within a component (useful for SerDes macros where IO corners are hotter than the centre).
- **Airflow distribution**: velocity and pressure maps for forced-air designs, showing whether airflow reaches critical components.

Accuracy: thermal simulation is typically accurate to 5-10C on absolute temperature for well-characterised components and boundary conditions. The dominant error source is usually the package thermal model (vendor-supplied models vary in quality) or the PCB copper density assumption. Correlation against lab measurement is essential for high-stakes designs.

Iteration: if the simulation predicts T_j above the target, the designer iterates on the design: add thermal vias, increase copper weight, move components, add a heat sink or cold plate, upgrade TIM. Each iteration is re-simulated. Typical AI accelerator boards go through 5-20 thermal iterations before tape-out.

For SerDes specifically, the simulation must include the temperature dependence of the electrical channel (through the material Dk and Df temperature coefficients). Some tools support co-simulation with SI tools, updating the channel model as the temperature changes. This is required for marginal designs where electrical closure depends on the operating temperature.

---

### Q7. What thermal-electrical coupling effects must be analysed for a SerDes PCB?

**Answer:**

Thermal-electrical coupling is the bidirectional interaction between temperature and electrical performance. Higher electrical activity produces more heat, and higher temperature changes electrical parameters. The loop is analysed by iterating between electrical and thermal simulation until both converge.

Coupling mechanisms:

- **Temperature-dependent loss**: the PCB dielectric loss (Df) and copper resistance rise with temperature. A SerDes channel with 5 dB loss at 25C has 6-7 dB loss at 85C. The loss increase consumes jitter and SNR margin.
- **Temperature-dependent impedance**: the dielectric constant (Dk) of PCB laminates has a small temperature coefficient, typically 0.5-1% per 50C. This shifts trace impedance by 0.2-0.5% — within the typical plus/minus 10% tolerance but potentially significant for marginal designs.
- **Temperature-dependent device behaviour**: transistor mobility, threshold, and leakage all change with temperature. SerDes transmitter edge rates slow at hot, receiver CTLE gain drops, PLL jitter increases. Equalisation must adapt.
- **Temperature-dependent refresh and retention**: not relevant for SerDes directly but relevant for adjacent DRAM that shares the thermal environment.
- **Electromigration (EM) acceleration at high temperature**: copper traces and via barrels can fail by electromigration if the current density is too high at elevated temperature. The EM failure rate doubles approximately every 10C. PCB current densities are usually low enough to ignore, but on-package and on-die EM must be considered.
- **Reliability (bathtub curve shifts)**: capacitor ESR increases with temperature and aging, gradually eroding PDN quality over the product lifetime. At 85C sustained operation, aluminium electrolytic caps lose significant capacitance over 5-7 years.

Analysis flow:

1. Run electrical simulation at nominal temperature (25C) to verify baseline eye closure.
2. Run thermal simulation with the resulting electrical power dissipation, finding worst-case component temperatures.
3. Update electrical simulation with temperature-adjusted models (channel loss, device models, PLL jitter) at the simulated temperatures.
4. Re-run electrical simulation, check if the eye still closes. If power dissipation changes significantly, go back to step 2.
5. Iterate until convergence, typically 2-4 iterations for a stable design.
6. Verify eye closure at all relevant temperature corners: cold (0C), nominal (25-45C), hot (85-105C), and any thermal transients if applicable.

For SerDes designs close to the loss budget, the thermal-electrical coupling analysis is not optional. A design that closes at 25C and fails at 85C is common when this analysis is skipped. The industry rule of thumb: budget 20-30% extra channel margin at 25C to accommodate the hot-corner degradation.

---

### Q8. How are AI accelerator boards cooled, and how does this drive PCB thermal design?

**Answer:**

AI accelerator boards dissipate 300-1000 W per GPU or TPU and cannot be cooled by air alone. The cooling strategy determines the PCB thermal constraints and drives layout decisions.

**Air cooling with heat sink + fan**: used for lower-power accelerators (under 300 W) and inference cards. A large copper or aluminium heat sink with 30-60 fins is mounted on top of the package lid via TIM. A system fan blows air across the fins. Thermal resistance: 0.1-0.3 C/W. PCB thermal constraint: the PCB itself is a secondary heat path but must not introduce bottlenecks. Decoupling caps and retimers must fit under or around the heat sink without blocking airflow.

**Liquid cooling with cold plate**: used for training accelerators (300-700 W). A copper cold plate with internal channels contacts the package lid via TIM. Coolant (typically water with glycol, 30-60C supply) flows through the channels. Thermal resistance: 0.02-0.1 C/W. PCB thermal constraint: the cold plate is large and mounts to the PCB with bolts; the PCB must provide mounting points and must not have tall components under the cold plate footprint. Hoses and fittings add mechanical complexity and risk leaks.

**Immersion cooling**: used for the highest-power designs (700+ W). The entire board is submerged in a dielectric fluid (fluorocarbon or synthetic oil) that is circulated and cooled externally. Thermal resistance: 0.05-0.15 C/W across the full board (no concentrated contact). PCB thermal constraint: all components must be compatible with the fluid (no standard electrolytic caps, limited materials). The PCB runs slightly hotter than the fluid temperature so the dielectric Dk/Df are measured at the operating temperature.

**Two-phase cooling (vapour chamber or heat pipe)**: used inside the heat sink or cold plate to move heat away from the die hotspot. Vapour chambers integrated into the lid or heat sink give excellent local spreading. Does not change PCB constraints directly but reduces junction hotspots.

PCB implications by cooling type:

- **Air-cooled**: thermal vias and copper spreading matter because the PCB carries 20-30% of the heat. Sensitive components must be kept out of the exhaust airflow to avoid cross-heating.
- **Cold plate**: thermal vias are less critical (most heat goes through the lid), but board flatness is critical for cold plate contact. Warpage over 100 um causes the cold plate to lift off in spots, creating hot zones.
- **Immersion**: all components see the fluid at a nearly uniform temperature, so hotspot mitigation is easier. But fluid compatibility constrains component selection.

For all cooling types, PCB thermal simulation must include the cooling solution in the model. A simulation assuming "no cold plate" dramatically under-predicts actual die temperature.

---

### Q9. How does component placement affect thermal management of SerDes PCBs?

**Answer:**

Component placement on a high-power PCB is a joint optimisation of electrical performance (signal length, via count, matching) and thermal performance (heat source spacing, airflow, cold plate coverage).

Placement rules for thermal:

- **Hotspot separation**: major heat sources (GPU/TPU, retimers, high-power VRMs) should be separated by 20-50 mm so their thermal fields do not overlap. Adjacent hotspots reinforce each other's temperature rise.
- **Cold plate coverage**: components that rely on the cold plate must fit within its footprint. A component placed outside the cold plate coverage area relies on PCB spreading alone and runs much hotter.
- **Airflow direction**: for forced-air designs, place heat-sensitive components upstream of major heat sources so they see the cool incoming air. Downstream components see pre-heated air and run hotter.
- **PMIC location**: PMICs dissipate 5-20 W each and are significant heat sources. Place them near the edge of the board or in a dedicated cold plate region, not adjacent to temperature-sensitive components.
- **Reference clock oscillator**: these are thermally sensitive (frequency drift with temperature) and should be placed in the coolest region of the board.
- **Sensitive analog**: LDOs, voltage references, and analog front-ends should be kept away from major heat sources.
- **Retimers**: each retimer dissipates 5-15 W and runs hot. If retimers are placed in a row along a cable exit, they can create a "hot wall" that affects air routing. Distribute them if possible.

Placement conflicts with electrical requirements:

- Short SerDes traces are required for electrical closure, which pushes components close together and creates thermal conflict.
- Decoupling caps need to be near the BGA, which crowds the space that thermal vias and heat spreaders want.
- Ground stitching vias compete with thermal vias for board area under the BGA.

The optimal placement is a negotiated compromise. Tools such as Cadence Allegro with thermal co-simulation or Mentor HyperLynx Thermal allow the designer to check temperature in near-real-time while placing components, speeding up the iteration.

For AI accelerator boards with dozens of high-power components, placement is typically done by a team with dedicated thermal and SI engineers collaborating. A single designer trying to balance everything usually makes compromises that cost margin on both sides.

---

### Q10. What thermal verification is performed on a SerDes PCB during bringup and qualification?

**Answer:**

Thermal verification uses lab measurement on prototype hardware to confirm the simulation predictions and to catch any unmodelled hotspots or thermal failures.

**Thermocouple measurement**: small-diameter thermocouples (type K or T, 0.1-0.25 mm) attached to component cases with thermally conductive epoxy. Measures case temperature directly. Fast and cheap. Limitation: measures case, not junction — junction temperature is estimated from theta-JC and the dissipated power.

**Infrared thermography**: infrared camera pointed at the board while running a workload. Produces a temperature map of the whole board in one image. Excellent for finding unexpected hotspots. Limitation: requires known surface emissivity (package plastic has emissivity around 0.9, metal lids need to be painted or taped for accurate reading), and the camera cannot see under a heat sink or cold plate.

**On-die temperature sensors**: most SerDes ASICs have built-in thermal sensors that report junction temperature via SMBus, I2C, or internal register access. Modern AI accelerators have dozens of sensors across the die, reporting hotspot locations. These are the most accurate measurements for die temperature.

**Diode measurement**: some packages expose a thermal diode on a dedicated pin. An external voltage measurement converts to temperature. Accurate and fast but only one measurement point per diode.

**Airflow measurement**: pitot tubes, hot-wire anemometers, or CFD validation tools measure airflow velocity at critical locations. Validates that the cooling system delivers the designed airflow.

**Thermal cycling**: accelerated temperature cycling (-40C to +125C or -55C to +125C, 500-1000 cycles per JEDEC JESD22-A104) tests long-term reliability. Solder joint failures, package warpage, and TIM degradation show up in these tests.

**Power cycling**: switching the component on and off repeatedly stresses the die-attach and solder joints differently than slow thermal cycling. Required for automotive qualification and useful for any product with frequent power state transitions.

**Dwell testing**: holding the board at peak temperature for 168 or 1000 hours while running a stress workload. Catches slow failures (cap aging, solder creep, insulation breakdown).

**Correlation step**: measured data is fed back into the thermal simulation to refine the model. Typical correlation targets: within 5C on absolute junction temperature, within 10% on airflow prediction. Large discrepancies require investigation — often reveal a missing thermal via, wrong TIM, or poor cold-plate contact.

**Production test**: for high-reliability products, each unit is subjected to a burn-in at elevated temperature (typically 85C) for 24-168 hours before shipment. Catches infant mortality failures before they reach the customer.

If thermal verification reveals problems, the fix depends on severity. Minor problems (hotspot 5-10C over target) may be fixed with a better TIM or improved airflow. Moderate problems (10-20C over) require component re-placement or additional thermal vias. Severe problems (over 20C over) typically require a PCB respin or a redesigned cooling solution. The cost of catching thermal problems late is very high, which is why extensive simulation and early prototype testing are standard for AI accelerator boards.

---

See also:
- [Package and PCB Routing](package_and_pcb_routing.md)
- [PCB Power Delivery](pcb_power_delivery.md)
- [Testing and Compliance](testing_and_compliance.md)
