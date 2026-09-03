# 02 — Top-Level Architecture

> Reverse-engineering checkpoint — **read-only**.
> All claims below are verified from RTL unless marked INFERRED or UNKNOWN.

---

## 1. Top Module

| Property | Value | RTL Evidence |
|----------|-------|-------------|
| Module name | `h265enc_top` | `rtl/top/enc_top.v:32` |
| File | `rtl/top/enc_top.v` | — |
| Testbench | `tb_enc_top` | `sim/top_testbench/tb_enc_top.v` |
| Clock domain | Single (`clk`) | All ports use single `clk` |
| Reset | Active-low asynchronous (`rstn`) | `always @(posedge clk or negedge rstn)` in all submodules |

The design top is `h265enc_top`. The testbench instantiates it directly.

---

## 2. First-Level Child Modules

`h265enc_top` instantiates exactly **three** child modules:

| Instance Name | Module | File | Role |
|---------------|--------|------|------|
| `u_enc_ctrl` | `enc_ctrl` | `rtl/top/enc_ctrl.v:34` | Pipeline FSM: sequencing, start/done, CTU coordinate tracking |
| `u_fetch_top` | `fetch_top` | `rtl/fetch/fetch_top.v:23` | Data fetch: external memory IF, pixel buffer management |
| `u_enc_core` | `enc_core` | `rtl/top/enc_core.v:22` | Datapath: all processing blocks + data pipeline registers |

**Instantiation evidence** (`enc_top.v:334`, `enc_top.v:413`, `enc_top.v:549`):

```verilog
enc_ctrl u_enc_ctrl ( ... );
fetch_top u_fetch_top ( ... );
enc_core u_enc_core ( ... );
```

### 2.1 enc_ctrl — Pipeline FSM Controller

- **No submodules** — pure combinational + sequential FSM.
- 12-state FSM: `IDLE`, `S0`–`S9`, `SA` (`enc_ctrl.v:116-127`).
- Generates `start` and `x/y` coordinates for each pipeline stage.
- Generates fetch load/store enable and address signals.
- Collects `done` signals from all subsystems to advance pipeline.

### 2.2 fetch_top — Data Fetch Subsystem

Instantiates 6 submodules:

| Instance | Module | File |
|----------|--------|------|
| `u_wrapper` | `fetch_wrapper` | `fetch_wrapper.v` |
| `u_cur_luma` | `fetch_cur_luma` | `fetch_cur_luma.v` |
| `u_ref_luma` | `fetch_ref_luma` | `fetch_ref_luma.v` |
| `u_cur_chroma` | `fetch_cur_chroma` | `fetch_cur_chroma.v` |
| `u_ref_chroma` | `fetch_ref_chroma` | `fetch_ref_chroma.v` |
| `u_db` | `fetch_db` | `fetch_db.v` |

`fetch_top` acts as the data-supply hub: all subsystems read current/reference pixels through it, and the DB writes reconstructed pixels through it.

### 2.3 enc_core — Encoder Datapath

Instantiates **8** child modules:

| Instance Name | Module | File | Role |
|---------------|--------|------|------|
| `u_prei_top_buf` | `prei_top_buf` | `top/prei_top_buf.v:16` | Pre-intra estimation + rate control (with MD RAM wrapper) |
| `u_posi_top_buf` | `posi_top_buf` | `top/posi_top_buf.v:22` | Post-intra SATD + partition decision (with MD RAM quad-rotation) |
| `u_ime_top_buf` | `ime_top_buf` | `top/ime_top_buf.v:22` | Integer motion estimation (with MV RAM ping-pong) |
| `u_fme_top_buf` | `fme_top_buf` | `top/fme_top_buf.v:22` | Fractional motion estimation (with MV/pred RAM rotation) |
| `u_rec_top` | `rec_top` | `rec/rec_top.v:23` | Reconstruction (intra pred + MC + TQ + buffer management) |
| `u_dbsao_top` | `dbsao_top` | `db/dbsao_top.v:22` | Deblocking filter + SAO |
| `u_cabac_top` | `cabac_top` | `cabac/cabac_top.v:32` | CABAC entropy coding |
| `u_data_pipeline` | `enc_data_pipeline` | `top/enc_data_pipeline.v:11` | Pipeline registers for QP/partition/CBF/skip |

---

## 3. Second-Level Major Modules

### 3.1 prei_top_buf

| Property | Detail |
|----------|--------|
| File | `rtl/top/prei_top_buf.v:16` |
| Inner module | `prei_top` (`rtl/prei/prei_top.v:13`) |
| Memories | `prei_md_ram_sp_85x6` × 2 (ping-pong mode storage) |
| Key function | Mode decision preprocessing + QP allocation |

### 3.2 posi_top_buf

| Property | Detail |
|----------|--------|
| File | `rtl/top/posi_top_buf.v:22` |
| Inner module | `posi_top` (`rtl/posi/posi_top.v:12`) |
| Memories | `posi_md_ram_sp_64x6` × 4 (quad-rotation: write/rec/blank/ec) |
| Key function | SATD-based intra partition decision |

### 3.3 ime_top_buf

| Property | Detail |
|----------|--------|
| File | `rtl/top/ime_top_buf.v:22` |
| Inner module | `ime_top` (`rtl/ime/ime_top.v:17`) |
| Memories | `ime_mv_ram_sp_64x13` × 2 (ping-pong MV storage) |
| Key function | Integer-pel motion search + MV output |

### 3.4 fme_top_buf

| Property | Detail |
|----------|--------|
| File | `rtl/top/fme_top_buf.v:22` |
| Inner module | `fme_top` (`rtl/fme/fme_top.v:27`) |
| Memories | `fme_buf_wrapper` × 2 (prediction pixel buffer), `fme_mv_ram_dp_64x20` × 3 (MV rotation: FME/MC/DB) |
| Key function | Fractional-pel interpolation + skip decision + cost output |

### 3.5 rec_top

| Property | Detail |
|----------|--------|
| File | `rtl/rec/rec_top.v:23` |
| Submodules | `intra_top`, `mc_top`, `tq_top`, `rec_buf_wrapper`, `IinP_flag_gen` |
| Key function | Intra/inter prediction → transform → quantize → reconstruct |

### 3.6 dbsao_top

| Property | Detail |
|----------|--------|
| File | `rtl/db/dbsao_top.v:22` |
| Submodules | `dbsao_controller`, `dbsao_datapath`, `db_filter`, `sao_top`, and supporting LUTs |
| Key function | Deblocking filter + sample adaptive offset |

### 3.7 cabac_top

| Property | Detail |
|----------|--------|
| File | `rtl/cabac/cabac_top.v:32` |
| Submodules | `cabac_binsort`, `cabac_bitpack`, `cabac_bina`, `cabac_ucontext`, `cabac_ulow`, `pipo`, and SE prepare modules |
| Key function | Context-adaptive binary arithmetic coding → bitstream output |

### 3.8 enc_data_pipeline

| Property | Detail |
|----------|--------|
| File | `rtl/top/enc_data_pipeline.v:11` |
| Submodules | None (pure pipeline registers) |
| Key function | Delays QP, intra/inter partition, CBF, IinP flag, skip flag across pipeline stages |

---

## 4. Input Interfaces (from `h265enc_top` ports)

### 4.1 Global

| Port | Width | Direction | Description |
|------|-------|-----------|-------------|
| `clk` | 1 | input | System clock |
| `rstn` | 1 | input | Active-low async reset |

### 4.2 System Configuration (`sys_cfg_if`)

| Port | Width | Dir | Description |
|------|-------|-----|-------------|
| `sys_start_i` | 1 | in | Start encoding |
| `sys_done_o` | 1 | out | Encoding complete |
| `sys_type_i` | 1 | in | Slice type: 0=P, 1=I |
| `sys_all_x_i` | `PIC_WIDTH` (13) | in | Frame width in pixels |
| `sys_all_y_i` | `PIC_HEIGHT` (12) | in | Frame height in pixels |
| `sys_init_qp_i` | 6 | in | Initial QP |
| `sys_IinP_ena_i` | 1 | in | Enable I-block-in-P-frame |
| `sys_db_ena_i` | 1 | in | Enable deblocking filter |
| `sys_sao_ena_i` | 1 | in | Enable SAO |
| `sys_posi4x4bit_i` | 5 | in | POSI 4×4 bit width |

### 4.3 Skip Cost Threshold

| Port | Width | Dir | Description |
|------|-------|-----|-------------|
| `skip_cost_thresh_08` | 32 | in | Skip cost threshold for 8×8 |
| `skip_cost_thresh_16` | 32 | in | Skip cost threshold for 16×16 |
| `skip_cost_thresh_32` | 32 | in | Skip cost threshold for 32×32 |
| `skip_cost_thresh_64` | 32 | in | Skip cost threshold for 64×64 |

### 4.4 Rate Control Configuration (`rc_cfg_if`)

| Port | Width | Dir | Description |
|------|-------|-----|-------------|
| `sys_rc_mod64_sum_o` | 32 | out | CTU sum for RC |
| `sys_rc_bitnum_i` | 32 | in | Actual bit count |
| `sys_rc_k` | 16 | in | RC parameter K |
| `sys_rc_roi_height` | 6 | in | ROI height |
| `sys_rc_roi_width` | 7 | in | ROI width |
| `sys_rc_roi_x` | 7 | in | ROI X offset |
| `sys_rc_roi_y` | 7 | in | ROI Y offset |
| `sys_rc_roi_enable` | 1 | in | ROI enable |
| `sys_rc_L1_frame_byte` | 10 | in | L1 frame byte budget |
| `sys_rc_L2_frame_byte` | 10 | in | L2 frame byte budget |
| `sys_rc_lcu_en` | 1 | in | Per-LCU RC enable |
| `sys_rc_max_qp` | 6 | in | Max QP |
| `sys_rc_min_qp` | 6 | in | Min QP |
| `sys_rc_delta_qp` | 6 | in | Delta QP |

### 4.5 IME Configuration (`ime_cfg_if`)

| Port | Width | Dir | Description |
|------|-------|-----|-------------|
| `sys_ime_cmd_num_i` | `CMD_NUM_WIDTH` (3) | in | Number of IME commands |
| `sys_ime_cmd_dat_i` | `CMD_DAT_WIDTH` (computed) | in | IME command data |

---

## 5. Output Interfaces (from `h265enc_top` ports)

### 5.1 External Memory Interface (`ext_if`)

| Port | Width | Dir | Description |
|------|-------|-----|-------------|
| `extif_start_o` | 1 | out | External access start |
| `extif_done_i` | 1 | in | External access done |
| `extif_mode_o` | 5 | out | Access mode |
| `extif_x_o` | 12 | out | X address |
| `extif_y_o` | 12 | out | Y address |
| `extif_width_o` | 8 | out | Transfer width |
| `extif_height_o` | 8 | out | Transfer height |
| `extif_wren_i` | 1 | in | Write enable (from external) |
| `extif_rden_i` | 1 | in | Read enable (from external) |
| `extif_data_i` | 128 | in | Write data (16 pixels × 8 bits) |
| `extif_data_o` | 128 | out | Read data (16 pixels × 8 bits) |

### 5.2 Bitstream Output (`bs_if`)

| Port | Width | Dir | Description |
|------|-------|-----|-------------|
| `bs_val_o` | 1 | out | Bitstream byte valid |
| `bs_dat_o` | 8 | out | Bitstream byte data |

---

## 6. Clock and Reset

| Property | Value | RTL Evidence |
|----------|-------|-------------|
| Clock signal | `clk` | `enc_top.v:105` |
| Reset signal | `rstn` (active-low) | `enc_top.v:106` |
| Reset type | Asynchronous | `always @(posedge clk or negedge rstn)` in `enc_ctrl.v:262`, `enc_core.v:482`, etc. |
| Clock domains | **Single** | No `CLKDIV`, `clk_div`, or multi-clock evidence in `enc_top.v` or `enc_core.v` |

---

## 7. Major Data Interfaces

These are the pixel-data read paths between `enc_core` processing blocks and `fetch_top`:

### 7.1 Current-Frame Pixel Read (enc_core → fetch)

All subsystems read current-frame pixels through a common address-multiplexed interface to `fetch_top`:

| Interface | Subsystem | Bus Width | Address Width | Signals |
|-----------|-----------|-----------|---------------|---------|
| `prei_cur_if` | prei_top_buf | 256 (32×8) | 4+4+5+2+2 | `ren`, `sel[1:0]`, `size[1:0]`, `4x4_x[3:0]`, `4x4_y[3:0]`, `idx[4:0]` |
| `posi_cur_if` | posi_top_buf | 256 (32×8) | 4+4+5+2+2 | `rd_ena`, `sel[1:0]`, `siz[1:0]`, `4x4_x[3:0]`, `4x4_y[3:0]`, `idx[4:0]` |
| `ime_cur_if` | ime_top_buf | 256 (32×8) | 4+4+5+2+2 | `rden`, `4x4_x[3:0]`, `4x4_y[3:0]`, `idx[4:0]`, `sel[1:0]`, `size[1:0]` |
| `fme_cur_if` | fme_top_buf | 256 (32×8) | 4+4+5 | `rden`, `4x4_x[3:0]`, `4x4_y[3:0]`, `idx[4:0]` |
| `rec_cur_if` | rec_top | 256 (32×8) | 4+4+5+2+2 | `rd_ena`, `sel[1:0]`, `siz[1:0]`, `4x4_x[3:0]`, `4x4_y[3:0]`, `idx[4:0]` |
| `db_cur_if` | dbsao_top | 256 (32×8) | 4+4+5+2+2 | `ren`, `sel[1:0]`, `siz[1:0]`, `4x4_x[3:0]`, `4x4_y[3:0]`, `idx[4:0]` |

**Multiplexing:** `enc_ctrl` arbitrates which subsystem drives the fetch address bus at any given pipeline stage. Only one subsystem reads pixels per cycle.

### 7.2 Reference-Frame Pixel Read (enc_core → fetch)

| Interface | Subsystem | Bus Width | Address Width | Signals |
|-----------|-----------|-----------|---------------|---------|
| `ime_ref_if` | ime_top_buf | 256 (32×8) | `IME_MV_WIDTH_X+1` + `IME_MV_WIDTH_Y+1` | `rden`, `x[7:0]`, `y[6:0]` |
| `fme_ref_if` | fme_top_buf | 512 (64×8) | 8+8 | `rden`, `idx_x[7:0]`, `idx_y[7:0]` |
| `rec_ref_if` | rec_top | 64 (8×8) | 8+8+2 | `rd_ena`, `sel[1:0]`, `idx_x[7:0]`, `idx_y[7:0]` |

### 7.3 Deblocking Reconstruction Write (enc_core → fetch)

| Interface | Bus Width | Address | Signals |
|-----------|-----------|---------|---------|
| `db_rec_if` | 128 (16×8) | 5+5 | `wen`, `w4x4_x[4:0]`, `w4x4_y[4:0]`, `wprevious`, `wdone`, `wsel[1:0]`, `wdata[127:0]` |

### 7.4 Deblocking Top-Row Read (fetch → enc_core)

| Interface | Bus Width | Address | Signals |
|-----------|-----------|---------|---------|
| `db_top_rec_if` | 32 (4×8) | 5+2 | `ren`, `r4x4[4:0]`, `ridx[1:0]`, `rdata[31:0]` |

### 7.5 MV Data Chain (internal to enc_core)

The MV data flows between subsystems through internal wires (not through fetch):

```
ime_top_buf → [inter_partition_w] → fme_top_buf (via fme_partition_i)
ime_top_buf → [fme_mv_data_w]     → fme_top_buf (via fme_mv_data_i, read from ime MV RAM)
fme_top_buf → [mc_mv_rd_dat_w]    → rec_top     (via mc_top mv_rd_dat_i)
fme_top_buf → [db_mv_rd_dat_w]    → dbsao_top   (via mb_mv_rdata_i)
```

---

## 8. Major Control Interfaces

### 8.1 Pipeline Start/Done (`enc_ctrl` → subsystems)

| Subsystem | Start Signal | Done Signal | CTU X/Y |
|-----------|-------------|-------------|---------|
| fetch | `enc_start` | `fetch_done` | (driven by load/store enables) |
| prei | `prei_start_o` | `prei_done_i` | `prei_x_o`, `prei_y_o` |
| posi | `posi_start_o` | `posi_done_i` | `posi_x_o`, `posi_y_o` |
| ime | `ime_start_o` | `ime_done_i` | `ime_x_o`, `ime_y_o` |
| fme | `fme_start_o` | `fme_done_i` | `fme_x_o`, `fme_y_o` |
| rec | `rec_start_o` | `rec_done_i` | `rec_x_o`, `rec_y_o` |
| db | `db_start_o` | `db_done_i` | `db_x_o`, `db_y_o` |
| cabac (ec) | `ec_start_o` | `ec_done_i` | `ec_x_o`, `ec_y_o` |
| db_store | `store_db_start_o` | — | `store_db_x_o`, `store_db_y_o` |

### 8.2 Pipeline FSM States

From `enc_ctrl.v:116-127`:

```
IDLE → S0 → S1 → S2 → S3 → S4 → S5 (steady-state) → S6 → S7 → S8 → S9 → SA → IDLE
```

- **S0–S4**: Pipeline fill (each state enables one additional stage).
- **S5**: Steady-state — all stages active simultaneously.
- **S6–SA**: Pipeline drain (each state disables one stage).
- Transition condition: `enc_start_w` (derived from subsystem done flags).

### 8.3 CTU Coordinate Tracking

`enc_ctrl` maintains independent x/y counters for each pipeline stage. On each `enc_done_r`, coordinates shift forward through the pipeline:

```
INTRA: pre_l → prei → posi → (ime=0) → (fme=0) → rec → db → ec → store_db
INTER: pre_l → prei → posi → ime → fme → rec → db → ec → store_db
```

---

## 9. Memory Interfaces

### 9.1 Internal Buffer Wrappers

The `top/` buffer wrappers (`*_top_buf`) exist to manage memory rotation between pipeline stages:

| Wrapper | Inner Module | Memory Type | Rotation Scheme | Purpose |
|---------|-------------|-------------|-----------------|---------|
| `prei_top_buf` | `prei_top` | `prei_md_ram_sp_85x6` × 2 | 2-way ping-pong | Mode results: prei writes, posi reads |
| `posi_top_buf` | `posi_top` | `posi_md_ram_sp_64x6` × 4 | 4-way rotation (mod 4) | Mode results: write / rec / blank / ec |
| `ime_top_buf` | `ime_top` | `ime_mv_ram_sp_64x13` × 2 | 2-way ping-pong | IME MVs: ime writes, fme reads |
| `fme_top_buf` | `fme_top` | `fme_mv_ram_dp_64x20` × 3, `fme_buf_wrapper` × 2 | 3-way MV rotation, 2-way pred rotation | MVs: FME/MC/DB rotation; prediction pixels: FME/rec rotation |

### 9.2 External Memory Interface

`fetch_top` provides the single external memory port (`ext_if`). It sequences load/store operations for:

- **Current luma**: `load_cur_luma_ena` → fetch → `fetch_cur_luma`
- **Current chroma**: `load_cur_chroma_ena` → fetch → `fetch_cur_chroma`
- **Reference luma**: `load_ref_luma_ena` → fetch → `fetch_ref_luma`
- **Reference chroma**: `load_ref_chroma_ena` → fetch → `fetch_ref_chroma`
- **DB load luma/chroma**: `load_db_*_ena` → fetch → `fetch_db`
- **DB store luma/chroma**: `store_db_*_ena` → fetch → `fetch_db`

### 9.3 Key Memory Parameters

From `enc_defines.v`:

| Macro | Value | Meaning |
|-------|-------|---------|
| `LCU_SIZE` | 64 | Largest Coding Unit (64×64 pixels) |
| `CU_DEPTH` | 3 | CU partition depth (64→32→16→8) |
| `PIXEL_WIDTH` | 8 | Pixel bit-width |
| `COEFF_WIDTH` | 16 | Transform coefficient width (`PIXEL_WIDTH+8`) |
| `FMV_WIDTH` | 10 | Fine motion vector width |
| `MVD_WIDTH` | 11 | Motion vector difference width |
| `IME_MV_WIDTH` | 13 | IME MV width (`IME_MV_WIDTH_X + IME_MV_WIDTH_Y`) |

---

## Evidence Traceability

| Claim | RTL Source |
|-------|-----------|
| Top module = `h265enc_top` | `enc_top.v:32` (module declaration) |
| 3 first-level children | `enc_top.v:334,413,549` (3 instantiations) |
| `enc_core` instantiates 8 blocks | `enc_core.v:512-998` (8 instantiation blocks) |
| 12-state FSM | `enc_ctrl.v:116-127` (state encoding) |
| Active-low async reset | `enc_ctrl.v:262` (`negedge rstn`) |
| Single clock domain | No multi-clock evidence in `enc_top.v` |
| ext_if width = 128 bits | `enc_top.v:152-153` (`16*PIXEL_WIDTH`) |
| bs_if = 8-bit byte | `enc_top.v:157` (`[7:0]`) |
| IME MV RAM ping-pong | `ime_top_buf.v:191-199` (sel_mod_2 muxing) |
| FME MV 3-way rotation | `fme_top_buf.v:327-407` (sel_mod_3 case statements) |
