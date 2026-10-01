# Quadstor

**Quadstor** is a modular, four-lane charger for **4S Li-ion, LiPo, and LiHV packs**, designed around the simple, balance-connector-only workflow of small FPV battery chargers such as Whoopstor and Toothstor. It is currently a **hardware/firmware design project**, not a validated or production-ready charger.

This README captures the intended product, established design decisions, explored alternatives, and unresolved engineering work so contributors and coding agents can pick up the project without reconstructing earlier discussions.

> **Status (30 September 2026):** The **power-source selection board schematic is drawn** (`Hardware/Power Board/Quadstor Power Board/`, 19 parts). Its parts are synced to a 45 × 45 mm, 2-layer PCB outline, but nothing is routed yet. A netlist review found **issues to fix before layout**, listed in [§3 Power board schematic](#power-board-schematic-as-drawn-2026-09-30). The project is a git repository published at [github.com/ItsReckliss/Quadstor](https://github.com/ItsReckliss/Quadstor). A **single-lane prototype** is the next hardware milestone before tiling the design four times. **BQ25792 (charge IC) is selected for the single-lane prototype**, with the **STM32G0B1 (microcontroller)** as the MCU. The BQ25713 (original charge controller) is kept as historical context and a fallback. No schematic, tested charging system, or firmware exists yet. Dated decisions are recorded in [§12 Decision log](#12-decision-log); datasheets are in `datasheets/`.

### Key parts at a glance

Parts are referred to by part number throughout; this table gives each one's job.

| Part | Purpose | Status |
|---|---|---|
| TI **BQ25792** | Charge IC: buck-boost battery charger with integrated FETs, one per lane | Selected for prototype |
| TI **BQ25713** | Charge controller: buck-boost charger needing external MOSFETs, one per lane | Superseded; fallback |
| ST **STM32G0B1** | Microcontroller (MCU): control, cell-tap ADC, UI, USB | Selected |
| TI **TCA9548A** | I²C multiplexer: gives each lane's charge IC its own bus (fixed-address workaround) | Selected |
| TI **TPS54160A** | Buck regulator: makes the logic supply (5 V/3.3 V) from the 24–25 V input | Selected |
| SSD1306-class OLED, 0.92" | Display (prototype): header row + one lane panel | Selected (exact module TBD) |
| Toshiba **TPHR8504PL** | Power MOSFET: BQ25713 switching stage (Q1–Q4) | BQ25713 design only |
| **AONR21307** | P-MOSFET: BATFET for the BQ25713 design | BQ25713 design only |
| **AO3401A** | P-MOSFET: bleed switch for cells 2–4 (high-side, cell-referenced) | Proposed (owner suggestion); datasheet in repo |
| **AO3400A** | N-MOSFET: bleed switch for cell 1 (ground-referenced) | Proposed |
| **2N7002** | Small N-MOSFET: level-shifting gate driver for each AO3401A | Proposed |
| BZT52C5V1 (5.1 V Zener) | Gate-source clamp protecting each AO3401A's ±12 V gate rating | Proposed |
| **NCE6003X** / **NTD5865NLT4G** | N-MOSFET: earlier bleed-switch candidates | Superseded by AO3401A/AO3400A proposal |
| TLV755P / AP2112K | LDO regulator: 5 V → clean 3.3 V | Proposed |
| 0603 10 kΩ NTC (B ≈ 3380–3435 K) | Temperature sensor on the charge IC's TS pin, next to the JST-XH | Decided (exact part TBD) |
| USBLC6-2SC6 | USB ESD protection on D+/D− | Proposed |
| TI **CD74HC4067SM96** | 16:1 analog multiplexer (SSOP-24, tape and reel): routes the 16 divided tap voltages to one MCU ADC pin | Selected (datasheet not yet in `datasheets/`) |
| **TLV9001** / **MCP6001** | Op-amp: unity-gain buffer between the analog mux and the ADC | Proposed |
| Tag-Connect TC2030 | Connector-less SWD programming/debug footprint | Proposed |
| Mean Well 24 V / 200 W | Internal power supply (model TBD) | Planned |
| **AOD409** (×3) | P-MOSFET, TO-252: power-board source switches (Q1 PSU path; Q2/Q3 back-to-back on the 6S path) | In power-board schematic |
| **2SA1213-Y** | PNP transistor, SC-62: power board, pulls the 6S FET gates off when the PSU is present (Q4) | In power-board schematic |
| **AO3400A** (power board) | N-MOSFET: power board PSU-present detector that drives Q4 (Q5) | In power-board schematic |
| **BZT52C12V** | 12 V Zener: power board gate-source clamp for the 6S FETs (U1) | In power-board schematic |
| **SMCJ30A-TR** (×2) | TVS diode, 30 V standoff: power board surge clamp on each input (D1 PSU, D2 6S) | In power-board schematic |
| Littelfuse **01550900M** (×2) | SMD fuse: power board, one per input (FH1 PSU, FH2 6S) | In power-board schematic; rating to confirm |

## 1. Product goals

- **Four independent 4S charging lanes**, each with a 5-pin JST-XH balance connector. Packs can be inserted/removed independently without disturbing other lanes.
- **Balance-connector-only charging:** the outer connector pins carry pack charging current; intermediate pins provide cell taps. No separate XT30/XT60 lead is required for the battery being charged.
- **User-selectable charging current:** 0.1–2.0 A per lane in 0.1 A steps. Earlier discussions refer both to a global current setting and per-lane adjustment; the UI behavior should be finalized in firmware requirements. The electrical architecture must support independently controlled lanes.
- **Global per-cell target voltage presets:** 3.80 V, 3.85 V, 4.20 V, and 4.35 V, corresponding to 15.20 V, 15.40 V, 16.80 V, and 17.40 V for a 4S pack. The 4.35 V setting is **LiHV only** and must not be applied to conventional 4.20 V/cell packs.
- **Passive cell balancing**, individual cell-voltage measurement, pack monitoring, and safe fault handling on every lane.
- Compact **OLED, two-button, and beeper** interface. The UI uses a **single OLED**. The prototype uses a small 0.92" OLED showing a **header** (input voltage, target voltage, set current) and **one lane panel**. The final version uses a larger single OLED with that lane panel **tiled four times** and a reorganized header (see §7).
- Operate from an **internal 24 V / 200 W Mean Well PSU** or an external **6S battery via XT60**, automatically preferring the PSU when available.
- Modular design: develop, assemble, and characterize one complete lane, then replicate it four times.

## 2. System overview

```text
                     Internal 24 V / 200 W PSU
                                |
External 6S pack -> XT60 -> input protection / source selection
                                |
                   Protected, selected VIN bus
                                |
                   Bulk decoupling near lanes
                                |
            +-------------------+-------------------+
            |                   |                   |
       Charger lane 1      Charger lane 2      ... lane 4
       4S JST-XH           4S JST-XH              4S JST-XH
       Cell taps           Cell taps              Cell taps
       4 bleed paths       4 bleed paths          4 bleed paths
            |                   |                   |
       I2C mux ch0         I2C mux ch1            I2C mux ch3
            +-------------------+-------------------+
                                |
                      TCA9548A I2C mux
                                |
        STM32G0B1: ADC (cell taps, VSENSE) / GPIO / USB
                                |
   Single OLED (header + lane panels) / 2 buttons / buzzer / LEDs / USB-C / SWD

Logic power: selected VIN -> TPS54160A buck -> 3.3 V (optionally 5 V -> LDO -> 3.3 V)
```

Each lane has its own charger power stage, pack sensing, balancing, and charging state. The **tiled hardware lane unit** is: BQ25792 charge IC + its inductor/capacitors, JST-XH, cell-tap sensing, four bleed paths, and NTC. The MCU, regulators, I²C mux, the single OLED, buttons, buzzer, USB-C, and SWD are **shared and are not tiled**. Tiling of the UI happens in **firmware**: one lane panel drawn four times on the one display (§7).

The BQ25792 charge IC has a fixed 7-bit I²C address (**0x6B**), so four of them cannot share one bus. A **TI TCA9548A** (8-channel I²C switch/multiplexer) gives each lane's charge IC its own downstream channel. The proposed channel map is ch0–ch3 for lanes 1–4 and ch4–ch7 spare. The single OLED (SSD1306-class, typically 0x3C) sits on the upstream bus, or on a spare channel if needed. If the final larger display uses SPI, it connects directly to the MCU instead. Tie A0–A2 low (address 0x70), connect RESET to an MCU GPIO to clear stuck buses, and fit pull-ups on the upstream bus and on each used downstream channel. The TCA9548A runs from 1.65–5.5 V at up to 400 kHz. This channel map is **proposed** until it is drawn in a schematic.

## 3. Power input and source selection

### Intended inputs

| Input | Nominal | Relevant range / notes |
|---|---:|---|
| Internal Mean Well PSU | 24 V, 200 W | Default source when present. Exact PSU model not yet recorded here. |
| External field battery | 6S Li-ion/LiPo via XT60 | Up to **25.2 V** for standard 4.20 V/cell 6S. Verify any LiHV input separately. |

The separately developed **power-selection board** accepts `VBAT`, `GND`, `GND`, and `VPSU`, and provides `VOUT`, `GND`, and `VSENSE`. `VSENSE` is intended to feed an MCU ADC through a resistor divider. The selected output feeds the charger board directly, with additional local bulk and high-frequency bypass capacitors at each charger IC.

**Confirmed by the owner (2026-09-29):** the existing power board already **handles source selection**, **electrically disconnects the XT60 input when the PSU is on**, and **blocks backfeed**. The charger lanes therefore do not need their own source selection or reverse-current blocking. ~~**Open item:** confirm whether the power board has a **TVS and/or soft-start**~~ **Answered by the schematic (2026-09-30):** both inputs have an SMCJ30A TVS diode, but it clamps far above the BQ25792 charge IC's 30 V absolute maximum, and there is no dedicated soft-start. See the review below. **Review note:** the drawn schematic does **not** block backfeed into the PSU (finding 2 below), so the owner's earlier confirmation needs to be rechecked.

Discussed input protection/components:

- 15 A fuse on the power input path (final placement/rating must be verified).
- Input TVS diode; proposed 100 µF bulk capacitor and 1 µF bypass capacitor.
- Back-to-back P-channel MOSFETs on the XT60 path for source isolation/reverse-current blocking.
- PSU-priority behavior: when PSU is present, block the external battery from supplying the bus; use the external battery when PSU is absent.
- `VSENSE` divider for firmware supply-voltage monitoring. Confirm which node it measures in the current schematic.

### Power board schematic (as drawn, 2026-09-30)

The KiCad 10 project is `Hardware/Power Board/Quadstor Power Board/`. Its parts were imported from LCSC with the EasyEDA importer into the project-local `EasyEDA.kicad_sym` / `EasyEDA.pretty` libraries. All 19 footprints are on a **45 × 45 mm, 2-layer** board outline (imported from DXF), with no tracks or copper zones yet.

| Ref | Part (LCSC) | Function |
|---|---|---|
| FH1, FH2 | Littelfuse 01550900M SMD fuse (C108518) | Input fuses: FH1 on `VPSU`, FH2 on `VBAT` (6S/XT60). Current rating not yet recorded. |
| D1, D2 | SMCJ30A-TR TVS diode (C1975473) | Surge clamps: D1 on `VPSU_Protected`, D2 on the fused 6S node |
| C1, C2 | 1 µF 100 V X7R 1206 (C91151) | C1 on `VPSU_Protected`, C2 on `VOUT` |
| Q1 | AOD409 P-MOSFET (C36220) | PSU path switch: source `VPSU_Protected`, drain `VOUT`. Gate is biased by the R1/R2 divider to Vgs ≈ −VPSU/2. |
| Q2, Q3 | AOD409 P-MOSFET (C36220) | 6S path, back-to-back with common sources. Q2 drain on the fused battery, Q3 drain on `VOUT`. |
| U1 | BZT52C12V 12 V Zener (C5182293) | Clamps the 6S FETs' gate (`VBAT_Gate`) to 12 V below their common source |
| Q4 | 2SA1213-Y PNP transistor (C396014) | When on, shorts `VBAT_Gate` to the common source, turning the 6S path off |
| Q5 | AO3400A N-MOSFET (C20917) | PSU-present detector: gate from the R6/R7 divider on `VPSU_Protected`; pulls Q4's base low through R4 |
| R1–R7 | Labeled "220k"; library part is RC0402FR-071KL (C106235) | Gate/bias dividers and pull-downs (see finding 3) |

**Intended operation:** with the PSU present, Q1 conducts and Q5 turns Q4 on, which holds the 6S FETs off, so the PSU has priority. With the PSU absent, Q5 and Q4 are off, and R3 pulls `VBAT_Gate` low (clamped by U1), turning Q2/Q3 on. Back-to-back Q2/Q3 block current in both directions when off.

Not yet in the schematic: input/output **connectors** (XT60, PSU input, output to the charger board; the nets are global labels only), the `VSENSE` divider, and the duplicate GND pins of the board interface.

**Review findings (Claude, from a netlist exported with kicad-cli on 2026-09-30; verify before layout):**

1. **Q2's gate is unconnected** (ERC net `unconnected-(Q2-G-Pad1)`). A third `VBAT_Gate` label exists but isn't attached. Without the gate, Q2 conducts only through its body diode or turns on unpredictably, and the 6S path can't block reverse current. Connect Q2's gate to `VBAT_Gate`.
2. **Q1 doesn't block backfeed into the PSU, and the PSU-detect can be fooled.** Q1's body diode conducts from drain to source (`VOUT` → `VPSU_Protected`). When running from the 6S pack with the PSU off, the pack feeds the PSU output through that diode. That also lifts `VPSU_Protected` to about VOUT − 0.7 V, which turns Q5 → Q4 on and shuts the 6S path off. The likely result is the battery path oscillating or failing to stay on. Fixing this needs a PSU-side switch that blocks in both directions (for example a back-to-back pair or an ideal-diode controller), or a PSU-present signal taken upstream of anything `VOUT` can back-drive.
3. **The resistor value doesn't match the part.** All seven resistors say **220k**, but their library part and LCSC number are **RC0402FR-071KL, which is 1 kΩ**. A BOM ordered by LCSC number would fit 1 kΩ. R1/R2 would then draw ~12 mA from 24 V and dissipate ~0.14 W each, above a 0402's typical 1/16 W rating. Pick a real 220 kΩ 0402 part and LCSC number.
4. **The TVS doesn't protect the BQ25792 charge IC's 30 V absolute maximum.** The SMCJ30A has a 30 V standoff, 33.3 V minimum breakdown, and ~48 V clamping at rated current (typical datasheet values). It catches large surges but lets `VOUT` exceed 30 V during a spike. This matches the reasoning behind "no per-lane TVS" in §4. Hot-plug ringing still needs scoping, and possibly a lower-voltage clamp or a soft-start/inrush limiter.
5. **Q5's gate margin is thin.** The R6/R7 divider gives VGS = VPSU/2, which is 12 V at 24 V. The AO3400A's gate rating is ±12 V (verify; datasheet not in repo), so a Mean Well PSU trimmed up, or a spike, overstresses it. Use a lower divider ratio (for example 220k over 100k) or a Zener clamp.

**Important:** Do not assume the power board provides a regulated 24 V output. The charger must tolerate the actual selected input voltage and transients. Absolute maximum is **not** a recommended continuous design target. Check the chosen IC's recommended operating range, transient headroom, TVS coordination, and hot-plug behavior before powering a prototype.

**BQ25792 (charge IC) input-voltage findings (datasheet SLUSDG1C, checked 2026-09-29):**

| Parameter | Value | Implication |
|---|---|---|
| VBUS/VAC absolute maximum | 30 V | A 25.2 V 6S pack will not damage the IC in steady state. Hot-plug ringing must stay below 30 V. That is the power board's job (TVS/soft-start still to be confirmed), because a per-lane TVS cannot do it (see §4). |
| VBUS recommended operating range | 3.6–24 V | The 24 V PSU is at the limit and acceptable. A full 6S pack (25.2 V) is ~1.2 V above the recommended range. |
| VAC OVP, highest setting (`VAC_OVP[1:0]=00`, default) | Rising: 25.2 / 26.0 / 26.8 V (min/typ/max). Falling: 24.4 / 25.2 / 26.0 V | A low-tolerance part could trip OVP on a fully charged 6S pack and stay off until the input falls below ~24.4 V. The expected result is a lane refusing to start or dropping out, not damage. |

**Decision (2026-09-29):** accept 6S direct input for the prototype and characterize it. Test with a freshly charged 6S pack, and scope VBUS during XT60 hot-plug. Firmware should detect and report the VAC OVP fault and retry. If lanes trip in practice, the options are a ~1 V drop on the 6S path (a diode, or preferably an ideal-diode controller for lower loss), a pre-regulator, or limiting the field input to 5S (21 V).

The proposed 200 W PSU is nominally sufficient for the planned four-lane output power under ideal conditions (4 × 17.4 V × 2 A = **139.2 W** at the maximum LiHV target), but input power, losses, balancing dissipation, wiring limits, and thermal derating must be measured.

## 4. Charger lane: baseline and alternative

### Original baseline: TI BQ25713 (charge controller)

The original architecture uses **one BQ25713 buck-boost charger IC per lane**, with an external switching power stage, an external BATFET, and MCU-controlled passive balancing. This is the historical baseline, not proof that the current schematic still uses it.

| Function | Discussed component | Quantity per lane | Status |
|---|---|---:|---|
| Buck-boost charger controller | TI **BQ25713** | 1 | Original baseline |
| Main switching MOSFETs Q1–Q4 | Toshiba **TPHR8504PL** N-MOSFET | 4 | Selected for original design |
| BATFET | **AONR21307** P-MOSFET | 1 | Selected/proposed; validate exact suffix and pinout |
| Passive balancing switches | **NCE6003X** N-MOSFET | 4 | Proposed; other candidates considered |
| Balance bleed resistors | **110 Ω, 1210, 0.5 W** | 4 | Proposed; requires thermal validation |
| Battery connector | **5-pin JST-XH** | 1 | Required interface |
| Cell sensing | MCU ADC with suitable front-end | 4 cell voltages derived from taps | Architecture only; protection/circuit not finalized |

The uploaded TPHR8504PL (power MOSFET) datasheet specifies a **40 V** drain-source rating and typical `RDS(on)` of **0.7 mΩ at VGS = 10 V** (0.85 mΩ max under its stated test conditions). It has substantial gate charge (103 nC typical at 10 V in the datasheet's stated test setup). Validate compatibility with the selected charger gate drivers, switching frequency, layout, and thermal design; low on-resistance alone does not establish suitability. The exact package variant/footprint must match the ordered device.

The **NTD5865NLT4G** was also investigated as a possible balancing MOSFET; it is **not** established as the final selection.

### Prototype selection: TI BQ25792 (charge IC)

**Decision (2026-09-29):** the single-lane prototype uses the **BQ25792**, a buck-boost charger with **integrated switching FETs** (datasheet in `datasheets/BQ25792_datasheet.pdf`, rev SLUSDG1C). It replaces the BQ25713's external Q1–Q4 stage, so the TPHR8504PL switches and the AONR21307 BATFET in the table above do **not** apply to this design. Never mix the two architectures' BOMs.

Datasheet facts relevant to Quadstor:

| Item | Datasheet value | Notes |
|---|---|---|
| Charge voltage (VREG) | 3.0–18.8 V, 10 mV steps | Covers all presets, including 17.40 V LiHV. |
| Charge voltage accuracy | +0.65% / −0.85% | At 17.40 V, +0.65% is +113 mV (≈4.378 V/cell average). Firmware must use per-cell measurements, not rely on VREG alone, to keep LiHV cells ≤4.35 V. |
| Charge current (ICHG) | 50 mA–5 A, 10 mA steps | Covers 0.1–2.0 A. Low-current accuracy is worse; for example, ±7.5% at 0.5 A. |
| Input current limit (IINDPM) | 0.1–3.3 A | Also limited by the ILIM_HIZ pin divider (V = 1 V + 0.8 × IINDPM). |
| Input voltage | See §3 | 24 V recommended max, 30 V abs max. |
| I²C | Fixed 7-bit address 0x6B | Requires the TCA9548A mux (§2). |
| Built-in ADC | VBUS, IBUS, VBAT, IBAT, TS, die temperature, and more | Lets the MCU read pack voltage/current and NTC temperature through I²C, which saves MCU ADC channels. |
| REGN | ~5 V LDO, 30 mA limit, needs 4.7 µF | For the IC's gate drive and TS bias only. **Do not** power the MCU/logic from it. |

Per-lane pin handling (proposed; verify against the datasheet when drawing the schematic):

- **PROG:** 17.4 kΩ selects 4S at 1.5 MHz (use a ~1 µH inductor); 27.0 kΩ selects 4S at 750 kHz (use a ~2.2 µH inductor). Choose based on efficiency and thermal testing.
- **VAC1/VAC2:** connect to VBUS, and tie **ACDRV1/ACDRV2** to GND. The power board does source selection, so the BQ25792's dual-input FETs are not fitted.
- **SDRV:** tie to GND or BAT because no ship FET is used.
- **CE (active low):** drive from an MCU GPIO with a **pull-up**, so charging is disabled while the MCU is in reset. It must never float.
- **INT and STAT:** open-drain outputs; use 10 kΩ pull-ups to 3.3 V and route to the MCU. INT is needed for fault handling. STAT is optional.
- **TS / temperature sensing (decided 2026-09-29):**
  - Fit **5.23 kΩ from REGN to TS** and **30.1 kΩ from TS to GND**, the TI typical-application divider in nearest E96 values. This gives the default window: charging suspends at ~0 °C and ~60 °C, with JEITA reductions near the edges.
  - Fit a **0603 10 kΩ NTC from TS to GND**. Use B ≈ 3380–3435 K; the datasheet recommends the 103AT-2 at B = 3435. 5% tolerance is acceptable.
  - Place the NTC **right next to the JST-XH connector**. It measures connector/board temperature, **not** the pack.
  - TS must **never float**. If temperature sensing is ever dropped, replace the NTC with a fixed 10 kΩ resistor.
- **D+/D−:** USB adapter detection is not used; follow the datasheet for unused handling.
- **OTG (reverse boost) mode (decided 2026-09-29):** the BQ25792 charge IC can boost the battery back out to VBUS. Firmware must **never enable OTG**, and must **read back that OTG is disabled** after configuring the IC, so a pack can never drive the input bus.
- **SYS:** the BQ25792 is a power-path (NVDC) charger, so SYS is powered from the pack when the input is removed. Fit only the required capacitors on SYS, with no other loads.
- **Input protection per lane (decided 2026-09-29):**
  - **No per-lane TVS.** The standoff would have to exceed 25.2 V (full 6S), and such a TVS clamps around 40 V, which is above the 30 V absolute maximum. Surge protection belongs on the power board (§3).
  - Add a **do-not-populate (DNP) footprint** for the datasheet's **input RC snubber (~2 Ω + 2.2 µF) on VBUS**. Fit it only if ringing shows up on the bench.
  - **Input capacitors:** VBUS gets 1× 0.1 µF + 2× 10 µF; PMID gets 1× 0.1 µF + 3× 10 µF. All are **50 V X7R**, derated for the 25.2 V input because ceramics lose capacitance under DC bias.
- **Other passives (datasheet values; ratings proposed):** SYS 1× 0.1 µF + 5× 10 µF (≥25 V, 35 V preferred at 17.4 V); BAT 2× 10 µF (≥25 V); BTST1 and BTST2 47 nF each; REGN 4.7 µF (10 V).

The BQ25713 charge-controller design remains a fallback if the BQ25792 charge IC fails characterization (input range, thermals, or accuracy). Switching back requires redesigning the power stage, not a part swap.

## 5. Battery connector and cell measurement

Each output is a **five-pin JST-XH connector for one 4S pack**. Number the contacts by **electrical function**, not assumed footprint orientation:

| Electrical node | Function |
|---|---|
| `B-` | Pack negative; charger negative connection |
| `B1` | Junction between cells 1 and 2 |
| `B2` | Junction between cells 2 and 3 |
| `B3` | Junction between cells 3 and 4 |
| `B+` | Pack positive; charger positive connection |

With taps referenced to `B-`, compute cell voltages as `Vcell1 = VB1`, `Vcell2 = VB2 - VB1`, `Vcell3 = VB3 - VB2`, and `Vcell4 = VB+ - VB3`.

**The MCU cannot directly connect upper-cell taps to ordinary 3.3 V ADC pins.** Provide properly rated dividers/buffers or a suitable multi-cell monitor, input protection, filtering, accurate reference/calibration, and safe handling of missing/intermittent contacts. Check voltage differences and resistor tolerances at every tap. Keep measurement input current and connector contact resistance in mind when charging through the same plug.

**Measurement architecture (decided 2026-09-29): an analog multiplexer feeding one MCU ADC pin.** The ADC is ground-referenced, so the system measures **tap voltages**, not cells directly. Per lane, the four taps B1, B2, B3, and B+ are measured, and cell voltages are computed by subtraction as above. Four lanes × 4 taps = **16 channels, one 16:1 analog mux**, with 4 MCU select GPIOs and 1 ADC input. `VSENSE` keeps its own ADC pin.

- **Pack voltage:** the B+ tap *is* the pack voltage, so no extra channel is needed. The BQ25792 charge IC also reports VBAT over I²C, which serves as an **independent cross-check**. A mismatch between the two ADCs should raise a fault.
- **Signal chain, per tap:** divider (0.1%) → series resistor + clamp → mux input. After the mux: a **unity-gain op-amp buffer** → MCU ADC.
  - Divide **before** the mux, because common muxes only pass signals within their 3.3–5 V supply.
  - The buffer isolates the divider's high source impedance and the mux on-resistance from the ADC sampling capacitor.
- **Settling:** allow settling time after each channel switch (tens of µs; confirm on the bench). Pause balancing on a lane while reading its taps.
- **Unpowered safety:** a pack can be connected while the board is unpowered, so the series resistors must limit injection current into the unpowered mux.
- **Standby drain:** the dividers drain packs left connected (e.g. ~170 µA at 100 kΩ total). Use high-value dividers (then the buffer is mandatory) or switch the dividers off when idle.
- **Accuracy:** all taps share one mux, buffer, and ADC path, so the offset is common. Differencing still amplifies divider error: 0.1% at ~13–17 V is roughly ±13–17 mV per tap. Plan two-point calibration per channel.
- **Candidate parts (proposed):**
  - **Selected (2026-09-29): TI CD74HC4067SM96** (16:1 analog multiplexer, SSOP-24, 0.65 mm pitch; the SOIC-24 variant CD74HC4067M96 is electrically identical).
    - Power it from 3.3 V (same as the ADC). Signals must stay within GND to VCC.
    - Tie E (active-low enable) to GND or to a GPIO.
    - On-resistance is ~70 Ω typical at 4.5 V and higher at 3.3 V. The buffer makes this irrelevant.
    - **Off-leakage × divider Thevenin resistance is an error source:** e.g. 1 µA × 25 kΩ = 25 mV at the ADC, ≈175 mV referred to a 17 V tap. Check the datasheet's leakage over temperature when choosing divider values.
  - Alternative: **ADG706** (Analog Devices, 16:1 analog multiplexer, ~2.5 Ω, lower leakage).
  - **TLV9001** or **MCP6001** (rail-to-rail op-amp) as the buffer.
- **Single-lane prototype:** fit the full 16:1 mux even though only 4 channels are used, to validate the four-lane measurement chain.

Tap front-end details still to design: divider values scaled to the ADC reference (for example B+ at 17.4 V max scaled to ≤2.5 V), clamp type, filter capacitor, and open/missing-tap detection (such as a weak bias that drives an unconnected tap to an implausible reading). A dedicated multi-cell monitor IC per lane remains an alternative if accuracy proves insufficient.

## 6. Passive balancing

The proposed architecture has **four independent bleed paths per lane**, one across each cell, each controlled by an external MOSFET under MCU supervision. The original candidate network is **110 Ω / 0.5 W (1210)** per cell, switched using NCE6003X MOSFETs (or a validated equivalent).

Approximate ideal resistor dissipation and bleed current:

| Cell voltage | Current through 110 Ω | Resistor dissipation |
|---:|---:|---:|
| 3.80 V | 34.5 mA | 0.131 W |
| 4.20 V | 38.2 mA | 0.160 W |
| 4.35 V | 39.5 mA | 0.172 W |

These values are estimates before switch losses and component tolerances. Verify each bleed path's **gate-drive voltage relative to its cell**, especially upper cells; do not connect high-side cell-referenced gates directly to ground-referenced MCU GPIOs. Upper-cell switches sit up to ~13 V above ground, so they need level shifting.

**Proposed bleed-switch circuit (2026-09-29).** The owner suggested the AO3401A (P-MOSFET). Its figures below were checked against `datasheets/AO3401A.pdf` (AOS Rev 3.1, Dec 2023) on 2026-09-30.

- **Cells 2–4 (upper cells):**
  - The **AO3401A** source goes to the cell's top node (B1/B2/B3/B+), and its drain goes through the 110 Ω bleed resistor to the cell's bottom node.
  - A 100 kΩ gate-to-source pull-up holds it off.
  - A **2N7002** N-MOSFET, gate driven by an MCU GPIO with a 100 kΩ pull-down, pulls the gate toward GND through ~47 kΩ.
  - A **5.1 V Zener** from gate to source clamps V_GS to about −5 V.
  - Why this is needed: the AO3401A's V_GS limit is **±12 V**, so the gate must never be pulled straight to GND. On the top cell that would apply ~−17 V.
  - AO3401A (datasheet): −30 V V_DS, ±12 V V_GS, 47 mΩ typical / 60 mΩ max at V_GS = −4.5 V, V_GS(th) −0.5 / −0.9 / −1.3 V (min/typ/max), SOT-23. At 40 mA the conduction loss is negligible.
  - Off-state leakage I_DSS is at most 1 µA at 25 °C (5 µA at 55 °C) per switch. This is small next to the tap-divider drain (§5) but adds to it on a stored, plugged-in pack.
- **Cell 1 (bottom cell):** B− is GND, so an **AO3400A** N-MOSFET is driven directly from a GPIO, with a 100 kΩ pull-down.
- **Why not N-channel on the upper cells:**
  - An upper-cell N-FET's source sits at that cell's bottom node (up to ~13 V). Its gate would need to be driven to about the cell's top node, which requires a high-side (P-type) switch plus its own driver. That is two driver transistors per upper cell.
  - A P-FET with its source on the cell's top node turns on by pulling its gate *down*, which one ground-referenced 2N7002 can do.
  - Totals per lane: 7 transistors with the proposed P-channel scheme, versus 10 for all N-channel.
- **Alternative:** one PhotoMOS solid-state relay per cell (for example the Panasonic AQY21x class). It needs no level shifting and fails off, but costs more (~$1–2 each) and is larger.
- **Fail-safe:** with the GPIO low, floating, or in reset, every bleed switch is off.
- **Per lane:** 3× AO3401A, 1× AO3400A, 3× 2N7002, 3× 5.1 V Zener.
- **To validate:** off-state leakage (a continuous drain on a stored, plugged-in pack), and tap-reading error while bleeding. Firmware should pause bleeding during cell measurements.

Four lanes × 4 bleed paths = **16 MCU GPIOs**, and every gate must have a pull-down or equivalent so all bleeders are **off during reset**. Ensure balancing cannot remain latched on after MCU reset and that balancing and charging decisions account for measured cell voltages, thermal limits, and the relatively low ~40 mA bleed rate compared with up to 2 A charging current.

## 7. MCU, firmware, and controls

### MCU: STM32G0B1 (microcontroller)

**Decision (2026-09-29):** STM32G0B1 (Cortex-M0+, 64 MHz; datasheet in `datasheets/STM32G0B1_datasheet.pdf`). It was chosen for its 16-channel 12-bit ADC, crystal-less USB 2.0 FS device, and ROM bootloader.

- **Package (proposed): LQFP64 (STM32G0B1RET6).** With the analog mux, the four-lane pin budget is 1 ADC for taps + 4 mux selects + 1 ADC for `VSENSE`, 16 balance GPIOs, 4 × CE + 4 × INT (optionally 4 × STAT), I²C, TCA9548A (I²C mux) RESET, USB D+/D−, SWD, 2 buttons, buzzer PWM, status LEDs, and possibly SPI for the final display. That is too tight for a 48-pin package. Use the same part on the single-lane prototype so firmware carries over.
- **Support circuitry:** 100 nF at every VDD pin plus 4.7 µF bulk; VDDA filtered with a ferrite bead, 1 µF, and 100 nF; 100 nF on NRST. Use VREF+ with the internal VREFBUF (2.048/2.5 V) or an external reference for ADC accuracy. No HSE crystal is needed, because USB uses HSI48 with clock recovery.
- **Safe defaults:** all charge-enable and balance outputs must default to off through external pulls, independent of firmware (see §6 and §8).

### Logic power

**Decision (2026-09-29):** a **TI TPS54160A** (3.5–60 V in, 1.5 A, non-synchronous buck; datasheet in `datasheets/tps54160a.pdf`) generates the logic supply from the selected VIN. An LDO from ~25 V was rejected because it would dissipate ~1 W at only 50 mA.

Design notes:

- **Switching frequency:** the minimum on-time is 130 ns typical. For 3.3 V output from up to 30 V in, f_SW(max) ≈ (3.3 + 0.5) / (30 V × 130 ns) ≈ 950 kHz. Use **~500–700 kHz**.
- **Catch diode:** requires an external Schottky from PH to GND, rated ~60 V / 1 A.
- **EN UVLO:** use a divider on EN so the regulator does not start until the input reaches roughly 8 V or more.
- **Components:** use 50 V-rated input capacitors, and compute compensation and the inductor with TI WEBBENCH.
- **Load:** the logic load is expected to be ≤ ~150 mA (MCU, OLEDs, mux, sensing), far below the 1.5 A rating.
- **Proposed topology:** buck to **5 V**, then a small low-noise LDO regulator (for example TLV755P or AP2112K) to 3.3 V. This gives a quieter ADC supply, a 5 V rail for the buzzer, and allows USB VBUS to be diode-ORed into the 5 V rail (see below). Buck directly to 3.3 V is acceptable if that is not wanted.

### Programming, USB, and diagnostics

**Decision (2026-09-29):** a **USB-C port on the controller/lane board** is for **flashing (USB DFU) and diagnostics (USB CDC virtual COM port)**. It is **not** a charging power input, since USB cannot supply the charging power, and it does not belong on the power board.

- **USB-C device wiring:** 5.1 kΩ Rd from each CC pin to GND; USB ESD protection (for example USBLC6-2SC6) on D+/D−; D+/D− to PA12/PA11.
- **Optional:** USB VBUS → Schottky → 5 V logic rail. This lets the MCU, OLEDs, buttons, buzzer, and USB be brought up with only a USB cable while the chargers stay unpowered, because their VBUS comes only from the power board. Ensure VBUS is never back-fed.
- **DFU entry:** a blank device boots the ROM bootloader automatically. Once flash is programmed, DFU entry needs either a firmware "jump to bootloader" command or the BOOT0 pin. On STM32G0 the BOOT0 function shares PA14 with SWCLK and is governed by option bytes (`nBOOT_SEL`/`nBOOT0`). Verify in RM0444 and AN2606 before relying on a BOOT button.
- **Buttons:** a **RESET button** (NRST to GND) and a **BOOT button** (PA14 to 3.3 V with a 10 kΩ pull-down) are planned for the prototype.
- **SWD is kept.** It is needed for debugging (breakpoints) and for recovery when bad firmware breaks USB. Earlier plans called for a JST-XH SWD header; the updated plan is a **Tag-Connect TC2030 footprint or bare pads** (3V3, SWDIO, SWCLK, NRST, GND) on both prototype and production, with no connector BOM cost. An optional UART TX test pad is useful for early logging.

### Intended UI

**Decision (2026-09-29):** the UI parts (OLED, buttons, buzzer, status LEDs) go on the single-lane prototype board so they can be verified before the four-lane build.

- **Single OLED (decided 2026-09-29):** there is exactly **one** display, never one per lane.
  - **Prototype:** a small **0.92" OLED**. Most modules this size are 128×32 SSD1306 I²C (display controller); confirm the actual resolution and controller of the purchased module. Connect it with a 4-pin header (VCC, GND, SCL, SDA) rather than a bare glass panel with FPC and charge-pump capacitors.
  - **Final:** a **larger single OLED** (part TBD). The prototype's lane panel is **tiled four times** on it, and the header is added or reorganized to fit.
- **Header row:** shows **input voltage, target voltage, and set current**, and may also show the power source and mute state if space allows. For example, `24.1V 4.20V 2.0A` is 16 characters, about 96 px in a 6 px-wide font, so it fits in 128 px.
- **Lane panel (the tiled unit in firmware):** connection/charging state, pack and cell voltages, charge current, and fault or completion. Implement it as a **self-contained, fixed-size render component** that takes a lane index and an (x, y) origin, so the final display simply draws it four times.
- **Proposed layout (not decided):**
  - On a 128×32 prototype display, use an **8 px header row + a 128×24 lane panel**.
  - A final **256×64** display (for example the SSD1322 class, commonly 2.8"–3.12") would fit the 8 px header plus four 128×24 panels in a 2×2 grid (8 + 48 = 56 px ≤ 64 px) without redesigning the panel.
  - Larger 256×64 panels are often SPI and 4-bit grayscale, which needs a different driver than the SSD1306. Keep SPI-capable MCU pins available.
- **Refresh budget:** a full 128×32 SSD1306 frame is 512 bytes, ~13 ms at 400 kHz I²C. A 256×64 grayscale frame is 8 KB, which is why the final display should use SPI and/or partial updates.
- **Two physical buttons:** tactile switches to GND with pull-ups and an optional 100 nF debounce, on interrupt-capable pins. One combines **Start/Stop + Current** and the other **Mute + Voltage**, using click/hold interactions. Final click/hold mappings, per-lane selection, and display layout need a firmware specification.
- **Beeper:** a passive magnetic buzzer driven from a timer PWM output (allowing distinct tones) through an N-MOSFET/NPN from the 5 V rail, with a flyback diode. It gives status, completion, and error notifications with mute support.
- **Status LEDs:** one power LED on 3.3 V plus one or two GPIO LEDs (~1 kΩ) for heartbeat and fault, useful before the OLED works.

The recorded product goals include a global current setting **and** a requirement to independently adjust each lane's current; this needs a deliberate UI decision rather than an assumed implementation. The planned header display shows one "set current," which implies a **global setpoint by default**. Whether per-lane overrides exist, and how they are shown in the lane panels, is still open. At minimum, the charger hardware and firmware should independently control and stop each lane.

### Suggested firmware modules (proposed, not existing code)

```text
firmware/
  board/             # GPIO, ADC, I2C, timer, and peripheral definitions
  power/             # VIN measurement and source status
  charger/           # Charger-IC abstraction, one instance per lane
  sensing/           # Tap acquisition, calibration, cell-voltage calculation
  balancing/         # Per-cell bleed control and safeguards
  safety/            # Fault detection, limits, watchdog, safe shutdown
  ui/                # OLED pages, button events, buzzer
  app/               # Global settings and four independent lane state machines
```

A sensible lane state machine to implement and validate is `DISCONNECTED -> DETECTED -> VALIDATING -> CHARGING -> COMPLETE`, with independently reachable `FAULT` and `STOPPED` states. This is a **proposed software design**, not an already implemented protocol.

Firmware must not enable charge solely because pack voltage appears plausible. Validate all cell taps and their ordering, detect absent/open wires and impossible voltages, configure and read back charger limits, check charger faults, and independently stop a lane when unsafe. Implement watchdog and brownout behavior that defaults all charge and balance switches to a safe state. Never enable the BQ25792 charge IC's OTG (reverse boost) mode, and read back that it is disabled after every configuration (§4).

## 8. Safety-critical design and test requirements

**Quadstor handles lithium batteries. This README is design context, not an assurance that the proposed circuitry is safe. Do not attach valuable packs or leave prototypes unattended before electrical and fault testing.**

Required engineering validation includes:

1. **Input:** PSU/6S switchover, reverse polarity, hot-plug, overvoltage/transients, reverse-current blocking, fuse and TVS selection, and 6S maximum voltage with design margin.
2. **Charging:** chosen IC's actual operating limits, current/voltage programming, charge termination, precharge/recovery policy, and pack temperature handling where sensing is available.
3. **Cell taps:** reversed connector, open/misaligned/intermittent taps, contact heating at up to 2 A, ADC protection, voltage accuracy, and independent overvoltage detection/shutdown strategy.
4. **Balancing:** upper-cell switch drive, leakage when off, stuck-on MOSFET, resistor temperature, ADC readings with bleeders active, and behavior during reset/power loss.
5. **Thermal/power:** inductor and MOSFET losses, IC temperature, copper/connector ratings, simultaneous four-lane load, and enclosure ventilation.
6. **Firmware:** startup with a pack already inserted, insertion/removal during charge, fault latching/recovery, brownouts, watchdog, and independent lane operation.

Prefer current-limited bench supplies and a battery simulator / resistive test fixtures for initial bring-up. Do not infer cell health from pack voltage alone.

## 9. Development plan and current open decisions

| Stage | Deliverable | State |
|---|---|---|
| Power board | PSU-priority selection, XT60 backup, protected `VOUT`, `VSENSE` | Schematic drawn (2026-09-30); 5 review findings to fix (§3); connectors and `VSENSE` not yet drawn; PCB outline only, unrouted |
| Single-lane schematic | BQ25792 lane (JST-XH, sensing, four bleed paths, NTC) plus STM32G0B1 MCU, TCA9548A I²C mux, TPS54160A buck regulator for the logic supply, 0.92" OLED, USB-C, SWD pads, buttons, buzzer, LEDs | Next milestone; ICs selected, schematic not started |
| Single-lane PCB | Assemble and test at low current | Planned |
| Firmware bring-up | Read charger registers, measure cell taps, control charge and bleed paths | Planned |
| Safety characterization | Voltage/current calibration, faults, switchover, thermal testing | Required before scaling |
| Four-lane board | Replicate proven lane, shared source bus and controller | Planned |
| Integrated UI/enclosure | Four independent statuses and user controls | Planned |

**Decisions that must be resolved before another agent finalizes a PCB:**

- ~~**BQ25713 vs BQ25792**~~: resolved as the BQ25792 charge IC for the prototype (§4). Still open is whether full-6S input (25.2 V vs the 24 V recommended max and ~25.2 V minimum OVP trip) works in practice (§3).
- **LiHV accuracy:** the BQ25792 charge IC's VREG (charge-voltage) tolerance (+0.65%) can exceed 4.35 V/cell at 17.40 V. Decide the firmware strategy, such as setting VREG slightly low or terminating on per-cell readings.
- **Current-setting semantics:** a global setpoint is shown in the header; are per-lane overrides needed, and how are lanes selected with two buttons?
- **Sensing and balancing:** ~~MCU choice~~ (STM32G0B1 microcontroller, LQFP64 proposed); ADC reference choice, tap front-end component values, upper-cell balance gate drivers, and whether a dedicated battery-monitor IC is needed.
- **Logic supply:** buck directly to 3.3 V, or buck to 5 V plus LDO (proposed); USB VBUS diode-OR yes or no.
- **Display:** the final larger single OLED (size, resolution, controller, I²C vs SPI); the lane panel pixel size (proposed 128×24) so four tiles plus the header fit; the I²C mux channel map (proposed ch0–3 lanes).
- **Protection:** battery temperature sensing, independent overvoltage cutoff, connector current rating, thermal sensors, fault thresholds, and charge enable interlocks.
- **Power board:** the schematic and part numbers are now in the repo (§3). Before layout, fix the unconnected Q2 gate, the backfeed path through Q1 / false PSU detection, the 220k vs 1 kΩ resistor mismatch, and Q5's gate margin. Also decide whether the SMCJ30A TVS diodes are enough given they clamp above 30 V, add the connectors and the `VSENSE` divider, and record the fuse rating. Test results are still needed.
- **Physical design:** OLED and connector part numbers, board dimensions, cooling/enclosure, PCB stackup, exact BOM, and firmware repository/toolchain.

## 10. Handoff instructions for contributors and agents

1. **Read this README and inspect the latest schematics, BOM, and PCB files.** Conversation-era part choices are historical context; the checked-in schematic is authoritative for what is actually built.
2. **Separate requirements from proposals.** Four 4S lanes, JST-XH-only pack connections, target-voltage presets, dual input with PSU priority, OLED/two-button UI, and passive balancing are project goals. Charger IC, MCU, sensing topology, and detailed firmware architecture may change.
3. **Do not assume tested functionality.** Mark every claim as proposed, schematic-complete, assembled, bench-tested, or battery-tested, with evidence and revision.
4. **When modifying hardware, update the README/BOM and record why** (input voltage margin, efficiency, thermal performance, supply availability, safety, etc.). Never swap the BQ25713 and BQ25792 charge ICs without redesigning the corresponding power stage.
5. **Before implementing charge control, confirm** the exact IC datasheet/register map, valid pack-voltage range, hardware safety defaults, cell-measurement calibration, and independent fault handling.
6. **Develop one lane first**, validate it under simulated and controlled loads, then replicate the proven circuitry to four lanes.
7. **Record every new determination in §12 Decision log** with its date and reasoning, and update the affected sections. Mark superseded options instead of deleting them.

### Suggested repository structure

```text
Quadstor/
├── README.md
├── docs/
│   ├── architecture.md
│   ├── decisions.md        # Dated design decisions and superseded options
│   ├── safety-test-plan.md
│   └── bringup-log.md      # Revision-specific measurements and failures
├── hardware/
│   ├── power-selection/
│   ├── single-lane/
│   └── four-lane/
├── firmware/
├── datasheets/             # Link to vendor docs or include where permitted
└── images/
```

The structure above is a **suggestion**. As of 2026-09-30, the repository contains:

- `README.md` and a `.gitignore`. The `.gitignore` excludes KiCad lock files (`~*.lck`), `*-backups/`, `fp-info-cache`, `*.kicad_prl`, and `_autosave-*`.
- `datasheets/` (see §11).
- `Hardware/Power Board/Quadstor Power Board/`: the KiCad 10 power board project (schematic drawn, PCB outline placed), plus its project-local `EasyEDA.kicad_sym`, `EasyEDA.pretty`, and `EasyEDA.3dshapes` libraries.
- `Hardware/Charge Board/` and `Hardware/Single Module (TEST)/` also exist locally but are empty. Git doesn't track empty folders, so they don't appear on GitHub yet.

The repository is published at [github.com/ItsReckliss/Quadstor](https://github.com/ItsReckliss/Quadstor) (`origin`, branch `main`).

**KiCad library gotcha:** the EasyEDA/LCSC importer once wrote the project's `fp-lib-table` with a `(sym_lib_table` header instead of `(fp_lib_table`. KiCad then ignores the footprint library, and "Update PCB from Schematic" reports every `EasyEDA:...` footprint as "not found." To fix it, change the first line to `(fp_lib_table` with the project closed, then reopen it. Also check that `sym-lib-table` starts with `(sym_lib_table`.

## 11. References

Datasheets in `datasheets/`:

- **`BQ25792_datasheet.pdf`:** TI BQ25792 charge IC (SLUSDG1C, Aug 2022), selected for the prototype.
- **`STM32G0B1_datasheet.pdf`:** ST STM32G0B1 microcontroller, the selected MCU. The reference manual RM0444 and bootloader note AN2606 are also needed and not yet in the repo.
- **`tca9548a.pdf`:** TI TCA9548A 8-channel I²C switch/multiplexer.
- **`tps54160a.pdf`:** TI TPS54160A 60 V buck regulator (SLVSB56C), for the logic supply.
- **`AO3401A.pdf`:** AOS AO3401A 30 V P-MOSFET (Rev 3.1, Dec 2023), the proposed bleed switch for cells 2–4.

Datasheets for the power board parts (AOD409, 2SA1213-Y, AO3400A, BZT52C12V, SMCJ30A, 01550900M) and for the CD74HC4067SM96 analog mux aren't in the repo yet.

Other:

- **TI BQ25713 (charge controller):** original external-MOSFET buck-boost charger baseline and current fallback; consult the current TI datasheet and reference design.
- **Toshiba TPHR8504PL (power MOSFET for the BQ25713 stage):** supplied project datasheet, including ratings, gate charge, electrical characteristics, and package drawings. Verify the actual revision and package against purchased parts.
- **AONR21307 (BATFET), NCE6003X and NTD5865NLT4G (balancing MOSFETs):** candidate/selected devices from earlier design discussions; retrieve authoritative datasheets before layout.

## 12. Decision log

Newest first. Status is **Decided** (the owner chose it), **Proposed** (recommended, not yet confirmed), or **Superseded**.

| Date | Decision | Status | Reasoning / notes |
|---|---|---|---|
| 2026-09-30 | Power board schematic: AOD409 P-MOSFET PSU switch (Q1); back-to-back AOD409 6S path (Q2/Q3) with BZT52C12V gate clamp; 2SA1213-Y PNP + AO3400A N-MOSFET PSU-present override; SMCJ30A TVS and 01550900M fuse on each input; 45 × 45 mm 2-layer outline | Decided (owner's schematic); fixes pending | Implements PSU priority with the 6S path off when the PSU is present. The netlist review in §3 found an unconnected Q2 gate, backfeed through Q1's body diode that can also falsely trigger PSU detection, 220k-labeled resistors whose part number is 1 kΩ, a TVS that clamps above 30 V, and thin Q5 gate margin. |
| 2026-09-30 | AO3401A bleed-switch figures verified against the datasheet | Verified | ±12 V V_GS confirmed, so the 5.1 V Zener gate clamp remains necessary; 60 mΩ max at −4.5 V; I_DSS ≤ 1 µA at 25 °C. |
| 2026-09-30 | Project tracked in git and published to github.com/ItsReckliss/Quadstor, with a KiCad `.gitignore` | Decided | Lock files, backups, autosaves, and per-user `.kicad_prl` view settings are excluded. |
| 2026-09-29 | TS: 5.23 kΩ REGN→TS, 30.1 kΩ TS→GND, 0603 10 kΩ NTC (B ≈ 3380–3435 K, 5%) next to the JST-XH. Fixed 10 kΩ if sensing is ever dropped. | Decided | TI typical-application divider (E96 values) keeps the datasheet's 0–60 °C / JEITA thresholds. Measures connector/board temperature, not the pack. |
| 2026-09-29 | Lane input protection: no per-lane TVS; DNP footprint for the ~2 Ω + 2.2 µF VBUS snubber; VBUS 0.1 µF + 2× 10 µF and PMID 0.1 µF + 3× 10 µF, all 50 V X7R | Decided | A TVS with >25.2 V standoff clamps ~40 V, above the 30 V abs max. The power board already does source selection, XT60 disconnect with the PSU on, and backfeed blocking. Its TVS/soft-start is still an open question. |
| 2026-09-29 | Firmware must never enable BQ25792 OTG (reverse boost) mode, and must read back that it is off after configuration | Decided | Prevents a pack from driving the input bus. |
| 2026-09-29 | **Single OLED.** The prototype uses a 0.92" OLED with a header (input V, target V, set current) and one lane panel. The final version uses a larger single OLED with the lane panel tiled 4× and a reorganized header. | Decided (final display part and panel pixel sizes proposed) | Tiling is done in firmware, not hardware. Lane-panel code must be a reusable fixed-size component. |
| 2026-09-29 | One OLED per lane plus a separate header display | Superseded | Logged in error from a misread of the owner's intent; replaced by the single-OLED decision above. |
| 2026-09-29 | OLED, buttons, buzzer, and status LEDs on the single-lane prototype | Decided | Verify the UI hardware before the four-lane build. |
| 2026-09-29 | SWD via Tag-Connect TC2030 footprint or bare pads instead of a JST-XH header | Proposed | USB DFU covers routine flashing; SWD is still needed for debugging and recovery. No connector BOM cost. |
| 2026-09-29 | USB-C on the controller/lane board for DFU flashing and CDC diagnostics, not power | Decided | USB can't deliver charging power; it is a data/service port, so it doesn't belong on the power board. |
| 2026-09-29 | RESET and BOOT buttons on the prototype | Proposed | BOOT0 needs the option-byte change on STM32G0 (verify in RM0444). |
| 2026-09-29 | TPS54160A buck regulator for the logic supply | Decided | 60 V input gives wide margin over the 30 V worst case. An LDO from 25 V would dissipate ~1 W. Run at ~500–700 kHz because of the 130 ns minimum on-time. |
| 2026-09-29 | Buck to 5 V, then LDO to 3.3 V; optional USB VBUS diode-OR into 5 V | Proposed | Quieter ADC rail, a 5 V buzzer rail, and USB-only bring-up of logic/UI. |
| 2026-09-29 | STM32G0B1 microcontroller, LQFP64 package | MCU decided; package proposed | 12-bit ADC (taps now read through an external analog mux, so few channels are needed), crystal-less USB with ROM DFU; the pin budget needs 64 pins. |
| 2026-09-29 | TCA9548A I²C mux; ch0–3 lanes, ch4–7 spare | Part decided; map proposed | The BQ25792 charge IC's address is fixed at 0x6B. |
| 2026-09-29 | Accept direct 6S input for the prototype and characterize OVP/hot-plug behavior | Decided | The 30 V abs max is not exceeded, but 25.2 V is above the 24 V recommended max and at the minimum OVP trip. Fallbacks are listed in §3. |
| 2026-09-29 | BQ25792 charge IC for the single-lane prototype | Decided | Integrated FETs reduce BOM and layout complexity; VREG up to 18.8 V covers 17.40 V LiHV. |
| 2026-09-29 | Cell measurement via one 16:1 analog mux (16 divided taps) + op-amp buffer into one MCU ADC pin; pack voltage from the B+ tap, cross-checked with the BQ25792 charge IC's VBAT reading | Decided (owner); mux = CD74HC4067SM96 (owner); op-amp proposed | Uses 1 ADC + 4 select pins instead of 13 ADC channels. Taps, not cells, are measured, and cells are computed by subtraction. Supersedes the 13-channel direct-ADC plan. |
| 2026-09-29 | Bleed switches: AO3401A P-MOSFET (cells 2–4) with 2N7002 level-shift + 5.1 V Zener gate clamp; AO3400A N-MOSFET (cell 1) | Proposed (owner suggested AO3401A; circuit is Claude's proposal) | P-channel with source on the cell's top node needs no floating gate supply. The Zener protects the ±12 V gate rating. Fails safe (off) at reset. Supersedes the NCE6003X/NTD5865NLT4G candidates. |
| earlier | BQ25713 charge controller + TPHR8504PL power MOSFETs ×4 + AONR21307 BATFET | Superseded (fallback) | Original baseline; see §4. |

---

**Project intent:** a convenient, independently controlled, four-port 4S balance-plug charger that works on the bench from its internal PSU and in the field from a 6S pack. The immediate priority is a **safe, instrumented, single-lane proof of concept**, not prematurely committing to a four-lane production PCB.
