# gm/ID Methodology

Detailed notes on the gm/ID sizing method used in the CMOS bandgap reference project (SkyWater SKY130 PDK).

---

## 1. Why the square-law is not enough

The textbook design equations are:

```
ID = ½ · µCox · (W/L) · Vov²        Vov = VGS - VTH
gm = 2·ID / Vov
```

They treat the overdrive voltage `Vov` as the main design knob. This only works for **long-channel devices in strong inversion**. In a 130 nm process such as SKY130:

- **Velocity saturation and mobility degradation** at short channel length make ID and gm deviate from the square law.
- **Analog circuits often operate in weak or moderate inversion** to save power, where `Vov` is close to zero or negative and loses its meaning.
- Hand-calculated results can sit far from the **BSIM SPICE** results, forcing many trial-and-error iterations.

## 2. The idea

The gm/ID methodology keeps the insight of hand analysis but replaces the square-law equations with **simulated, width-independent device data** taken straight from the SKY130 library. The inversion level (how strongly the transistor is on), not the absolute width or length alone, decides the device behaviour, and gm/ID measures that inversion level.

```
gm/ID  =  transconductance efficiency  [V⁻¹]
```

It tells you **how much transconductance you get for each unit of DC current**. Equivalently, `ID = gm ÷ (gm/ID)`: once you know the gm a circuit needs and pick a gm/ID, the current follows directly.

## 3. Why gm/ID is width-independent

Both `gm` and `ID` scale in proportion to the width W. Their ratio therefore stays the same for any W, as long as VGS, VDS and L are fixed. This is why one set of charts, made with a single fixed width, can be used to size a transistor of any width.

## 4. Limiting behaviour

| Region | Expression for gm/ID | Notes |
|---|---|---|
| Weak inversion | `gm/ID ≈ 1 / (n · VT)` | Exponential current, set by thermal voltage and slope factor n; this is the maximum efficiency the device can reach |
| Strong inversion | `gm/ID = 2 / Vov` | Square-law behaviour; falls as Vov increases |

For example, `Vov = 0.2 V` in strong inversion gives `gm/ID = 2 / 0.2 = 10 V⁻¹`. Between the two limits is moderate inversion, where no simple hand equation exists, so simulated charts or lookup tables are essential.

## 5. Regions of inversion

| Region | gm/ID | Current mechanism | Efficiency | Speed (fT) | Device size | Typical use |
|---|---|---|---|---|---|---|
| **Weak** | ≳ 20 V⁻¹ | Diffusion (exponential in VGS) | Highest | Lowest | Large width, large capacitance | Ultra-low-power blocks |
| **Moderate** | about 10 to 20 V⁻¹ | Mix of drift and diffusion | Good | Good | Medium | OTA input pairs, gain devices |
| **Strong** | ≲ 10 V⁻¹ | Drift (square-law trend) | Lowest | Highest | Small | Current mirrors, high-speed paths, mismatch-critical devices |

## 6. Characterisation setup

The devices characterised are the SKY130 1.8 V transistors **`nfet_01v8`** and **`pfet_01v8`**. For each one:

1. **Fix the width W** (the results are width-independent).
2. **Bias the drain** at a representative VDS and tie the source and bulk appropriately (sweep VGS for the NMOS and VSG for the PMOS).
3. **Sweep the gate voltage** over the full range, from weak to strong inversion.
4. **Step the channel length L** over several values, so each chart shows one curve per L.
5. **Run the simulation in NGSpice** and record, at every bias point:
   - drain current `ID`
   - transconductance `gm`
   - output conductance `gds`
   - gate capacitance `Cgg`
6. **Compute the derived quantities** from these:

```
gm/ID           transconductance efficiency
gm/gds          intrinsic gain
fT ≈ gm / (2π · Cgg)   transit frequency
ID/W            current density
```

The NMOS and PMOS testbenches are identical in principle, with only the polarity swapped.

## 7. The four design charts

| Chart | Axes | What it tells you | How it is used |
|---|---|---|---|
| **gm/ID vs VGS** (VSG for PMOS) | gm/ID against gate voltage | Maps the bias voltage to the inversion level: high values are weak inversion, low values are strong | Find the gate voltage that gives the chosen gm/ID, and the current required (`ID = gm ÷ gm/ID`) |
| **Intrinsic gain (gm/gds) vs gm/ID** | Gain against gm/ID, one curve per L | The maximum single-stage voltage gain the device can give | Gain rises with gm/ID and grows strongly with L, so pick the **channel length** that meets the gain target |
| **fT vs gm/ID** | Transit frequency against gm/ID | The speed limit of the transistor | Shows the key trade-off: **efficiency and speed work against each other**, since fT falls as gm/ID rises |
| **ID/W vs gm/ID** | Current density against gm/ID | The current each micron of width carries at that inversion level | Final step: `W = ID ÷ (ID/W)` |

## 8. The central trade-off

```
High gm/ID  →  more gm per µA (low power, high gain per current)
               but lower fT, bigger devices, more capacitance

Low gm/ID   →  high speed, small devices
               but needs more current for the same gm, and more headroom
```

The choice of gm/ID is therefore the main design decision for each transistor, and it is made from the charts rather than from a guessed overdrive voltage.

## 9. Sizing procedure

For any transistor in the circuit:

1. **Find the target gm** from the circuit requirement (gain, bandwidth, loop gain).
2. **Choose gm/ID** to balance efficiency, speed and headroom (use the regions table in section 5).
3. **Choose the channel length L** from the intrinsic-gain chart, so that `gm/gds` meets the gain requirement.
4. **Calculate the drain current:** `ID = gm ÷ (gm/ID)`.
5. **Read the current density ID/W** from the ID/W chart at the chosen gm/ID and L.
6. **Calculate the width:** `W = ID ÷ (ID/W)`.
7. **Verify in NGSpice** with the full circuit, since interactions with other devices and the real VDS can shift the values.

## 10. Example: OTA input pair

The five-transistor OTA in this design has a 50 µA tail current, so each input transistor carries 25 µA. The hand design used `gm = 250 µS`. The corresponding inversion level is:

```
gm/ID = 250 µS / 25 µA = 10 V⁻¹
```

This is at the boundary between moderate and strong inversion, so it is a good compromise between gain and speed for an OTA input pair, and it matches the 0.2 V overdrive used in the hand calculation (`2 / 0.2 V = 10 V⁻¹`).

From here the remaining steps are:
- choose L from the intrinsic-gain chart so the OTA reaches the gain target,
- read ID/W at `gm/ID = 10 V⁻¹` for that L,
- compute `W = 25 µA ÷ (ID/W)`.

The same procedure sizes the PMOS active loads and the PMOS current-mirror devices in the bandgap core. For mismatch-critical mirrors, a lower gm/ID (stronger inversion) and longer L are usually preferred.

## 11. Compared with the square-law method

| | Square-law hand design | gm/ID method |
|---|---|---|
| Model | Simplified equations | Simulated BSIM data from the PDK |
| Valid in | Long-channel, strong inversion | Weak, moderate and strong inversion |
| Short-channel effects | Ignored | Included automatically |
| Design knob | Overdrive voltage Vov | Inversion level gm/ID |
| Accuracy vs SPICE | Often far off | Close |
| Iterations | Many | Few |
| Result | Estimate only | Deterministic sizing from gain, bandwidth and power targets |
