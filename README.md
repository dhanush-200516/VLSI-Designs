
## Author

**Irukuvajula Dhanush**
B.Tech ECE, Jain (Deemed-to-be University), Bengaluru

## License

This project is licensed under the MIT License — see the [LICENSE](LICENSE) file for details.



# 16-bit Pipelined Wallace Tree Multiplier

A 5-stage pipelined RTL implementation of a 16×16 unsigned multiplier in Verilog HDL, using a Wallace Tree partial-product reduction network and a 32-bit Carry Lookahead Adder (CLA) for the final addition stage.

Verified with a self-checking, latency-aware testbench across **1,012 test vectors (12 directed + 1,000 random) — 100% pass rate, zero mismatches.**

---

## Overview

This project implements a 16-bit × 16-bit unsigned multiplier entirely from first principles — no use of Verilog's `*` operator anywhere in the actual hardware. Multiplication is built up using:

- Bitwise AND gates for partial product generation
- Half adders and full adders (Wallace Tree 2:2 / 3:2 compressors) for parallel reduction
- A Carry Lookahead Adder (CLA) for the final fast addition

The design is pipelined across **5 register stages**, giving a fixed **4-cycle latency** and a steady-state **throughput of one result per clock cycle**.

### Why Wallace Tree?

A naive multiplier adds 16 partial product rows sequentially. A Wallace Tree instead reduces all rows in parallel, column by column, using half/full adders — cutting the row count down logarithmically (6 reduction levels) instead of linearly, before a single fast adder finishes the job.

---

## Architecture

```
A (16b), B (16b)
        │
        ▼
[Stage 1] partial_product_gen  → 256 raw AND-bits (16 rows × 16 bits)
        │ (pipeline register)
        ▼
[Stage 2] wallace_stage1  → Wallace reduction Levels 1 & 2
        │ (pipeline register)
        ▼
[Stage 3] wallace_stage2  → Wallace reduction Levels 3 & 4
        │ (pipeline register)
        ▼
[Stage 4] wallace_stage3  → Wallace reduction Levels 5 & 6 (≤2 bits/column)
        │ (pipeline register)
        ▼
[Stage 5] cla_32bit  → Final 32-bit Carry Lookahead Addition
        │ (pipeline register)
        ▼
P (32b), valid_out
```

A result for an operand pair applied at cycle **T** becomes valid on cycle **T+4**, confirmed both analytically and via simulation.

### Top-Level Interface

| Signal      | Direction | Width | Description                                                  |
|-------------|-----------|-------|----------------------------------------------------------------|
| `clk`       | Input     | 1     | System clock (frequency-agnostic, purely synchronous design) |
| `rst_n`     | Input     | 1     | Active-low synchronous reset; clears all pipeline registers   |
| `valid_in`  | Input     | 1     | Asserted when `A` and `B` are valid on the current cycle      |
| `A[15:0]`   | Input     | 16    | Multiplicand                                                  |
| `B[15:0]`   | Input     | 16    | Multiplier                                                    |
| `P[31:0]`   | Output    | 32    | Registered final product, `P = A × B`                        |
| `valid_out` | Output    | 1     | Asserted exactly 4 cycles after the corresponding `valid_in`   |

### Module Hierarchy

```
tb_pipelined_wallace_mult_16x16      (self-checking testbench)
 └── pipelined_wallace_mult_16x16    (top-level DUT)
      ├── partial_product_gen        (Stage 1 — 256 AND-gate partial products)
      ├── wallace_stage1             (Stage 2 — Wallace Levels 1–2: 124 FA, 21 HA)
      ├── wallace_stage2             (Stage 3 — Wallace Levels 3–4: 53 FA, 23 HA)
      ├── wallace_stage3             (Stage 4 — Wallace Levels 5–6: 19 FA, 35 HA)
      └── cla_32bit                  (Stage 5 — 32-bit Carry Lookahead Adder)
           └── 8 × 4-bit lookahead blocks
```

---

## Design Specifications

| Parameter        | Value                                  |
|-------------------|-----------------------------------------|
| Architecture      | Wallace Tree, 6 reduction levels       |
| Operand width     | 16 bits (unsigned)                     |
| Product width     | 32 bits                                |
| Pipeline stages   | 5 (4-cycle latency)                    |
| Throughput        | 1 result / clock cycle (steady state)  |
| Clock frequency   | 50 MHz (20 ns period, testbench-defined) |
| Reset type        | Active-low, synchronous                |
| HDL               | Verilog HDL (Verilog-2001 style)       |
| Simulation tool   | Icarus Verilog (`iverilog` / `vvp`)    |
| Waveform viewer   | GTKWave                                |

### Resource Summary

| Metric                                  | Count                        |
|------------------------------------------|-------------------------------|
| Partial product bits generated (Stage 1) | 256 (16 × 16 AND terms)       |
| Total Full Adders (Stages 2–4)            | 196                           |
| Total Half Adders (Stages 2–4)            | 79                            |
| Total adder primitives                    | 275                           |
| Final adder                               | 32-bit, 8 × 4-bit CLA blocks  |
| Pipeline registers (flip-flops)           | 550, across 5 stage boundaries|
| Fixed pipeline latency                    | 4 clock cycles                |
| Steady-state throughput                   | 1 result / clock cycle        |

---

## Verification

The design was verified with a **self-checking, scoreboard-based testbench** (`tb_pipelined_wallace_mult_16x16.v`):

- **Self-checking:** every output is automatically compared against a behavioral reference model (`A * B`, using Verilog's native operator — used *only* in the testbench, never in the DUT).
- **Latency-aware scoreboard:** since the DUT is pipelined, inputs and outputs are temporally decoupled. A FIFO queue pushes every sent `(A, B)` pair and pops/compares the oldest entry against each result as it emerges — correctly matching results to inputs regardless of pipeline depth.
- **Directed + random coverage:** 12 hand-picked corner cases (`0×0`, `max×max`, alternating-bit patterns, etc.) followed by 1,000 back-to-back pseudo-random vectors.
- **Watchdog timeout:** guarantees the simulation always terminates, even if a bug prevents `valid_out` from ever asserting.
- **Strict pass condition:** verdict requires both zero failures *and* a minimum number of checks performed, preventing false positives from a silently broken `valid_out` chain.

### Results

| Metric                  | Result                     |
|---------------------------|----------------------------|
| Total results checked     | 1012                       |
| Passed                    | 1012                       |
| Failed                    | 0                          |
| Overall verdict           | **PASS**                   |
| Verified pipeline latency | 4 clock cycles             |
| Clock period              | 20 ns (50.0 MHz)           |

Sample verified transactions:

```
PASS : A=0x0000  B=0x0000  expected P=0x00000000  got P=0x00000000
PASS : A=0xFFFF  B=0xFFFF  expected P=0xFFFE0001  got P=0xFFFE0001
PASS : A=0xD609  B=0x5663  expected P=0xFFFC0004  got P=0xFFFC0004
```

Waveform inspection in GTKWave independently confirms:
- The 4-cycle latency between `valid_in` and `valid_out`
- The `valid_stage1` → `valid_stage2` → `valid_stage3` → `valid_stage4` chain propagating alongside the data path
- Back-to-back random vectors toggling every clock cycle with no stalls, confirming full pipeline throughput

---

## Repository Structure

```
16bit-Pipelined-Wallace-Tree-Multiplier/
├── rtl/
│   ├── pipelined_wallace_mult_16x16.v   # Top-level module
│   ├── partial_product_gen.v            # Stage 1
│   ├── wallace_stage1.v                 # Stage 2 (Wallace L1+L2)
│   ├── wallace_stage2.v                 # Stage 3 (Wallace L3+L4)
│   ├── wallace_stage3.v                 # Stage 4 (Wallace L5+L6)
│   ├── cla_32bit.v                      # Stage 5 (32-bit CLA)
│   ├── full_adder.v                     # 3:2 compressor primitive
│   └── half_adder.v                     # 2:2 compressor primitive
├── testbench/
│   └── tb_pipelined_wallace_mult_16x16.v
├── sim_results/
│   ├── sim_output.txt                   # Full simulation log
│   └── wallace_mult_waveform.vcd        # GTKWave waveform dump
├── docs/
│   ├── capstone_report.pdf              # Full IEEE-style capstone report
│   └── design_documentation.pdf         # Design documentation
└── README.md
```

---

## How to Reproduce

```bash
iverilog -o sim_out tb_pipelined_wallace_mult_16x16.v \
  pipelined_wallace_mult_16x16.v wallace_stage1.v wallace_stage2.v \
  wallace_stage3.v partial_product_gen.v cla_32bit.v full_adder.v half_adder.v

vvp sim_out

gtkwave wallace_mult_waveform.vcd
```

This compiles the design and testbench, runs the simulation (producing `sim_output.txt`), and opens the resulting waveform in GTKWave for inspection.

---

## Synthesizability

The design under test contains no simulation-only constructs — no `*` operator, no `$display`/`$finish` calls, no delay statements. All eight RTL source files are written in clean, synthesizable Verilog-2001 and can be brought into Quartus Prime, Vivado, or any standard synthesis tool, with `pipelined_wallace_mult_16x16` set as the top-level entity. (`tb_pipelined_wallace_mult_16x16.v` is simulation-only and must be excluded from synthesis.)

---

## Key Design Concepts

1. **Multiplication = AND array + parallel reduction.** `partial_product_gen` builds the 16×16 AND array; the Wallace Tree compresses it in parallel rather than adding rows sequentially.
2. **Wallace Tree over sequential multiplication.** 6 reduction levels (split across 3 pipeline stages) collapse the matrix to 2 rows, instead of ~16 sequential additions.
3. **Carry Lookahead over ripple-carry.** Avoids reintroducing a slow 32-bit serial carry chain after the fast Wallace reduction.
4. **Pipelining trades latency for throughput.** 4-cycle fixed latency in exchange for a new result every single clock cycle.
5. **Verification via scoreboard.** A FIFO-based scoreboard is the only correct way to check a pipelined datapath, since inputs and outputs are temporally decoupled.

---
