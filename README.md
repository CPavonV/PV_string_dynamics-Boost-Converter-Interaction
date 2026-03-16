# PV_string_dynamics-Boost-Converter-Interaction

LTSpice simulation framework for studying the dynamic interaction between a photovoltaic (PV) string and a synchronous boost converter under duty-cycle step perturbations.

The goal is to characterize the PV-converter system transient response by applying controlled step stimuli around an operating point, enabling impedance extraction and linearity assessment directly from time-domain waveforms.

![Schematic overview](schematic_overview.pdf)

## Repository Structure

```
PV_string_dynamics-Boost-Converter-Interaction/
├── LTSpice_files/
│   └── PV_Dynamics_Assessment.asc      # Main LTSpice schematic
|   ├── .gitignore
|   ├── .gitignore
|   └── schematic_overview.pdf          # Annotated schematic screenshot
├── .gitignore
├── LICENSE
├── README.md
└── schematic_overview.pdf          # Annotated schematic screenshot
```

## Circuit Description

The circuit consists of three main blocks:

**PV Source (X1)** — An enhanced single-diode model (eSDM) subcircuit representing a PV string with Ns=32 series cells and Np=1 parallel string. The model includes junction capacitance and transit-time parameters for dynamic behavior. An auxiliary `IV_curve` source is provided for initial I–V characterization via DC sweep.

**Synchronous Boost Converter** — Built around two IRFP4668 MOSFETs (M1 low-side, M2 high-side) with 80 µH inductor, 18 µF input capacitor, and 4.7 µF output capacitor. Switching frequency is 100 kHz with 100 ns dead time. The output is loaded by a 48 V battery model (with 2.4 Ω series resistance). Gate drivers (U1, U3) are powered by independent 12 V supplies.

**Control Signal** — A comparator-based PWM modulator (A1) compares the duty-cycle reference (`Vdelta`) against a 100 kHz sawtooth. The `Vdelta` signal is loaded from an external PWL file (`Single_Boost_STEPS.txt`), allowing arbitrary duty-cycle waveforms without modifying the schematic.

### Key Parameters

| Parameter | Value | Description |
|-----------|-------|-------------|
| Ns, Np | 32, 1 | Series cells (2-panels), parallel strings |
| Iph | 5.9829 A | Photogenerated current |
| Rsh | 20 Ω | Shunt resistance |
| Rs | 500 µΩ | Series resistance |
| Is | 150 nA | Saturation current |
| eta | 1.52 | Ideality factor |
| Cjo | 10 mF | Junction capacitance |
| Vj | 1.0 V | Junction potential |
| Vsun | 1 kW/m² | Irradiance |
| fsw | 100 kHz | Switching frequency |
| L1 | 80 µH | Boost inductor |
| Cin / Cout | 18 µF / 4.7 µF | Input / output capacitance |
| Vbatt | 48 V | Battery voltage |
| Rser | 2.4 Ω | Battery series resistance |

## Simulation Guide

The workflow has two phases. Run them one at a time by enabling/disabling the appropriate SPICE directive in LTSpice (right-click the directive text and toggle with `;`).

### Phase 1 — I–V Curve Characterization

This step identifies the operating point of the PV string so you can choose meaningful duty-cycle values for the step perturbations.

1. **Enable** the `.dc IV_curve 0 20 0.1` directive.
2. **Disable** the `.tran` directive (prefix with `;`).
3. Connect the `IV_curve` voltage source across the PV output (in place of the converter).
4. Run the simulation and plot `I(IV_curve)` vs. `V(Vp)` to obtain the full I–V curve.
5. Identify the voltage at which you want to operate (e.g., near MPP) and compute the corresponding duty cycle using the boost relation:

$$\delta = 1 - \frac{V_{PV}}{V_{out}}$$

### Phase 2 — Dynamic Step-Response Analysis

1. **Disable** the `.dc` directive.
2. **Enable** `.tran 19m`.
3. Disconnect the `IV_curve` source and reconnect the boost converter input to the PV output.
4. Ensure `Vdelta` points to the correct PWL file:
   ```
   Vdelta net+ net- PWL file=Single_Boost_STEPS.txt
   ```
5. Run the transient simulation.
6. Plot signals of interest: `V(Vp)`, `I(L1)`, `V(vload)`, `V(V_sw1)`.

### What to Expect

The simulation applies the following duty-cycle sequence around **δ₀ = 0.675**:

```
     d_up ─ ─ ─ ─ 0.685 ┌────┐                 ┌────┐
                         │    │                 │    │
     d0 ─ ─ ─ ─ ─ 0.675─┤    ├────┐     ┌─────┤    │     ┌────
                         │    │    │     │     │    │     │
     d_dn ─ ─ ─ ─ 0.665 │    │    └────┘     │    └────┘
                         │    │               │
           ╱─────────────┘    │               │
          ╱  ramp   hold  sm. sm. sm. sm. lrg. lrg. lrg. settle
         0   1   3   5   7   9  11  13  15  17  19 ms
```

The first four steps (±0.01 around center) test the small-signal regime. The final three steps (full ±0.02 swing) test large-signal linearity. If the transient settling shape changes significantly between the two regimes, the operating point is near the boundary of the linear region.

## Modifying the PWL Stimulus

All perturbation parameters are in `stimuli/Single_Boost_STEPS.txt`. To adapt for a different operating point:

1. **Change the center duty cycle (δ₀):** Replace all occurrences of `0.675` with your value from the I–V curve analysis.

2. **Change the perturbation amplitude (Δ):** The default is ±0.01 for small steps and ±0.02 for large steps. Adjust `d_up` and `d_dn` symmetrically around your new δ₀.

3. **Change the hold time:** Each step currently holds for 2 ms to allow the system to reach steady state. If your converter has a slower response (larger L or C), increase the hold intervals accordingly.

4. **Inline alternative:** If you prefer to avoid an external file, use this directly on the `Vdelta` source:
   ```
   Vdelta net+ net- PWL(0 0  1m {d0}  3m {d0}  3.000001m {d_up}  5m {d_up}
   + 5.000001m {d0}  7m {d0}  7.000001m {d_dn}  9m {d_dn}  9.000001m {d0}
   + 11m {d0}  11.000001m {d_dn}  13m {d_dn}  13.000001m {d_up}  15m {d_up}
   + 15.000001m {d_dn}  17m {d_dn}  17.000001m {d0}  19m {d0})
   .param d0=0.675  d_up=0.685  d_dn=0.665
   .tran 19m
   ```
   This allows parametric sweeps with `.step param` without editing the file.

## Requirements

- **LTSpice XVII** (or newer) — available free from [Analog Devices](https://www.analog.com/en/design-center/design-tools-and-calculators/ltspice-simulator.html).
- Place the `.asc` schematic, `.lib` files, and PWL file in the same directory, or update the file paths in the schematic accordingly.

## License

This project is licensed under the MIT License — see [LICENSE](LICENSE) for details.
