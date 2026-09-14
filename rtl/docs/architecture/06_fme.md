# 06 — FME (Fractional Motion Estimation)

## 1. Overview

The FME subsystem refines integer motion vectors (IMV) from IME to quarter-pixel
accuracy using HEVC-compliant interpolation, evaluates 9 candidate fractional
positions per 8×8 block via SATD cost, and performs skip/merge mode decision.

**File listing** (17 files under `rtl/fme/`):

| File | Role |
|------|------|
| `fme_top.v` | Top-level wrapper; wires all submodules |
| `fme_ctrl.v` | Main FSM + reference pixel address generation |
| `fme_interpolator_8x8.v` | 8×8 block interpolation orchestrator |
| `fme_interpolator_8pel.v` | 8-pel parallel horizontal interpolator |
| `fme_interpolator.v` | 7-tap / 4-tap filter primitive + clip |
| `fme_ip_half_ver.v` | Vertical half-pel + diagonal interpolators |
| `fme_ip_quarter_ver.v` | Vertical quarter-pel interpolators |
| `fme_satd_gen.v` | 9 parallel SATD engines + current pixel fetch |
| `fme_satd_8x8.v` | 8×8 Hadamard SATD core |
| `fme_cost.v` | Cost = SATD + λ·MVD; partition-aware best select |
| `fme_pred.v` | Prediction pixel MUX from best candidate |
| `fme_mv_buffer.v` | Top/left MV buffer (RAM + ping-pong regs) |
| `fme_mv_candidate_prepare.v` | Spatial MV candidate validity check (A0–B2) |
| `fme_skip.v` | Skip mode: (0,0) MV test + cost threshold |
| `getbits.v` | MV bit cost (leading-one detection) |
| `qp_lambda_table.v` | QP → λ lookup (combinational) |
| `enc_defines.v` (shared) | Global macro definitions |

---

## 2. Parameters

```verilog
FMV_WIDTH       = 10      // fractional MV field width (signed, quarter-pel units)
FME_COST_WIDTH  = 20      // cost accumulator width
PIXEL_WIDTH     = 8       // pixel bit-depth
PIC_X_WIDTH     = 6       // CTU X index width
PIC_Y_WIDTH     = 6       // CTU Y index width
SATD_WIDTH      = PIXEL_WIDTH + 10  // = 18
DLY_NUM         = 1       // pipeline delay for ctrl→interp signals
```

---

## 3. Module Hierarchy

```
fme_top
├── fme_ctrl                           // FSM + address generation
├── fme_interpolator_8x8               // 8×8 block interpolation
│   ├── fme_interpolator_8pel          // 8-pel parallel horizontal interp
│   │   └── fme_interpolator × 9       // half-pel / quarter-pel filter
│   ├── fme_ip_half_ver                // vertical half-pel + diagonal
│   │   └── fme_interpolator × 17      // 8 vert-half + 9 diagonal
│   └── fme_ip_quarter_ver             // vertical quarter-pel
│       └── fme_interpolator × 48      // 16 q1 + 16 q3 + 16 half-ref
├── fme_satd_gen                       // 9× SATD + current pixel fetch
│   └── fme_satd_8x8 × 9              // Hadamard SATD core
│       ├── fme_abs × 8                // column absolute difference
│       └── hadamard_trans_1d × 3      // 1D Hadamard (row+col)
├── fme_cost                           // Cost accumulator + best select
├── fme_pred                           // Prediction pixel MUX
├── fme_mv_buffer                      // Top/left MV storage
├── fme_mv_candidate_prepare           // Spatial candidate check
├── fme_skip                           // Skip mode decision
├── qp_lambda_table                    // QP → λ
└── db_mv_ram_sp_64x20                 // FME cost RAM
```

---

## 4. Interface

### fme_top ports

| Port | Width | Dir | Description |
|------|-------|-----|-------------|
| `clk` | 1 | I | Clock |
| `rstn` | 1 | I | Active-low async reset |
| `sysif_cmb_x_i` | PIC_X_WIDTH | I | Current CTU X index |
| `sysif_cmb_y_i` | PIC_Y_WIDTH | I | Current CTU Y index |
| `sysif_qp_i` | 6 | I | QP value |
| `sysif_start_i` | 1 | I | FME start |
| `sysif_done_o` | 1 | O | FME done |
| `fimeif_partition_i` | 42 | I | IME partition info |
| `fimeif_mv_rden_o` | 1 | O | IMV read enable |
| `fimeif_mv_rdaddr_o` | 6 | O | IMV read address |
| `fimeif_mv_data_i` | 2×FMV_WIDTH | I | IMV from IME |
| `cur_rden_o` | 1 | O | Current pixel read enable |
| `cur_4x4_idx_o` | 5 | O | Current 4×4 index |
| `cur_4x4_x_o` | 4 | O | Current 4×4 X pos |
| `cur_4x4_y_o` | 4 | O | Current 4×4 Y pos |
| `cur_pel_i` | 32×PIXEL_WIDTH | I | Current pixels (32 bytes) |
| `ref_rden_o` | 1 | O | Reference pixel read enable |
| `ref_idx_x_o` | 8 | O | Reference X index |
| `ref_idx_y_o` | 8 | O | Reference Y index |
| `ref_pel_i` | 64×PIXEL_WIDTH | I | Reference pixels (64 bytes, 16 pels) |
| `mcif_mv_rden_o` | 1 | O | FMV read enable |
| `mcif_mv_rdaddr_o` | 6 | O | FMV read address |
| `mcif_mv_data_i` | 2×FMV_WIDTH | I | FMV data in |
| `mcif_mv_wren_o` | 1 | O | FMV write enable |
| `mcif_mv_wraddr_o` | 6 | O | FMV write address |
| `mcif_mv_data_o` | 2×FMV_WIDTH | O | FMV data out |
| `mcif_pre_pixel_o` | 32×PIXEL_WIDTH | O | Predicted pixels |
| `mcif_pre_wren_o` | 1 | O | Predicted pixel write enable |
| `mcif_wr_4x4_x_o` | 4 | O | Predicted 4×4 X |
| `mcif_wr_4x4_y_o` | 4 | O | Predicted 4×4 Y |
| `mcif_wr_idx_o` | 5 | O | Predicted index |
| `mcif_siz_o` | 2 | O | Block size code |
| `fme_cost_o` | FME_COST_WIDTH | O | Accumulated FME cost |
| `skip_idx_o` | 85×4 | O | Skip merge indices |
| `skip_flag_o` | 85 | O | Skip flags per CU |
| `skip_cost_thresh_08/16/32/64` | 32 | I | Skip cost thresholds |

---

## 5. FSM

**fme_ctrl.v** — 13 states, one-hot encoding:

```
IDLE(0) → PRE_HALF(1) → HALF(2) → DONE_HALF(3)
         → PRE_QUAR(4) → QUAR(5) → DONE_QUAR(6)
         → PRE_SKIP(7) → SKIP(8) → DONE_SKIP(9)
         → PRE_MC(10) → MC(11) → DONE_MC(12)
```

| State | Action |
|-------|--------|
| IDLE | Wait for `sysif_start_i` |
| PRE_HALF | Setup half-pixel interpolation; generate ref pixel addresses |
| HALF | Run 8×8 interpolation for half-pixel candidates (3 positions) |
| DONE_HALF | Latch half-pixel results |
| PRE_QUAR | Setup quarter-pixel interpolation |
| QUAR | Run 8×8 interpolation for quarter-pixel candidates (5 positions) |
| DONE_QUAR | Latch quarter-pixel results; latch best MV into cost RAM |
| PRE_SKIP | Setup skip mode evaluation |
| SKIP | Run skip SATD evaluation per 8×8 block |
| DONE_SKIP | Finalize skip cost; compare with FME best cost |
| PRE_MC | Setup MC prediction write-back |
| MC | Write predicted pixels to MC buffer; write FMV to SRAM |
| DONE_MC | Write final MV; done |

---

## 6. Interpolation Pipeline

### 6.1 Filter Primitive — `fme_interpolator`

```verilog
module fme_interpolator #(
    parameter TYPE     // 0=half(7-tap), 1=quarter-1(4-tap), 2=quarter-3(4-tap)
    parameter HOR      // 1=horizontal, 0=vertical
    parameter LAST     // 1=output clip, 0=output expand
    parameter IN_EXPAND  // 1=input 16-bit, 0=input 8-bit
    parameter OUT_EXPAND // 1=output 16-bit, 0=output 8-bit
) (
    input  [PIXEL_WIDTH-1:0]    tap_0..tap_7  // 8 taps (8-bit or 16-bit)
    output [PIXEL_WIDTH-1:0]    val_o          // filtered pixel
);
```

**Half-pixel (TYPE=0):** 7-tap filter `[-1, 4, -11, 40, 40, -11, 4, -1]`, right-shift by 6, clip [0,255].

**Quarter-pixel (TYPE=1/2):** 4-tap filter, right-shift by 12, clip.

### 6.2 Horizontal Interpolation — `fme_interpolator_8pel`

- Takes 16 reference pixels (`ref_pel0..15`) from external 64-byte bus
- Produces 9 outputs: `hhalf_pel[0..8]` (half), `q1_buf[0..8]` (quarter-1), `q3_buf[0..8]` (quarter-3)
- Instantiates 9 × `fme_interpolator` (TYPE=0, HOR=1 for half; TYPE=1/2, HOR=1 for quarter)

### 6.3 Vertical Half-Pixel — `fme_ip_half_ver`

- 7-stage shift register per column (8 columns) to align 8 rows of reference pixels
- 8 × vertical half-pixel interpolators (TYPE=0, HOR=0, LAST=1) → `vhalf_pel[0..7]`
- 9 × diagonal interpolators (TYPE=0, HOR=0, LAST=0, IN_EXPAND=1) → `d_pel[0..8]`
  - Apply vertical 7-tap filter to the horizontal half-pixel buffer, producing diagonal (half×half) positions

### 6.4 Vertical Quarter-Pixel — `fme_ip_quarter_ver`

- Shifts for `q1_buf`, `q3_buf`, `h_buf`, `ref_pel` (7-stage each)
- Produces 48 outputs organized as 6 groups × 8 pixels:
  - `vquarter_1_x` — from quarter-1 horizontal buffer (TYPE=1)
  - `vquarter_3_x` — from quarter-3 horizontal buffer (TYPE=2)
  - `vhalf_2_x` — from half horizontal buffer (TYPE=1/2)
  - `vpel_0_x` — from reference pixel (TYPE=1/2)
- `frac_x_i` / `frac_y_i` select which quarter positions to output

### 6.5 Interpolation Flow Summary

For each 8×8 candidate position:
1. CTRL generates ref pixel address → external SRAM returns 64 pixels (16 pels × 4 bytes)
2. `fme_interpolator_8pel` runs horizontal interpolation → half, q1, q3 buffers
3. `fme_ip_half_ver` runs vertical interpolation on horizontal results → 9 candidate pixel sets
4. `fme_ip_quarter_ver` runs vertical interpolation on q1/q3/h buffers → quarter-pixel candidates
5. Results fed to SATD engine

---

## 7. SATD Computation

### 7.1 SATD Core — `fme_satd_8x8`

- **Inputs:** 8 current pixels, 8 predicted pixels
- **Process:**
  1. `fme_abs`: column absolute difference `|cur - pred|` → 9-bit
  2. `hadamard_trans_1D` (horizontal): 8-point 1D Hadamard on rows
  3. `hadamard_trans_1D` (vertical): 8-point 1D Hadamard on columns
  4. Sum absolute values of 2D Hadamard coefficients → SATD
- **Output:** `satd_8x8_o` [SATD_WIDTH-1:0] (18 bits), valid pulse

### 7.2 SATD Generator — `fme_satd_gen`

- 9 parallel `fme_satd_8x8` instances (one per candidate)
- Manages current block pixel fetch from external SRAM via `cur_rden_o`
- Pipelines candidate pixel alignment
- Outputs 9 SATD values + `satd_valid_o`

---

## 8. Cost Computation — `fme_cost`

**Formula:** `cost = Σ(SATD) + λ × MVD_bits`

- Accumulates SATD across 8×8 blocks within a PU
- Supports 4 partitions: 16×16, 8×16, 16×8, 8×8
- `partition_i[42:0]` from IME encodes partition tree
- MVD bit cost from `getbits` (leading-one detection on absolute MV)
- Best candidate selection per partition: outputs `fmv_best_o`, `fmv_wren_o`, `fmv_addr_o`
- Final cost written to `db_mv_ram_sp_64x20` (64×20-bit SRAM) for skip comparison

---

## 9. MV Buffer — `fme_mv_buffer`

### Top MV (SRAM)

- `db_mv_ram_sp_512x20` — 512×20-bit single-port SRAM
- Stores FMVs for top CTU row (across picture width)
- Write: during PRE_MC→DONE_MC from `mv_wr_ena_i`
- Read: during PRE_HALF/PRE_QUAR/PRE_SKIP for candidate lookup
- Top-left MV corner: captured in `tl_mv_r` register

### Left MV (Register Array)

- `fmv_lft_r_0[7:0]` and `fmv_lft_r_1[7:0]` — two 8×20-bit register arrays
- Ping-pong buffer: `fme_ctu_x_i[0]` selects which bank to read/write
- Write: when `mv_wr_adr_d1[2:0] == 3'b111` (last 8×8 in column)
- Read: combinational from selected bank

---

## 10. MV Candidate Preparation — `fme_mv_candidate_prepare`

Checks existence of 5 spatial MV candidates:

```
    ----     -------
    |B2|     |B1|B0|
    -------------------
       |        |
       |   PU   |
    ---|        |
    |A1|        |
    -------------
    |A0|
    ----
```

- **FSM:** 6 states (IDLE, A0, A1, B0, B1, B2)
- Sequentially checks each candidate, reads MV from current/top/left buffer
- Outputs `mv_candi_XX_val_o` and `mv_candi_XX_dat_o` (5 candidates)
- Boundary checks against picture edges and CTU boundaries
- `IinP_flag_i` controls availability of top/left/TL/TR neighbors

---

## 11. Skip Mode — `fme_skip`

- Evaluates skip (0,0) MV cost for each 8×8 block within a CU
- Uses dedicated `fme_satd_8x8` instance for skip SATD
- Candidate priority: A1 → B1 → B0 → A0 → B2 (skip order, not AMVP order)
- Duplicate MV elimination (if A1==B1, B1 marked invalid)
- Cost threshold comparison: `cost_mv0_sum_r < skip_cost_thresh_XX`
  - Thresholds are CU-size dependent (8×8, 16×16, 32×32, 64×64)
- Skip flag/index stored in per-CU registers for all CU sizes
- `mc_skip_flg_o` output used during MC state to force (0,0) MV
- `skip_idx_o` (85×4) and `skip_flag_o` (85) output to system

---

## 12. Prediction — `fme_pred`

- MUXes predicted pixels from 9 candidates based on best candidate index
- Double-buffered: `best_r0`/`best_r1` with `flag` toggle
- Outputs `pred_pixel_o` [32×PIXEL_WIDTH], `pred_wren_o` [3:0], `pred_addr_o` [6:0]
- Byte-lane selection: `{cnt16[0], cnt08[0]}` selects which 8-byte lane

---

## 13. Utilities

### getbits.v
- Combinational leading-one detector
- Input: unsigned integer; Output: bit count (1–31)
- Used for MV bit cost in skip mode

### qp_lambda_table.v
- Pure combinational QP → λ mapping
- QP 0–15 → λ=1; QP 16–19 → λ=2; ... QP 51 → λ=91
- λ is 8-bit unsigned

---

## 14. Memory Map

| Instance | Type | Depth | Width | Ports | Description |
|----------|------|-------|-------|-------|-------------|
| `db_mv_ram_sp_512x20` | SRAM | 512 | 20 | 1R/1W (sp) | Top MV buffer |
| `db_mv_ram_sp_64x20` | SRAM | 64 | 20 | 1R/1W (sp) | FME cost RAM |
| `fmv_lft_r_0[7:0]` | Reg array | 8 | 20 | R/W | Left MV bank 0 |
| `fmv_lft_r_1[7:0]` | Reg array | 8 | 20 | R/W | Left MV bank 1 |
| `tl_mv_r` | Register | 1 | 20 | R/W | Top-left MV |

---

## 15. Clock and Reset

- **Clock:** Single clock domain `clk`
- **Reset:** Active-low asynchronous reset `rstn`
- All registers reset to 0 on `negedge rstn`
- FSM reset state: `IDLE` (state 0)

---

## 16. Pipeline Stages

| Stage | Source Reg | Dest Reg | Latency |
|-------|-----------|----------|---------|
| Ctrl→Interp | `ip_start_ctrl_r` | `ip_start_ctrl_w` | 1 cycle (DLY_NUM=1) |
| Ref pixel valid | `ref_rden_o` | `refpel_valid` | 1 cycle |
| Ref pixel valid→interp | `refpel_valid` | `refpel_valid_w` | 1 cycle (DLY_NUM=1) |
| Satd valid | `fme_satd_8x8` output | `satd_valid` | combinational from satd_start |
| Cost write | `fmv_wren` | cost RAM write | combinational |
| Skip cost | `cost_mv0_sum` accum | `cost_done_r` | variable (PU-size dependent) |

---

## 17. Evidence Traceability

| Block | Status | RTL Evidence |
|-------|--------|-------------|
| fme_top | VERIFIED | `fme_top.v:27` module declaration |
| fme_ctrl FSM | VERIFIED | `fme_ctrl.v` localparam IDLE..DONE_MC, case statements |
| fme_interpolator_8x8 | VERIFIED | `fme_top.v:465` instantiation as `ip8x8` |
| fme_interpolator_8pel | VERIFIED | `fme_interpolator_8pel.v` module declaration |
| fme_interpolator | VERIFIED | `fme_interpolator.v` module declaration |
| fme_ip_half_ver | VERIFIED | `fme_ip_half_ver.v` module, 17 interpolator instances |
| fme_ip_quarter_ver | VERIFIED | `fme_ip_quarter_ver.v` module, 48 interpolator instances |
| fme_satd_gen | VERIFIED | `fme_top.v:535` instantiation as `satd_gen` |
| fme_satd_8x8 | VERIFIED | `fme_satd_8x8.v` module declaration |
| fme_cost | VERIFIED | `fme_top.v:597` instantiation as `sp_cost` |
| fme_pred | VERIFIED | `fme_top.v:703` instantiation as `fme_pred` |
| fme_mv_buffer | VERIFIED | `fme_top.v:736` instantiation as `u_mv_buffer` |
| fme_mv_candidate_prepare | VERIFIED | `fme_top.v:755` instantiation as `u_mv_candidates` |
| fme_skip | VERIFIED | `fme_top.v:799` instantiation as `u_fme_skip` |
| qp_lambda_table | VERIFIED | `fme_top.v:460` instantiation |
| db_mv_ram_sp_64x20 | VERIFIED | `fme_top.v:639` instantiation |
| db_mv_ram_sp_512x20 | VERIFIED | `fme_mv_buffer.v:116` instantiation |
| getbits | INFERRED | `getbits.v` module exists but commented out in `fme_skip.v:731-753` |
| Cost RAM | VERIFIED | `fme_top.v:639` `db_mv_ram_sp_64x20` instantiation |

---

## 18. Cross-Check: Diagram vs RTL

| Diagram Block | RTL File | Verified |
|--------------|----------|----------|
| fme_top | fme_top.v | ✓ |
| fme_ctrl | fme_ctrl.v | ✓ |
| fme_interpolator_8x8 | fme_interpolator_8x8.v | ✓ |
| fme_interpolator_8pel | fme_interpolator_8pel.v | ✓ |
| fme_interpolator | fme_interpolator.v | ✓ |
| fme_ip_half_ver | fme_ip_half_ver.v | ✓ |
| fme_ip_quarter_ver | fme_ip_quarter_ver.v | ✓ |
| fme_satd_gen | fme_satd_gen.v | ✓ |
| fme_satd_8x8 | fme_satd_8x8.v | ✓ |
| fme_cost | fme_cost.v | ✓ |
| fme_pred | fme_pred.v | ✓ |
| fme_mv_buffer | fme_mv_buffer.v | ✓ |
| fme_mv_candidate_prepare | fme_mv_candidate_prepare.v | ✓ |
| fme_skip | fme_skip.v | ✓ |
| qp_lambda_table | qp_lambda_table.v | ✓ |
| db_mv_ram_sp_64x20 | fme_top.v (inst) | ✓ |
| db_mv_ram_sp_512x20 | fme_mv_buffer.v (inst) | ✓ |
| getbits | getbits.v (commented out) | ✓ |
