# 8×8 6T SRAM — Full-Custom Design in Cadence (gpdk090)

A 64-bit (8×8) 6T SRAM designed full-custom in Cadence Virtuoso on the Generic 90 nm PDK: bit-cell array, 3-to-8 row decoder, precharge, write drivers and sense amplifiers, taken from schematic through layout, DRC, LVS, parasitic extraction and pre/post-layout simulation.

> **Status:** Complete. DRC clean, LVS matched, extracted-view simulation verified. Supply VDD = 1.2 V.

![Top-level layout](docs/images/top_layout.png)

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
![Top-level layout](docs/images/top_layout.png)
*Final top-level layout.*

![Hierarchy view](docs/images/hierarchy_view.png)
*Hierarchy view: decoder, array, precharge, write driver, sense amp.*

![Top-level schematic](docs/images/top_schematic.png)
*Top-level schematic.*

### 6T Bit Cell
![Cell test schematic](docs/images/cell_test_schematic.png)
*1-bit cell with precharge, write driver and sense amp.*

![Cell layout](docs/images/cell_layout.png)
*6T cell layout.*

![Cell waveforms](docs/images/cell_waveforms.png)
*1-bit read / write waveforms.*

![Hold SNM](docs/images/hold_snm.png)
*Hold static noise margin.*

### Peripheral Circuits
| Schematic | Layout |
|---|---|
| ![Precharge schematic](docs/images/precharge_schematic.png) | ![Precharge layout](docs/images/precharge_layout.png) |
| ![Write driver schematic](docs/images/write_driver_schematic.png) | ![Write driver layout](docs/images/write_driver_layout.png) |
| ![Sense amp schematic](docs/images/sense_amp_schematic.png) | ![Sense amp layout](docs/images/sense_amp_layout.png) |

### Row Decoder
![Decoder layout](docs/images/decoder_layout.png)
*3-to-8 row decoder layout.*

![Decoder truth table](docs/images/decoder_truth_table.png)
![Decoder inputs](docs/images/decoder_inputs.png)
![Decoder waveforms](docs/images/decoder_waveforms.png)
*Truth table, address / enable inputs and one-hot word-line outputs.*

### Physical Verification
![Extracted view](docs/images/av_extracted_view.png)
*Extracted view of the full array.*

![LVS match](docs/images/lvs_match.png)
*LVS: schematic and layout match.*

### Simulation
![Functional waveforms](docs/images/functional_waveforms.png)
*Functional read / write on all 8 outputs.*

![Pre-layout read/write](docs/images/prelayout_read_write.png)
*Pre-layout read / write.*

![Post-layout read/write](docs/images/postlayout_read_write.png)
*Post-layout (extracted) read / write.*

![Pre-layout hold](docs/images/prelayout_hold.png)
*Pre-layout bit hold.*

![Post-layout hold](docs/images/postlayout_hold.png)
*Post-layout (extracted) bit hold.*

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