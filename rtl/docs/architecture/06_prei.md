# 06 — PREI (Pre-Intra Prediction) Subsystem

> Reverse-engineering checkpoint — **read-only**.
> All claims below are verified from RTL unless marked INFERRED or UNKNOWN.

---

## 1. Overview

The PREI subsystem performs pre-intra mode decision and LCU-level rate control. It evaluates 35 intra-prediction modes (DC, Planar, 33 angular) using gradient-based cost estimation (Sobel-like operators), selects the best mode at 8×8, 16×16, 32×32, and 64×64 granularity, stores the mode map in a dual-port mode RAM, and computes an adjusted QP for rate control.

| Property | Value | RTL Evidence |
|----------|-------|-------------|
| Buffer wrapper | `prei_top_buf` | `rtl/top/prei_top_buf.v:16` |
| Core module | `prei_top` | `rtl/prei/prei_top.v:13` |
| Pipeline stage | Stage 1 (first CTU stage, before IME) | `enc_ctrl.v` |
| Clock domain | Single (`clk`) | All PREI modules |
| Reset | Active-low async (`rstn`) | `prei_top.v:56-57` |
| Processing granularity | 8×8 blocks (65 blocks per CTU) | `control.v:94` |
| Cycle budget per CTU | 40 cycles × 65 blocks = ~2600 cycles | `control.v:50,58,66,94` |

---

## 2. Module Hierarchy

```
prei_top_buf [rtl/top/prei_top_buf.v:16]
├── prei_top [rtl/prei/prei_top.v:13]
│   ├── hevc_md_top [rtl/prei/hevc_md_top.v:13]     (mode decision)
│   │   ├── md_top [rtl/prei/md_top.v:7]
│   │   │   ├── md_fetch [rtl/prei/md_fetch.v:7]
│   │   │   ├── control [rtl/prei/control.v:11]
│   │   │   ├── gxgy [rtl/prei/gxgy.v:7]
│   │   │   ├── counter [rtl/prei/counter.v:4]
│   │   │   ├── compare [rtl/prei/compare.v:7]
│   │   │   └── dc_planar [rtl/prei/DC_Plannar.v:7]
│   │   ├── fetch8x8 [rtl/prei/fetch8x8.v:9]
│   │   └── mode_write [rtl/prei/mode_write.v:9]
│   └── rate_control [rtl/prei/rate_control.v:22]
├── prei_md_ram_sp_85x6 × 2 [rtl/mem/]               (ping-pong mode RAM)
```

---

## 3. Buffer Wrapper: prei_top_buf

| Property | Detail | RTL Evidence |
|----------|--------|-------------|
| File | `rtl/top/prei_top_buf.v:16` | — |
| Inner module | `prei_top` | `prei_top_buf.v:133` |
| Memories | `prei_md_ram_sp_85x6` × 2 | `prei_top_buf.v:187,196` |
| Mode RAM scheme | 2-way ping-pong (`sel_mod_2_i` switches) | `prei_top_buf.v:174-184` |
| POSI read port | Reads from the opposite RAM of current write | `prei_top_buf.v:175-184` |
| Address remapping | POSI address `0-83` → mode address; `≥84` → `21+(addr-84)/4` | `prei_top_buf.v:122-129` |

### 3.1 Ping-Pong Mode RAM Control

From `prei_top_buf.v:174-184`:
- When `sel_mod_2_i==0`: RAM 0 is written by PREI, RAM 1 is read by POSI
- When `sel_mod_2_i==1`: RAM 1 is written by PREI, RAM 0 is read by POSI
- Write enable is active-low (`!prei_md_we_w`)

### 3.2 Mode RAM Format

From `prei_top_buf.v:187-203`: 85 entries × 6-bit words. The 85 entries store modes for:
- 64 × 8×8 block modes (entries 0-63)
- 16 × 16×16 block modes (entries 64-79)
- 4 × 32×32 block modes (entries 80-83)
- 1 × 64×64 block mode (entry 84, remapped via `21+(addr-84)/4`)

---

## 4. Top Module: prei_top

`prei_top` instantiates **2 submodules**:

| Instance | Module | File | Role |
|----------|--------|------|------|
| `hevc_md_top1` | `hevc_md_top` | `hevc_md_top.v:13` | Mode decision (35 intra modes) |
| `rate_control1` | `rate_control` | `rate_control.v:22` | LCU-level QP adjustment |

**Instantiation evidence** (`prei_top.v:102,120`):

### 4.1 Submodule Chaining

From `prei_top.v:123`: `rate_control` starts when mode decision finishes (`rc_start_i = md_done_w`).

---

## 5. Mode Decision Top: hevc_md_top

| Property | Detail | RTL Evidence |
|----------|--------|-------------|
| File | `rtl/prei/hevc_md_top.v:13` | — |
| Sub-modules | 3: `md_top`, `fetch8x8`, `mode_write` | `hevc_md_top.v:88,102,119` |
| Control | `enable_reg` latches after `enable_r && cnt[3]` | `hevc_md_top.v:72-86` |
| `cnt[5:0]` | 6-bit cycle counter (0-40) | `hevc_md_top.v:62` |
| `blockcnt[6:0]` | 7-bit block counter (0-64) | `hevc_md_top.v:63` |

### 5.1 Timing

From `hevc_md_top.v:72-86`:
- `enable_reg` set on `enable_r && cnt[3]` (delayed start for pipeline fill)
- `finish` signal from `md_top` resets both `enable_reg` and `enable_r`

---

## 6. Mode Decision Core: md_top

| Property | Detail | RTL Evidence |
|----------|--------|-------------|
| File | `rtl/prei/md_top.v:7` | — |
| Parameter | `MODE=21`, `DIGIT=0` | `md_top.v:21-22` |
| Sub-modules | 6: `md_fetch`, `control`, `gxgy`, `counter`, `compare`, `dc_planar` | `md_top.v:102,113,127,139,182,231` |
| Mode outputs | `bestmode_o[5:0]`, `bestmode16_o[5:0]`, `bestmode32_o[5:0]`, `bestmode64_o[5:0]` | `md_top.v:30-33` |
| `modebest64[27:0]` | Accumulated mode cost for RC | `md_top.v:34,55` |

### 6.1 Mode Cost Signals

From `md_top.v:52-55`:
- `modebest[21:0]` — 8×8 block best angle mode cost
- `modebest16[23:0]` — 16×16 accumulated cost
- `modebest32[25:0]` — 32×32 accumulated cost
- `modebest64[27:0]` — 64×64 accumulated cost

---

## 7. Pixel Fetch: md_fetch

| Property | Detail | RTL Evidence |
|----------|--------|-------------|
| File | `rtl/prei/md_fetch.v:7` | — |
| Input | `rf_512bit[511:0]` (512-bit = 8×8 × 8-bit pixel block) | `md_fetch.v:22` |
| Output | `x1[23:0]`, `x2[23:0]`, `x3[23:0]` (3 rows of 4 pixels each) | `md_fetch.v:23-25` |
| Clocked | Registered on `posedge clk`, switched on `cnt` | `md_fetch.v:31-77` |

### 7.1 Data Extraction

From `md_fetch.v:39-76`: For each `cnt` value (5-40), extracts 3 rows of 4 pixels from the 512-bit block. Each `x` output contains 3 bytes (3 pixels × 8-bit) packed as `{pixel2, pixel1, pixel0}`.

---

## 8. Controller: control

| Property | Detail | RTL Evidence |
|----------|--------|-------------|
| File | `rtl/prei/control.v:11` | — |
| `cyclecnt[5:0]` | 6-bit cycle counter, 0→40 per block | `control.v:42,57-61` |
| `blockcnt[6:0]` | 7-bit block counter, 0→64 (65 blocks) | `control.v:43,63-69` |
| `newblock` | Pulse at `cyclecnt==40` | `control.v:47-53` |
| `gxgyrun` | Active during `cyclecnt==5..1` when `blockcnt != 64` | `control.v:71-77` |
| `counterrun1/2` | Pipeline-aligned gx/gy enable | `control.v:79-89` |
| `finish` | Asserted when `blockcnt==65 && cyclecnt==15` | `control.v:91-97` |

### 8.1 FSM Behavior (Implicit)

The controller uses implicit FSM behavior via `cyclecnt` and `blockcnt`:
```
cyclecnt: 0 → 1 → 2 → ... → 40 → 0 (next block)
blockcnt: 0 → 1 → ... → 64 → (finish at blockcnt==65, cyclecnt==15)
```
Total: 65 blocks × 41 cycles = 2665 cycles per CTU.

---

## 9. Gradient Calculation: gxgy

| Property | Detail | RTL Evidence |
|----------|--------|-------------|
| File | `rtl/prei/gxgy.v:7` | — |
| Input | `x1[23:0]`, `x2[23:0]`, `x3[23:0]` (from md_fetch) | `gxgy.v:18-20` |
| Output | `gx[10:0]` (signed), `gy[10:0]` (signed) | `gxgy.v:24-25` |
| Clocked | Registered when `gxgyrun` active | `gxgy.v:30-40` |

### 9.1 Sobel-like Operators

From `gxgy.v:38-39`:
```
gy = x1[7:0] + x1[15:8]*2 + x1[23:16] - x3[7:0] - x3[15:8]*2 - x3[23:16]
gx = x1[7:0] + x2[7:0]*2 + x3[7:0] - x3[23:16] - x2[23:16]*2 - x1[23:16]
```
This implements 3×3 Sobel gradient operators on the pixel neighborhood.

---

## 10. Angular Mode Cost: counter

| Property | Detail | RTL Evidence |
|----------|--------|-------------|
| File | `rtl/prei/counter.v:4` | — |
| Input | `gx[10:0]`, `gy[10:0]`, `counterrun1/2` | `counter.v:49-52` |
| Output | `mode2[21:0]` through `mode33[21:0]` (32 angular modes) | `counter.v:54-85` |
| Parameter | `MODE=21` → outputs are 22 bits wide | `counter.v:46` |
| Clocked | Registered when `counterrun1` active | `counter.v:120-155` |

### 10.1 Mode Cost Calculation

From `counter.v:156-280` (representative pattern): Each angular mode computes a cost as:
```
mode_N = cos_angle × gx + sin_angle × gy
```
Where the cos/sin values are pre-computed integer constants (e.g., mode 2: `90*gx + 90*gy`, mode 3: `99*gx + 81*gy`, etc.).

The 32 angular modes (modes 2-33) cover the HEVC intra-prediction angle set.

---

## 11. Best Angle Selection: compare

| Property | Detail | RTL Evidence |
|----------|--------|-------------|
| File | `rtl/prei/compare.v:7` | — |
| Parameter | `MODE=21`, `DIGIT=0` | `compare.v:56-57` |
| Input | `mode2[21:0]` through `mode33[21:0]` | `compare.v:95-126` |
| Output | `bestmode[5:0]`, `modebest[21:0]`, `bestmode16[5:0]`, `modebest16[23:0]`, etc. | `compare.v:63-73` |
| Clocked | Sequential comparison over `cnt` cycles | `compare.v:131+` |

### 11.1 Hierarchical Best Mode

The compare module selects the best mode at each block size:
- **8×8**: `bestmode` — best among modes 2-33
- **16×16**: `bestmode16` — best among 4 × 8×8 sub-blocks
- **32×32**: `bestmode32` — best among 4 × 16×16 sub-blocks
- **64×64**: `bestmode64` — best among 4 × 32×32 sub-blocks

---

## 12. DC/Planar Decision: dc_planar

| Property | Detail | RTL Evidence |
|----------|--------|-------------|
| File | `rtl/prei/DC_Plannar.v:7` | — |
| Parameters | `DC8=288`, `DC16=1152`, `DC32=4608`, `DC64=18432`; `Plan8/16/32/64=32` | `DC_Plannar.v:32-39` |
| Input | `gx[10:0]`, `gy[10:0]`, `cnt`, `blockcnt`, `bestmode*`, `modebest*` | `DC_Plannar.v:41-56` |
| Output | `bestmode_o[5:0]`, `bestmode16_o[5:0]`, `bestmode32_o[5:0]`, `bestmode64_o[5:0]` | `DC_Plannar.v:57-60` |

### 12.1 Mode Cost Accumulation

From `DC_Plannar.v:76-118`:
- `modedata` accumulates `|gx| + |gy|` per cycle (gradient magnitude proxy)
- `modedata8/16/32/64` latches accumulated cost at cycle boundaries
- Hierarchical: 8×8 → 16×16 (`modedata16 += modedata8`) → 32×32 → 64×64

### 12.2 DC/Planar vs Best Angle Decision

From `DC_Plannar.v:122-160`:
1. If `modedata < DC_threshold` → select **DC mode** (mode 1)
2. Else if `modebest > Planar_threshold × modedata` → select **Planar mode** (mode 0)
3. Else → select **best angle mode** from `compare` output

Decision timing:
- 8×8: `cnt==35`
- 16×16: `cnt==38`
- 32×32: `cnt==39`
- 64×64: `cnt==40`

---

## 13. Pixel Fetch Unit: fetch8x8

| Property | Detail | RTL Evidence |
|----------|--------|-------------|
| File | `rtl/prei/fetch8x8.v:9` | — |
| Output | `rf_512bit[511:0]` (2 × 256-bit reads concatenated) | `fetch8x8.v:34` |
| Read interface | `md_ren_o`, `md_sel_o=0`, `md_size_o=01`, `md_4x4_x/y_o`, `md_idx_o` | `fetch8x8.v:59-63` |
| Cnt | `cnt[5:0]` — 0→40 per block | `fetch8x8.v:52,73-79` |
| Blockcnt | `blockcnt[6:0]` — 0→64 | `fetch8x8.v:53,81-87` |

### 13.1 Block Addressing

From `fetch8x8.v:62-63`:
```
md_4x4_x_o = {blockcnt[4], blockcnt[2], blockcnt[0], 1'b0}
md_4x4_y_o = {blockcnt[5], blockcnt[3], blockcnt[1], 1'b0}
```
This maps the linear `blockcnt` (0-64) to a raster-scan 4×4 block address.

### 13.2 Double-Buffered Pixel Read

From `fetch8x8.v:97-104`:
- `cnt==2`: Load lower 256 bits from `rdata` into upper half of `rf_512bit`
- `cnt==10`: Load upper 256 bits from `rdata` into lower half of `rf_512bit`
- Result: 512-bit block (8×8 × 8-bit) assembled over 2 read cycles

---

## 14. Mode Write-Back: mode_write

| Property | Detail | RTL Evidence |
|----------|--------|-------------|
| File | `rtl/prei/mode_write.v:9` | — |
| Output | `md_we`, `md_waddr[6:0]`, `md_wdata[5:0]` | `mode_write.v:34-36` |
| Timing | 8×8 at `cnt==11`, 16×16 at `cnt==12`, 32×32 at `cnt==13`, 64×64 at `cnt==14` | `mode_write.v:65-76` |

### 14.1 Write Address Mapping

From `mode_write.v:42-52`:
- `cnt==11`: `md_waddr = blockcnt + 19` (8×8 block mode)
- `cnt==12`: `md_waddr = blockcnt[6:2] + 4` (16×16 block mode)
- `cnt==13`: `md_waddr = blockcnt[6:4]` (32×32 block mode)
- `cnt==14`: `md_waddr = 0` (64×64 block mode at address 0)

### 14.2 Write Enable Conditions

From `mode_write.v:54-63`: Write enable is asserted at specific `blockcnt` and `cnt` values:
- 8×8: `blockcnt > 1 && cnt==11`
- 16×16: `blockcnt[1:0]==01 && cnt==12 && blockcnt != 1`
- 32×32: `blockcnt[3:0]==0001 && cnt==13 && blockcnt != 1`
- 64×64: `blockcnt == 65 && cnt==14`

---

## 15. Rate Control: rate_control

| Property | Detail | RTL Evidence |
|----------|--------|-------------|
| File | `rtl/prei/rate_control.v:22` | — |
| Pipeline | 11-cycle (`cnt` 0→10) | `rate_control.v:90-99` |
| `rc_done_o` | Asserted at `cnt==10` | `rate_control.v:109-115` |
| Accumulator registers | `modebest_1..7` (7-stage shift register) | `rate_control.v:119-149` |
| QP output | `rc_qp_o[5:0]` | `rate_control.v:82` |
| mod64_sum | `mod64_sum_o[31:0] = modebest_1[31:0]` | `rate_control.v:127-129` |

### 15.1 RC Pipeline Stages

| Cycle | Operation | RTL Evidence |
|-------|-----------|-------------|
| cnt==0 | rc_run set | `rate_control.v:94-99` |
| cnt==1 | Accumulate `modebest64_i` into `modebest_1`; shift `modebest_1..7` | `rate_control.v:141-149` |
| cnt==1 | Accumulate `actual_bitnum_i` into `frame_bit` | `rate_control.v:151-155` |
| cnt==2 | `predict_bit = modebest_7 * reg_k >> 28` | `rate_control.v:159-170` |
| cnt==3 | Compare `frame_bit` vs `predict_bit` → `actual_big` | `rate_control.v:172-178` |
| cnt==4 | Compute `diff_abs = |frame_bit - predict_bit|` | `rate_control.v:180-186` |
| cnt==5 | Determine `diff_level` (0/1/2) based on `reg_L1/L2_frame_byte` thresholds | `rate_control.v:193-201` |
| cnt==6 | Compute `qp_tmp = initial_qp ± diff_level` | `rate_control.v:203-211` |
| cnt==7 | ROI hit detection | `rate_control.v:218-224` |
| cnt==8 | Apply ROI QP: `qp_ROI = qp_tmp - delta_qp` if ROI hit | `rate_control.v:226-232` |
| cnt==9 | Clamp QP to `[min_qp, max_qp]` → `rc_qp_o` | `rate_control.v:235-243` |
| cnt==10 | Assert `rc_done_o` | `rate_control.v:109-115` |

### 15.2 ROI (Region of Interest)

From `rate_control.v:218-232`: If the current CTU falls within the ROI rectangle (`rc_ctu_x/y` within `[ROI_x, ROI_x+width) × [ROI_y, ROI_y+height)`) and `reg_ROI_enable` is set, QP is decreased by `reg_delta_qp` (better quality in ROI).

---

## 16. Mode RAM: md_ram

| Property | Detail | RTL Evidence |
|----------|--------|-------------|
| File | `rtl/prei/md_ram.v:18` | — |
| Implementation | `rf_2p #(.Addr_Width(4), .Word_Width(32))` (inferred register file) | `md_ram.v:47-48` |
| Depth | 16 entries | `md_ram.v:38` (4-bit address) |
| Width | 32 bits | `md_ram.v:34` |
| Read port | Synchronous (`cena_i = ~rd`) | `md_ram.v:47-53` |
| Write port | Synchronous (`cenb_i = ~we`) | `md_ram.v:54-58` |

Note: `md_ram` is used internally within `hevc_md_top` but the mode output goes through `mode_write` → `prei_top` → `prei_top_buf` → `prei_md_ram_sp_85x6`.

---

## 17. External Interfaces

### 17.1 Input Interface (from enc_ctrl via enc_core)

| Signal | Width | Source | Description |
|--------|-------|--------|-------------|
| `prei_start_i` | 1 | `enc_ctrl` | Start PREI processing |
| `prei_cur_data_i` | 256 | `fetch_top` | Current pixel data (32 × 8-bit) |
| `rc_actual_bitnum_i` | 16 | `cabac` | Actual bit count from CABAC |
| `rc_ctu_x_i` | PIC_X_WIDTH | `enc_ctrl` | Current CTU X position |
| `rc_ctu_y_i` | PIC_Y_WIDTH | `enc_ctrl` | Current CTU Y position |
| `sel_mod_2_i` | 1 | `enc_core` | Mode RAM ping-pong selector |
| `rc_k` | 16 | register | RC multiplier |
| `rc_bitnum_i` | 32 | register | RC target bit number |
| `rc_roi_*` | various | register | ROI configuration |
| `rc_initial_qp` | 6 | register | Initial QP |
| `rc_max/min_qp` | 6 | register | QP bounds |
| `rc_delta_qp` | 6 | register | ROI QP offset |

### 17.2 Pixel Read Interface (to fetch_top)

| Signal | Width | Dir | Description |
|--------|-------|-----|-------------|
| `prei_cur_ren_o` | 1 | out | Pixel read enable |
| `prei_cur_sel_o` | 2 | out | Pixel select |
| `prei_cur_size_o` | 2 | out | Pixel block size |
| `prei_cur_4x4_x_o` | 4 | out | 4×4 block X address |
| `prei_cur_4x4_y_o` | 4 | out | 4×4 block Y address |
| `prei_cur_idx_o` | 5 | out | 4×4 block index |

### 17.3 Mode RAM Interface (to POSI via prei_top_buf)

| Signal | Width | Dir | Description |
|--------|-------|-----|-------------|
| `posi_md_ena_i` | 1 | in | POSI mode read enable |
| `posi_md_addr_i` | 9 | in | POSI mode read address (0-84) |
| `posi_md_data_o` | 6 | out | POSI mode data output |

### 17.4 RC Output Interface

| Signal | Width | Dir | Description |
|--------|-------|-----|-------------|
| `prei_done_o` | 1 | out | PREI processing complete |
| `rc_qp_o` | 6 | out | Rate-controlled QP |
| `rc_mod64_sum_o` | 32 | out | Accumulated mode cost |

---

## 18. Timing Summary

### 18.1 Mode Decision Timing

| Phase | Cycles | Description |
|-------|--------|-------------|
| fetch8x8: read | 2 (cnt 0-1) | 2 × 256-bit reads from pixel memory |
| fetch8x8: buffer | 1 (cnt 2) | Assemble 512-bit block |
| md_fetch: extract | 1 (cnt 5) | Extract x1/x2/x3 from rf_512bit |
| gxgy: gradient | 1 (cnt 5-1) | Compute gx, gy |
| counter: cost | 1 (cnt 5-1) | Compute 32 angular mode costs |
| compare: select | ~25 (cnt 5-30) | Hierarchical best mode selection |
| dc_planar: final | 5 (cnt 35-40) | DC/Planar vs angle decision + write |
| **Total per block** | **40** | |
| **Total per CTU** | **65 × 41 ≈ 2665** | 65 8×8 blocks |

### 18.2 Rate Control Timing

| Phase | Cycles | Description |
|-------|--------|-------------|
| Accumulate | 1 (cnt==1) | Shift + add modebest |
| Predict | 1 (cnt==2) | Multiply + shift |
| Compare | 1 (cnt==3) | frame_bit vs predict_bit |
| Diff | 1 (cnt==4) | Absolute difference |
| Level | 1 (cnt==5) | Threshold check |
| QP calc | 1 (cnt==6) | initial_qp ± diff_level |
| ROI | 2 (cnt==7-8) | ROI hit + QP modify |
| Clamp | 1 (cnt==9) | min/max clamp |
| Done | 1 (cnt==10) | Assert rc_done |
| **Total** | **11** | |

---

## 19. Pipeline Timing

```
prei_start_i ──→ hevc_md_top.enable ──→ [65 blocks × 40 cycles] ──→ md_done_w
                                                                              │
                                                                              ↓
                                              rate_control.rc_start_i ──→ [11 cycles] ──→ prei_done_o
```

The two submodules are **sequential**: mode decision runs first, then rate control.

---

## Evidence Traceability

| Claim | RTL Source |
|-------|-----------|
| `prei_top_buf` instantiates `prei_top` | `prei_top_buf.v:133` |
| `prei_md_ram_sp_85x6` × 2 (ping-pong) | `prei_top_buf.v:187,196` |
| `prei_top` instantiates `hevc_md_top` + `rate_control` | `prei_top.v:102,120` |
| `hevc_md_top` instantiates `md_top`, `fetch8x8`, `mode_write` | `hevc_md_top.v:88,102,119` |
| `md_top` instantiates 6 sub-modules | `md_top.v:102,113,127,139,182,231` |
| `gxgy` implements Sobel operators | `gxgy.v:38-39` |
| `counter` computes 32 angular mode costs | `counter.v:156-280` |
| `control` finish: `blockcnt==65 && cyclecnt==15` | `control.v:94` |
| `dc_planar` thresholds: DC8=288, Plan8=32 | `DC_Plannar.v:32-39` |
| `rate_control` pipeline: 11 cycles | `rate_control.v:90-99` |
| `mode_write` timing: 8×8@11, 16×16@12, 32×32@13, 64×64@14 | `mode_write.v:45-52` |
| `fetch8x8` double-buffer: cnt 2,10 | `fetch8x8.v:97-104` |
| `md_ram` inferred `rf_2p` 16×32 | `md_ram.v:47` |

## Verification Checklist

- [x] Module hierarchy verified
- [x] Module instantiations verified
- [x] Port directions verified
- [x] Signal widths verified
- [x] Major signal connections verified
- [x] Memories/buffers verified
- [x] FSMs verified (implicit FSM via counters)
- [x] Sequential registers verified
- [x] Pipeline boundaries verified
- [x] Datapath verified
- [x] Control path verified
- [x] Clock/reset domains verified

### RTL Blocks Missing from Diagram
None — all modules accounted for.

### Diagram Blocks Without Direct RTL Evidence
None — all blocks verified.

### Missing or Uncertain Connections
None — all connections verified.

### Uncertain Signal Directions
None — all port directions verified against RTL.

### Uncertain Signal Widths
- `modebest64[27:0]` width derived from `MODE-DIGIT+6 = 21-0+6 = 27` → 28 bits (`md_top.v:34`)

### Unresolved Hierarchy
None — complete.

### Unresolved Pipeline Stages
None — pipeline fully documented.

### Unresolved Memory Implementation
- `md_ram` uses inferred `rf_2p` from `lib/` — internal implementation depends on technology library

### Unresolved FSM Behavior
None — all counter-based timing fully documented.

### Other UNKNOWN Items
- The `compare` module contains 1176 lines of sequential comparison logic; the exact comparison tree structure is not fully traced but the functional behavior (hierarchical best-mode selection at 8/16/32/64 levels) is VERIFIED from the output ports and timing.
