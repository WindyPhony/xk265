# 00 — Repository Map

> Reverse-engineering checkpoint — **read-only reconnaissance**.
> Nothing in this document is a redesign claim. Every statement below is
> traceable to actual RTL syntax unless marked INFERRED or UNKNOWN.

---

## A. Repository Hierarchy

```
rtl/
├── enc_defines.v              ← shared macros (LCU size, widths, partition codes)
├── top/                       ← system-level integration
│   ├── enc_top.v              ← h265enc_top (DESIGN TOP)
│   ├── enc_ctrl.v             ← pipeline FSM + start/done sequencing
│   ├── enc_core.v             ← datapath: instantiates all processing blocks
│   ├── enc_data_pipeline.v    ← QP/partition/CBF/skip pipeline registers
│   ├── prei_top_buf.v         ← prei_top + MD RAM ping-pong wrapper
│   ├── posi_top_buf.v         ← posi_top + MD RAM quad-rotation wrapper
│   ├── ime_top_buf.v          ← ime_top + MV RAM ping-pong wrapper
│   └── fme_top_buf.v          ← fme_top + MV/prediction RAM rotation wrapper
├── fetch/                     ← pixel data fetch & external memory interface
│   ├── fetch_top.v            ← fetch subsystem top
│   ├── fetch_wrapper.v        ← external-IF sequencing / done generation
│   ├── fetch_wrapper/
│   │   └── fetch_buf_wrapper_yuv.v
│   ├── fetch_cur_luma.v       ← current-frame luma pixel buffer
│   ├── fetch_cur_chroma.v     ← current-frame chroma pixel buffer
│   ├── fetch_ref_luma.v       ← reference-frame luma search window
│   ├── fetch_ref_chroma.v     ← reference-frame chroma search window
│   ├── fetch_db.v             ← DB rec-pixel store/load buffer
│   └── mem_bilo_db.v          ← DB top-rec memory
├── prei/                      ← pre-intra estimation + rate control
│   ├── prei_top.v             ← subsystem top
│   ├── hevc_md_top.v          ← HEVC mode-decision kernel
│   ├── md_top.v               ← mode-decision FSM
│   ├── md_fetch.v             ← mode-decision pixel fetch
│   ├── md_ram.v               ← MD result storage
│   ├── rate_control.v         ← QP / bit allocation
│   ├── DC_Plannar.v           ← DC/Planar mode evaluator
│   ├── gxgy.v                 ← gradient (SAD) helpers
│   ├── fetch8x8.v             ← 8×8 sub-block fetch
│   ├── control.v              ← prei FSM
│   ├── compare.v              ← cost comparator
│   ├── counter.v              ← iteration counter
│   └── mode_write.v           ← MD result writer
├── posi/                      ← post-intra SATD cost & partition decision
│   ├── posi_top.v             ← subsystem top
│   ├── posi_ctrl.v            ← FSM
│   ├── posi_prediction.v      ← intra predictor
│   ├── posi_satd_cost.v       ← SATD cost computation
│   ├── posi_satd_cost_engine.v
│   ├── posi_satd_cost_transpose.v
│   ├── posi_rate_estimation.v ← rate estimation
│   ├── posi_partition_decision.v ← best partition selector
│   ├── posi_reference.v       ← reference-pixel supply
│   ├── posi_buffer.v          ← internal buffer
│   ├── posi_memory_wrapper.v  ← MD RAM access
│   └── posi_transfer.v        ← result output mux
├── ime/                       ← integer motion estimation
│   ├── ime_top.v              ← subsystem top
│   ├── ime_ctrl.v             ← search FSM
│   ├── ime_sad_array.v        ← SAD computation array
│   ├── ime_dat_array.v        ← data (pixel) array
│   ├── ime_addressing.v       ← search-window address generator
│   ├── ime_cost_store.v       ← best-cost register file
│   ├── ime_partition_decision.v
│   ├── ime_partition_decision_engine.v
│   ├── ime_mv_dump.v          ← MV result output
│   ├── ime_transfer.v         ← result transfer
│   └── ime_ver_mem.v          ← vertical search-memory (optional)
├── fme/                       ← fractional motion estimation
│   ├── fme_top.v              ← subsystem top
│   ├── fme_ctrl.v             ← FME FSM
│   ├── fme_interpolator.v     ← fractional-pel interpolation core
│   ├── fme_interpolator_8pel.v
│   ├── fme_interpolator_8x8.v
│   ├── fme_ip_half_ver.v      ← half-pel vertical filter
│   ├── fme_ip_quarter_ver.v   ← quarter-pel vertical filter
│   ├── fme_cost.v             ← SATD cost
│   ├── fme_satd_8x8.v
│   ├── fme_satd_gen.v
│   ├── fme_pred.v             ← prediction assembly
│   ├── fme_mv_buffer.v        ← MV storage
│   ├── fme_mv_candidate_prepare.v ← MV candidate list
│   ├── fme_buf_wrapper.v      ← prediction-pixel buffer
│   ├── fme_skip.v             ← skip-mode decision
│   ├── getbits.v              ← bit-count for rate estimation
│   └── qp_lambda_table.v      ← QP→λ lookup
├── rec/                       ← reconstruction path
│   ├── rec_top.v              ← subsystem top (intra + MC + TQ + buffer)
│   ├── IinP_flag_gen.v        ← I-block-in-P-frame flag generator
│   ├── rec_intra/             ← intra prediction
│   │   ├── intra_top.v
│   │   ├── intra_ctrl.v
│   │   ├── intra_pred.v
│   │   ├── intra_ref.v
│   │   └── intra_buf_wrapper.v
│   ├── rec_mc/                ← motion compensation
│   │   ├── mc_top.v
│   │   ├── mc_ctrl.v
│   │   ├── mc_tq.v
│   │   ├── mc_chroma_top.v
│   │   ├── mc_chroma_filter.v
│   │   ├── mc_chroma_ip_1p.v
│   │   ├── mc_chroma_ip4x4.v
│   │   ├── mvd_top.v
│   │   ├── mvd_can_mv_addr.v
│   │   └── mvd_getBits.v
│   ├── rec_tq/                ← transform & quantization
│   │   ├── tq_top.v
│   │   ├── dct_top_2d.v
│   │   ├── quan.v
│   │   ├── re.v, re_level0..3.v, re_level0_cal..2_cal.v
│   │   ├── re_in_ctl.v, re_out_ctl.v
│   │   ├── pe.v, pe_i.v
│   │   ├── be.v, be_level0.v, be_level1.v, be_delay.v
│   │   ├── ctl0..3.v, mux0..3.v, mux32_1.v
│   │   ├── addr_ctl.v, row_ctl.v
│   │   ├── chroma_qp.v, offset_shift.v, mod.v
│   │   └── transform_mtr.v
│   └── rec_wrapper/           ← buffer management for rec/coeff/MVD
│       ├── rec_buf_wrapper.v
│       ├── rec_buf_pre.v
│       ├── rec_buf_rec.v, rec_buf_rec_rot.v
│       ├── rec_buf_cef.v, rec_buf_cef_rot.v
│       └── rec_buf_mvd_rot.v
├── db/                        ← deblocking filter + SAO
│   ├── dbsao_top.v            ← subsystem top
│   ├── dbsao_controller.v     ← DB/SAO scheduling FSM
│   ├── dbsao_datapath.v       ← DB + SAO datapath
│   ├── db_filter.v            ← DB filter core
│   ├── db_normal_filter.v
│   ├── db_strong_filter.v
│   ├── db_chroma_filter.v
│   ├── db_bs.v                ← boundary-strength computation
│   ├── db_pu_edge.v           ← PU edge detection
│   ├── db_tu_edge.v           ← TU edge detection
│   ├── db_qp.v                ← QP derivation
│   ├── db_mv.v                ← MV read for BS
│   ├── db_clip3_str.v         ← clip helper
│   ├── db_lut_beta.v          ← β LUT
│   ├── db_lut_tc.v            ← tc LUT
│   ├── sao_top.v              ← SAO top
│   ├── sao_statistic.v        ← SAO band/cat statistics
│   ├── sao_cal_offset.v       ← SAO offset computation
│   ├── sao_mode.v             ← SAO mode decision
│   ├── sao_type_decision.v
│   ├── sao_bo_predecision.v   ← band offset pre-decision
│   ├── sao_sum_diff.v
│   └── sao_add_offset.v
├── cabac/                     ← CABAC entropy coding
│   ├── cabac_top.v            ← subsystem top
│   ├── cabac_binsort.v        ← bin-slice scheduler
│   ├── cabac_binmix.v         ← bin arithmetic mixing
│   ├── cabac_bina.v           ← binarization
│   ├── cabac_bina_tools.v     ← binarization helpers
│   ├── cabac_bina_lut.v       ← binarization LUTs
│   ├── cabac_bitpack.v        ← bitstream packing
│   ├── cabac_se_prepare.v     ← SE preparation top
│   ├── cabac_se_prepare_*.v   ← per-SE-type preparation (cu, tu, coeff, mvd, etc.)
│   ├── cabac_ucontext.v       ← context-model update
│   ├── cabac_ucontext_t.v, cabac_ucontext_tt.v
│   ├── cabac_ulow.v           ← low-level encoding
│   ├── cabac_ulow_1bin.v
│   ├── cabac_ulow_refine.v
│   ├── cabac_urange4.v, cabac_urange4_full.v
│   ├── cabac_rlps4.v, cabac_rlps4_1bin.v
│   ├── coe_addr_trans.v       ← coefficient address transform
│   └── pipo.v                 ← pipeline FIFO
└── mem/                       ← memory wrappers (34 files)
    ├── buf_ram_1p_*.v         ← buffer RAMs (1-port)
    ├── ram_sp_*.v             ← single-port RAMs
    ├── ram_sp_be_*.v          ← single-port RAMs with byte-enable
    ├── ram_tp_be_*.v          ← two-port RAMs with byte-enable
    ├── fetch_ram_*.v          ← fetch-subsystem RAMs
    ├── fetch_rf_*.v           ← fetch-subsystem register files
    ├── fme_mv_ram_dp_*.v      ← FME MV dual-port RAM
    ├── ime_mv_ram_sp_*.v      ← IME MV single-port RAM
    ├── mc_mv_ram_sp_*.v       ← MC MV single-port RAM
    ├── cabac_ram_sp_*.v       ← CABAC RAM
    ├── db_*_ram_sp_*.v        ← DB/SAO RAMs
    ├── posi_md_ram_sp_*.v     ← POSI mode-decision RAM
    ├── prei_md_ram_sp_*.v     ← PREI mode-decision RAM
    ├── tq_ram_sp_*.v          ← TQ coefficient RAM
    └── mem_lipo_*.v           ← line-pixel buffers
```

**Total .v files:** 159 (including `enc_defines.v`)

---

## B. Functional Subsystem Map

| # | Directory | Subsystem | Purpose (RTL-inferred) | File Count |
|---|-----------|-----------|----------------------|------------|
| 1 | `top/` | System Integration | Top module, pipeline FSM, buffer wrappers | 8 |
| 2 | `fetch/` | Data Fetch | External IF, current/ref pixel buffers, DB buffer | 8 |
| 3 | `prei/` | Pre-Intra Estimation | Mode decision, rate control, QP | 13 |
| 4 | `posi/` | Post-Intra Prediction | SATD cost, partition decision | 12 |
| 5 | `ime/` | Integer Motion Estimation | SAD search, MV output | 11 |
| 6 | `fme/` | Fractional Motion Estimation | Interpolation, SATD, skip decision | 17 |
| 7 | `rec/` | Reconstruction | Intra pred, MC, TQ, buffer management | 2+subdirs |
| 8 | `rec/rec_intra/` | Intra Prediction | Directional intra prediction | 5 |
| 9 | `rec/rec_mc/` | Motion Compensation | MC filtering, MVD | 10 |
| 10 | `rec/rec_tq/` | Transform & Quantization | DCT, quant, residue coding | 28 |
| 11 | `rec/rec_wrapper/` | Rec Buffer Wrapper | Rotation buffers for rec/coeff/MVD | 7 |
| 12 | `db/` | Deblocking + SAO | Loop filter, sample adaptive offset | 23 |
| 13 | `cabac/` | CABAC Entropy Coding | Context-adaptive binary arithmetic coding | 31 |
| 14 | `mem/` | Memory Wrappers | RAM/RF behavioral models | 34 |

---

## C. Candidate Top Modules

| Candidate | File | Role | Verdict |
|-----------|------|------|---------|
| `h265enc_top` | `top/enc_top.v` | Design top: instantiates `enc_ctrl`, `fetch_top`, `enc_core` | **DESIGN TOP** |
| `enc_ctrl` | `top/enc_ctrl.v` | Pipeline FSM sequencing start/done for all subsystems | Controller only |
| `enc_core` | `top/enc_core.v` | Datapath: instantiates all processing blocks + data pipeline | Datapath wrapper |
| `tb_enc_top` | `sim/.../tb_enc_top.v` | Testbench wrapper (drives `h265enc_top`) | Testbench |

**Confirmed:** `h265enc_top` is the single design top module.

---

## D. Module Inventory

### D.1 Top-level (`top/`)

| Module | File | Instantiated By | Submodules Instantiated |
|--------|------|-----------------|------------------------|
| `h265enc_top` | `enc_top.v` | `tb_enc_top` (testbench) | `enc_ctrl`, `fetch_top`, `enc_core` |
| `enc_ctrl` | `enc_ctrl.v` | `h265enc_top` | — (pure FSM) |
| `enc_core` | `enc_core.v` | `h265enc_top` | `prei_top_buf`, `posi_top_buf`, `ime_top_buf`, `fme_top_buf`, `rec_top`, `dbsao_top`, `cabac_top`, `enc_data_pipeline` |
| `enc_data_pipeline` | `enc_data_pipeline.v` | `enc_core` | — (pipeline registers) |
| `prei_top_buf` | `prei_top_buf.v` | `enc_core` | `prei_top`, `prei_md_ram_sp_85x6` ×2 |
| `posi_top_buf` | `posi_top_buf.v` | `enc_core` | `posi_top`, `posi_md_ram_sp_64x6` ×4 |
| `ime_top_buf` | `ime_top_buf.v` | `enc_core` | `ime_top`, `ime_mv_ram_sp_64x13` ×2 |
| `fme_top_buf` | `fme_top_buf.v` | `enc_core` | `fme_top`, `fme_buf_wrapper` ×2, `fme_mv_ram_dp_64x20` ×3 |

### D.2 Fetch (`fetch/`)

| Module | File | Instantiated By |
|--------|------|-----------------|
| `fetch_top` | `fetch_top.v` | `h265enc_top` |
| `fetch_wrapper` | `fetch_wrapper.v` | `fetch_top` |
| `fetch_buf_wrapper_yuv` | `fetch_wrapper/fetch_buf_wrapper_yuv.v` | `fetch_wrapper` |
| `fetch_cur_luma` | `fetch_cur_luma.v` | `fetch_top` |
| `fetch_cur_chroma` | `fetch_cur_chroma.v` | `fetch_top` |
| `fetch_ref_luma` | `fetch_ref_luma.v` | `fetch_top` |
| `fetch_ref_chroma` | `fetch_ref_chroma.v` | `fetch_top` |
| `fetch_db` | `fetch_db.v` | `fetch_top` |
| `mem_bilo_db` | `mem_bilo_db.v` | `fetch_db` |

### D.3 Pre-Intra Estimation (`prei/`)

| Module | File | Instantiated By |
|--------|------|-----------------|
| `prei_top` | `prei_top.v` | `prei_top_buf` |
| `hevc_md_top` | `hevc_md_top.v` | `prei_top` |
| `md_top` | `md_top.v` | `hevc_md_top` |
| `md_fetch` | `md_fetch.v` | `md_top` |
| `md_ram` | `md_ram.v` | `md_top` |
| `rate_control` | `rate_control.v` | `prei_top` |
| `DC_Plannar` | `DC_Plannar.v` | `hevc_md_top` |
| `gxgy` | `gxgy.v` | `hevc_md_top` |
| `fetch8x8` | `fetch8x8.v` | `hevc_md_top` |
| `control` | `control.v` | `prei_top` |
| `compare` | `compare.v` | `hevc_md_top` |
| `counter` | `counter.v` | `prei_top` |
| `mode_write` | `mode_write.v` | `hevc_md_top` |

### D.4 Post-Intra Prediction (`posi/`)

| Module | File | Instantiated By |
|--------|------|-----------------|
| `posi_top` | `posi_top.v` | `posi_top_buf` |
| `posi_ctrl` | `posi_ctrl.v` | `posi_top` |
| `posi_prediction` | `posi_prediction.v` | `posi_top` |
| `posi_satd_cost` | `posi_satd_cost.v` | `posi_top` |
| `posi_satd_cost_engine` | `posi_satd_cost_engine.v` | `posi_satd_cost` |
| `posi_satd_cost_transpose` | `posi_satd_cost_transpose.v` | `posi_satd_cost` |
| `posi_rate_estimation` | `posi_rate_estimation.v` | `posi_top` |
| `posi_partition_decision` | `posi_partition_decision.v` | `posi_top` |
| `posi_reference` | `posi_reference.v` | `posi_top` |
| `posi_buffer` | `posi_buffer.v` | `posi_top` |
| `posi_memory_wrapper` | `posi_memory_wrapper.v` | `posi_top` |
| `posi_transfer` | `posi_transfer.v` | `posi_top` |

### D.5 Integer Motion Estimation (`ime/`)

| Module | File | Instantiated By |
|--------|------|-----------------|
| `ime_top` | `ime_top.v` | `ime_top_buf` |
| `ime_ctrl` | `ime_ctrl.v` | `ime_top` |
| `ime_sad_array` | `ime_sad_array.v` | `ime_top` |
| `ime_dat_array` | `ime_dat_array.v` | `ime_top` |
| `ime_addressing` | `ime_addressing.v` | `ime_top` |
| `ime_cost_store` | `ime_cost_store.v` | `ime_top` |
| `ime_partition_decision` | `ime_partition_decision.v` | `ime_top` |
| `ime_partition_decision_engine` | `ime_partition_decision_engine.v` | `ime_partition_decision` |
| `ime_mv_dump` | `ime_mv_dump.v` | `ime_top` |
| `ime_transfer` | `ime_transfer.v` | `ime_top` |
| `ime_ver_mem` | `ime_ver_mem.v` | `ime_top` (ifdef `IME_HAS_VER_MEM`) |

### D.6 Fractional Motion Estimation (`fme/`)

| Module | File | Instantiated By |
|--------|------|-----------------|
| `fme_top` | `fme_top.v` | `fme_top_buf` |
| `fme_ctrl` | `fme_ctrl.v` | `fme_top` |
| `fme_interpolator` | `fme_interpolator.v` | `fme_top` |
| `fme_interpolator_8pel` | `fme_interpolator_8pel.v` | `fme_interpolator` |
| `fme_interpolator_8x8` | `fme_interpolator_8x8.v` | `fme_interpolator` |
| `fme_ip_half_ver` | `fme_ip_half_ver.v` | `fme_interpolator` |
| `fme_ip_quarter_ver` | `fme_ip_quarter_ver.v` | `fme_interpolator` |
| `fme_cost` | `fme_cost.v` | `fme_top` |
| `fme_satd_8x8` | `fme_satd_8x8.v` | `fme_cost` |
| `fme_satd_gen` | `fme_satd_gen.v` | `fme_cost` |
| `fme_pred` | `fme_pred.v` | `fme_top` |
| `fme_mv_buffer` | `fme_mv_buffer.v` | `fme_top` |
| `fme_mv_candidate_prepare` | `fme_mv_candidate_prepare.v` | `fme_top` |
| `fme_buf_wrapper` | `fme_buf_wrapper.v` | `fme_top_buf` |
| `fme_skip` | `fme_skip.v` | `fme_top` |
| `getbits` | `getbits.v` | `fme_top` |
| `qp_lambda_table` | `qp_lambda_table.v` | `fme_top` |

### D.7 Reconstruction (`rec/`)

| Module | File | Instantiated By |
|--------|------|-----------------|
| `rec_top` | `rec/rec_top.v` | `enc_core` |
| `IinP_flag_gen` | `rec/IinP_flag_gen.v` | `rec_top` |
| **rec_intra:** | | |
| `intra_top` | `rec_intra/intra_top.v` | `rec_top` |
| `intra_ctrl` | `rec_intra/intra_ctrl.v` | `intra_top` |
| `intra_pred` | `rec_intra/intra_pred.v` | `intra_top` |
| `intra_ref` | `rec_intra/intra_ref.v` | `intra_top` |
| `intra_buf_wrapper` | `rec_intra/intra_buf_wrapper.v` | `intra_top` |
| **rec_mc:** | | |
| `mc_top` | `rec_mc/mc_top.v` | `rec_top` |
| `mc_ctrl` | `rec_mc/mc_ctrl.v` | `mc_top` |
| `mc_tq` | `rec_mc/mc_tq.v` | `mc_top` |
| `mc_chroma_top` | `rec_mc/mc_chroma_top.v` | `mc_top` |
| `mc_chroma_filter` | `rec_mc/mc_chroma_filter.v` | `mc_chroma_top` |
| `mc_chroma_ip_1p` | `rec_mc/mc_chroma_ip_1p.v` | `mc_chroma_top` |
| `mc_chroma_ip4x4` | `rec_mc/mc_chroma_ip4x4.v` | `mc_chroma_top` |
| `mvd_top` | `rec_mc/mvd_top.v` | `mc_top` |
| `mvd_can_mv_addr` | `rec_mc/mvd_can_mv_addr.v` | `mvd_top` |
| `mvd_getBits` | `rec_mc/mvd_getBits.v` | `mvd_top` |
| **rec_tq:** | | |
| `tq_top` | `rec_tq/tq_top.v` | `rec_top` |
| `dct_top_2d` | `rec_tq/dct_top_2d.v` | `tq_top` |
| `quan` | `rec_tq/quan.v` | `tq_top` |
| `re` | `rec_tq/re.v` | `tq_top` |
| `re_level0`..`re_level3` | `rec_tq/re_level*.v` | `re` |
| `re_level0_cal`..`re_level2_cal` | `rec_tq/re_level*_cal.v` | `re` |
| `re_in_ctl` | `rec_tq/re_in_ctl.v` | `re` |
| `re_out_ctl` | `rec_tq/re_out_ctl.v` | `re` |
| `pe`, `pe_i` | `rec_tq/pe.v`, `pe_i.v` | `tq_top` |
| `be`, `be_level0`, `be_level1`, `be_delay` | `rec_tq/be*.v` | `tq_top` |
| `ctl0`..`ctl3` | `rec_tq/ctl*.v` | `tq_top` |
| `mux0`..`mux3`, `mux32_1` | `rec_tq/mux*.v` | `tq_top` |
| `addr_ctl` | `rec_tq/addr_ctl.v` | `tq_top` |
| `row_ctl` | `rec_tq/row_ctl.v` | `tq_top` |
| `chroma_qp` | `rec_tq/chroma_qp.v` | `tq_top` |
| `offset_shift` | `rec_tq/offset_shift.v` | `tq_top` |
| `mod` | `rec_tq/mod.v` | `tq_top` |
| `transform_mtr` | `rec_tq/transform_mtr.v` | `tq_top` |
| **rec_wrapper:** | | |
| `rec_buf_wrapper` | `rec_wrapper/rec_buf_wrapper.v` | `rec_top` |
| `rec_buf_pre` | `rec_wrapper/rec_buf_pre.v` | `rec_buf_wrapper` |
| `rec_buf_rec` | `rec_wrapper/rec_buf_rec.v` | `rec_buf_wrapper` |
| `rec_buf_rec_rot` | `rec_wrapper/rec_buf_rec_rot.v` | `rec_buf_wrapper` |
| `rec_buf_cef` | `rec_wrapper/rec_buf_cef.v` | `rec_buf_wrapper` |
| `rec_buf_cef_rot` | `rec_wrapper/rec_buf_cef_rot.v` | `rec_buf_wrapper` |
| `rec_buf_mvd_rot` | `rec_wrapper/rec_buf_mvd_rot.v` | `rec_buf_wrapper` |

### D.8 Deblocking + SAO (`db/`)

| Module | File | Instantiated By |
|--------|------|-----------------|
| `dbsao_top` | `dbsao_top.v` | `enc_core` |
| `dbsao_controller` | `dbsao_controller.v` | `dbsao_top` |
| `dbsao_datapath` | `dbsao_datapath.v` | `dbsao_top` |
| `db_filter` | `db_filter.v` | `dbsao_datapath` |
| `db_normal_filter` | `db_normal_filter.v` | `db_filter` |
| `db_strong_filter` | `db_strong_filter.v` | `db_filter` |
| `db_chroma_filter` | `db_chroma_filter.v` | `db_filter` |
| `db_bs` | `db_bs.v` | `dbsao_datapath` |
| `db_pu_edge` | `db_pu_edge.v` | `db_bs` |
| `db_tu_edge` | `db_tu_edge.v` | `db_bs` |
| `db_qp` | `db_qp.v` | `dbsao_datapath` |
| `db_mv` | `db_mv.v` | `dbsao_datapath` |
| `db_clip3_str` | `db_clip3_str.v` | `db_filter` |
| `db_lut_beta` | `db_lut_beta.v` | `db_filter` |
| `db_lut_tc` | `db_lut_tc.v` | `db_filter` |
| `sao_top` | `sao_top.v` | `dbsao_datapath` |
| `sao_statistic` | `sao_statistic.v` | `sao_top` |
| `sao_cal_offset` | `sao_cal_offset.v` | `sao_top` |
| `sao_mode` | `sao_mode.v` | `sao_top` |
| `sao_type_decision` | `sao_type_decision.v` | `sao_top` |
| `sao_bo_predecision` | `sao_bo_predecision.v` | `sao_top` |
| `sao_sum_diff` | `sao_sum_diff.v` | `sao_top` |
| `sao_add_offset` | `sao_add_offset.v` | `sao_top` |

### D.9 CABAC (`cabac/`)

| Module | File | Instantiated By |
|--------|------|-----------------|
| `cabac_top` | `cabac_top.v` | `enc_core` |
| `cabac_binsort` | `cabac_binsort.v` | `cabac_top` |
| `cabac_binmix` | `cabac_binmix.v` | `cabac_binsort` |
| `cabac_bina` | `cabac_bina.v` | `cabac_binsort` |
| `cabac_bina_tools` | `cabac_bina_tools.v` | `cabac_bina` |
| `cabac_bina_lut` | `cabac_bina_lut.v` | `cabac_bina` |
| `cabac_bitpack` | `cabac_bitpack.v` | `cabac_top` |
| `cabac_se_prepare` | `cabac_se_prepare.v` | `cabac_binsort` |
| `cabac_se_prepare_cu` | `cabac_se_prepare_cu.v` | `cabac_se_prepare` |
| `cabac_se_prepare_tu` | `cabac_se_prepare_tu.v` | `cabac_se_prepare` |
| `cabac_se_prepare_coeff` | `cabac_se_prepare_coeff.v` | `cabac_se_prepare` |
| `cabac_se_prepare_coeff_last_sig_xy` | `cabac_se_prepare_coeff_last_sig_xy.v` | `cabac_se_prepare` |
| `cabac_se_prepare_amplitude_of_coeff` | `cabac_se_prepare_amplitude_of_coeff.v` | `cabac_se_prepare` |
| `cabac_se_prepare_sig_coeff_ctx` | `cabac_se_prepare_sig_coeff_ctx.v` | `cabac_se_prepare` |
| `cabac_se_prepare_intra` | `cabac_se_prepare_intra.v` | `cabac_se_prepare` |
| `cabac_se_prepare_intra_luma` | `cabac_se_prepare_intra_luma.v` | `cabac_se_prepare` |
| `cabac_se_prepare_mvd` | `cabac_se_prepare_mvd.v` | `cabac_se_prepare` |
| `cabac_se_prepare_mv` | `cabac_se_prepare_mv.v` | `cabac_se_prepare` |
| `cabac_se_prepare_sao_offset` | `cabac_se_prepare_sao_offset.v` | `cabac_se_prepare` |
| `cabac_ucontext` | `cabac_ucontext.v` | `cabac_binsort` |
| `cabac_ucontext_t` | `cabac_ucontext_t.v` | `cabac_ucontext` |
| `cabac_ucontext_tt` | `cabac_ucontext_tt.v` | `cabac_ucontext` |
| `cabac_ulow` | `cabac_ulow.v` | `cabac_binsort` |
| `cabac_ulow_1bin` | `cabac_ulow_1bin.v` | `cabac_ulow` |
| `cabac_ulow_refine` | `cabac_ulow_refine.v` | `cabac_ulow` |
| `cabac_urange4` | `cabac_urange4.v` | `cabac_ulow` |
| `cabac_urange4_full` | `cabac_urange4_full.v` | `cabac_urange4` |
| `cabac_rlps4` | `cabac_rlps4.v` | `cabac_ulow` |
| `cabac_rlps4_1bin` | `cabac_rlps4_1bin.v` | `cabac_rlps4` |
| `coe_addr_trans` | `coe_addr_trans.v` | `cabac_top` |
| `pipo` | `pipo.v` | `cabac_binsort` |

### D.10 Memory (`mem/`)

34 memory wrapper files. Key memories (identified from instantiations in top-level buffers):

| Wrapper Module | Depth × Width | Used By | Type |
|----------------|---------------|---------|------|
| `prei_md_ram_sp_85x6` | 85 × 6 | `prei_top_buf` | MD storage (intra modes) |
| `posi_md_ram_sp_64x6` | 64 × 6 | `posi_top_buf` | MD storage (4-way rotation) |
| `ime_mv_ram_sp_64x13` | 64 × 13 | `ime_top_buf` | IME MV storage |
| `fme_mv_ram_dp_64x20` | 64 × 20 | `fme_top_buf` | FME MV storage (dual-port) |
| `fme_buf_wrapper` | — | `fme_top_buf` | Prediction pixel buffer |
| `fetch_ram_1p_128x32` | 128 × 32 | `fetch/` | Fetch buffer |
| `fetch_ram_2p_64x208` | 64 × 208 | `fetch/` | Fetch dual-port buffer |
| `fetch_rf_1p_128x512` | 128 × 512 | `fetch/` | Fetch register file |
| `fetch_rf_1p_64x256` | 64 × 256 | `fetch/` | Fetch register file |
| `ram_sp_be_*` | various | `rec/`, `db/` | Byte-enable RAMs |
| `ram_tp_be_32x64` | 32 × 64 | `rec/` | Two-port RAM |

---

## E. Preliminary Dependency Graph

```
                    ┌─────────────────────────────────────────┐
                    │            tb_enc_top (TB)              │
                    └─────────────────┬───────────────────────┘
                                      │ clk, rstn, ext_if, bs_if
                                      ▼
                    ┌─────────────────────────────────────────┐
                    │         h265enc_top (DESIGN TOP)        │
                    │  enc_top.v                              │
                    └──┬──────────┬────────────┬──────────────┘
                       │          │            │
          ┌────────────▼──┐  ┌────▼─────┐  ┌──▼──────────────┐
          │   enc_ctrl    │  │fetch_top │  │   enc_core      │
          │  (pipeline    │  │(data     │  │ (all processing │
          │   FSM)        │  │ fetch)   │  │  blocks)        │
          └───────────────┘  └──────────┘  └──┬──────────────┘
                                              │
         ┌────────┬─────────┬────────┬────────┼────────┬──────────┐
         ▼        ▼         ▼        ▼        ▼        ▼          ▼
    ┌─────────┐┌───────┐┌────────┐┌──────┐┌────────┐┌────────┐┌────────┐
    │prei_top ││posi   ││ime_top ││fme   ││rec_top ││dbsao   ││cabac   │
    │_buf     ││top_buf││_buf    ││top_  ││        ││_top    ││_top    │
    │         ││       ││        ││buf   ││        ││        ││        │
    └────┬────┘└───┬───┘└───┬────┘└──┬───┘└───┬────┘└───┬────┘└───┬────┘
         │         │        │        │        │         │         │
         ▼         ▼        ▼        ▼        ▼         ▼         ▼
    ┌─────────┐┌───────┐┌────────┐┌──────┐┌────────┐┌────────┐┌────────┐
    │prei_top ││posi   ││ime_top ││fme   ││intra_  ││dbsao_  ││cabac_  │
    │         ││top    ││        ││top   ││top     ││control ││binsort │
    └─────────┘└───────┘└────────┘└──────┘│mc_top  ││dbsao_  ││cabac_  │
                                          │tq_top  ││datapath││bitpack │
                                          │rec_buf ││sao_top ││...     │
                                          │_wrapper│└────────┘└────────┘
                                          └────────┘

    enc_data_pipeline (pipeline registers, inside enc_core)
```

**Data-flow direction (LCU pipeline):**

```
fetch → prei → ime → fme → rec → db → cabac → bs_out
         ↓      ↓      ↓      ↓      ↓
       (mode   (MV)   (MV,  (pixel  (filtered
        dec)          pred)  reco)   pixels)
```

---

## F. Recommended Analysis Order

Based on RTL dependency analysis, the following order minimizes forward references:

| Priority | Subsystem | Rationale |
|----------|-----------|-----------|
| 1 | `enc_defines.v` | All modules depend on shared macros |
| 2 | `top/` | Establishes top-level hierarchy and pipeline FSM |
| 3 | `mem/` | Memory primitives used by all subsystems |
| 4 | `fetch/` | Data supply; all subsystems read pixels through fetch |
| 5 | `prei/` | First stage in LCU pipeline; produces mode decisions |
| 6 | `posi/` | Second stage; SATD cost and partition selection |
| 7 | `ime/` | Integer MV search; depends on fetch for pixel data |
| 8 | `fme/` | Fractional refinement; depends on ime MV output |
| 9 | `rec/rec_intra` | Intra prediction + reconstruction |
| 10 | `rec/rec_mc` | Motion compensation; depends on fme MV |
| 11 | `rec/rec_tq` | Transform + quantization; core of reconstruction |
| 12 | `rec/rec_wrapper` | Buffer management; ties rec path together |
| 13 | `db/` | Deblocking + SAO; post-reconstruction filtering |
| 14 | `cabac/` | Entropy coding; last stage, produces bitstream |

---

## Evidence Traceability

All items in this document are based on direct RTL inspection:

- **Module names and file paths**: Verified by reading actual file contents.
- **Instantiation relationships**: Verified by reading module body of each parent module (e.g., `enc_core.v` lines 512-998).
- **Interface signals**: Verified by reading port declarations of each module.
- **Memory types and sizes**: Verified by reading `mem/` wrapper file names and instantiation in buffer wrappers.
- **Pipeline FSM states**: Verified by reading `enc_ctrl.v` state machine (lines 116-128, 281-300).
- **Configuration macros**: Verified by reading `enc_defines.v` (150 lines).
