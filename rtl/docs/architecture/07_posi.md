---
title: POSI Subsystem Architecture
version: 1.0
created: 2026-09-04
status: DRAFT
---

# 07 — POSI Subsystem Architecture

## 1. Overview

POSI (Post Intra) performs intra mode decision for the current CTU. It evaluates multiple intra prediction modes across hierarchical block sizes (4×4 → 8×8 → 16×16 → 32×32) by:
1. Loading original pixels into internal row/column/frame RAMs
2. Extracting reference pixels (top, left, top-left, right, down) from these RAMs
3. Running the intra prediction engine for each mode
4. Computing residuals (original − predicted)
5. Computing SATD cost (2D Hadamard transform + absolute sum + rate estimation)
6. Selecting the best partition and mode

**Key Parameters** (from `enc_defines.v`):
- `PIXEL_WIDTH`: 8 bits
- `POSI_COST_WIDTH`: cost output width (from defines)
- `PIC_X_WIDTH`: 6, `PIC_Y_WIDTH`: 6 (CTU grid coordinates)
- `SIZE_04`, `SIZE_08`, `SIZE_16`, `SIZE_32`: block size encodings

## 2. Module Hierarchy

```
posi_top                                    (rtl/posi/posi_top.v:12)
├── posi_ctrl                              (rtl/posi/posi_ctrl.v:12)
│   └── [inline: 9-state FSM + position counter + mode/traversal counters]
├── posi_transfer                          (rtl/posi/posi_transfer.v:12)
│   └── [inline: 4-state FSM + pixel rearrangement + row/col/fra write]
├── posi_reference                         (rtl/posi/posi_reference.v:20)
│   └── [inline: raw→pad→flt pipeline + reference assembly]
├── posi_prediction                        (rtl/posi/posi_prediction.v:12)
│   └── intra_pred                         (rtl/rec/rec_intra/intra_pred.v)
├── posi_buffer                            (rtl/posi/posi_buffer.v:12)
│   └── [inline: residual calculation + 3-stage pipeline]
├── posi_satd_cost                         (rtl/posi/posi_satd_cost.v:17)
│   ├── posi_satd_cost_engine ×4           (rtl/posi/posi_satd_cost_engine.v)
│   ├── posi_satd_cost_transpose           (rtl/posi/posi_satd_cost_transpose.v)
│   └── posi_rate_estimation               (rtl/posi/posi_rate_estimation.v)
├── posi_partition_decision                (rtl/posi/posi_partition_decision.v)
└── posi_memory_wrapper                    (rtl/posi/posi_memory_wrapper.v)
    └── [inline: row/col/fra RAM wrappers]
```

## 3. Function

### 3.1 Actual Purpose (from RTL)

POSI evaluates intra prediction modes for the current CTU by:
- Iterating through 4×4 block positions within the CTU
- For each position, evaluating `num_mode_i` intra modes (up to 35 HEVC intra modes)
- Computing SATD cost for each mode at each block size
- Tracking best mode per 4×4 block and best partition overall
- Outputting final partition map and best cost

### 3.2 Processing Flow

```
start_i → Transfer (read original → row/col/fra RAMs)
        → Reference (load refs, pad, filter)
        → Prediction (intra_pred engine)
        → Buffer (compute residuals: original − predicted)
        → SATD Cost (2D Hadamard + abs + sum + rate)
        → Partition Decision (track best per block + best partition)
        → done_o
```

### 3.3 Block Size Hierarchy

The controller traverses block sizes in order:
1. **SIZE_04** (4×4): 16 positions per 16×16 area, 1 mode at a time
2. **SIZE_08** (8×8): 4 positions per 16×16 area, after all 4×4 done
3. **SIZE_16** (16×16): 1 position per 16×16 area, after all 8×8 done
4. **SIZE_32** (32×32): 1 position per CTU, after all 16×16 done

Then: **WAIT** (pipeline flush) → **DECI** (partition decision) → **TRA_POS** (write best mode) → done

## 4. Datapath

### 4.1 Transfer Stage (`posi_transfer`)

**Input**: Original pixels from external buffer (256-bit = 32×8b per read)

**Processing**:
- Reads 4×4 blocks of original pixels in raster scan order
- Rearranges into three internal formats:
  - **row_wr**: Row-organized pixels for horizontal reference
  - **col_wr**: Column-organized pixels for vertical reference
  - **fra_wr**: Frame-organized pixels for frame reference

**4-State FSM** (posi_transfer.v:180-201):
| State | Description |
|-------|-------------|
| IDLE | Waiting for start |
| PRE | Load original pixels for reference (row + col + fra) |
| POS_COL | Load column data for post-processing |
| POS_FRA | Load frame data for post-processing |

**Pixel Rearrangement** (posi_transfer.v:134-165):
- Input 256-bit = 32 pixels (4×8 block, two 4-pixel groups A and B)
- `row_wr_dat_o`: bottom row of 4×4 block (pixels a_3_0..a_3_3)
- `col_wr_dat_o`: right column of 4×4 block (pixels a_0_3, a_1_3, a_2_3, a_3_3)
- `fra_wr_dat_o`: top row of 4×4 block (pixels a_3_0..a_3_3 at y=15)

**2-Cycle Pipeline Delay**: val_dly_r, cur_state_dly_r, idx_4x4_x/y_dly_r for write enable timing

### 4.2 Reference Stage (`posi_reference`)

**Input**: Reference pixels from row/col/fra RAMs + mode RAM

**Processing**: Assembles reference borders (top, left, top-left, right, down) with padding and filtering.

**Reference Assembly Pipeline** (from header comment):
```
| i         | d0        | d1        | d2        | d3  | d4     |
| raw_pre_1 | raw_pre_0 | raw       |           |     |        |
|           |           | pad_pre   | pad       |     |        |
|           | flt_pre_2 | flt_pre_1 | flt_pre_0 | flt |        |
|           |           |           |           |     | output |
```

**Reference Borders** (each 32 pixels × 8b = 256b):
- `ref_t_o` (top): 32 pixels above the current block
- `ref_l_o` (left): 32 pixels to the left of the current block
- `ref_r_o` (right): 32 pixels to the right
- `ref_d_o` (down): 32 pixels below
- `ref_tl_o` (top-left): 1 pixel at top-left corner

**Mode RAM Access** (posi_reference.v:298-300):
- Reads from `mod_rd_ena_o`/`mod_rd_adr_o`/`mod_rd_dat_i` — external mode decision RAM (9-bit address, 6-bit mode data)

**Address Calculation** (posi_reference.v:369-370):
- `idx_4x4_x_w = {position_i[6],position_i[4],position_i[2],position_i[0]}` (bit-interleaved X)
- `idx_4x4_y_w = {position_i[7],position_i[5],position_i[3],position_i[1]}` (bit-interleaved Y)

### 4.3 Prediction Stage (`posi_prediction`)

**Input**: Reference borders (T, L, TL, R, D) + mode/size/position

**Processing**: Instantiates the shared `intra_pred` engine (from `rec/rec_intra/intra_pred.v`) to compute predicted pixels.

**Key Connections** (posi_prediction.v:260-316):
- `intra_pred` receives `pre_sel_i = TYPE_Y` (luma only)
- Iterates through 4×4 sub-blocks via `pre_cnt_r` counter
- For each sub-block, computes `idx_4x4_x/y_offset_w` from `pre_cnt_r` bits

**3-Cycle Pipeline Delay** (posi_prediction.v:321-338):
- val_r, mode_r, size_r, position_r, idx_4x4_x/y_ahd_r
- `val_ahd_o` = 1 cycle ahead (for buffer read address)
- `val_o` = final valid (for buffer compute)

### 4.4 Buffer Stage (`posi_buffer`)

**Input**: Predicted pixels (16×8b = 128b) + original pixels (via ori_rd interface)

**Processing**: Computes residuals: `residual = original − predicted`

**Residual Computation** (posi_buffer.v:288-335):
- `dat_dif_d0_X_Y_r = {1'b0, original_X_Y} - {1'b0, predicted_X_Y}` (9-bit signed)
- 3-stage pipeline: d0 → d1 → d2

**Output MUX** (posi_buffer.v:365-382):
- SIZE_04: single 4×4 residual block from d1
- Larger sizes: two 4×4 blocks (cnt_r==0: top half from d1+d0; cnt_r==1: bottom half from d2+d1)

**Read Interface** (posi_buffer.v:236-240):
- Issues `ori_rd_ena_o` 1 cycle ahead (via `val_ahd_i`) to read original pixels
- Address from `idx_4x4_x/y_ahd_i`

### 4.5 SATD Cost Stage (`posi_satd_cost`)

**Input**: Residual blocks (16×(PIXEL_WIDTH+1) bits) + QP

**Processing**: 2D Hadamard transform + absolute value + summation + rate estimation

**Pipeline Stages**:
1. **1D Transform** (7 cycles): Two parallel `posi_satd_cost_engine` instances process top/bottom halves
2. **Transpose** (via `posi_satd_cost_transpose`): Reorders 4×4 blocks for 2D transform
3. **2D Transform** (7 cycles): Two more `posi_satd_cost_engine` instances
4. **Summation** (3 cycles): abs → partial sums → final sum
5. **Rate Estimation**: `posi_rate_estimation` computes bitrate from mode/size/position/QP

**Total SATD Latency**: DELAY_SATD(7) + DELAY_SUMM(3) = 10 cycles per 4×4 block

**Cost Calculation** (posi_satd_cost.v:484):
- `dat_o = dat_sum_shift_r + bitrate_w` (SATD + rate)

### 4.6 Partition Decision (`posi_partition_decision`)

**Input**: Mode/size/position + cost value for each candidate

**Processing**: Tracks best mode per 4×4 block and best partition for the CTU

**Outputs**:
- `partition_o [85:0]`: Partition decision bits
- `mod_wr_ena_o/mod_wr_adr_o/mod_wr_dat_o`: Best mode write to external mode RAM
- `bst_cost_o`: Best cost value

### 4.7 Memory Wrapper (`posi_memory_wrapper`)

Wraps row/col/fra RAMs used by `posi_transfer` (write) and `posi_reference` (read).

**RAM Types**:
- `row_ram`: Row-organized reference pixels (8-bit × depth)
- `col_ram`: Column-organized reference pixels (8-bit × depth)
- `fra_ram`: Frame-organized reference pixels (8-bit × depth)

## 5. Control

### 5.1 Main FSM (`posi_ctrl`)

**RTL**: `posi_ctrl.v:116-158`

9-state FSM (`cur_state_r`/`nxt_state_w`):

| State | Value | Description |
|-------|-------|-------------|
| IDLE | 0 | Waiting for start |
| TRA_PRE | 1 | Transfer original pixels for reference |
| SIZE_04 | 2 | Process 4×4 blocks |
| SIZE_08 | 3 | Process 8×8 blocks |
| SIZE_16 | 4 | Process 16×16 blocks |
| SIZE_32 | 5 | Process 32×32 blocks |
| WAIT | 6 | Pipeline flush |
| DECI | 7 | Partition decision |
| TRA_POS | 8 | Write best mode to external |

**State Transitions** (posi_ctrl.v:116-158):
- IDLE → TRA_PRE (on start_i)
- TRA_PRE → SIZE_04 (on done_tra_i)
- SIZE_04 → SIZE_08 (when position_r[1:0]==2'b11)
- SIZE_08 → SIZE_16 (when position_r[3:0]==4'b1100) or WAIT
- SIZE_16 → SIZE_32 (when position_r[5:0]==6'b110000) or WAIT
- SIZE_32 → DECI (when position_r[7:0]==8'b11000000) or WAIT
- WAIT → SIZE_04 (on cnt_wait_done_w)
- DECI → TRA_POS (on done_dec_i)
- TRA_POS → IDLE (on done_tra_i)

**Position Counter** (posi_ctrl.v:162-192):
- 8-bit `position_r` tracks current 4×4 block position
- Incremented by 1 (SIZE_04), 4 (SIZE_08), 16 (SIZE_16), 64 (SIZE_32)
- Reset at state boundaries

**Mode/Traversal Counters** (posi_ctrl.v:217-282):
- `cnt_mode_r [2:0]`: Current mode index (0 to num_mode_i)
- `cnt_trav_r [5:0]`: Traversal count within current block size
- `done_size_w = ref_done_w && pre_done_w` (both reference and prediction complete)

**Wait Counter** (posi_ctrl.v:302-327):
- `cnt_wait_r [4:0]`: Pipeline flush delay
- Duration depends on previous size:
  - SIZE_08: `(2-1)*2-1 = 1` cycle
  - SIZE_16: `(4-1)*2 = 6` cycles
  - SIZE_32: `(8-1)*2 = 14` cycles

### 5.2 Transfer FSM (`posi_transfer`)

**RTL**: `posi_transfer.v:180-201`

4-state FSM:
| State | Description |
|-------|-------------|
| IDLE | Waiting for start |
| PRE | Load original pixels (row+col+fra) |
| POS_COL | Load column data (positional) |
| POS_FRA | Load frame data (positional) |

### 5.3 Reference FSMs (`posi_reference`)

**RTL**: `posi_reference.v:418-440` (raw), `posi_reference.v:300-301` (flt)

Two 1-state FSMs:
- `cur_state_raw_r`: RAW_IDLE/RAW_BUSY — controls raw reference loading
- `cur_state_flt_r`: FLT_IDLE/FLT_BUSY — controls filtered reference output

### 5.4 Prediction FSM (`posi_prediction`)

**RTL**: `posi_prediction.v:197-218`

2-state FSM (IDLE/BUSY):
- Iterates `pre_cnt_r` through all 4×4 sub-blocks
- Done when `pre_cnt_done_w` (depends on block size)

## 6. Memory

### 6.1 Internal RAMs (via `posi_memory_wrapper`)

| RAM | Width | Depth | Ports | Purpose |
|-----|------:|------:|-------|---------|
| row_ram | 32 × 8b | 256 rows | 1W/1R | Row-organized reference pixels |
| col_ram | 32 × 8b | 256 rows | 1W/1R | Column-organized reference pixels |
| fra_ram | 32 × 8b | 256 rows | 1W/1R | Frame-organized reference pixels |

### 6.2 Register Arrays in `posi_reference`

The reference module uses extensive register arrays (NOT RAM) for buffering:
- `ref_raw_t00..t31_r` (32 × 8b): Raw top reference
- `ref_raw_r00..r31_r` (32 × 8b): Raw right reference
- `ref_raw_l00..l31_r` (32 × 8b): Raw left reference
- `ref_raw_d00..d31_r` (32 × 8b): Raw down reference
- `ref_raw_tl_r` (8b): Raw top-left reference
- Same sets for `ref_pad_*_r` (padded) and `ref_flt_*_r` (filtered)

### 6.3 Register Arrays in `posi_buffer`

Residual register arrays (3-stage pipeline):
- `dat_dif_d0_X_Y_r` (10b): Current cycle residuals
- `dat_dif_d1_X_Y_r` (10b): 1-cycle delayed residuals
- `dat_dif_d2_X_Y_r` (10b): 2-cycle delayed residuals (for bottom rows)

## 7. Pipeline

### 7.1 Pipeline Stages

POSI has a multi-stage pipeline:

```
┌─────────────┐   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
│   Transfer   │ → │  Reference   │ → │  Prediction  │ → │    Buffer    │ → │  SATD Cost   │ → │   Decision   │
│  (4 states)  │   │ (raw→pad→flt)│   │ (intra_pred) │   │ (residual)   │   │ (Hadamard+Σ) │   │ (best mode)  │
└─────────────┘   └──────────────┘   └──────────────┘   └──────────────┘   └──────────────┘   └──────────────┘
     1 cycle           5 cycles           3 cycles           2 cycles           10 cycles           1 cycle
```

### 7.2 Latency

The total latency per mode per block size is approximately:
- Transfer: ~256 cycles (16×16 pixels, 32 pixels/cycle)
- Reference: variable (depends on padding/filtering)
- Prediction: ~(N×N/16) cycles (N = block size)
- Buffer: ~(N×N/16) + 2 cycles
- SATD Cost: 10 cycles per 4×4 block

### 7.3 Throughput

POSI processes one CTU at a time. Within a CTU, it evaluates all modes for all block sizes sequentially.

## 8. Interfaces

### 8.1 External Control Interface

| Source | Signal | Width | Destination | Type | Description |
|--------|--------|------:|-------------|------|-------------|
| enc_ctrl | `start_i` | 1 | posi_top | CONTROL | Start POSI processing |
| posi_top | `done_o` | 1 | enc_ctrl | CONTROL | POSI processing complete |
| enc_ctrl | `num_mode_i` | 3 | posi_top | CONTROL | Number of modes to evaluate |
| enc_ctrl | `qp_i` | 6 | posi_top | CONTROL | Quantization parameter |

### 8.2 CTU Configuration Interface

| Source | Signal | Width | Destination | Type | Description |
|--------|--------|------:|-------------|------|-------------|
| enc_ctrl | `ctu_x_all_i` | PIC_X_WIDTH | posi_top | ADDRESS | CTU grid width |
| enc_ctrl | `ctu_y_all_i` | PIC_Y_WIDTH | posi_top | ADDRESS | CTU grid height |
| enc_ctrl | `ctu_x_res_i` | 4 | posi_top | ADDRESS | CTU X residual |
| enc_ctrl | `ctu_y_res_i` | 4 | posi_top | ADDRESS | CTU Y residual |
| enc_ctrl | `ctu_x_cur_i` | PIC_X_WIDTH | posi_top | ADDRESS | Current CTU X |
| enc_ctrl | `ctu_y_cur_i` | PIC_Y_WIDTH | posi_top | ADDRESS | Current CTU Y |

### 8.3 Original Pixel Read Interface (posi → fetch_cur_luma)

| Source | Signal | Width | Destination | Type | Description |
|--------|--------|------:|-------------|------|-------------|
| posi_top | `ori_rd_ena_o` | 1 | fetch | CONTROL | Read enable |
| posi_top | `ori_rd_sel_o` | 2 | fetch | ADDRESS | Component select (TYPE_Y) |
| posi_top | `ori_rd_siz_o` | 2 | fetch | ADDRESS | Block size |
| posi_top | `ori_rd_4x4_x_o` | 4 | fetch | ADDRESS | 4×4 block X index |
| posi_top | `ori_rd_4x4_y_o` | 4 | fetch | ADDRESS | 4×4 block Y index |
| posi_top | `ori_rd_idx_o` | 5 | fetch | ADDRESS | Block index |
| fetch | `ori_rd_dat_i` | 256 | posi_top | DATA | Original pixel data (32×8b) |

### 8.4 Mode RAM Interface (posi ↔ mode decision RAM)

| Source | Signal | Width | Destination | Type | Description |
|--------|--------|------:|-------------|------|-------------|
| posi_ref | `mod_rd_ena_o` | 1 | mode RAM | CONTROL | Read enable |
| posi_ref | `mod_rd_adr_o` | 9 | mode RAM | ADDRESS | Read address |
| mode RAM | `mod_rd_dat_i` | 6 | posi_ref | DATA | Mode data |
| posi_dec | `mod_wr_ena_o` | 1 | mode RAM | CONTROL | Write enable |
| posi_dec | `mod_wr_adr_o` | 8 | mode RAM | ADDRESS | Write address |
| posi_dec | `mod_wr_dat_o` | 6 | mode RAM | DATA | Best mode data |

### 8.5 Output Interface

| Source | Signal | Width | Destination | Type | Description |
|--------|--------|------:|-------------|------|-------------|
| posi_top | `partition_o` | 85 | enc_ctrl | DATA | Partition decision |
| posi_top | `cost_o` | POSI_COST_WIDTH | enc_ctrl | DATA | Best cost |

## 9. Diagrams

See `diagrams/posi/posi_hierarchy.mmd` and `diagrams/posi/posi_rtl.mmd`.

## 10. RTL Evidence

### 10.1 Module Hierarchy
- **Status**: VERIFIED
- **RTL**: `posi_top.v:206-461`
- **Evidence**: 7 child module instantiations: posi_ctrl, posi_transfer, posi_reference, posi_prediction, posi_buffer, posi_satd_cost, posi_partition_decision, posi_memory_wrapper

### 10.2 Control FSM
- **Status**: VERIFIED
- **RTL**: `posi_ctrl.v:116-158` (FSM), `posi_ctrl.v:162-192` (position), `posi_ctrl.v:217-282` (counters)
- **Evidence**: 9-state FSM with explicit IDLE through TRA_POS states, position counter, mode counter, traversal counter

### 10.3 Transfer FSM
- **Status**: VERIFIED
- **RTL**: `posi_transfer.v:180-201`
- **Evidence**: 4-state FSM (IDLE/PRE/POS_COL/POS_FRA), pixel rearrangement at lines 134-165

### 10.4 Reference Assembly Pipeline
- **Status**: VERIFIED
- **RTL**: `posi_reference.v:418-1067+`
- **Evidence**: raw→pad→flt pipeline, register arrays for ref_raw_t/r/l/d, fra/row/col RAM read interfaces

### 10.5 Prediction Engine
- **Status**: VERIFIED
- **RTL**: `posi_prediction.v:260-316`
- **Evidence**: `intra_pred` instantiation with TYPE_Y, 3-cycle pipeline delay

### 10.6 Residual Computation
- **Status**: VERIFIED
- **RTL**: `posi_buffer.v:288-335`
- **Evidence**: `dat_dif_d0/d1/d2_X_Y_r` registers, subtraction logic `{1'b0,ori} - {1'b0,pre}`

### 10.7 SATD Cost Engine
- **Status**: VERIFIED
- **RTL**: `posi_satd_cost.v:183-272`
- **Evidence**: 4× `posi_satd_cost_engine` instances (1D×2 + 2D×2), `posi_satd_cost_transpose`, `posi_rate_estimation`

### 10.8 Memory Wrapper
- **Status**: VERIFIED
- **RTL**: `posi_memory_wrapper.v` (inferred from posi_top.v:433-461)
- **Evidence**: row/col/fra write and read port connections

## 11. Uncertainties

1. **`intra_pred` internal implementation**: The actual intra prediction modes (Planar, DC, Angular 2-34) are implemented in `rec/rec_intra/intra_pred.v` which is shared with the REC subsystem. The exact prediction computation is not analyzed here.

2. **`posi_memory_wrapper` RAM implementation**: The exact memory technology (register array vs inferred RAM) is defined inside `posi_memory_wrapper.v` which wraps row/col/fra RAMs.

3. **`posi_partition_decision` internal logic**: The partition decision algorithm (how best mode per 4×4 block is combined into final partition) is in `posi_partition_decision.v` — detailed analysis pending.

4. **`posi_rate_estimation` model**: The rate estimation model (how mode/size/position map to bitrate) is in `posi_rate_estimation.v` — detailed analysis pending.

5. **`posi_satd_cost_engine` Hadamard implementation**: The actual Hadamard transform kernel is in `posi_satd_cost_engine.v` — detailed analysis pending.

6. **Position bit interleaving**: The `position_r` uses bit-interleaved format where `position_i[7:0]` encodes `{y3,x3,y2,x2,y1,x1,y0,x0}` — this is VERIFIED from the bit extraction in `posi_reference.v:369-370`.
