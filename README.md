# sky130 SRAM — ACT Implementation

Filippo Pruzzi, s250237 \
Stefano Garbari, s260328

***

## Overview

A 128×64 SRAM macro implemented in ACT (Asynchronous Circuit Toolkit) PRS (Production Rule Set) language, targeting the SkyWater SKY130 130nm PDK. Provides a 32-bit read word via 2:1 column multiplexing.

***

## Architecture

| Component | Location | Description |
|---|---|---|
| 6T Bitcell | `src/mem/bitcell.act` | SKY130 6T SRAM bitcell with cross-coupled inverters and access transistors |
| Bitcell Array | `src/mem/bitcell_array.act` | 128 rows × 64 columns |
| Row Decoder | `src/decoder/hierarchical_decoder.act` | Hierarchical decoder for 128 row selection (7-bit address) |
| Wordline Driver | `src/wordline_driver/wordline_driver.act` | NAND + inverter per row; drives selected wordline |
| Column Mux | `src/column_mux/column_mux_array.act` | 2:1 read multiplexer providing a 32-bit word from 64 physical columns |
| Precharge | `src/precharge/precharge_array.act` | Active-low precharge; charges all physical bitlines to VDD before each access |

***

## Interface

```act
defproc sram (bool? addr[8]; bool? enable_wl; bool? pre_n; bool bl[32]; bool _bl[32])
```

- `addr[8]`: 8-bit address (`addr[0]` = column select, `addr[1..7]` = row)
- `enable_wl`: Wordline enable — assert after address has settled
- `pre_n`: Precharge enable, active low — assert before each access, release before opening the wordline
- `bl[32] / _bl[32]`: 32-bit differential read output (bitlines discharge on the side matching the stored value)

***

## Device Selection

### Bitcell devices

Device types and sizes for the 6T bitcell were taken from the VLSIDA sky130 SRAM macros reference:
https://github.com/VLSIDA/sky130_sram_macros

The actual bitcell SPICE netlist is also present locally in the PDK under `libs.ref/sky130_sram_macros/spice/`.

### Why not `nfet_01v8` / `pfet_01v8_hvt` for the bitcell

Those are the standard logic devices. The SKY130 PDK defines them as SPICE subckts with bin boundaries that require a minimum width of ~420nm (NFET) and ~360nm (PFET). A 6T SRAM cell requires devices as small as W=140nm (PFET) and W=210nm (NFET) — below what those standard models support. Using them causes ngspice to fail with "could not find a valid modelname".

### Correct devices for the bitcell

The PDK provides dedicated SRAM devices with exactly one bin, sized for minimum-width SRAM layout:

| Device | W (µm) | L (µm) |
|---|---|---|
| `sky130_fd_pr__special_nfet_latch` | 0.21 | 0.15 |
| `sky130_fd_pr__special_pfet_latch` | 0.14 | 0.15 |

Both are included automatically by `sky130.lib.spice` via `all.spice`, so no extra `.include` is needed.

### Peripheral logic devices

All peripheral cells (decoder, wordline driver, precharge) use standard logic devices:
`sky130_fd_pr__nfet_01v8` and `sky130_fd_pr__pfet_01v8`, confirmed against the VLSIDA sky130 SRAM macros reference.

***

## Transistor Sizing

### 6T Bitcell

| Role | Device | W (µm) | L (µm) |
|---|---|---|---|
| Pull-down NFET (drive) | `special_nfet_latch` × 2 in parallel | 0.21 × 2 = 0.42 | 0.15 |
| Pull-up PFET | `special_pfet_latch` | 0.14 | 0.15 |
| Access NFET | `special_nfet_latch` | 0.21 | 0.15 |

`special_nfet_latch` only supports W=0.21µm. To get the drive transistor width of W=0.42µm (β ratio = 2 for read stability), two instances are placed in parallel — the same approach used in the VLSIDA reference macros.

### Decoder gates

Sizes confirmed against the VLSIDA OpenRAM `sky130_custom_nand{2,3,4}_dec.sp` files:

| Cell | Device | W (µm) | L (µm) |
|---|---|---|---|
| NAND pull-down | `nfet_01v8` | 0.74 | 0.15 |
| NAND pull-up | `pfet_01v8` | 1.12 | 0.15 |
| INV pull-down | `nfet_01v8` | 0.36 | 0.15 |
| INV pull-up | `pfet_01v8` | 1.12 | 0.15 |

### Wordline driver

The NAND gate uses the same sizes as the decoder NAND. The inverter is upsized to drive the large wordline capacitance:

| Cell | Device | W (µm) | L (µm) |
|---|---|---|---|
| NAND pull-down | `nfet_01v8` | 0.74 | 0.15 |
| NAND pull-up | `pfet_01v8` | 1.12 | 0.15 |
| INV pull-down | `nfet_01v8` | 7.0 | 0.15 |
| INV pull-up | `pfet_01v8` | 7.0 | 0.15 |

### Precharge

Two PFET pull-ups per column (one for BL, one for BLB):

| Cell | Device | W (µm) | L (µm) |
|---|---|---|---|
| BL pull-up | `pfet_01v8` | 0.84 | 0.21 |
| BLB pull-up | `pfet_01v8` | 0.84 | 0.21 |

***

## Key Implementation Notes

### M vs X instances in SPICE

SKY130 devices are defined as SPICE `.subckt`, not `.model` primitives. They must be instantiated as `X` (subcircuit) instances, not `M` (MOSFET) instances. W and L are passed as plain numbers in µm because `sky130.lib.spice` sets `options scale=1e-6`.

### PRS simulation and SRAM writes

ACT/actsim PRS simulation is a digital model. SRAM write operations rely on transistor size ratios to overcome the bitcell PFET pull-up — an inherently analog effect that PRS simulation cannot reproduce. For this reason the design is verified at the read path level: the bitcell is initialised to a known state and the read (precharge → wordline open → bitline discharge) is simulated with actsim.

### Startup interference warnings

actsim reports `WARNING: weak-interference` on the decoder outputs at startup. These are expected: the predecoder NAND gates and their inverters race to resolve from an uninitialized state. They settle within a few time units and do not affect functional correctness.

***

## Simulation Results

PRS-level simulation with actsim (`docker run --rm -i -v $(pwd)/src:/work actsim actsim test.act test < src/test.actsim`):

Bitcell initialised to Q=1, _Q=0. Sequence: precharge → wordline open → read.

```
[  0] Q := 1, _Q := 0          cell initialised
[ 10] phys_bl[0]  := 1         precharge charges BL  ✓
[ 10] phys__bl[0] := 1         precharge charges BLB ✓
[ 80] phys__bl[0] := 0         _Q=0 discharges BLB through open wordline ✓
[ 90] _bl[0]      := 0         column mux routes BLB to output ✓
```

`bl[0]` and `phys_bl[0]` do not appear — they remain at 1 throughout, as expected for a stored Q=1. The one-unit X glitch on `phys__bl[0]` at time 11 is a known simulation artefact caused by the wordline driver inverter chain not yet having resolved at the moment the precharge fires; it clears before the actual read.

***

## References

- VLSIDA sky130 SRAM macros: https://github.com/VLSIDA/sky130_sram_macros
- VLSIDA OpenRAM sky130 sp_lib (decoder gate sizes): https://github.com/VLSIDA/OpenRAM/tree/dev/technology/sky130/sp_lib
- SKY130 PDK device details: https://skywater-pdk.readthedocs.io/en/main/rules/device-details.html
- ACT language documentation: https://avlsi.csl.yale.edu/act/doku.php
