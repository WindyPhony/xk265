# `h265enc_top` I/O Reference

This document records the external ports of `h265enc_top` and the RTL-supported routing and behavior needed to use them in a microarchitecture description. It describes the RTL interface, not a software API or a complete external-memory protocol.

## Evidence labels and sources

- **VERIFIED**: directly declared or connected in RTL.
- **INFERRED**: behavior derived from RTL logic but not declared as a separate interface contract.
- **UNKNOWN**: not determined by the inspected RTL.

Primary evidence locations:

- `top/enc_top.v:32,89-157` — top declaration, parameter, and public ports; `329-330` — CTU-grid calculation; `334`, `413`, and `549` — controller, fetch, and core instances; `534-544`, `582-611`, and `691-692` — representative external port connections.
- `enc_defines.v:49-50,84,92-101` — IME coordinate, LCU, frame, and pixel width macros.
- `fetch/fetch_top.v:23,278-288,397-407` — fetch module and external-interface ports/forwarding.
- `fetch/fetch_wrapper.v:205-215,309-336,364-447,531-545` — external-interface declarations, sequencer conditions, operation fields/start, and input-data capture.
- `top/enc_core.v:338-339,946-947` and `cabac/cabac_top.v:857-858` — bitstream port connection into CABAC output ports.

## Top-level context

| Property | Value | Status and evidence |
|---|---|---|
| Top module | `h265enc_top` | VERIFIED — `top/enc_top.v`, module declaration. |
| Default parameter | `CMD_NUM_WIDTH = 3` | VERIFIED — `top/enc_top.v`, parameter declaration. |
| Clock input | `clk` | VERIFIED — top port and connections to `enc_ctrl`, `fetch_top`, and `enc_core`. |
| Reset input | `rstn` | VERIFIED — top port and connections to those child modules. The signal name indicates active-low polarity; reset timing at the top interface is UNKNOWN without treating child implementation as a global interface guarantee. |
| First-level instances | `u_enc_ctrl: enc_ctrl`, `u_fetch_top: fetch_top`, `u_enc_core: enc_core` | VERIFIED — instantiations in `top/enc_top.v`. |

The macro values used to resolve widths are `PIC_X_WIDTH=6`, `PIC_Y_WIDTH=6`, `PIC_WIDTH=13`, `PIC_HEIGHT=12`, `PIXEL_WIDTH=8`, `IME_MV_WIDTH_X=7`, and `IME_MV_WIDTH_Y=6` in `enc_defines.v`.

## External port inventory

All widths below are resolved for the checked-in macro definitions and default top parameter. Directions are from the perspective of `h265enc_top`.

### Clock and system configuration

| Port | Direction | Width | RTL-supported description | Status |
|---|---:|---:|---|---|
| `clk` | Input | 1 | Clock connected to the top-level controller, fetch subsystem, and encoder core. | VERIFIED |
| `rstn` | Input | 1 | Reset signal connected to the same first-level modules. | VERIFIED |
| `sys_start_i` | Input | 1 | Start control connected to `enc_ctrl`; the controller generates internal `enc_start`. | VERIFIED |
| `sys_done_o` | Output | 1 | Completion output from `enc_ctrl`. | VERIFIED |
| `sys_type_i` | Input | 1 | Mode/type input used by `enc_ctrl` and `enc_core`. Exact external encoding is UNKNOWN: the RTL defines multiple type constants (`INTRA`/`INTER` and `SLICE_TYPE_I`/`SLICE_TYPE_P`) with differing values. | VERIFIED / UNKNOWN |
| `sys_all_x_i` | Input | 13 | Frame horizontal size input; comment in `enc_top.v` says pixel number in X. Used to calculate the last CTU X index and passed into the core/fetch logic. | VERIFIED |
| `sys_all_y_i` | Input | 12 | Frame vertical size input; comment says pixel number in Y. Used to calculate the last CTU Y index and passed into the core/fetch logic. | VERIFIED |
| `sys_init_qp_i` | Input | 6 | Initial QP input routed to `enc_core`. | VERIFIED |
| `sys_IinP_ena_i` | Input | 1 | Control input routed to `enc_core`; comment says enable I block in P frame. | VERIFIED |
| `sys_db_ena_i` | Input | 1 | Deblocking enable input routed to `enc_core`. | VERIFIED |
| `sys_sao_ena_i` | Input | 1 | SAO enable input routed to `enc_core`. | VERIFIED |
| `sys_posi4x4bit_i` | Input | 5 | POSI configuration input routed to `enc_core`; exact encoding is UNKNOWN. | VERIFIED |

`top/enc_top.v` derives internal CTU-grid maxima as `ceil(sys_all_x_i / 64) - 1` and `ceil(sys_all_y_i / 64) - 1`, using `LCU_SIZE=64`, before passing them to the controller/core. This calculation is VERIFIED; the legal frame-dimension range and behavior for zero dimensions are UNKNOWN.

### Skip-cost thresholds

| Port | Direction | Width | RTL-supported description | Status |
|---|---:|---:|---|---|
| `skip_cost_thresh_08` | Input | 32 | Threshold input associated by name with 8×8 skip cost; routed to `enc_core`. | VERIFIED for declaration/routing; meaning inferred from name |
| `skip_cost_thresh_16` | Input | 32 | Threshold input associated by name with 16×16 skip cost; routed to `enc_core`. | VERIFIED for declaration/routing; meaning inferred from name |
| `skip_cost_thresh_32` | Input | 32 | Threshold input associated by name with 32×32 skip cost; routed to `enc_core`. | VERIFIED for declaration/routing; meaning inferred from name |
| `skip_cost_thresh_64` | Input | 32 | Threshold input associated by name with 64×64 skip cost; routed to `enc_core`. | VERIFIED for declaration/routing; meaning inferred from name |

### Rate-control interface (`rc_cfg_if`)

| Port | Direction | Width | RTL-supported description | Status |
|---|---:|---:|---|---|
| `sys_rc_mod64_sum_o` | Output | 32 | Rate-control output routed from `enc_core`. | VERIFIED; semantic interpretation beyond signal name UNKNOWN |
| `sys_rc_bitnum_i` | Input | 32 | Rate-control input routed to `enc_core`. | VERIFIED; precise measurement/window UNKNOWN |
| `sys_rc_k` | Input | 16 | Rate-control parameter input routed to `enc_core`. | VERIFIED; numeric meaning UNKNOWN |
| `sys_rc_roi_height` | Input | 6 | ROI height configuration routed to `enc_core`. | VERIFIED |
| `sys_rc_roi_width` | Input | 7 | ROI width configuration routed to `enc_core`. | VERIFIED |
| `sys_rc_roi_x` | Input | 7 | ROI X configuration routed to `enc_core`. | VERIFIED |
| `sys_rc_roi_y` | Input | 7 | ROI Y configuration routed to `enc_core`. | VERIFIED |
| `sys_rc_roi_enable` | Input | 1 | ROI control input routed to `enc_core`. | VERIFIED |
| `sys_rc_L1_frame_byte` | Input | 10 | Frame-byte configuration routed to `enc_core`; exact level/interpretation UNKNOWN. | VERIFIED |
| `sys_rc_L2_frame_byte` | Input | 10 | Frame-byte configuration routed to `enc_core`; exact level/interpretation UNKNOWN. | VERIFIED |
| `sys_rc_lcu_en` | Input | 1 | Per-LCU-related control input routed to `enc_core`; exact behavior UNKNOWN from top-level connectivity alone. | VERIFIED |
| `sys_rc_max_qp` | Input | 6 | Maximum-QP configuration routed to `enc_core`. | VERIFIED |
| `sys_rc_min_qp` | Input | 6 | Minimum-QP configuration routed to `enc_core`. | VERIFIED |
| `sys_rc_delta_qp` | Input | 6 | Delta-QP configuration routed to `enc_core`. | VERIFIED |

### IME command interface (`ime_cfg_if`)

| Port | Direction | Width | RTL-supported description | Status |
|---|---:|---:|---|---|
| `sys_ime_cmd_num_i` | Input | 3 (default) | Command-count/configuration field routed to the IME wrapper in `enc_core`. Its exact count encoding is UNKNOWN. | VERIFIED for width/routing |
| `sys_ime_cmd_dat_i` | Input | 232 (default) | Packed command-data bus routed to the IME wrapper in `enc_core`. | VERIFIED for width/routing |

The width expression in `top/enc_top.v` calculates one slot as 29 bits: 7-bit center X, 6-bit center Y, 6-bit length X, 5-bit length Y, 2-bit slope, and three 1-bit fields (downsample, partition, feedback). With the default `CMD_NUM_WIDTH=3`, the aggregate bus is `29 × (1 << 3) = 232` bits. These component widths are VERIFIED by the expression and its comments. Slot ordering, bit offsets, signedness, and the meaning of the 3-bit command-number value are UNKNOWN from this expression alone.

### External memory interface (`ext_if`)

| Port | Direction | Width | RTL-supported description | Status |
|---|---:|---:|---|---|
| `extif_start_o` | Output | 1 | Fetch wrapper request/start indication. | VERIFIED |
| `extif_done_i` | Input | 1 | Completion indication used by fetch sequencing. | VERIFIED |
| `extif_mode_o` | Output | 5 | Operation/mode code generated by `fetch_wrapper`. | VERIFIED; mode-code contract should be taken from fetch RTL |
| `extif_x_o` | Output | 12 | X coordinate/address field (`PIC_X_WIDTH + 6`). | VERIFIED; units are operation-dependent in fetch RTL |
| `extif_y_o` | Output | 12 | Y coordinate/address field (`PIC_Y_WIDTH + 6`). | VERIFIED; units are operation-dependent in fetch RTL |
| `extif_width_o` | Output | 8 | Transfer width field. | VERIFIED |
| `extif_height_o` | Output | 8 | Transfer height field. | VERIFIED |
| `extif_wren_i` | Input | 1 | Input handshake/control used by fetch logic when capturing incoming memory data and advancing load handling. | VERIFIED; exact external timing contract UNKNOWN |
| `extif_rden_i` | Input | 1 | Input handshake/control used by fetch logic when advancing store-data handling. | VERIFIED; exact external timing contract UNKNOWN |
| `extif_data_i` | Input | 128 | 16 pixels × 8 bits of input data to the fetch subsystem. | VERIFIED |
| `extif_data_o` | Output | 128 | 16 pixels × 8 bits of output data from the fetch subsystem. | VERIFIED |

**Routing evidence:** `h265enc_top` connects these ports directly to `u_fetch_top`; `fetch_top` forwards them to `fetch_wrapper`. The wrapper generates mode, coordinates, dimensions, and start; it uses `extif_done_i` to sequence operations, captures `extif_data_i` under `extif_wren_i`, and advances store handling under `extif_rden_i`. The supported mode cases and operation-dependent coordinate/dimension values are implemented in `fetch/fetch_wrapper.v`. No separate bus-standard name or complete cycle-level external protocol is established by the top-level RTL.

### Encoded byte-stream interface (`bs_if`)

| Port | Direction | Width | RTL-supported description | Status |
|---|---:|---:|---|---|
| `bs_val_o` | Output | 1 | Valid/ready-style signal exported from `enc_core` and connected to the CABAC output interface. | VERIFIED for routing; external interpretation UNKNOWN |
| `bs_dat_o` | Output | 8 | Byte data from `enc_core`/CABAC. | VERIFIED |

`enc_core` connects these signals to `cabac_top` as `bs_val_o` and `bs_data_o`. In `cabac/cabac_top.v`, `bs_val_o` is connected to an internal `out_ready` port and `bs_data_o` to `output_byte`; this is VERIFIED internal wiring. Whether an external consumer should interpret `bs_val_o` as ready, valid, or another contract is UNKNOWN from the top-level boundary alone.

## Signal-flow summary

```text
sys_start_i, sys_type_i, frame dimensions
                    │
                    ▼
               enc_ctrl ───────────────► sys_done_o
                    │ internal stage start/done and CTU coordinates
                    ▼
                enc_core ◄───────────── configuration, RC inputs, IME commands
                    │
                    ├──────────────────► bs_dat_o, bs_val_o
                    │
                    └── pixel requests/data ◄──► fetch_top / fetch_wrapper
                                                  │
                           extif_start/mode/x/y/width/height/data_o ──► external memory side
                           extif_done/rden/wren/data_i                ◄── external memory side
```

**Status:** module boundaries and the named top-level connections are VERIFIED in `top/enc_top.v`. The diagram is a signal-flow summary, not a cycle-accurate protocol diagram.

## Evidence and open items

| Item | Status | Evidence or limitation |
|---|---|---|
| Top-level port names, directions, declared widths | VERIFIED | Declarations in `top/enc_top.v`. |
| Macro-resolved frame, coordinate, and pixel-bus widths | VERIFIED | `enc_defines.v` plus port width expressions in `top/enc_top.v`. |
| Default IME command bus width | VERIFIED | `CMD_NUM_WIDTH` and localparam expression in `top/enc_top.v`. |
| First-level routing of public ports | VERIFIED | Named connections in the three top-level instances in `top/enc_top.v`. |
| External fetch operation sequencing and data capture/store controls | VERIFIED | `fetch/fetch_wrapper.v` state/control logic. |
| System-type encoding | UNKNOWN | RTL defines multiple type constants with different conventions; integration-level encoding must be checked against the intended configuration. |
| IME command packing order, signedness, and exact count encoding | UNKNOWN | Width expression does not define field offsets or encoding. |
| External memory bus-standard and full timing contract | UNKNOWN | No standard-specific channel/address/response protocol is declared at this top. |
| `bs_val_o` external consumer semantics | UNKNOWN | Internal CABAC wiring exists, but top-level consumer contract is not declared here. |
| Top-interface reset assertion/deassertion timing | UNKNOWN | `rstn` is passed through; child reset behavior does not alone define integration timing requirements. |
| Legal frame sizes and zero-dimension handling | UNKNOWN | Top computes CTU maxima but does not declare an external range contract. |

## Review checklist

- [x] Port inventory cross-checked against `h265enc_top` declarations.
- [x] Widths resolved from active macros and default parameter.
- [x] Main child-module routes checked against named instantiations.
- [x] External-memory path checked through `fetch_top` to `fetch_wrapper`.
- [x] Byte-stream path checked from `enc_core` to `cabac_top`.
- [x] Unverified encodings and protocol details marked UNKNOWN.
