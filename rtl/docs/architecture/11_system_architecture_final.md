---
title: xk265 RTL System Architecture — Final Integration
version: 1.0
created: 2026-09-04
status: FINAL
---

# 11 — Final System Architecture

> Reverse-engineering integration — **read-only**.
> Every claim below is traceable to actual RTL syntax unless marked INFERRED or UNKNOWN.

---

## 1. Final System Overview

The xk265 encoder is a pipelined HEVC hardware encoder implemented in pure Verilog RTL. It processes LCU (Largest Coding Unit, 64×64 pixels) tiles in a deep pipeline with 6 logical stages. Each stage operates on a different LCU concurrently, achieving high throughput.

**Key Parameters** (from `enc_defines.v`):

| Parameter | Value | Meaning |
|-----------|-------|---------|
| `LCU_SIZE` | 64 | Largest Coding Unit (64×64 pixels) |
| `CU_DEPTH` | 3 | CU partition depth (64→32→16→8) |
| `PIXEL_WIDTH` | 8 | Pixel bit-width |
| `INIT_QP` | 22 | Default initial QP |
| `PIC_X_WIDTH` | 6 | Max 64 CTUs in X |
| `PIC_Y_WIDTH` | 6 | Max 64 CTUs in Y |
| `COEFF_WIDTH` | 16 | Transform coefficient width |
| `FMV_WIDTH` | 10 | Fine motion vector width |
| `MVD_WIDTH` | 11 | Motion vector difference width |
| `IME_MV_WIDTH` | 13 | IME MV width (X+Y) |

**Design Top**: `h265enc_top` (`rtl/top/enc_top.v:32`)
**Clock**: Single domain (`clk`)
**Reset**: Active-low asynchronous (`rstn`)
**External Interfaces**: 128-bit external memory bus, 8-bit bitstream output

---

## 2. Verified RTL Topology

### 2.1 Module Hierarchy

```
h265enc_top                          (rtl/top/enc_top.v:32)
├── enc_ctrl                         (rtl/top/enc_ctrl.v:34)    — Pipeline FSM (12 states)
├── fetch_top                        (rtl/fetch/fetch_top.v:23) — Data fetch hub (6 submodules)
└── enc_core                         (rtl/top/enc_core.v:22)    — Encoder datapath (8 blocks)
    ├── prei_top_buf                 (rtl/top/prei_top_buf.v:16)
    │   ├── prei_top                 (rtl/prei/prei_top.v:13)
    │   ├── prei_md_ram_sp_85x6 [0]  (rtl/mem/)
    │   └── prei_md_ram_sp_85x6 [1]  (rtl/mem/)
    ├── posi_top_buf                 (rtl/top/posi_top_buf.v:22)
    │   ├── posi_top                 (rtl/posi/posi_top.v:12)
    │   ├── posi_md_ram_sp_64x6 [0]  (rtl/mem/)
    │   ├── posi_md_ram_sp_64x6 [1]  (rtl/mem/)
    │   ├── posi_md_ram_sp_64x6 [2]  (rtl/mem/)
    │   └── posi_md_ram_sp_64x6 [3]  (rtl/mem/)
    ├── ime_top_buf                  (rtl/top/ime_top_buf.v:22)
    │   ├── ime_top                  (rtl/ime/ime_top.v:17)
    │   ├── ime_mv_ram_sp_64x13 [0]  (rtl/mem/)
    │   └── ime_mv_ram_sp_64x13 [1]  (rtl/mem/)
    ├── fme_top_buf                  (rtl/top/fme_top_buf.v:22)
    │   ├── fme_top                  (rtl/fme/fme_top.v:27)
    │   ├── fme_buf_wrapper [0]      (rtl/fme/fme_buf_wrapper.v)
    │   ├── fme_buf_wrapper [1]      (rtl/fme/fme_buf_wrapper.v)
    │   ├── fme_mv_ram_dp_64x20 [0]  (rtl/mem/)
    │   ├── fme_mv_ram_dp_64x20 [1]  (rtl/mem/)
    │   └── fme_mv_ram_dp_64x20 [2]  (rtl/mem/)
    ├── rec_top                      (rtl/rec/rec_top.v:23)
    │   ├── intra_top                (rtl/rec/rec_intra/intra_top.v)
    │   ├── mc_top                   (rtl/rec/rec_mc/mc_top.v)
    │   ├── tq_top                   (rtl/rec/rec_tq/tq_top.v)
    │   ├── rec_buf_wrapper          (rtl/rec/rec_wrapper/rec_buf_wrapper.v)
    │   └── IinP_flag_gen            (rtl/rec/IinP_flag_gen.v)
    ├── dbsao_top                    (rtl/db/dbsao_top.v:22)
    │   ├── dbsao_controller         (rtl/db/dbsao_controller.v)
    │   ├── dbsao_datapath           (rtl/db/dbsao_datapath.v)
    │   ├── db_filter                (rtl/db/db_filter.v)
    │   ├── db_bs                    (rtl/db/db_bs.v)
    │   ├── db_mv                    (rtl/db/db_mv.v)
    │   └── sao_top                  (rtl/db/sao_top.v)
    ├── cabac_top                    (rtl/cabac/cabac_top.v:32)
    │   ├── cabac_se_prepare         (rtl/cabac/cabac_se_prepare.v)
    │   ├── cabac_bina               (rtl/cabac/cabac_bina.v)
    │   ├── cabac_binsort            (rtl/cabac/cabac_binsort.v)
    │   ├── cabac_ucontext           (rtl/cabac/cabac_ucontext.v)
    │   ├── cabac_rlps4              (rtl/cabac/cabac_rlps4.v)
    │   ├── cabac_urange4            (rtl/cabac/cabac_urange4.v)
    │   ├── cabac_binmix             (rtl/cabac/cabac_binmix.v)
    │   ├── cabac_ulow               (rtl/cabac/cabac_ulow.v)
    │   ├── cabac_ulow_refine        (rtl/cabac/cabac_ulow_refine.v)
    │   ├── cabac_bitpack            (rtl/cabac/cabac_bitpack.v)
    │   └── coe_addr_trans           (rtl/cabac/coe_addr_trans.v)
    └── enc_data_pipeline            (rtl/top/enc_data_pipeline.v:11)
```

### 2.2 Instantiation Evidence

| Instance | Module | File:Line | Parent |
|----------|--------|-----------|--------|
| `u_enc_ctrl` | `enc_ctrl` | `enc_top.v:334` | `h265enc_top` |
| `u_fetch_top` | `fetch_top` | `enc_top.v:413` | `h265enc_top` |
| `u_enc_core` | `enc_core` | `enc_top.v:549` | `h265enc_top` |
| `u_prei_top_buf` | `prei_top_buf` | `enc_core.v:512` | `enc_core` |
| `u_posi_top_buf` | `posi_top_buf` | `enc_core.v:560` | `enc_core` |
| `u_ime_top_buf` | `ime_top_buf` | `enc_core.v:606` | `enc_core` |
| `u_fme_top_buf` | `fme_top_buf` | `enc_core.v:653` | `enc_core` |
| `u_rec_top` | `rec_top` | `enc_core.v:728` | `enc_core` |
| `u_dbsao_top` | `dbsao_top` | `enc_core.v:830` | `enc_core` |
| `u_cabac_top` | `cabac_top` | `enc_core.v:900` | `enc_core` |
| `u_data_pipeline` | `enc_data_pipeline` | `enc_core.v:951` | `enc_core` |

---

## 3. Major Subsystems

### 3.1 FETCH — Data Fetch Hub

**RTL**: `rtl/fetch/fetch_top.v:23`

**Purpose**: Manages all external memory access and pixel buffer management. Time-multiplexes 8 DMA channels across a single 128-bit external interface.

**Submodules**: `fetch_wrapper`, `fetch_cur_luma`, `fetch_ref_luma`, `fetch_cur_chroma`, `fetch_ref_chroma`, `fetch_db`

**Key Memories**:
- `cur_luma`: 10× `mem_lipo_1p_128x64x4` (5 INTRA + 3 INTER + 2 IME rotation banks)
- `cur_chroma`: 3× `mem_lipo_1p_64x64x4`
- `ref_luma IME`: 4× `fetch_rf_1p_128x512`
- `ref_luma FME`: 5× `fetch_rf_1p_128x512`
- `ref_chroma`: 8× `fetch_rf_1p_64x256` (4 U + 4 V)
- `db store`: 3× `mem_bilo_db` (triple-buffer)
- `db rec`: 2× `fetch_ram_1p_128x32`

**See**: `docs/architecture/03_fetch.md`

### 3.2 PREI — Pre-Intra Estimation

**RTL**: `rtl/top/prei_top_buf.v:16` → `rtl/prei/prei_top.v:13`

**Purpose**: Fast intra mode pre-estimation using Sobel gradient operators. Reduces 35 HEVC intra modes to a shortlist for POSI.

**Key Details**: 40-cycle per 8×8 block, 65 blocks per CTU → ~2,665 cycles. Outputs mode candidates to POSI via ping-pong mode RAM.

**Key Memories**: 2× `prei_md_ram_sp_85x6` (ping-pong, `sel_mod_2_i`)

**See**: `docs/architecture/04_prei.md`

### 3.3 IME — Integer Motion Estimation

**RTL**: `rtl/top/ime_top_buf.v:22` → `rtl/ime/ime_top.v:17`

**Purpose**: Full-pel motion estimation with configurable search patterns (center, slope, direction). 4 partitions: 2N×2N, 1N×2N, 2N×1N, 1N×1N.

**Key Details**: 5-state FSM, up to 8 search commands per CTU, zigzag scan, 4× downsampling option.

**Key Memories**: 2× `ime_mv_ram_sp_64x13` (ping-pong, `sel_mod_2_i`)

**See**: `docs/architecture/05_ime.md`

### 3.4 POSI — Post-Intra SATD Cost & Partition Decision

**RTL**: `rtl/top/posi_top_buf.v:22` → `rtl/posi/posi_top.v:12`

**Purpose**: Full intra mode decision with SATD-based RD cost calculation, using modes from PREI.

**Key Details**: Reads PREI modes, computes SATD cost at multiple partition sizes, outputs 85-bit partition tree and 20-bit cost.

**Key Memories**: 4× `posi_md_ram_sp_64x6` (4-way rotation: write/rec/blank/ec)

**See**: `docs/architecture/07_posi.md`

### 3.5 FME — Fractional Motion Estimation

**RTL**: `rtl/top/fme_top_buf.v:22` → `rtl/fme/fme_top.v:27`

**Purpose**: Sub-pel motion refinement using interpolated reference pixels. SATD-based cost evaluation.

**Key Details**: Reads IME MVs, produces fractional MVs, skip mode decision, prediction pixel output.

**Key Memories**: 3× `fme_mv_ram_dp_64x20` (3-way rotation, `sel_mod_3_i`), 2× `fme_buf_wrapper` (MC pred double-buffer)

**See**: `docs/architecture/06_fme.md`

### 3.6 REC — Reconstruction

**RTL**: `rtl/rec/rec_top.v:23`

**Purpose**: Intra/inter prediction, transform, quantization, inverse transform, inverse quantization, reconstruction, CBF generation.

**Submodules**: `intra_top`, `mc_top`, `tq_top`, `rec_buf_wrapper`, `IinP_flag_gen`

**Key Details**: Dual-path (intra + MC), shared TQ engine. Produces reconstructed pixels for DB, coefficients for CABAC.

**Key Memories**: `rec_buf_wrapper` contains: pre (4× `ram_tp_be_32x64`), rec (4× `ram_sp_be_192x64`), cef (4× `ram_sp_be_192x128`), mvd (3× `ram_sp_be_64x23`)

**See**: `docs/architecture/08_rec.md`

### 3.7 DB/SAO — Deblocking Filter + Sample Adaptive Offset

**RTL**: `rtl/db/dbsao_top.v:22`

**Purpose**: In-loop deblocking filter and sample adaptive offset.

**Submodules**: `dbsao_controller`, `dbsao_datapath`, `db_filter`, `db_bs`, `db_mv`, `sao_top`

**Key Details**: Reads reconstructed pixels from REC, MVs from FME MV RAM, original pixels from fetch. Writes filtered pixels back to fetch DB buffer.

**Key Memories**: `db_cbf_ram_sp_64x16`, `db_tupu_ram_sp_64x32`, `db_qp_ram_sp_64x20`, `db_mv_ram_sp_64x20`, `db_mv_ram_sp_512x20`

**See**: `docs/architecture/09_db.md`

### 3.8 CABAC — Entropy Coding

**RTL**: `rtl/cabac/cabac_top.v:32`

**Purpose**: Context-adaptive binary arithmetic coding of syntax elements. Final stage producing bitstream.

**Submodules**: `cabac_se_prepare`, `cabac_bina`, `cabac_binsort`, `cabac_ucontext`, `cabac_rlps4`, `cabac_urange4`, `cabac_binmix`, `cabac_ulow`, `cabac_ulow_refine`, `cabac_bitpack`, `coe_addr_trans`

**Key Details**: 8-stage pipeline: SE prepare → PIPO → binarize → bin sort → context+RLPS → bin mix → low update → bit pack.

**Key Memories**: `cabac_ram_sp_64x16` (neighboring MB info), 188-entry context table (flip-flops), 50-entry binsort FIFO.

**See**: `docs/architecture/10_cabac.md`

### 3.9 Data Pipeline Registers

**RTL**: `rtl/top/enc_data_pipeline.v:11`

**Purpose**: Shift registers propagating coding parameters through pipeline stages, aligned with CTU flow.

**Pipelined Parameters**:

| Parameter | Width | Pipeline Path |
|-----------|-------|---------------|
| QP | 6 | rc → posi → fme → rec → db → ec |
| Intra partition | 85 | posi → rec → db → ec |
| Inter partition | 42 | ime → fme → rec → db → ec |
| CBF (Y/U/V) | 256 each | rec → db → ec |
| Skip flag | 85 | fme → rec → db → ec |
| IinP flag | 3 | rec → db → ec |
| Actual bitnum | 16 | cabac → rc (feedback) |

All registered on `enc_done_i` (pipeline advance pulse).

---

## 4. Complete Dataflow

### 4.1 Pipeline Stages

The pipeline has 6 logical stages, mapped to FSM enable groups in `enc_ctrl`:

| Stage | Subsystem | Start Signal | Done Signal | Description |
|-------|-----------|-------------|-------------|-------------|
| 0 | FETCH | `enc_start` | `fetch_done_i` | External memory load/store |
| 1a | PREI | `prei_start_o` | `prei_done_i` | Intra mode pre-est (always) |
| 1b | IME | `ime_start_o` | `ime_done_i` | Integer ME (inter only) |
| 2a | POSI | `posi_start_o` | `posi_done_i` | Intra SATD + partition |
| 2b | FME | `fme_start_o` | `fme_done_i` | Fractional ME (inter only) |
| 3 | REC | `rec_start_o` | `rec_done_i` | Reconstruction |
| 4 | DB/SAO | `db_start_o` | `db_done_i` | Deblocking + SAO |
| 5 | CABAC | `ec_start_o` | `ec_done_i` | Entropy coding |

**Stage grouping** (`enc_ctrl.v:437-438`):
- `stg_1_flg`: INTRA → `prei_flg`; INTER → `prei_flg && ime_flg`
- `stg_2_flg`: INTRA → `posi_flg`; INTER → `posi_flg && fme_flg`

### 4.2 Inter-Mode Dataflow (Complete)

```
                    ┌─────────────────────────────────────────────────┐
                    │                  EXTERNAL MEMORY                │
                    │                 (SDRAM / DPB)                   │
                    └────────────────────┬────────────────────────────┘
                                         │ 128-bit ext_if
                                         ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           fetch_top                                         │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────────┐   │
│  │ fetch_wrapper│ │fetch_cur_luma│ │fetch_ref_luma│ │fetch_cur_chroma  │   │
│  │ (DMA arbiter)│ │(10× LIPO)   │ │(9× rf_1p)   │ │(3× LIPO)        │   │
│  └──────────────┘ └──────┬───────┘ └──────┬───────┘ └────────┬─────────┘   │
│                          │                │                   │             │
│  ┌──────────────┐ ┌──────┴───────┐ ┌──────┴───────┐ ┌───────┴──────────┐  │
│  │  fetch_db    │ │fetch_ref_chrm│ │              │ │                  │  │
│  │(3×db+2×rec)  │ │(8× rf_1p)   │ │              │ │                  │  │
│  └──────────────┘ └──────────────┘ └──────────────┘ └──────────────────┘  │
└──────────┬──────────────┬──────────────┬──────────────┬────────────┬────────┘
           │              │              │              │            │
    256b   │    256b      │    256b      │    256b      │    256b    │ 128b
           ▼              ▼              ▼              ▼            ▼
     ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐  ┌─────────┐
     │  PREI   │    │  IME    │    │  POSI   │    │   FME   │  │   DB    │
     │prei_top │    │ime_top  │    │posi_top │    │ fme_top │  │dbsao_top│
     │_buf     │    │_buf     │    │_buf     │    │  _buf   │  │         │
     └────┬────┘    └────┬────┘    └────┬────┘    └────┬────┘  └─────────┘
          │              │              │              │
     6b   │    42b       │         85b  │    20b       │
     mode │   partition  │        part  │   cost       │
          │    + 13b MV  │         + 6b │   + skip     │
          │              │         mode │   flags      │
          ▼              ▼              ▼              ▼
     ┌─────────────────────────────────────────────────────────┐
     │                 enc_data_pipeline                        │
     │  (QP, partition, CBF, skip, IinP — shift registers)     │
     └───────────────────────┬─────────────────────────────────┘
                             │
                             ▼
     ┌─────────┐    ┌─────────┐    ┌─────────┐
     │   REC   │───▶│  DB/SAO │───▶│  CABAC  │───▶ BITSTREAM
     │rec_top  │    │dbsao_top│    │cabac_top│     (8-bit)
     └─────────┘    └─────────┘    └─────────┘
```

### 4.3 Intra-Mode Dataflow

In intra mode, IME and FME are disabled. The pipeline simplifies to:

```
FETCH → PREI → POSI → REC (intra) → DB/SAO → CABAC
```

### 4.4 Feedback Paths

| Feedback Path | Source | Destination | Signal | Width | Purpose |
|--------------|--------|-------------|--------|-------|---------|
| Bitnum → RC | CABAC | PREI | `rc_actual_bitnum_i` | 16 | Rate control feedback |
| Pred pixels → REC | FME | REC | `fme_rd_dat_w` | 256b | MC prediction read |
| Pred pixels ← REC | REC | FME | `fme_wr_dat_w` | 256b | MC prediction write |
| MV → REC/DB | FME MV RAM | REC, DB | `mc_mv_rd_dat_w`, `db_mv_rd_dat_w` | 20b×64 | MV read |
| Rec pixels → DB | REC | DB | `db_rec_rd_dat_w` | 256b | Reconstruction read |
| Rec pixels ← DB | DB | REC | `db_rec_wr_dat_w` | 256b | Reconstruction write |
| Filtered → fetch | DB | FETCH | `fetch_wen_o`, `fetch_wdata_o` | 1b + 128b | Store filtered pixels |
| IinP decision | REC | FME | `fme_IinP_flag_w` | 4b | I-in-P frame flag |

---

## 5. Control Architecture

### 5.1 Pipeline FSM (`enc_ctrl`)

12-state FSM (`enc_ctrl.v:116-127`):

```
IDLE → S0 → S1 → S2 → S3 → S4 → S5 (steady-state) → S6 → S7 → S8 → S9 → SA → IDLE
```

| State | Value | Active Stages | Description |
|-------|-------|---------------|-------------|
| IDLE | 0 | — | Waiting for start |
| S0 | 1 | fetch | First CTU fill |
| S1 | 2 | fetch, stg_1 | + PREI or PREI+IME |
| S2 | 3 | fetch, stg_1, stg_2 | + POSI or POSI+FME |
| S3 | 4 | fetch, stg_1, stg_2, rec | + Reconstruction |
| S4 | 5 | fetch, stg_1, stg_2, rec, db | + Deblocking |
| S5 | 6 | all stages | Full pipeline (stall until last CTU) |
| S6 | 7 | stg_1–db, ec | Drain fetch, enter EC |
| S7 | 8 | stg_2–ec | Drain stg_1 |
| S8 | 9 | rec–ec | Drain stg_2 |
| S9 | 10 | db, ec | Drain rec |
| SA | 11 | ec | Drain db, finish EC |

### 5.2 Start/Done Handshake

Each subsystem uses a start/done handshake (`enc_ctrl.v:356-390`):
1. `enc_ctrl` asserts `xxx_start_o` for one cycle when the stage is enabled
2. Subsystem processes the CTU
3. Subsystem asserts `xxx_done_o` for one cycle when complete
4. `enc_ctrl` latches `xxx_flg` (sticky flag, stays high until `enc_done_r`)
5. `enc_done_r` asserted when all required flags match expected pattern

### 5.3 CTU Coordinate Pipeline

Each stage receives independent x/y coordinates, shifted one CTU behind (`enc_ctrl.v:529-566`):

```
INTRA: pre_l → prei → posi → (ime=0) → (fme=0) → rec → db → ec → store_db
INTER: pre_l → prei → posi → ime → fme → rec → db → ec → store_db
```

### 5.4 Fetch Enable Generation (`enc_ctrl.v:331-352`)

| Enable Signal | Condition | Purpose |
|---------------|-----------|---------|
| `load_cur_luma_ena` | `pre_l_ena` | Load current luma for PREI/IME |
| `load_ref_luma_ena` | `pre_l_ena` (inter) | Load reference luma for IME |
| `load_cur_chroma_ena` | `stg_2_ena` | Load current chroma for REC |
| `load_ref_chroma_ena` | `stg_2_ena` (inter) | Load reference chroma for REC |
| `load_db_luma_ena` | `rec_ena && (rec_y != 0)` | Load DB top rec pixels |
| `store_db_luma_ena` | `store_db_ena && (store_db_x != 0)` | Store filtered pixels |

### 5.5 Global Rotation Counters (`enc_core.v:482-501`)

| Counter | Width | Increment | Purpose |
|---------|-------|-----------|---------|
| `sel_r` | 1 bit | Per CTU | Ping-pong toggle (PREI mode, IME MV, FME pred) |
| `sel_mod_3_r` | 2 bits | Per CTU | 3-way rotation (FME MV RAM) |

### 5.6 POSI Mode Rotation (`posi_top_buf.v:174-179`)

| Counter | Width | Increment | Purpose |
|---------|-------|-----------|---------|
| `sel_mode_4_r` | 2 bits | Per sys_start | 4-way rotation (POSI mode RAM) |

Rotation scheme:
```
mod_4  | 0         1           2           3
ram0   | wr        rec         blank       ec
ram1   | ec        wr          rec         blank
ram2   | blank     ec          wr          rec
ram3   | rec       blank       ec          wr
```

---

## 6. Memory Integration

### 6.1 Rotation Schemes

| Scheme | Buffers | Control | Subsystems |
|--------|---------|---------|------------|
| **Ping-pong (2-way)** | PREI mode RAM, IME MV RAM, FME MC pred buffer | `sel_r` (1-bit) | PREI↔POSI, IME↔FME, FME↔REC |
| **Triple rotation (3-way)** | FME MV RAM | `sel_mod_3_r` (2-bit) | FME write / MC+DB read / FME next read |
| **Quad rotation (4-way)** | POSI mode RAM | `sel_mode_4_r` (2-bit) | POSI write / REC read / blank / CABAC read |

### 6.2 External Memory Interface

`fetch_wrapper` arbitrates 8 DMA channels (`enc_ctrl.v:331-352`):

| Channel | Mode | Direction | Data Width | Purpose |
|---------|------|-----------|------------|---------|
| load_cur_luma | read | ext→fetch | 128b | Current CTU luma |
| load_ref_luma | read | ext→fetch | 128b | Reference search window luma |
| load_cur_chroma | read | ext→fetch | 128b | Current CTU chroma |
| load_ref_chroma | read | ext→fetch | 128b | Reference chroma |
| load_db_luma | read | ext→fetch | 128b | DB top-row reconstruction |
| load_db_chroma | read | ext→fetch | 128b | DB top-row chroma reconstruction |
| store_db_luma | write | fetch→ext | 128b | Filtered luma → external |
| store_db_chroma | write | fetch→ext | 128b | Filtered chroma → external |

### 6.3 Memory Summary

| Category | Count | Total Instances |
|----------|-------|-----------------|
| Fetch buffers | 32 | ref_luma(9) + cur_luma(10) + cur_chroma(3) + ref_chroma(8) + db(5) + bilinear(4) + CEF(4) |
| IME | 3 | MV RAM(2) + vertical mem(1) |
| FME | 9 | MV dp(3) + cost(1) + top MV(1) + pred buf(2) + CEF(4) |
| PREI | 2 | mode RAM(2) |
| POSI | 7 | mode RAM(4) + row(1) + col(1) + frame(1) |
| REC | 27 | intra ref(3) + MVD(3) + pre(4) + rec(4) + cef(4) + MC MV(1) + TQ transpose(32) |
| DB | 5 | CBF(1) + TU/PU(1) + QP(1) + MV(2) |
| CABAC | 1 | neighbor(1) |
| **Total** | **~95** | |

---

## 7. Cross-Subsystem Interfaces

### 7.1 Pixel Data Interfaces (enc_core ↔ fetch)

| Interface | Subsystem | Bus Width | Address Signals | Purpose |
|-----------|-----------|-----------|-----------------|---------|
| `prei_cur_if` | prei_top_buf | 256b | `ren, sel[1:0], size[1:0], 4x4_x[3:0], 4x4_y[3:0], idx[4:0]` | Current luma |
| `posi_cur_if` | posi_top_buf | 256b | `rd_ena, sel[1:0], siz[1:0], 4x4_x[3:0], 4x4_y[3:0], idx[4:0]` | Current luma |
| `ime_cur_if` | ime_top_buf | 256b | `rden, 4x4_x[3:0], 4x4_y[3:0], idx[4:0], sel[1:0], size[1:0]` | Current luma |
| `ime_ref_if` | ime_top_buf | 256b | `rden, x[7:0], y[6:0]` | Reference luma |
| `fme_cur_if` | fme_top_buf | 256b | `rden, 4x4_idx[4:0], 4x4_x[3:0], 4x4_y[3:0]` | Current luma |
| `fme_ref_if` | fme_top_buf | 512b | `rden, idx_x[7:0], idx_y[7:0]` | Reference luma |
| `rec_cur_if` | rec_top | 256b | `rd_ena, sel[1:0], siz[1:0], 4x4_x[3:0], 4x4_y[3:0], idx[4:0]` | Current luma+chroma |
| `rec_ref_if` | rec_top | 64b | `rd_ena, sel[1:0], idx_x[7:0], idx_y[7:0]` | Reference chroma |
| `db_cur_if` | dbsao_top | 256b | `ren, sel[1:0], siz[1:0], 4x4_x[3:0], 4x4_y[3:0], idx[4:0]` | Current luma+chroma |
| `db_rec_if` | dbsao_top | 128b | `wen, w4x4_x[4:0], w4x4_y[4:0], wprevious, wdone, wsel[1:0]` | Rec store |
| `db_top_rec_if` | dbsao_top | 32b | `ren, r4x4[4:0], ridx[1:0]` | Top rec read |

### 7.2 MV Data Chain (internal to enc_core)

| Source | Destination | Signal | Width | Purpose |
|--------|-------------|--------|-------|---------|
| ime_top_buf | fme_top_buf | `fme_mv_data_w` (via MV RAM) | 20b (2×FMV_WIDTH) | Integer MVs |
| ime_top_buf | enc_data_pipeline | `inter_partition_w` | 42b | Partition tree |
| fme_top_buf | enc_data_pipeline | `fme_skip_idx_w`, `fme_skip_flag_w` | 85×4b, 85b | Skip decisions |
| fme_top_buf | rec_top | `mc_mv_rd_dat_w` (via MV RAM) | 20b×64 | MC MVs |
| fme_top_buf | dbsao_top | `db_mv_rd_dat_w` (via MV RAM) | 20b×64 | DB MVs |
| fme_top_buf | rec_top | `fme_rd_dat_w` | 256b | MC prediction pixels |

### 7.3 Reconstruction Data Chain

| Source | Destination | Signal | Width | Purpose |
|--------|-------------|--------|-------|---------|
| rec_top | dbsao_top | `db_rec_rd_dat_w` | 256b | Reconstructed pixels (read) |
| dbsao_top | rec_top | `db_rec_wr_dat_w` | 256b | Filtered pixels (write) |
| dbsao_top | fetch_top | `fetch_wen_o`, `fetch_wdata_o` | 1b + 128b | Store to external |

### 7.4 Mode/Coefficient/CABAC Chain

| Source | Destination | Signal | Width | Purpose |
|--------|-------------|--------|-------|---------|
| prei_top_buf | posi_top_buf | Mode RAM | 6b×85 | Mode candidates (ping-pong) |
| posi_top_buf | enc_core | `posi_partition_w`, `posi_cost_w` | 85b, 20b | Partition + cost |
| rec_top | enc_core | `cbf_y_w`, `cbf_u_w`, `cbf_v_w` | 256b each | Coded block flags |
| rec_top | cabac_top | `ec_coe_rd_dat_w` | 128b | Transform coefficients |
| rec_top | cabac_top | `ec_mvd_rd_data_w` | 22b | Motion vector diffs |
| posi_top_buf | cabac_top | Mode RAM (4-way) | 6b×64 | Mode decisions |
| dbsao_top | cabac_top | `sao_data_w` | 62b | SAO parameters |
| cabac_top | h265enc_top | `bs_dat_o`, `bs_val_o` | 8b, 1b | Bitstream output |
| enc_data_pipeline | prei_top_buf | `rc_actual_bitnum_i` | 16b | RC feedback |

### 7.5 Control Signals

| Source | Destination | Signal | Width | Purpose |
|--------|-------------|--------|-------|---------|
| enc_ctrl | all subsystems | `xxx_start_o` | 1b each | Stage start |
| all subsystems | enc_ctrl | `xxx_done_i` | 1b each | Stage completion |
| enc_ctrl | all subsystems | `xxx_x_o`, `xxx_y_o` | PIC_X_WIDTH, PIC_Y_WIDTH | CTU coordinates |
| enc_ctrl | fetch_top | `load_xxx_ena_o` + `load_xxx_x/y_o` | 1b + 2×PW | Fetch channel enable + address |
| enc_core | rec_top | `sys_type_i` | 1b | INTRA/INTER select |

---

## 8. Final System Block Diagram

See: `diagrams/11_system/system_architecture_final.mmd`

The diagram shows:
- **Top module** `h265enc_top` with external interfaces
- **Three first-level children**: `enc_ctrl`, `fetch_top`, `enc_core`
- **Eight processing blocks** inside `enc_core` with data/control connections
- **Pipeline registers** (`enc_data_pipeline`) propagating parameters
- **Memory rotation buffers** as shared resources between stages
- **External memory interface** and **bitstream output**
- Solid arrows = DATA, dashed arrows = CONTROL
- Important bus widths labeled

---

## 9. RTL Evidence

### 9.1 Top-Level Verification

| Block | RTL Module | File:Line | Status |
|-------|-----------|-----------|--------|
| `h265enc_top` | `enc_top` | `enc_top.v:32` | VERIFIED |
| `enc_ctrl` | `enc_ctrl` | `enc_ctrl.v:34` | VERIFIED |
| `fetch_top` | `fetch_top` | `fetch_top.v:23` | VERIFIED |
| `enc_core` | `enc_core` | `enc_core.v:22` | VERIFIED |
| `enc_data_pipeline` | `enc_data_pipeline` | `enc_data_pipeline.v:11` | VERIFIED |

### 9.2 Subsystem Verification

| Block | RTL Module | File | Status |
|-------|-----------|------|--------|
| `prei_top_buf` | `prei_top_buf` | `prei_top_buf.v:16` | VERIFIED |
| `prei_top` | `prei_top` | `prei_top.v:13` | VERIFIED |
| `posi_top_buf` | `posi_top_buf` | `posi_top_buf.v:22` | VERIFIED |
| `posi_top` | `posi_top` | `posi_top.v:12` | VERIFIED |
| `ime_top_buf` | `ime_top_buf` | `ime_top_buf.v:22` | VERIFIED |
| `ime_top` | `ime_top` | `ime_top.v:17` | VERIFIED |
| `fme_top_buf` | `fme_top_buf` | `fme_top_buf.v:22` | VERIFIED |
| `fme_top` | `fme_top` | `fme_top.v:27` | VERIFIED |
| `rec_top` | `rec_top` | `rec_top.v:23` | VERIFIED |
| `dbsao_top` | `dbsao_top` | `dbsao_top.v:22` | VERIFIED |
| `cabac_top` | `cabac_top` | `cabac_top.v:32` | VERIFIED |

### 9.3 Memory Verification

| Memory | Module | File | Instances | Status |
|--------|--------|------|-----------|--------|
| PREI mode | `prei_md_ram_sp_85x6` | `prei_top_buf.v:279` | 2 | VERIFIED |
| POSI mode | `posi_md_ram_sp_64x6` | `posi_top_buf.v:279` | 4 | VERIFIED |
| IME MV | `ime_mv_ram_sp_64x13` | `ime_top_buf.v:204` | 2 | VERIFIED |
| FME MV dp | `fme_mv_ram_dp_64x20` | `fme_top_buf.v:409` | 3 | VERIFIED |
| FME pred | `fme_buf_wrapper` | `fme_top_buf.v:287` | 2 | VERIFIED |

### 9.4 Pipeline Verification

| Feature | RTL Location | Status |
|---------|-------------|--------|
| 12-state FSM | `enc_ctrl.v:116-127` | VERIFIED |
| Start/done handshake | `enc_ctrl.v:356-390` | VERIFIED |
| CTU coordinate pipeline | `enc_ctrl.v:529-566` | VERIFIED |
| Fetch enable generation | `enc_ctrl.v:331-352` | VERIFIED |
| QP pipeline registers | `enc_data_pipeline.v:139-154` | VERIFIED |
| Partition pipeline registers | `enc_data_pipeline.v:157-186` | VERIFIED |
| CBF pipeline registers | `enc_data_pipeline.v:201-218` | VERIFIED |
| Skip pipeline registers | `enc_data_pipeline.v:221-238` | VERIFIED |
| Bitnum feedback | `enc_data_pipeline.v:241-257` | VERIFIED |

---

## 10. Resolved Conflicts

No conflicts found between subsystem analyses. All existing documents are mutually consistent.

---

## 11. Remaining Unknowns

| Item | Status | Notes |
|------|--------|-------|
| Exact cycle counts per stage | UNKNOWN | Depends on CTU content and configuration |
| IME_HAS_VER_MEM compile-time guard | INFERRED | Vertical memory optimization; compile-time define |
| External memory controller behavior | UNKNOWN | Not in RTL; only `ext_if` protocol defined |
| Rate control algorithm details | INFERRED | `rate_control.v` has 11-cycle pipeline; exact algorithm not fully traced |
| SAO mode decision thresholds | UNKNOWN | Internal to `sao_mode.v` |
| CABAC context initialization values | UNKNOWN | Flip-flop initial values; reset behavior not traced |

---

## Verification Checklist

- [x] Module hierarchy verified — all instantiations traced in RTL
- [x] Module instantiations verified — 11 top-level instances, all confirmed
- [x] Port directions verified — input/output declarations read
- [x] Signal widths verified — all bus widths confirmed from port declarations
- [x] Major signal connections verified — pixel, MV, mode, CBF chains traced
- [x] Memories/buffers verified — all rotation schemes confirmed
- [x] FSMs verified — 12-state enc_ctrl FSM confirmed
- [x] Sequential registers verified — pipeline registers in enc_data_pipeline
- [x] Pipeline boundaries verified — 6 stages mapped to FSM states
- [x] Datapath verified — complete dataflow from pixels to bitstream
- [x] Control path verified — start/done/coordinate pipeline
- [x] Clock/reset domains verified — single clock, async active-low reset
