# sky130 SRAM — ACT Implementation

Filippo Pruzzi, s250237 \
Stefano Garbari, s260328

***

## Overview

A 128×64 SRAM macro implemented in ACT (Asynchronous Circuit Toolkit) PRS (Production Rule Set) language, targeting the SkyWater SKY130 130nm PDK. Provides a 32-bit word via 2:1 column multiplexing.

***

## Architecture

| Component | Location | Description |
|---|---|---|
| 6T Bitcell | `src/mem/bitcell.act` | SKY130 6T SRAM bitcell with cross-coupled inverters and access transistors |
| Bitcell Array | `src/mem/bitcell_array.act` | 128 rows × 64 columns |
| Row Decoder | `src/decoder/hierarchical_decoder.act` | Hierarchical decoder for 128 row selection (7-bit address) |
| Wordline Driver | `src/wordline_driver/wordline_driver.act` | Drives selected wordline |
| Column Mux | `src/column_mux/column_mux_array.act` | 2:1 multiplexer to provide 32-bit words from 64 columns |

***

## Interface

```act
defproc sram (bool? addr[8]; bool? enable_wl; bool bl[32]; bool _bl[32])
```

- `addr[8]`: 8-bit address (addr[0] = column mux, addr[1..7] = row)
- `enable_wl`: Wordline enable signal
- `bl[32] / _bl[32]`: 32-bit bidirectional bitlines (read output, write input)

***

## Device Selection

### Where the devices and sizes come from

Device types and sizes were taken from the VLSIDA sky130 SRAM macros reference repository:  
https://github.com/VLSIDA/sky130_sram_macros

The actual bitcell SPICE netlist is also present locally in the PDK under `libs.ref/sky130_sram_macros/spice/`. Inspecting that file confirmed which device models and W/L values are used in a working, validated SRAM macro.

### Why not `nfet_01v8` / `pfet_01v8_hvt`

Those are the standard logic devices. The SKY130 PDK defines them as SPICE subckts with bin boundaries that require a minimum width of ~420nm (NFET) and ~360nm (PFET). A 6T SRAM cell requires devices as small as W=140nm (PFET) and W=210nm (NFET) — below what those standard models support. Using them causes ngspice to fail with "could not find a valid modelname".

### Correct devices for SRAM

The PDK provides dedicated SRAM devices with exactly one bin, sized for minimum-width SRAM layout:

| Device | W (µm) | L (µm) |
|---|---|---|
| `sky130_fd_pr__special_nfet_latch` | 0.21 | 0.15 |
| `sky130_fd_pr__special_pfet_latch` | 0.14 | 0.15 |

Both are included automatically by `sky130.lib.spice` via `all.spice`, so no extra `.include` is needed.

***

## Transistor Sizing

| Role | Device | W (µm) | L (µm) |
|---|---|---|---|
| Pull-down NFET (drive) | `special_nfet_latch` × 2 in parallel | 0.21 × 2 = 0.42 | 0.15 |
| Pull-up PFET | `special_pfet_latch` | 0.14 | 0.15 |
| Access NFET | `special_nfet_latch` | 0.21 | 0.15 |

`special_nfet_latch` only supports W=0.21µm. To get the drive transistor width of W=0.42µm (β ratio = 2 for read stability), two instances are placed in parallel — the same approach used in the VLSIDA reference macros.

***

## Key Implementation Notes

### M vs X instances in SPICE

SKY130 devices are defined as SPICE `.subckt`, not `.model` primitives. They must be instantiated as `X` (subcircuit) instances, not `M` (MOSFET) instances. W and L are passed as plain numbers in µm because `sky130.lib.spice` sets `options scale=1e-6`.

### Model loading

The testbench loads models via `.lib "path/to/sky130.lib.spice" tt`. ngspice resolves all relative sub-includes inside the `.lib` file relative to the `.lib` file's own location, making the simulation runnable from any directory.

***

## Simulation Results

Running `ngspice -b sky130_6T_bitcell_tb.spice`:

```
q_wr1 = 1.80000e+00   → PASS write-1: Q = 1.8 V
q_wr0 = 2.45722e-09   → PASS write-0: Q ≈ 0 V
```

Both write operations pass. The cell correctly stores a logic 1 (Q = VDD = 1.8V) and a logic 0 (Q ≈ 0V).

***

## References

- VLSIDA sky130 SRAM macros: https://github.com/VLSIDA/sky130_sram_macros
- SKY130 PDK device details: https://skywater-pdk.readthedocs.io/en/main/rules/device-details.html
- ACT language documentation: https://avlsi.csl.yale.edu/act/doku.php