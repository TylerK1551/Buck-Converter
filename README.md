# 12V to 5V Synchronous Buck Converter

A 5V, 3A synchronous buck converter built around the Renesas ISL6545A voltage-mode PWM controller with external MOSFETs, designed end to end in KiCad.

I took this board from schematic capture through a custom controller symbol, footprint and 3D model assignment, two-layer placement and routing, DRC, and fabrication outputs. Most of the design effort went into the layout decisions that make or break a switching converter: the hot loop, the switch node, the ground plane, and the feedback path.

**Status:** Designed and DRC-clean. Not yet fabricated or tested. The plan for bring-up is at the end of this page.



## Specs

| Parameter | Value |
|---|---|
| Input | 12V (controller VCC and power stage VIN share the rail) |
| Output | 5.0V |
| Output current | 3A |
| Controller | Renesas ISL6545A, voltage mode, 600kHz fixed |
| Switches | 2x AOS AOD4184A, 40V N-channel, DPAK |
| Inductor | 4.7µH, Bourns SRP7028A-4R7M (10A sat) |
| Output capacitor | 330µF 10V polymer, Panasonic 10SVP330MX |
| Board | 55 x 58 mm, 2 layers |
| Tools | KiCad 10 |

## Design notes

### Hot loop
The loop from the input ceramic (C8) through the high-side FET, the switch node, the low-side FET and back to C8 carries the fast current edges, so I placed it first and kept it as tight as I could. It took a few iterations: rotating Q2 shortened the switch node connection but made the ground return long, so I moved C8 to the other side of the FETs where it bridges Q1's drain tab and Q2's source directly.

The +12V leg of the loop is a filled zone, and both the +12V zone and the power pads use solid connections instead of thermal reliefs so the pulsed current isn't forced through thin spokes.

### Switch node
The switch node copper is only wide where load current actually flows (Q1 source to Q2 tab to L1). The branch to PHASE and the bootstrap cap carries only gate drive current, so it's a thin trace that leaves from right next to Q1's source instead of from the inductor end.

### Input path
J1 feeds the bulk cap (C7) first and then the FETs, so the bulk capacitor sits in the current path rather than on a side branch.

### Compensation
The datasheet's typical application shows Type II compensation, but with a low-ESR polymer output cap the ESR zero sits too high for Type II to give comfortable phase margin across part tolerances. I added R3 and C5 across the upper feedback resistor to make it Type III, which is what the datasheet recommends for general use. Values follow the datasheet's design procedure. Stability still needs to be confirmed with a load step once the board is built.

### Output voltage and remote sensing
Vout = 0.6V x (1 + 2.2k / 300) = 5.0V. The top of the feedback divider connects to J2's +5V pad with its own trace (a Kelvin connection), so the converter regulates the voltage at the output terminal and no load current flows through the sense path.

### Overcurrent
The ISL6545A senses current across the low-side FET's Rds(on). ROCSET (R5 = 1.5k) puts the worst-case trip (hot FET, minimum OCSET current) around 5A, above the roughly 3.5A peak inductor current at full load, and the typical trip stays under the inductor's 10A saturation rating.

### Ground plane and layer use
The bottom layer is a solid GND plane. Top-side ground pads drop to it through vias, with the most vias at C8 and Q2's source where the pulsed return current is. Only two signals use the bottom layer: LGATE (boxed in by the switch node and PHASE on top) and VCC. Both are thin and kept away from the area under the hot loop to limit slots in the plane.

### Tradeoffs
- PHASE runs under U1 between the pad rows. It was the only single-layer path that didn't cross UGATE or the FB/COMP traces, and I chose it over another via to keep the ground plane intact. The downside is that it passes near the FB pin. In a revision I'd look at re-orienting U1 so PHASE doesn't need to go there.
- UGATE and PHASE can't run side by side the whole way because of where the pins sit, so I prioritized keeping UGATE short and direct.

## Bill of materials

| Ref | Value | Part / spec | Function |
|---|---|---|---|
| U1 | ISL6545A | ISL6545AIBZ, SOIC-8 | Controller |
| Q1 | 40V N-FET | AOS AOD4184A, DPAK | High-side switch |
| Q2 | 40V N-FET | AOS AOD4184A, DPAK | Low-side switch |
| L1 | 4.7µH | Bourns SRP7028A-4R7M | Output inductor |
| C6 | 330µF 10V | Panasonic 10SVP330MX | Output capacitor |
| C7 | 100µF 25V | Low-ESR aluminum electrolytic | Input bulk |
| C8 | 10µF 25V | X7R, 1206 | Input HF decoupling (hot loop) |
| C4 | 0.1µF 25V | X7R, 0603 | Bootstrap |
| C3 | 1µF 25V | X7R, 0805 | VCC decoupling |
| R4 | 2.2k 1% | 0603 | Feedback divider, top |
| R2 | 300 1% | 0603 | Feedback divider, bottom |
| R1 | 3.3k 1% | 0603 | Compensation |
| C2 | 22nF | 0603 | Compensation |
| C1 | 1.5nF | C0G, 0603 | Compensation |
| R3 | 22 1% | 0603 | Compensation (Type III) |
| C5 | 15nF | 0603 | Compensation (Type III) |
| R5 | 1.5k 1% | 0603 | Overcurrent set (ROCSET) |
| J1 | 2-pin | MaiXu MX126-5.0-02P, 5mm screw terminal | 12V input |
| J2 | 2-pin | MaiXu MX126-5.0-02P, 5mm screw terminal | 5V output |

## Repository layout

- `Buck Converter.kicad_pro`, `.kicad_sch`, `.kicad_pcb`: KiCad project, schematic and PCB
- `controller_symbol.kicad_sym`: custom ISL6545A symbol
- `gerbers/`: Gerber and drill files for fabrication
- `3D Files/`: STEP models that aren't in the default KiCad library

## Next steps

- Fabricate and verify: switch node ringing, load-step response, efficiency and FET temperatures at 3A
- Re-orient U1 to get PHASE away from FB
- Add mounting holes
