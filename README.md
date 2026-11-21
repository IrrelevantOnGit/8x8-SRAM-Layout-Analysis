# 8×8 6T SRAM — Full-Custom Design in Cadence (gpdk090)

A 64-bit (8×8) 6T SRAM designed full-custom in Cadence Virtuoso on the Generic 90 nm PDK: bit-cell array, 3-to-8 row decoder, precharge, write drivers and sense amplifiers, taken from schematic through layout, DRC, LVS, parasitic extraction and pre/post-layout simulation.

> **Status:** Complete. DRC clean, LVS matched, extracted-view simulation verified. Supply VDD = 1.2 V.

![Top-level layout](docs/images/top_layout_drc.png)

---

## Contents

- [Architecture](#architecture)
- [Block Specifications](#block-specifications)
- [Verification Status](#verification-status)
- [Results](#results)
- [Images](#images)
- [Tools and Technology](#tools-and-technology)
- [Future Work](#future-work)
- [License](#license)
- [Author](#author)

---

## Architecture

| Item | Value |
|---|---|
| Capacity | 8 words × 8 bits (64 bits) |
| Word lines / bit-line pairs | 8 / 8 |
| Address | 3-bit row address + enable |
| Cell | 6T, 4.5 µm × 3.5 µm, rails top and bottom |
| Array | 8×8 tiles, odd rows mirrored in Y so neighbouring rows share a rail |
| Bounding box | ≈ 45 µm × 53 µm (decoder included) |
| Library / top cell | `Tanuj` / `sram_8x8` |

**Floorplan** (per column, 4.5 µm pitch, bottom to top):

```
 sense amp  →  write driver (mirrored)  →  8×1 array column  →  precharge
```

The row decoder sits left of the array on the same 3.5 µm row pitch. WL0–WL7 run on Metal3 into the array. PRE, WE and SE run across all 8 columns on Metal3.

**Top-level pins:** `A0 A1 A2 EN` · `PRE WE SE` · `DIN0–DIN7` · `Vout0–Vout7` (`DOUT` in testbenches) · `VDD GND`

---

## Block Specifications

| Block | Function | Details |
|---|---|---|
| **6T cell** | Stores one bit | Cross-coupled inverters + two access transistors on WL; BL / BLB |
| **Precharge** | Precharges and equalises BL / BLB before a read | Two 2 µm PMOS pull-ups + one 1 µm PMOS equaliser, L = 100 nm |
| **Write driver** | Drives BL / BLB during a write | Inverter (480n / 240n) makes DINB; two 1 µm series-NMOS stacks gated by WE discharge BL or BLB |
| **Sense amp** | Detects small BL differential, amplifies to logic level | Input pair 2 µm, mirror load 4 µm, tail 4 µm (enabled by SE), output inverter 2.4 µm / 1 µm |
| **Row decoder** | One-hot word-line select | 3 address inverters (480n / 240n); per row a NAND4 (EN, A2, A1, A0) plus a 960n / 480n word-line driver, L = 100 nm |

---

## Verification Status

| Check | Result |
|---|---|
| Block-level DRC / LVS (precharge, write driver, sense amp, decoder) | Pass |
| Top-level DRC (`sram_8x8`) | Pass |
| Top-level LVS (`sram_8x8`) | **Schematic and layout match** |
| Functional read / write (schematic) | Pass |
| Extracted-view (parasitic) simulation | Pass |

**Top-level DRC issues found and fixed:**

| Rule | Cause | Fix |
|---|---|---|
| `NW.SP.1` (×7) | Precharge Nwell gap between columns, 0.48 µm | Nwell fill |
| `NW.SP.1` (×8) | Sense-amp Nwell pocket, 0.49 µm | Nwell fill |
| `NIMP.SP.1` (×7) | Write-driver Nimp gap between columns, 0.14 µm | Nimp fill |
| `METAL1.SP.1.1` (×8) | 0.11 µm notch beside the precharge BLB pad | Metal1 fill |
| `psubstrate_StampError*` | Precharge VDD, write-driver/sense-amp VDD and sense-amp GND rails were floating | Decoder Metal2 GND/VDD spines extended, Metal1 straps added |

**LVS note:** `CBL_0`, a bit-line load capacitor in the schematic, has no layout counterpart and was excluded from the comparison; the loading belongs in the testbench, and extraction supplies the real bit-line capacitance.

---

## Results

### Hold static noise margin
Butterfly curves show two stable states (Q = 0, Q = 1) and one metastable point at ≈ 0.48 V, with large symmetric wings at VDD = 1.2 V.

### Decoder
For A2 A1 A0 = 000 → 111 the matching WL0–WL7 goes high in turn (10 ns per step). With EN = 0 (80–100 ns) no word line is selected.

### Read / write, pre-layout vs extracted
All eight outputs return the expected data at both sample points.

| Output | Expected @84n | Pre @84n | Post @84n | Expected @184n | Pre @184n | Post @184n |
|---|---|---|---|---|---|---|
| DOUT0 | 0 | 0 V | 0.005 V | 1 | 1.2 V | 1.184 V |
| DOUT1 | 1 | 1.2 V | 1.182 V | 0 | 0 V | 0.005 V |
| DOUT2 | 0 | 0 V | 0.005 V | 1 | 1.2 V | 1.180 V |
| DOUT3 | 1 | 1.2 V | 1.179 V | 0 | 0 V | 0.005 V |
| DOUT4 | 1 | 1.2 V | 1.178 V | 0 | 0 V | 0.006 V |
| DOUT5 | 0 | 0 V | 0.005 V | 1 | 1.2 V | 1.177 V |
| DOUT6 | 1 | 1.2 V | 1.177 V | 0 | 0 V | 0.005 V |
| DOUT7 | 0 | 0 V | 0.006 V | 1 | 1.2 V | 1.176 V |

**Extracted-view figures:**

| Metric | Post-layout |
|---|---|
| Average supply current (0–210 ns) | 173 µA |
| Average power at 1.2 V | 208 µW |
| Peak supply current | 4.08 mA |
| Output high (at read) | 1.176 – 1.184 V |
| Output low (at read) | 5 – 6 mV |

### Bit hold
Over a 10 µs hold window both the schematic and extracted runs retain the stored bits; the extracted run shows gradual output decay beginning around 5.2 µs.

---

## Images

### Top Level

![Top-level layout — DRC clean](docs/images/top_layout_drc.png)
*Final top-level layout of `sram_8x8` — DRC: No errors found.*

![Top-level schematic](docs/images/top_schematic.png)
*Top-level testbench schematic (`tb_sram_8x8_ret_v3`) showing the `sram_8x8` block with all pins.*

![Top-level LVS — match](docs/images/top_lvs_match.png)
*Top-level LVS run: Schematic and Layout Match (0 net / device / pin / parameter mismatches).*

![Top-level LVS Debug](docs/images/top_lvs_debug.png)
*LVS Debug window for `sram_8x8` confirming "Schematic and Layout Match".*

---

### 6T Bit Cell

![6T cell layout](docs/images/cell_layout.png)
*6T SRAM bit-cell layout (`sram6t_gen`) — DRC: No errors found.*

![6T cell layout detail](docs/images/cell_layout_detail.png)
*6T bit-cell layout with BL, BLB and WL pin labels visible.*

![Cell LVS — match](docs/images/cell_lvs_match.png)
*Cell-level LVS run for `sram6t_gen`: Schematic and Layout Match.*

![Cell LVS Debug](docs/images/cell_lvs_debug.png)
*LVS Debug for `sram6t_gen` — no mismatches.*

---

### 8×8 Array

![8×8 array layout — DRC](docs/images/array_layout_drc.png)
*8×8 tiled bit-cell array (`sram_array_8x8`) — DRC: No errors found.*

![8×8 array LVS — match](docs/images/array_lvs_match.png)
*Array-level LVS for `sram_array_8x8`: Schematic and Layout Match.*

---

### Peripheral Circuits

#### Precharge

![Precharge layout — DRC](docs/images/precharge_layout_drc.png)
*Precharge circuit layout — DRC: No errors found.*

#### Write Driver

| Schematic | Layout — DRC clean |
|---|---|
| ![Write driver schematic](docs/images/write_driver_schematic.png) | ![Write driver layout](docs/images/write_driver_layout_drc.png) |

*Write driver symbol schematic (left); write driver layout with DRC: No errors (right).*

![Write driver LVS Debug](docs/images/write_driver_lvs_debug.png)
*Write driver LVS Debug: Schematic and Layout Match.*

#### Sense Amplifier

| Schematic | Layout — LVS match |
|---|---|
| ![Sense amp schematic](docs/images/sense_amp_schematic.png) | ![Sense amp layout](docs/images/sense_amp_layout_lvs.png) |

*Sense amp symbol schematic (left); sense amp layout with LVS: Schematic and Layout Match (right).*

![Sense amp LVS Debug](docs/images/sense_amp_lvs_debug.png)
*Sense amp LVS Debug: Schematic and Layout Match.*

---

### Row Decoder

| Schematic | Layout — DRC clean |
|---|---|
| ![Decoder schematic](docs/images/decoder_schematic.png) | ![Decoder layout](docs/images/decoder_layout_drc.png) |

*3-to-8 row decoder schematic (left); decoder layout with DRC: No errors (right).*

![Decoder LVS — match](docs/images/decoder_lvs_match.png)
*Decoder LVS: Schematic and Layout Match.*

![Decoder LVS Debug](docs/images/decoder_lvs_debug.png)
*Decoder LVS Debug window: Schematic and Layout Match.*

---

### Physical Verification — Top Level

![Top-level DRC — clean](docs/images/top_layout_drc.png)
*Top-level DRC on `sram_8x8`: No DRC errors found (after all fills applied).*

![Top-level LVS — match](docs/images/top_lvs_match.png)
*Top-level LVS summary: 0 net mismatches, 0 device mismatches, 0 pin mismatches.*

![Top-level LVS Debug](docs/images/top_lvs_debug.png)
*LVS Debug for `sram_8x8` — "Schematic and Layout Match".*

---

### Extracted (Parasitic) View

![Extracted view — Virtuoso](docs/images/av_extracted_virtuoso.png)
*Extracted view of the full `sram_8x8` array open in Cadence Virtuoso (av_extracted cellview).*

![Extracted view — parasitics](docs/images/av_extracted_view.png)
*Full parasitic-extracted view showing Metal1–Metal6 routing with RC parasitics overlaid.*

---

### Simulation — Bit Hold

![Pre-layout bit hold](docs/images/prelayout_hold.png)
*Pre-layout bit retention over 10 µs: all 8 outputs hold their stored values stably.*

![Post-layout bit hold](docs/images/postlayout_hold.png)
*Post-layout (extracted) bit hold: outputs remain valid; gradual decay begins around 5.2 µs due to parasitic leakage.*

![Post-layout bit hold (run 2)](docs/images/postlayout_hold2.png)
*Post-layout bit hold simulation — alternative run confirming decay profile.*

---

## Tools and Technology

- Cadence Virtuoso (schematic, layout)
- Cadence Assura 4.1 (DRC, LVS, extraction)
- Cadence Spectre (simulation)
- Generic 90 nm PDK (`gpdk090`) — **not included**; requires your own licensed copy

---

## Future Work

- Column multiplexing and a larger array
- Read-delay and power characterisation across process / voltage / temperature
- Comparison against a flip-flop register file

---

## License

Copyright (c) 2026 Tanuj. **All rights reserved.**

This repository is proprietary. No licence is granted to use, copy, modify, distribute or create derivative works from any part of it without prior written permission from the copyright holder. See [LICENSE](LICENSE).

---

## Author

**Tanuj** — Reg. No. 24BEC0539
VLSI Systems Design (BECE303P)
