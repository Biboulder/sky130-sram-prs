# sky130 6T SRAM Bitcell — Implementation Notes

## Files

| File | Description |
|---|---|
| `sky130_6T_bitcell.act` | ACT PRS description of the 6T bitcell |
| `sizes.act` | sky130-specific transistor sizing parameters |
| `sky130_6T_bitcell_tb.spice` | ngspice testbench — write-1 and write-0 operations |
| `test.act` / `test.actsim` | actsim structural test for the PRS |
| `sim.sh` | runs ngspice from the project folder |

---

## Device Selection

### Reference source

Device types and sizes were taken from the VLSIDA sky130 SRAM macros repository:
[https://github.com/VLSIDA/sky130_sram_macros](https://github.com/VLSIDA/sky130_sram_macros)

The bitcell SPICE netlist (e.g. `sky130_sram_1kbyte_1rw1r_32x256_8.spice`) is also
present locally in the PDK at:

```
~/.volare/volare/sky130/versions/.../sky130A/libs.ref/sky130_sram_macros/spice/
```

Inspecting that file confirmed the device names and sizes used in a validated SRAM macro.

### Why not `nfet_01v8` / `pfet_01v8_hvt`

Those are the standard logic devices. The sky130 PDK defines them as SPICE subckts
(not `.model` primitives), with bin boundaries in their pm3 model files:

- `sky130_fd_pr__nfet_01v8__tt.pm3.spice` — minimum W covered by bins is ~420 nm
- `sky130_fd_pr__pfet_01v8_hvt__tt.pm3.spice` — minimum W covered by bins is ~360 nm

A 6T SRAM cell requires a pull-up PFET of W=140 nm and access/drive NFETs as small
as W=210 nm. These are below the minimum covered by the standard logic devices, so
those models cannot simulate these sizes.

### Correct devices for SRAM

The PDK provides purpose-built SRAM devices with exactly one bin, sized for the
minimum-width SRAM layout:

| Device | W (µm) | L (µm) | Bin wmin (m) | Bin wmax (m) |
|---|---|---|---|---|
| `sky130_fd_pr__special_nfet_latch` | 0.21 | 0.15 | 2.095e-7 | 2.105e-7 |
| `sky130_fd_pr__special_pfet_latch` | 0.14 | 0.15 | 1.395e-7 | 1.405e-7 |

Model files found at:
```
~/.volare/volare/sky130/versions/.../sky130A/libs.ref/sky130_fd_pr/spice/
  sky130_fd_pr__special_nfet_latch.pm3.spice
  sky130_fd_pr__special_pfet_latch.pm3.spice
```

Both are included automatically by `sky130.lib.spice` via `all.spice`.

---

## Transistor Sizing

| Role | Device | W (µm) | L (µm) |
|---|---|---|---|
| Pull-down NFET (drive) | `special_nfet_latch` × 2 in parallel | 0.21 × 2 = 0.42 | 0.15 |
| Pull-up PFET | `special_pfet_latch` | 0.14 | 0.15 |
| Access NFET | `special_nfet_latch` | 0.21 | 0.15 |

`special_nfet_latch` only supports W=0.21 µm. To achieve the drive transistor
width of W=0.42 µm (β ratio = 2 for read stability), two instances are placed
in parallel — the same approach used in the VLSIDA reference macros.

---

## SPICE Model Loading

### Problem

`sky130.lib.spice tt` uses relative `.include` paths internally. These only resolve
correctly when ngspice is run from the PDK ngspice directory:

```
~/.volare/.../sky130A/libs.tech/ngspice/
```

Running from the project folder caused "can't find model" errors. The `set skywaterpdk`
flag in `~/.spiceinit` does not fix this in ngspice-46.

### Fix

Use `.lib` with an absolute path in the testbench:

```spice
.lib "/home/bibo/.volare/.../sky130A/libs.tech/ngspice/sky130.lib.spice" tt
```

ngspice resolves all relative sub-includes inside the `.lib` file relative to the
`.lib` file's own location, not the CWD. This makes the simulation runnable from
any directory.

### M vs X instances

Because sky130 devices are defined as SPICE `.subckt` (not `.model`), they must
be instantiated as `X` (subcircuit) instances, not `M` (MOSFET) instances.
W and L are passed in µm (unitless numbers) because `sky130.lib.spice` sets
`options scale=1e-6`.

---

## Simulation Results

Running `ngspice -b sky130_6T_bitcell_tb.spice` from the project folder:

```
q_wr1 = 1.80000e+00   → PASS write-1: Q = 1.8 V
q_wr0 = 2.45722e-09   → PASS write-0: Q ≈ 0 V
```

## PRS Structural Test

Running `actsim test.act test < test.actsim` verifies that the ACT PRS compiles
and that the access transistors correctly connect the bitlines to the storage nodes
when the wordline is raised — matching the expected output of the reference
implementation in `asram-fusion-refactor`.
