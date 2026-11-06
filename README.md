## Abstract

A Bandgap Reference (BGR) is a fundamental building block of analog and mixed-signal ICs. It provides a stable, temperature-independent reference voltage by combining a **CTAT** (Complementary-to-Absolute-Temperature) voltage and a **PTAT** (Proportional-to-Absolute-Temperature) voltage. BGRs are used in power management ICs, ADCs, DACs, LDO regulators, sensors and battery management systems.

This project covers the design and layout of a **CMOS bandgap reference** using the open-source **SkyWater SKY130 (130 nm) PDK**. The goal is a stable reference of about **1.2 V from a 1.8 V supply** with low power consumption. The flow uses **Xschem** for schematics, **NGSpice** for simulation and **KLayout** for custom layout, followed by DRC, LVS and post-layout simulation.

---

## Design Flow at a Glance

```
1. Target specifications
        ↓
2. Study the physics: CTAT (VBE) and PTAT (ΔVBE)
        ↓
3. Study and choose the circuit blocks
   (startup, core, amplifier, output stage)
        ↓
4. Characterise SKY130 transistors (gm/ID method)
        ↓
5. Hand-design the ideal bandgap (PTAT, scaling, OTA, resistors)
        ↓
6. Simulate in NGSpice (PTAT/CTAT, VREF vs temperature, transient)
        ↓
7. Check power against the budget
        ↓
8. Next: startup circuit → layout → DRC/LVS → post-layout simulation
```

---

## Step 1: Target Specifications

| Parameter | Target | Reason |
|---|---|---|
| Supply voltage | 1.8 V | Nominal core supply for SKY130 |
| Reference voltage | about 1.2 V | Sum of the CTAT voltage and the scaled PTAT voltage |
| Temperature range | -40 °C to 125 °C | Practical IC operating range |
| Power consumption | < 100 µW | At 1.8 V this means a total supply current below about 55.6 µA |
| Startup time | < 10 µs | Reference should settle quickly after power-up |

The power limit comes from `P = VDD · IDD`, so `IDD < 100 µW / 1.8 V ≈ 55.6 µA`.

**How each spec is checked:**

| Check | What it verifies |
|---|---|
| DC operating point | Reference voltage, bias currents, transistor operating regions |
| Temperature sweep | Reference variation and temperature coefficient |
| Power analysis | Total supply current and power |
| Transient analysis | Startup behaviour and settling time |
| Post-layout simulation | Effect of extracted parasitics |

---

## Step 2: Understanding the Physics

### The CTAT voltage (VBE)

The base-emitter voltage of a diode-connected BJT is:

```
|VBE| = VT · ln(I0 / IS),      VT = kT/q
```

Two things depend on temperature and pull in opposite directions:
- `VT` is proportional to T, with `dVT/dT = k/q ≈ 0.086 mV/K` (PTAT).
- `IS`, the saturation current, grows very quickly with temperature (`IS ∝ T^(4+m) · exp(-Eg/kT)`), which makes `ln(I0/IS)` fall (CTAT).

Differentiating gives:

```
dVBE/dT = [VBE - (4+m)·VT - Eg/q] / T
```

With VBE ≈ 0.7 V, m = -3/2, VT = 26 mV, Eg/q ≈ 1.2 V and T = 300 K, this is about **-1.9 mV/K** (about -1.6 mV/K using Eg/q = 1.12 V). So `VBE` is CTAT, with a slope of roughly -1.5 to -2 mV/K.

Notes from the derivation:
- The slope depends on VBE itself: a larger VBE gives a less negative slope.
- The behaviour is slightly curved, which limits the accuracy of a first-order bandgap.
- If the bias current is itself PTAT, it adds a small positive term of about +0.086 mV/K.

### The PTAT voltage (ΔVBE)

Two BJTs Q1 and Q2 with different emitter areas are used. Their saturation currents are in the ratio of their areas:

```
IS2 / IS1 = AE2 / AE1 = n
```

Subtracting their base-emitter voltages:

```
ΔVBE = VT · ln( (IC1/IC2) · n )
```

With op-amp feedback forcing equal currents (`IC1 = IC2`):

```
ΔVBE = VT · ln(n) = (kT/q) · ln(n)
d(ΔVBE)/dT = (k/q) · ln(n)
```

This is PTAT. The strong temperature dependence of `IS` cancels exactly in the difference because both junctions share the same `JS(T)`. For n = 16, the slope is `0.086 mV/K × ln(16) ≈ +0.24 mV/K`.

The unit-device approach is used: Q1 is made of 4 unit transistors and Q2 of 64, giving n = 16 with good matching.

### The PTAT current

When ΔVBE is developed across R1:

```
IPTAT = ΔVBE / R1 = (kT / q·R1) · ln(n)
```

### Combining them for zero temperature coefficient

```
VREF = VBE + K · VT · ln(n)
dVREF/dT = (dVBE/dT) + K · (k/q) · ln(n) = 0
```

This gives `VREF ≈ 0.75 V + 18.6 × 26 mV ≈ 1.2 V`, close to the silicon bandgap voltage. The gain `K` is set by a resistor ratio.

### Forcing equal currents with feedback

An op-amp A1 senses nodes X and Y and drives the common gate node P of two PMOS transistors M1 and M2. Feedback drives `VX ≈ VY`. With equal W/L and equal VGS, `ID1 = ID2`, so:

```
|ID| · R1 = VT · ln(n)
```

The branch current is therefore PTAT, and it can be copied into more branches with extra PMOS devices.

---

## Step 3: Choosing the Circuit Blocks

The system is split into four blocks, and the main options for each were studied.

### 3.1 Startup circuit

A self-biased bandgap has a stable **zero-current state**. If it powers up there, it stays off. A startup circuit pulls node P low at power-up to push the core into its active region, then switches itself off.

| Topology | How it works | Pros | Cons |
|---|---|---|---|
| **Timing-based** | An RC network (RS, CS) makes a time-dependent voltage at node G that turns on startup transistor MS, which pulls node P low. MS turns off as the RC voltage returns to rest. | Simple; no extra amplifier | Must work across supply ramp rates, process and temperature; a very slow ramp (for example 1 ms) needs impractically large R and C |
| **Voltage-comparison** | A comparator/OTA (A3) compares Vout with a reference Vr. If Vout is too low it turns on MS, and it turns MS off once the output is up. | Responds to the real output condition; no huge RC | Extra amplifier and transistor; threshold depends on Vr and offset; must disengage cleanly |

### 3.2 Bandgap core

The core was chosen after considering the following:

- **Current-mode (PMOS-mirror) core** instead of voltage-mode: mirroring a PTAT current into resistors lets the output be scaled below 1.2 V, which suits low supply voltages. The extra resistors R2, R3 and the output branch (M3, RL) set the output level.
- **Op-amp forcing VX = VY** instead of a simple mirror: gives accurate equal currents that are not corrupted by channel-length modulation.
- **Vertical parasitic PNP BJTs (Q1, Q2):** the only source of a predictable VBE in a CMOS process.
- **Large emitter-area ratio:** reduces |VBE| (helping headroom) and reduces the effect of op-amp offset, which appears directly in the PTAT term. A compromise of 4 and 64 unit devices (n = 16) balances area and offset sensitivity.
- **Current ratio m = 1:** scaling one branch current up would reduce offset sensitivity, but it raises |VBE| and worsens low-temperature MOS headroom, so m = 1 is kept and offset is handled by good op-amp design.

**Component roles:**

| Component | Role |
|---|---|
| Q1, Q2 | Generate CTAT (VBE) and PTAT (ΔVBE) voltages; matching is essential |
| R1 | Converts ΔVBE into the PTAT current; sets current and power |
| Op-amp A1 | Forces VX = VY; its gain sets how well currents match, its offset degrades the PTAT term |
| PMOS M1, M2 | Supply equal currents to the BJT branches and copy current to the output |
| R2, R3 | Provide the constant-current (VBE/R) component for the CTAT contribution |

**Limitations accepted:** op-amp offset, large silicon area, limited headroom at low temperature, and the need for a startup circuit.

### 3.3 Amplifier

| Topology | Strengths | Weaknesses |
|---|---|---|
| **Five-transistor OTA** (chosen for the ideal design) | Only 5 MOSFETs, low power (50 µA tail), compact, PMOS load helps supply tracking | Gain only about 20, limited PSRR, limited input common-mode range, input pair may approach triode at low temperature |
| **Two-stage CMOS op-amp** | Much higher gain, tighter regulation, lower input error | More transistors, more power, compensation and stability concerns |

The five-transistor OTA works as follows: an NMOS differential pair (MX, MY) senses VX and VY, PMOS active loads (Ma, Mb) turn the current imbalance into a voltage at node P, and that voltage changes the gate voltage of M1 and M2 to correct the imbalance.

### 3.4 Output stage

| Topology | Description | Main limitation |
|---|---|---|
| **Current-output core** | PTAT current from ΔVBE/R1 plus CTAT current via R2 and R3 gives a temperature-compensated current | Accuracy depends on resistor ratios and device matching |
| **Voltage-output core** | PMOS M3 copies the current into a branch with load RL: `Vout = I(M3) · RL` | Mirror mismatch and the temperature coefficient of RL affect Vout |
| **Regulated-cascode output** | An amplifier A2 senses internal nodes VY and VN and adjusts the gate of cascode transistor M4 so `VN ≈ VY`, reducing current-copying error between M2 and M3 | Extra transistors, amplifier, power, and loop stability must be verified |

---

## Step 4: Tools Setup

| Tool | Purpose in this project |
|---|---|
| **Xschem** | Hierarchical schematic capture; runs NGSpice directly from the schematic; supports parametric sweeps |
| **NGSpice** | DC, AC and transient simulation; temperature sweeps; power and stability checks |
| **KLayout** | Custom layout with SKY130 layers; DRC and LVS workflows; GDSII export |
| **SkyWater SKY130 PDK** | Open-source 130 nm process models and rules |

---

## Step 5: Transistor Characterisation with gm/ID

### Why not the square-law?

The textbook model (`ID = ½·µCox·(W/L)·Vov²`, `gm = 2ID/Vov`) is only valid for long-channel devices in strong inversion. Modern short-channel devices show velocity saturation and mobility degradation, and `Vov` loses meaning near and below threshold. Hand results then differ a lot from BSIM SPICE results.

The **gm/ID methodology** keeps the insight of hand analysis but uses simulated, width-independent device data from the SKY130 library.

### Regions of inversion

| Region | gm/ID | Behaviour |
|---|---|---|
| Weak inversion | ≳ 20 V⁻¹ | Highest efficiency and lowest power, but lowest fT and large devices |
| Moderate inversion | about 10 to 20 V⁻¹ | Good balance of efficiency and speed; typical for OTA input pairs |
| Strong inversion | ≲ 10 V⁻¹ | Fast and small, but least efficient and needs more headroom |

### Characterisation procedure

For `nfet_01v8` (sweeping VGS) and `pfet_01v8` (sweeping VSG), the gate voltage is swept at fixed width while stepping the channel length, and four width-independent charts are extracted:

| Chart | Used for |
|---|---|
| gm/ID vs VGS | Finding the bias point and the current needed (`ID = gm ÷ (gm/ID)`) |
| Intrinsic gain (gm/gds) vs gm/ID | Choosing channel length to meet gain targets; gain grows with L |
| fT vs gm/ID | Checking speed; fT falls as gm/ID rises |
| ID/W vs gm/ID | Final sizing: `W = ID ÷ (ID/W)` |

### Sizing procedure
1. Pick gm/ID to balance gain, speed and efficiency.
2. Pick L from the intrinsic-gain chart.
3. Compute ID from the required gm.
4. Read ID/W and compute `W = ID / (ID/W)`.

---

## Step 6: Hand Design of the Ideal Bandgap Reference

### Initial parameters

| Parameter | Value |
|---|---|
| Technology | SKY130, 130 nm CMOS |
| Supply voltage | 1.8 V |
| Target VREF | 1.2 V |
| Temperature range | -40 °C to 125 °C |
| Nominal temperature | 27 °C |
| BJT emitter-area ratio | 1 : 16 |
| OTA | Five-transistor OTA |

The circuit contains a BJT core, a five-transistor OTA, MOSFET current mirrors, resistors and an output branch. Because ideal device models were used, no startup circuit was added in this first simulation.

### 6.1 PTAT voltage and current

```
T    = 27 + 273.15 = 300.15 K
VT   = k·T / q = 25.86 mV
ΔVBE = VT · ln(16) = 25.86 mV × 2.7726 = 71.72 mV
R1   = 2 kΩ                       (chosen)
IPTAT = ΔVBE / R1 = 35.86 µA
```

### 6.2 PTAT scaling resistor

Starting from an assumed `VBE = 0.662 V`:

```
VPTAT = VREF - VBE = 1.2 - 0.662 = 0.538 V
R2    = VPTAT / IPTAT = 0.538 / 35.86 µA ≈ 15 kΩ
K     = R2 / R1 = 15 kΩ / 2 kΩ = 7.5
```

### 6.3 Reference voltage

```
VREF = VBE + K · ΔVBE = 0.662 + 7.5 × 71.72 mV ≈ 1.200 V
```

If the measured VBE differs, use `K = (VREF,target - VBE) / ΔVBE`. For example, VBE = 0.680 V gives `K ≈ 7.25`. The final resistor ratio must be tuned using the actual BJT operating point.

### 6.4 Temperature compensation check

```
d(ΔVBE)/dT = (k/q) · ln(16) = 86.17 µV/K × 2.7726 = 238.9 µV/K
K = -(dVBE/dT) / d(ΔVBE)/dT = 1.8 mV/K ÷ 0.2389 mV/K ≈ 7.53
```

This is close to the chosen ratio of 7.5, which confirms the resistor choice. The CTAT slope of -1.8 mV/K is illustrative and must be extracted from the BJT simulation.

### 6.5 Five-transistor OTA sizing

| Step | Calculation | Result |
|---|---|---|
| Tail current | chosen | 50 µA |
| Branch currents | 50 µA / 2 | 25 µA each |
| Transconductance (Vov = 0.20 V) | `gm = 2·ID / Vov` | 250 µS |
| Aspect ratio (µCox = 200 µA/V², illustrative) | `W/L = 2·ID / (µCox · Vov²)` | 6.25 |
| Mirror loads | equal currents | `(W/L)2 = (W/L)1` |
| Open-loop gain (Rout = 100 kΩ, illustrative) | `AOL ≈ gm · Rout` | 25 V/V (about 28 dB) |

These W/L and gain values are only illustrative. Final dimensions must be taken from the SKY130 models using the gm/ID charts and checked in simulation.

### 6.6 Resistors and output load

```
R1 = 2 kΩ,  R2 = 15 kΩ,  R3 = 15 kΩ
VR1 = IPTAT × R1 = 71.72 mV
VR2 = IPTAT × R2 = 0.538 V
IR3 = 0.538 V / 15 kΩ = 35.87 µA
RL  = 15.4 kΩ  →  IL = 1.2 V / 15.4 kΩ = 77.92 µA
```

---

## Step 7: Simulation Results

Preliminary NGSpice simulations with ideal models show:

| Simulation | Observation |
|---|---|
| PTAT and CTAT vs temperature | CTAT falls and PTAT rises with temperature, as expected |
| VREF vs temperature | Output stays nearly constant at about **1.205 V** |
| Transient | Output must settle within 1.188 V to 1.212 V (1% of 1.2 V) |

Still to be extracted: the exact minimum and maximum VREF for the temperature coefficient (ppm/°C), and the startup time from the transient waveform.

---

## Step 8: Power Check

```
Ptotal = VDD × IDD = 1.8 V × 50 µA = 90 µW       (within the 100 µW target)
```

This holds only if 50 µA is the **total** supply current rather than just the OTA tail current.

The 15.4 kΩ load alone dissipates:

```
PL = VREF² / RL = 1.2² / 15.4 kΩ = 93.51 µW
```

With this load, the 100 µW total budget cannot be met once internal bias power is added. The load and output stage therefore need to be revisited in the next phase.

---

## Next Steps

1. **Switch to actual SKY130 devices** (replace ideal models) and sweep operating point, temperature, supply and transient.
2. **Add a startup circuit** (timing-based or voltage-comparison) and measure startup time.
3. **Extract performance numbers:** temperature coefficient, line regulation, power, startup time.
4. **Resize using gm/ID charts** and fix the load and output stage to meet the power budget.
5. **Draw the layout in KLayout** with good analog practice (unit-device matching, common-centroid style placement for the BJT array, minimising parasitics).
6. **Run DRC and LVS** to check design rules and layout-versus-schematic.
7. **Run post-layout simulation** with extracted parasitics and compare with the schematic results.

---

## Conclusion

This phase built the theory and a preliminary design of a CMOS bandgap reference in SKY130. The PTAT-CTAT principle was derived, the circuit blocks were studied and selected, SKY130 transistors were characterised with the gm/ID method, and an ideal design was calculated by hand (VT = 25.86 mV, ΔVBE = 71.72 mV, IPTAT = 35.86 µA, K = 7.5, VREF about 1.2 V). Preliminary NGSpice simulation gives about 1.205 V. The remaining work is full verification with real models, a startup circuit, layout, DRC/LVS and post-layout simulation.

---

## References

1. B. Razavi, "The design of a low-voltage bandgap reference," *IEEE Solid-State Circuits Magazine*, vol. 13, no. 3, pp. 6-13, Summer 2021.
2. K. M. Cañonero, J. S. Bernales, A. G. Macapundag, S. A. Khalid and S. E. Maestre, "Design of a bandgap voltage reference for an 8-bit SAR-ADC for powerline monitoring using SKY130 PDK," *Proc. 10th Int. Conf. Integrated Circuits and Microsystems (ICICM)*, 2025.
3. M. Gunawan, G. C. M. Meijer, J. Fonderie and J. H. Huijsing, "A curvature-corrected low-voltage bandgap reference," *IEEE J. Solid-State Circuits*, vol. 28, no. 6, pp. 667-670, Jun. 1993.
4. Y. Osaki, T. Hirose, N. Kuroki and M. Numa, "1.2-V supply, 100-nW, 1.09-V bandgap and 0.7-V supply, 52.5-nW, 0.55-V subbandgap reference circuits for nanowatt CMOS LSIs," *IEEE J. Solid-State Circuits*, vol. 48, no. 6, pp. 1530-1540, Jun. 2013.
5. G. Zhu, X. Huang, Y. You, Y. Li, W. Guo and Z. Zhu, "Multi-cell battery sensing and protection IC with integrated low-temperature-drift reference for series battery pack management," *IEEE Trans. Circuits Syst. I*, vol. 72, no. 10, pp. 5634-5645, Oct. 2025.
6. Y.-Y. Huang, M. Takamiya and P.-H. Chen, "Serial-stacked single-inductor dual-source triple-output energy harvesting interface with 2D-MPPT for battery-free IoT devices," *IEEE Trans. Circuits Syst. I*, vol. 72, no. 11, pp. 6592-6603, Nov. 2025.
7. K. Chen, L. Petruzzi, R. Hulfachor and M. Onabajo, "A 1.16-V 5.8-to-13.5-ppm/°C curvature-compensated CMOS bandgap reference circuit with a shared offset-cancellation method for internal amplifiers," *IEEE J. Solid-State Circuits*, vol. 56, no. 1, pp. 267-276, Jan. 2021.
8. X. Ming, L. Hu, Y.-L. Xin, X. Zhang, D. Gao and B. Zhang, "A high-precision resistor-less CMOS compensated bandgap reference based on successive voltage-step compensation," *IEEE Trans. Circuits Syst. I*, vol. 65, no. 12, pp. 4086-4097, Dec. 2018.
9. B. L. Hunter and W. E. Matthews, "A ±3 ppm/°C single-trim switched capacitor bandgap reference for battery monitoring applications," *IEEE Trans. Circuits Syst. I*, vol. 64, no. 4, pp. 777-786, Apr. 2017.
