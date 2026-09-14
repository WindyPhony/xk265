---
title: xk265 RTL System Architecture
version: 1.0
created: 2026-09-04
status: DRAFT
---

# System Architecture

## 1. Overview

The xk265 encoder is a pipelined HEVC hardware encoder implemented in pure Verilog RTL. The design processes LCU (Largest Coding Unit, 64×64 pixels) tiles in a deep pipeline with 8 processing stages. Each stage operates on a different LCU concurrently, achieving high throughput.

**Key Parameters** (from `enc_defines.v`):
- `LCU_SIZE`: 64
- `CU_DEPTH`: 3 (64→32→16→8)
- `PIXEL_WIDTH`: 8 bits
- `INIT_QP`: 22
- `PIC_X_WIDTH`: 6, `PIC_Y_WIDTH`: 6 (max 64×64 CTU grid)

## 2. Top-Level Module Hierarchy

```
h265enc_top                          (rtl/top/enc_top.v)
├── enc_ctrl                         (rtl/top/enc_ctrl.v)    — Pipeline FSM controller
├── fetch_top                        (rtl/fetch/fetch_top.v) — Data fetch subsystem
└── enc_core                         (rtl/top/enc_core.v)    — Datapath (all processing stages)
    ├── prei_top_buf                 (rtl/top/prei_top_buf.v)
    │   └── prei_top                 (rtl/prei/prei_top.v)
    ├── posi_top_buf                 (rtl/top/posi_top_buf.v)
    │   └── posi_top                 (rtl/posi/posi_top.v)
    ├── ime_top_buf                  (rtl/top/ime_top_buf.v)
    │   └── ime_top                  (rtl/ime/ime_top.v)
    ├── fme_top_buf                  (rtl/top/fme_top_buf.v)
    │   └── fme_top                  (rtl/fme/fme_top.v)
    ├── rec_top                      (rtl/rec/rec_top.v)
    ├── dbsao_top                    (rtl/db/dbsao_top.v)
    ├── cabac_top                    (rtl/cabac/cabac_top.v)
    └── enc_data_pipeline            (rtl/top/enc_data_pipeline.v)
```

## 3. Pipeline Architecture

### 3.1 Pipeline FSM (`enc_ctrl`)

The pipeline is controlled by a 12-state FSM in `enc_ctrl.v:116-127`:

| State | Value | Active Stages | Description |
|-------|-------|---------------|-------------|
| IDLE  | 0     | —             | Waiting for start |
| S0    | 1     | fetch         | Fetch only (first CTU fill) |
| S1    | 2     | fetch, stg_1  | + PREI (intra) or PREI+IME (inter) |
| S2    | 3     | fetch, stg_1, stg_2 | + POSI (intra) or POSI+FME (inter) |
| S3    | 4     | fetch, stg_1, stg_2, rec | + Reconstruction |
| S4    | 5     | fetch, stg_1, stg_2, rec, db | + Deblocking |
| S5    | 6     | all stages    | Full pipeline (stall until last CTU fetched) |
| S6    | 7     | stg_1–db, ec  | Drain fetch, enter EC |
| S7    | 8     | stg_2–ec      | Drain stg_1 |
| S8    | 9     | rec–ec        | Drain stg_2 |
| S9    | 10    | db, ec        | Drain rec |
| SA    | 11    | ec            | Drain db, finish EC |

### 3.2 Pipeline Stages

The pipeline has 8 logical stages, mapped to 8 FSM enable groups:

| Stage | Start Signal | Done Signal | Subsystem | Description |
|-------|-------------|-------------|-----------|-------------|
| 0: Fetch | — | `fetch_done_i` | `fetch_top` | Load current/ref/store DB from external memory |
| 1a: PREI | `prei_start_o` | `prei_done_i` | `prei_top_buf` | Intra mode pre-estimation (always active) |
| 1b: IME | `ime_start_o` | `ime_done_i` | `ime_top_buf` | Integer motion estimation (inter only) |
| 2a: POSI | `posi_start_o` | `posi_done_i` | `posi_top_buf` | Intra mode decision + RD cost |
| 2b: FME | `fme_start_o` | `fme_done_i` | `fme_top_buf` | Fractional motion estimation (inter only) |
| 3: REC | `rec_start_o` | `rec_done_i` | `rec_top` | Transform, quantization, inverse, recon |
| 4: DB/SAO | `db_start_o` | `db_done_i` | `dbsao_top` | Deblocking filter + SAO |
| 5: CABAC | `ec_start_o` | `ec_done_i` | `cabac_top` | Entropy coding (bitstream output) |

**Stage grouping** (`enc_ctrl.v:437-438`):
- `stg_1_flg`: INTRA → `prei_flg`; INTER → `prei_flg && ime_flg`
- `stg_2_flg`: INTRA → `posi_flg`; INTER → `posi_flg && fme_flg`

### 3.3 Pipeline Data Flow (Inter Mode)

```
CTU N:    [FETCH] → [PREI+IME] → [POSI+FME] → [REC] → [DB] → [CABAC]
CTU N+1:       [FETCH] → [PREI+IME] → [POSI+FME] → [REC] → [DB] → [CABAC]
CTU N+2:              [FETCH] → [PREI+IME] → [POSI+FME] → [REC] → [DB] → [CABAC]
...
```

At steady state, up to 8 CTUs are in-flight simultaneously.

### 3.4 CTU Coordinate Pipeline

Each stage receives its own CTU (x,y) coordinate, shifted one cycle behind the previous stage (`enc_ctrl.v:529-566`):

```
prei_x/y ← pre_l_x/y    (current fetch position)
posi_x/y ← prei_x/y     (1 CTU behind prei)
ime_x/y  ← pre_l_x/y    (same as prei for inter)
fme_x/y  ← ime_x/y      (1 CTU behind ime)
rec_x/y  ← fme_x/y      (1 CTU behind fme)
db_x/y   ← rec_x/y      (1 CTU behind rec)
ec_x/y   ← db_x/y       (1 CTU behind db)
```

## 4. Subsystem Descriptions

### 4.1 Fetch Subsystem (`fetch_top`)

**RTL**: `rtl/fetch/fetch_top.v`

**Purpose**: Manages all external memory access — loading current frame pixels, reference frame pixels, storing deblocked output, and loading deblocked reconstruction from previous CTUs.

**Internal Structure**:
- `fetch_wrapper` — External IF sequencing FSM; arbitrates between 8 load/store channels via time-division multiplexing
- `fetch_cur_luma` — Current frame luma pixel buffer (32×256-bit = 1KB); serves PREI, POSI, IME, FME, REC, DB
- `fetch_ref_luma` — Reference frame luma pixel buffer (128×256-bit); serves IME, FME
- `fetch_cur_chroma` — Current frame chroma pixel buffer; serves REC, DB
- `fetch_ref_chroma` — Reference frame chroma pixel buffer; serves REC
- `fetch_db` — Deblock reconstruction pixel buffer; read/write for DB and store-to-external

**Key Interfaces**:
- 8 external memory channels: `load_cur_luma`, `load_ref_luma`, `load_cur_chroma`, `load_ref_chroma`, `load_db_luma`, `load_db_chroma`, `store_db_luma`, `store_db_chroma`
- Pixel read ports to: PREI, POSI, IME (current+reference), FME (current+reference), REC (current+reference), DB (current+reconstruction)
- DB write/read for reconstruction storage

### 4.2 PREI — Pre-Intra Estimation (`prei_top_buf`)

**RTL**: `rtl/top/prei_top_buf.v` → `rtl/prei/prei_top.v`

**Purpose**: Fast intra mode pre-estimation using Sobel gradient operators. Reduces the 35 HEVC intra modes to a shortlist for POSI.

**Key Details**:
- 40-cycle processing per 8×8 block, 65 blocks per CTU → ~2,665 cycles
- Sobel gradient (gx, gy) → 32 angular mode costs → hierarchical best angle selection
- DC/Planar vs angular decision with magnitude thresholds (DC=288, Planar=32)
- Mode results stored in `prei_md_ram_sp_85x6` (inferred rf_2p RAM)
- Ping-pong mode RAM via `sel_mod_2_i` for CONCURRENT read by POSI

**Buffer Wrapper** (`prei_top_buf`):
- Instantiates 2× `prei_md_ram_sp_85x6` (mode storage ping-pong)
- `sel_mod_2_i` switches write vs POSI read port

### 4.3 IME — Integer Motion Estimation (`ime_top_buf`)

**RTL**: `rtl/top/ime_top_buf.v` → `rtl/ime/ime_top.v`

**Purpose**: Full-pel motion estimation with configurable search patterns (center, slope, direction).

**Key Details**:
- 5-state FSM: IDLE→UPDATE→BUSY_ADR→BUSY_DEC→BUSY_DMP
- 4 partitions: 2N×2N, 1N×2N, 2N×1N, 1N×1N
- Up to 8 search commands per CTU (CMD_NUM_WIDTH=3)
- 4 search directions (quadrants): down, right, up, left
- Zigzag scan pattern with slope control
- 4× downsampling for fast search
- MV storage: 2× `ime_mv_ram_sp_64x13` ping-pong
- IME_HAS_VER_MEM enabled (vertical memory optimization)

**Buffer Wrapper** (`ime_top_buf`):
- 2× `ime_mv_ram_sp_64x13` ping-pong MV storage
- `sel_mod_2_i` switches write vs FME read port
- Pixel width conversion (4→8 bit)
- FME MV sign extension

### 4.4 POSI — Post-Intra Decision (`posi_top_buf`)

**RTL**: `rtl/top/posi_top_buf.v` → `rtl/posi/posi_top.v`

**Purpose**: Full intra mode decision with RD cost calculation, using modes from PREI.

**Key Details**:
- Reads mode candidates from PREI mode RAM
- Full pixel comparison with current block (4×4 granularity)
- Outputs: `posi_partition_o` (85 bits, CU/PU partition tree), `posi_cost_o` (20-bit RD cost)
- Mode results written to 4× `posi_md_ram_sp_64x6` with rotation

**Buffer Wrapper** (`posi_top_buf`):
- 4× `posi_md_ram_sp_64x6` mode storage with 4-way rotation (`sel_mode_4_r`)
- Rotation scheme: `{wr, rec, blank, ec}` across 4 RAMs, cycling each `sys_start_i`
- Dual read ports: `rec_md_rd_*` for REC, `ec_md_rd_*` for CABAC

### 4.5 FME — Fractional Motion Estimation (`fme_top_buf`)

**RTL**: `rtl/top/fme_top_buf.v` → `rtl/fme/fme_top.v`

**Purpose**: Sub-pel motion refinement using interpolated reference pixels.

**Key Details**:
- Reads IME MV results from IME MV RAM
- 3× `fme_mv_ram_dp_64x20` for triple-buffered MV storage (3-way rotation via `sel_mod_3_i`)
- 2× `fme_buf_wrapper` for MC prediction pixel double-buffering
- Outputs: `fme_cost_o` (20-bit), `skip_idx_o`/`skip_flag_o` (skip mode decision)
- Interfaces: current pixels, reference pixels, MV read (for REC/DB)

**Buffer Wrapper** (`fme_top_buf`):
- 3× `fme_mv_ram_dp_64x20` triple-rotation MV memory
- 2× `fme_buf_wrapper` MC prediction double-buffer
- `sel_mod_2_i` for MC pred buffer selection
- `sel_mod_3_i` for MV memory rotation

### 4.6 REC — Reconstruction (`rec_top`)

**RTL**: `rtl/rec/rec_top.v`

**Purpose**: Transform, quantization, inverse transform, inverse quantization, reconstruction, residual coding.

**Key Details**:
- Reads current pixels from fetch, modes from POSI, MV from FME
- Produces reconstructed pixels for DB
- Produces CBF (coded block flag) per channel: `cbf_y_w`, `cbf_u_w`, `cbf_v_w` (256 bits each)
- Outputs coefficient data for CABAC: `ec_coe_rd_*`
- Outputs MVD data for CABAC: `ec_mvd_rd_*`
- IinP mode support: compares intra vs inter cost

### 4.7 DB/SAO — Deblocking Filter + SAO (`dbsao_top`)

**RTL**: `rtl/db/dbsao_top.v`

**Purpose**: In-loop deblocking filter and sample adaptive offset.

**Key Details**:
- Reads reconstructed pixels from REC
- Reads original pixels from fetch for filtering decisions
- Reads MV data from FME MV RAM for boundary strength calculation
- Writes filtered pixels back to fetch reconstruction buffer
- Outputs SAO data to CABAC: `sao_data_w` (62 bits)
- DB starts after REC completes for previous CTU; stores result to external memory via `fetch_wen_o`

### 4.8 CABAC — Entropy Coding (`cabac_top`)

**RTL**: `rtl/cabac/cabac_top.v`

**Purpose**: Context-adaptive binary arithmetic coding of syntax elements.

**Key Details**:
- Final stage — reads all coding decisions from pipeline
- Reads modes from POSI mode RAM
- Reads coefficients from REC
- Reads MVD from REC
- Reads partition/skip/CBF from `enc_data_pipeline`
- Reads SAO data from DB/SAO
- Outputs: `bs_data_o` (8-bit bitstream), `bs_val_o` (bitstream valid)
- Slice-level output

### 4.9 Data Pipeline Registers (`enc_data_pipeline`)

**RTL**: `rtl/top/enc_data_pipeline.v`

**Purpose**: Shift registers that propagate coding parameters through the pipeline stages, aligned with CTU flow.

**Pipelined Parameters**:
| Parameter | Width | Stages |
|-----------|-------|--------|
| QP | 6 | rc→posi→fme→rec→db→ec |
| Intra partition | 85 | posi→rec→db→ec |
| Inter partition | 42 | ime→fme→rec→db→ec |
| CBF (Y/U/V) | 256 each | rec→db→ec |
| Skip flag | 85 | fme→rec→db→ec |
| IinP flag | 3 | rec→db→ec |
| Actual bitnum | 16 | cabac→rc (feedback) |

All registered on `enc_done_i` (pipeline advance pulse).

## 5. Memory Architecture

### 5.1 Buffer Rotation Schemes

The design uses three rotation schemes to decouple pipeline stages:

**Ping-Pong (sel_mod_2)**: 2 buffers, toggled per CTU
- PREI mode RAM: 2× `prei_md_ram_sp_85x6`
- IME MV RAM: 2× `ime_mv_ram_sp_64x13`
- FME MC pred buffer: 2× `fme_buf_wrapper`

**3-Way Rotation (sel_mod_3)**: 3 buffers, rotated per CTU
- FME MV RAM: 3× `fme_mv_ram_dp_64x20`
- Functions: FME write, REC/DB MV read, FME MV read (next CTU)

**4-Way Rotation (sel_mode_4)**: 4 buffers, rotated per CTU
- POSI mode RAM: 4× `posi_md_ram_sp_64x6`
- Functions: POSI write, REC mode read, blank, CABAC mode read

### 5.2 External Memory Interface

`fetch_wrapper` arbitrates 8 DMA channels through a shared external interface (`ext_if`):
- `extif_start_o`, `extif_done_i`
- `extif_mode_o` (5-bit channel selector)
- `extif_x_o`, `extif_y_o` (address)
- `extif_width_o`, `extif_height_o` (burst size)
- `extif_rden_i`/`extif_wren_i` + `extif_data_i`/`extif_data_o` (128-bit data bus)

## 6. Control Architecture

### 6.1 Start/Done Handshake

Each subsystem uses a start/done handshake:
1. `enc_ctrl` asserts `xxx_start_o` for one cycle when the pipeline stage is enabled
2. Subsystem processes the CTU
3. Subsystem asserts `xxx_done_o` for one cycle when complete
4. `enc_ctrl` latches `xxx_flg` (sticky flag, stays high until `enc_done_r`)
5. `enc_done_r` asserted when all required flags match expected pattern for current state

### 6.2 enc_done_r Pattern (`enc_ctrl.v:448-461`)

| State | Required Pattern `{fetch,stg1,stg2,rec,db,ec}` |
|-------|------------------------------------------------|
| S0    | `1_00_00` |
| S1    | `1_10_00` |
| S2    | `1_11_00` |
| S3    | `1_11_10` |
| S4    | `1_11_11` |
| S5    | `1_11_11` |
| S6    | `1_11_11` |
| S7    | `1_01_11` |
| S8    | `1_00_11` |
| S9    | `1_000_11` |
| SA    | `1_000_01` |

### 6.3 Fetch Enable Generation (`enc_ctrl.v:331-352`)

Fetch channels are enabled based on pipeline stage and frame type:
- `load_cur_luma_ena_o`: `pre_l_ena` (always during PREI/IME)
- `load_ref_luma_ena_o`: `pre_l_ena` (inter only)
- `load_cur_chroma_ena_o`: `stg_2_ena` (for REC)
- `load_ref_chroma_ena_o`: `stg_2_ena` (inter only)
- `load_db_luma_ena_o`: `rec_ena && (rec_y != 0)` (skip first row)
- `store_db_luma_ena_o`: `store_db_ena && (store_db_x != 0)` (skip first column)

## 7. External Interface

```
h265enc_top
├── clk, rstn                         (global)
├── sys_start_i, sys_type_i           (encode control)
├── sys_ctu_all_x_i, sys_ctu_all_y_i  (picture dimensions in CTUs)
├── extif_*                           (external memory IF — 128-bit bus)
├── bs_data_o, bs_val_o               (bitstream output — 8-bit)
└── sys_done_o                        (frame done)
```

## 8. RTL Block Verification

| Block | RTL Module | File | Status |
|-------|-----------|------|--------|
| `h265enc_top` | `enc_top` | `rtl/top/enc_top.v:32` | VERIFIED |
| `enc_ctrl` | `enc_ctrl` | `rtl/top/enc_ctrl.v:34` | VERIFIED |
| `fetch_top` | `fetch_top` | `rtl/fetch/fetch_top.v:23` | VERIFIED |
| `enc_core` | `enc_core` | `rtl/top/enc_core.v:22` | VERIFIED |
| `prei_top_buf` | `prei_top_buf` | `rtl/top/prei_top_buf.v:22` | VERIFIED |
| `prei_top` | `prei_top` | `rtl/prei/prei_top.v` | VERIFIED |
| `posi_top_buf` | `posi_top_buf` | `rtl/top/posi_top_buf.v:22` | VERIFIED |
| `posi_top` | `posi_top` | `rtl/posi/posi_top.v` | VERIFIED |
| `ime_top_buf` | `ime_top_buf` | `rtl/top/ime_top_buf.v:22` | VERIFIED |
| `ime_top` | `ime_top` | `rtl/ime/ime_top.v` | VERIFIED |
| `fme_top_buf` | `fme_top_buf` | `rtl/top/fme_top_buf.v:22` | VERIFIED |
| `fme_top` | `fme_top` | `rtl/fme/fme_top.v` | VERIFIED |
| `rec_top` | `rec_top` | `rtl/rec/rec_top.v` | VERIFIED |
| `dbsao_top` | `dbsao_top` | `rtl/db/dbsao_top.v` | VERIFIED |
| `cabac_top` | `cabac_top` | `rtl/cabac/cabac_top.v` | VERIFIED |
| `enc_data_pipeline` | `enc_data_pipeline` | `rtl/top/enc_data_pipeline.v:11` | VERIFIED |
| `fetch_wrapper` | `fetch_wrapper` | `rtl/fetch/fetch_wrapper.v` | VERIFIED |
| `fetch_cur_luma` | `fetch_cur_luma` | `rtl/fetch/fetch_cur_luma.v` | VERIFIED |
| `fetch_ref_luma` | `fetch_ref_luma` | `rtl/fetch/fetch_ref_luma.v` | VERIFIED |
| `fetch_cur_chroma` | `fetch_cur_chroma` | `rtl/fetch/fetch_cur_chroma.v` | VERIFIED |
| `fetch_ref_chroma` | `fetch_ref_chroma` | `rtl/fetch/fetch_ref_chroma.v` | VERIFIED |
| `fetch_db` | `fetch_db` | `rtl/fetch/fetch_db.v` | VERIFIED |
| Ping-pong buffer (PREI) | `prei_md_ram_sp_85x6` | `rtl/top/prei_top_buf.v:279` | VERIFIED |
| Ping-pong buffer (IME) | `ime_mv_ram_sp_64x13` | `rtl/top/ime_top_buf.v` | VERIFIED |
| Triple-rotation buffer (FME) | `fme_mv_ram_dp_64x20` | `rtl/top/fme_top_buf.v:409` | VERIFIED |
| 4-way rotation buffer (POSI) | `posi_md_ram_sp_64x6` | `rtl/top/posi_top_buf.v:279` | VERIFIED |
| MC pred double-buffer (FME) | `fme_buf_wrapper` | `rtl/top/fme_top_buf.v:287` | VERIFIED |
| FSM (enc_ctrl) | 12-state FSM | `rtl/top/enc_ctrl.v:116-127` | VERIFIED |
| Data pipeline registers | shift regs | `rtl/top/enc_data_pipeline.v:139-257` | VERIFIED |
| sel_r (ping-pong toggle) | 1-bit reg | `rtl/top/enc_core.v:482-488` | VERIFIED |
| sel_mod_3_r (3-way counter) | 2-bit reg | `rtl/top/enc_core.v:491-501` | VERIFIED |
| sel_mode_4_r (4-way counter) | 2-bit reg | `rtl/top/posi_top_buf.v:174-179` | VERIFIED |
