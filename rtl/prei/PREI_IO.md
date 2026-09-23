# I/O và kết nối các khối PREI

Tài liệu này được lập trực tiếp từ RTL trong thư mục `rtl/prei`. Không giả định kiến trúc HEVC chung.

## Quy ước bằng chứng

- **VERIFIED**: cổng, độ rộng và kết nối được xác nhận trực tiếp từ khai báo/instance/assignment RTL.
- **INFERRED**: ý nghĩa chức năng được suy ra từ biểu thức và hành vi RTL.
- **UNKNOWN**: RTL trong phạm vi khảo sát không cho biết chắc chắn.
- `Khối ngoài (UNKNOWN)` nghĩa là cổng top-level có thật nhưng module producer/consumer nằm ngoài hierarchy `prei_top` chưa được truy trong tài liệu này.
- Mọi bus `[N:0]` có độ rộng `N+1` bit. Với tham số mặc định `MODE=21`, `DIGIT=0`: `mode2..mode33` = 22 bit, `modebest` = 22 bit, `modebest16` = 24 bit, `modebest32` = 26 bit, `modebest64` = 28 bit.
- ``PIC_X_WIDTH`` và ``PIC_Y_WIDTH`` đều được resolve thành 6 từ `rtl/enc_defines.v:92-93`.

## Hierarchy và luồng chính

```text
prei_top
├── hevc_md_top
│   ├── fetch8x8
│   ├── md_top
│   │   ├── control
│   │   ├── md_fetch
│   │   ├── gxgy
│   │   ├── counter
│   │   ├── compare
│   │   └── dc_planar
│   └── mode_write
└── rate_control

md_ram                 (có source RTL nhưng không được instantiate trong hierarchy trên)
└── rf_2p              (module thư viện ngoài thư mục prei)
```

Luồng dữ liệu chính đã xác minh:

```text
md_data_i[255:0] -> fetch8x8 -> rf_512bit[511:0] -> md_fetch
md_fetch -> x1/x2/x3[23:0] -> gxgy -> gx/gy[10:0]
gxgy -> counter -> mode2..mode33[21:0] -> compare
compare -> bestmode*/modebest* -> dc_planar -> bestmode*_o[5:0]
bestmode*_o -> mode_write -> md_we/md_waddr/md_wdata
compare.modebest64[27:0] -> rate_control -> rc_qp_o/prei_done/mod64_sum_o
```

Status của các cạnh trên: **VERIFIED** về kết nối; tên gọi chức năng như “gradient”, “cost” là **INFERRED** từ phép toán RTL, trừ khi mô tả file nói rõ.

## 1. `prei_top`

Top-level PREI; instantiate `hevc_md_top` và `rate_control` (`prei_top.v:102-149`).

| Cổng | Hướng | Rộng | Nguồn -> Đích | Mô tả | Status |
|---|---:|---:|---|---|---|
| `clk` | in | 1 | Khối ngoài -> `prei_top` -> hai submodule và toàn bộ logic tuần tự | Clock, tác động cạnh lên | VERIFIED; endpoint ngoài UNKNOWN |
| `rstn` | in | 1 | Khối ngoài -> toàn hierarchy | Reset active-low, bất đồng bộ trong các `always @(posedge clk or negedge rstn)` | VERIFIED; endpoint ngoài UNKNOWN |
| `prei_start` | in | 1 | Khối ngoài -> `hevc_md_top.enable` | Bắt đầu nhánh mode decision | VERIFIED |
| `prei_done` | out | 1 | `rate_control.rc_done_o` -> khối ngoài | Xung hoàn tất rate control khi bộ đếm RC đạt 10 | VERIFIED |
| `md_data_i` | in | 256 | Khối ngoài -> `hevc_md_top` -> `fetch8x8.md_data_i` | Mỗi lần nhận 256 bit; ghép hai lần thành block 512 bit | VERIFIED; dữ liệu pixel 8x8 là INFERRED từ slicing/comment |
| `md_ren_o` | out | 1 | `fetch8x8.md_ren_o` -> khối ngoài | Read-enable của giao tiếp lấy dữ liệu gốc | VERIFIED |
| `md_sel_o` | out | 1 | `fetch8x8.md_sel_o` -> khối ngoài | Luôn bằng `0` | VERIFIED; ý nghĩa select ngoài UNKNOWN |
| `md_size_o` | out | 2 | `fetch8x8.md_size_o` -> khối ngoài | Luôn bằng `2'b01` | VERIFIED; encoding size ngoài UNKNOWN |
| `md_4x4_x_o` | out | 4 | `fetch8x8` -> khối ngoài | Tọa độ X tạo từ `{blockcnt[4],blockcnt[2],blockcnt[0],1'b0}` | VERIFIED |
| `md_4x4_y_o` | out | 4 | `fetch8x8` -> khối ngoài | Tọa độ Y tạo từ `{blockcnt[5],blockcnt[3],blockcnt[1],1'b0}` | VERIFIED |
| `md_idx_o` | out | 5 | `fetch8x8` -> khối ngoài | `{2'b00,flag,2'b00}`, với `flag <= cnt[3]` | VERIFIED; ý nghĩa index ngoài UNKNOWN |
| `md_we` | out | 1 | `mode_write.md_we` -> khối ngoài | Write-enable cho nơi lưu mode | VERIFIED; consumer ngoài UNKNOWN |
| `md_waddr` | out | 7 | `mode_write.md_waddr` -> khối ngoài | Địa chỉ write mode 8/16/32/64 | VERIFIED |
| `md_wdata` | out | 6 | `mode_write.md_wdata` -> khối ngoài | Mode được chọn để ghi | VERIFIED |
| `actual_bitnum_i` | in | 16 | Khối ngoài -> `rate_control.actual_bitnum_i` | Cộng vào `frame_bit`; comment nói từ CABAC nhưng connection ngoài chưa xác minh | VERIFIED port/use; nguồn CABAC UNKNOWN |
| `rc_ctu_x_i` | in | 6 | Khối ngoài -> `rate_control.rc_ctu_x_i` | Tọa độ CTU X dùng kiểm tra ROI | VERIFIED |
| `rc_ctu_y_i` | in | 6 | Khối ngoài -> `rate_control.rc_ctu_y_i` | Tọa độ CTU Y; hàng 0 ép dùng QP khởi tạo | VERIFIED |
| `reg_k` | in | 16 | Khối ngoài -> `rate_control.reg_k` | Hệ số nhân với lịch sử `modebest_7` | VERIFIED |
| `reg_bitnum_i` | in | 32 | Khối ngoài -> `rate_control.reg_bitnum_i` | Khai báo và nối cổng nhưng không được đọc trong thân `rate_control` | VERIFIED unused |
| `reg_ROI_height` | in | 6 | Khối ngoài -> `rate_control` | Chiều cao vùng ROI trong phép kiểm tra tọa độ | VERIFIED |
| `reg_ROI_width` | in | 7 | Khối ngoài -> `rate_control` | Chiều rộng vùng ROI trong phép kiểm tra tọa độ | VERIFIED |
| `reg_ROI_x` | in | 7 | Khối ngoài -> `rate_control` | Gốc X vùng ROI | VERIFIED |
| `reg_ROI_y` | in | 7 | Khối ngoài -> `rate_control` | Gốc Y vùng ROI | VERIFIED |
| `reg_ROI_enable` | in | 1 | Khối ngoài -> `rate_control` | Cho phép hiệu chỉnh QP khi `ROI_hit` | VERIFIED |
| `reg_L1_frame_byte` | in | 10 | Khối ngoài -> `rate_control` | Ngưỡng sai lệch mức 1 | VERIFIED; đơn vị “byte” theo tên, UNKNOWN về đặc tả |
| `reg_L2_frame_byte` | in | 10 | Khối ngoài -> `rate_control` | Ngưỡng sai lệch mức 2 | VERIFIED; đơn vị “byte” theo tên, UNKNOWN về đặc tả |
| `reg_lcu_rc_en` | in | 1 | Khối ngoài -> `rate_control` | Enable điều chỉnh QP mức LCU/CTU | VERIFIED |
| `reg_initial_qp` | in | 6 | Khối ngoài -> `rate_control` | QP cơ sở | VERIFIED |
| `reg_max_qp` | in | 6 | Khối ngoài -> `rate_control` | Giới hạn trên của QP output | VERIFIED |
| `reg_min_qp` | in | 6 | Khối ngoài -> `rate_control` | Giới hạn dưới của QP output | VERIFIED |
| `reg_delta_qp` | in | 6 | Khối ngoài -> `rate_control` | Giá trị trừ khỏi QP khi CTU thuộc ROI | VERIFIED |
| `rc_qp_o` | out | 6 | `rate_control.rc_qp_o` -> khối ngoài | QP sau RC, ROI và clamp min/max | VERIFIED |
| `mod64_sum_o` | out | 32 | `rate_control.modebest_1[31:0]` -> khối ngoài | 32 bit thấp của bộ tích lũy metric 64x64 | VERIFIED |

Kết nối nội bộ riêng của top:

| Signal | Rộng | Nguồn -> Đích | Mô tả | Status |
|---|---:|---|---|---|
| `md_done_w` | 1 | `hevc_md_top.finish` -> `rate_control.rc_start_i` | Hoàn tất mode decision đồng thời khởi động RC | VERIFIED |
| `modebest64_w` | 28 | `hevc_md_top.modebest64` -> `rate_control.modebest64_i` | Metric/cost góc tốt nhất cấp 64x64; **không phải mã mode** | VERIFIED connectivity; “cost” INFERRED từ compare-min logic |

## 2. `hevc_md_top`

Wrapper mode-decision (`hevc_md_top.v:36-132`). Các cổng `clk`, `rstn`, `md_*`, `enable`, `finish` nối thẳng/lần lượt với `prei_top` như bảng trên.

| Cổng | Hướng | Rộng | Nguồn -> Đích | Mô tả | Status |
|---|---:|---:|---|---|---|
| `clk`, `rstn` | in | 1 mỗi | `prei_top` -> `md_top`, `fetch8x8`, `mode_write`, logic enable | Clock/reset chung | VERIFIED |
| `enable` | in | 1 | `prei_top.prei_start` -> logic `enable_r/enable_reg`; `enable_r||enable` -> `fetch8x8`; `enable_reg` -> `md_top` | Start được giữ và tạo enable trễ cho xử lý mode | VERIFIED |
| `md_data_i` | in | 256 | `prei_top` -> `fetch8x8` | Dữ liệu đầu vào cần fetch | VERIFIED |
| `md_ren_o`, `md_sel_o` | out | 1 mỗi | `fetch8x8` -> `prei_top` | Điều khiển đọc/select | VERIFIED |
| `md_size_o` | out | 2 | `fetch8x8` -> `prei_top` | Mã size cố định `01` | VERIFIED |
| `md_4x4_x_o`, `md_4x4_y_o` | out | 4 mỗi | `fetch8x8` -> `prei_top` | Tọa độ fetch | VERIFIED |
| `md_idx_o` | out | 5 | `fetch8x8` -> `prei_top` | Index fetch | VERIFIED |
| `md_we` | out | 1 | `mode_write` -> `prei_top` | Write-enable mode | VERIFIED |
| `md_waddr` | out | 7 | `mode_write` -> `prei_top` | Địa chỉ mode | VERIFIED |
| `md_wdata` | out | 6 | `mode_write` -> `prei_top` | Mode write-back | VERIFIED |
| `modebest64` | out | 28 | `md_top.compare.modebest64` -> `prei_top.modebest64_w` | Metric 64x64 đưa sang RC | VERIFIED |
| `finish` | out | 1 | `md_top.control.finish` -> `fetch8x8`, `mode_write`, logic enable và `prei_top` | Kết thúc mode decision | VERIFIED |

Tín hiệu nội bộ quan trọng:

| Signal | Rộng | Nguồn -> Đích | Mô tả | Status |
|---|---:|---|---|---|
| `rf_512bit` | 512 | `fetch8x8` -> `md_top.md_fetch` | Block dữ liệu ghép từ hai từ 256 bit | VERIFIED |
| `cnt`, `blockcnt` | 6, 7 | `fetch8x8` -> `mode_write` và logic enable của wrapper | Counter fetch/write-back | VERIFIED |
| `bestmode`, `bestmode16`, `bestmode32`, `bestmode64` | 6 mỗi | `md_top.dc_planar` -> `mode_write` | Mode cuối cho bốn kích thước | VERIFIED |

Lưu ý: `cnt/blockcnt` ở bảng này do `fetch8x8` tạo. `md_top` có một cặp `cnt/blockcnt` khác do `control` tạo; hai cặp không nối trực tiếp với nhau. Trạng thái: **VERIFIED**.

## 3. `fetch8x8`

Khai báo tại `fetch8x8.v:28-55`, hành vi tại `fetch8x8.v:59-103`.

| Cổng | Hướng | Rộng | Nguồn -> Đích | Mô tả | Status |
|---|---:|---:|---|---|---|
| `clk`, `rstn` | in | 1 mỗi | `hevc_md_top` -> `fetch8x8` | Clock và reset active-low bất đồng bộ | VERIFIED |
| `enable` | in | 1 | `hevc_md_top.(enable_r || enable)` -> `fetch8x8` | Cho counter/fetch chạy | VERIFIED |
| `finish` | in | 1 | `md_top.finish` -> `fetch8x8` | Xóa `cnt` và `blockcnt` | VERIFIED |
| `md_data_i` | in | 256 | Giao tiếp top -> `fetch8x8.rdata` | Dữ liệu đọc về | VERIFIED |
| `md_ren_o` | out | 1 | `fetch8x8` -> giao tiếp top | Assert từ `cnt=0`, deassert tại `cnt=17` | VERIFIED |
| `md_sel_o` | out | 1 | `fetch8x8` -> giao tiếp top | Hằng `0` | VERIFIED |
| `md_size_o` | out | 2 | `fetch8x8` -> giao tiếp top | Hằng `01` | VERIFIED |
| `md_4x4_x_o`, `md_4x4_y_o` | out | 4 mỗi | `fetch8x8` -> giao tiếp top | Địa chỉ/tọa độ tạo từ bit `blockcnt` | VERIFIED |
| `md_idx_o` | out | 5 | `fetch8x8` -> giao tiếp top | Index phụ thuộc `flag=cnt[3]` trễ một clock | VERIFIED |
| `rf_512bit` | out | 512 | `fetch8x8` -> `md_top.md_fetch` | Thanh ghi block: nạp nửa trên tại `cnt=2`, nửa dưới tại `cnt=10` | VERIFIED |
| `cnt` | out | 6 | `fetch8x8` -> `mode_write`, wrapper enable | Counter chu kỳ 0..40 | VERIFIED |
| `blockcnt` | out | 7 | `fetch8x8` -> `mode_write`, tạo tọa độ | Tăng tại `cnt=32`, xóa bởi `finish` | VERIFIED |

## 4. `md_top`

Top nội bộ của datapath mode (`md_top.v:7-256`).

| Cổng | Hướng | Rộng | Nguồn -> Đích | Mô tả | Status |
|---|---:|---:|---|---|---|
| `clk`, `rstn` | in | 1 mỗi | `hevc_md_top` -> toàn bộ submodule `md_top` | Clock/reset | VERIFIED |
| `enable` | in | 1 | `hevc_md_top.enable_reg` -> `control` và `md_fetch` | Cho controller chạy; cổng `md_fetch.enable` thực tế không được đọc | VERIFIED |
| `rf_512bit` | in | 512 | `fetch8x8` -> `md_fetch` | Block đầu vào 8x8 | VERIFIED |
| `finish` | out | 1 | `control.finish` -> `hevc_md_top` | Báo hoàn tất | VERIFIED |
| `bestmode_o` | out | 6 | `dc_planar.bestmode_o` -> `hevc_md_top.mode_write` | Mode cuối cấp 8x8 | VERIFIED |
| `bestmode16_o` | out | 6 | `dc_planar.bestmode16_o` -> `hevc_md_top.mode_write` | Mode cuối cấp 16x16 | VERIFIED |
| `bestmode32_o` | out | 6 | `dc_planar.bestmode32_o` -> `hevc_md_top.mode_write` | Mode cuối cấp 32x32 | VERIFIED |
| `bestmode64_o` | out | 6 | `dc_planar.bestmode64_o` -> `hevc_md_top.mode_write` | Mode cuối cấp 64x64 | VERIFIED |
| `modebest64` | out | 28 | `compare.modebest64` -> `hevc_md_top` -> `rate_control` | Giá trị metric nhỏ nhất ở cấp 64x64 | VERIFIED connectivity; chức năng INFERRED |

## 5. `control`

Controller dùng counter, không có FSM state encoding riêng (`control.v:24-97`).

| Cổng | Hướng | Rộng | Nguồn -> Đích | Mô tả | Status |
|---|---:|---:|---|---|---|
| `clk`, `rstn` | in | 1 mỗi | `md_top` -> `control` | Clock/reset | VERIFIED |
| `enable` | in | 1 | `md_top.enable` -> `control` | Cho `cyclecnt/blockcnt` chạy, hạ `finish` | VERIFIED |
| `cyclecnt` | out | 6 | `control` -> `md_fetch.cnt`, `compare.cnt`, `dc_planar.cnt` | Counter chu kỳ 0..40 | VERIFIED |
| `blockcnt` | out | 7 | `control` -> `compare.blockcnt`, `dc_planar.blockcnt` | Counter block; tăng khi `cyclecnt=40` | VERIFIED |
| `gxgyrun` | out | 1 | `control` -> `gxgy.gxgyrun` | Enable tính `gx/gy`, set tại cycle 5 | VERIFIED |
| `counterrun1` | out | 1 | `control` -> `counter`, `dc_planar` | `gxgyrun` trễ 1 clock | VERIFIED |
| `counterrun2` | out | 1 | `control` -> `counter`, `dc_planar` | `counterrun1` trễ 1 clock | VERIFIED |
| `finish` | out | 1 | `control` -> `md_top` và các consumer cấp trên | Assert tại `blockcnt=65 && cyclecnt=15` | VERIFIED |
| `newblock` | out | 1 | `control` -> wire `md_top.newblock` -> không có consumer | Xung khi `cyclecnt=40` | VERIFIED; unused |

## 6. `md_fetch`

Chọn ba cửa sổ 24-bit từ block 512-bit theo `cnt` (`md_fetch.v:18-77`).

| Cổng | Hướng | Rộng | Nguồn -> Đích | Mô tả | Status |
|---|---:|---:|---|---|---|
| `clk`, `rstn` | in | 1 mỗi | `md_top` -> `md_fetch` | Clock/reset | VERIFIED |
| `enable` | in | 1 | `md_top.enable` -> `md_fetch` | Cổng được khai báo/nối nhưng không được dùng trong thân module | VERIFIED unused |
| `cnt` | in | 6 | `control.cyclecnt` -> `md_fetch` | Chọn slice tại các cycle 5..40 | VERIFIED |
| `rf_512bit` | in | 512 | `fetch8x8` -> `md_fetch` | Dữ liệu block nguồn | VERIFIED |
| `x1`, `x2`, `x3` | out | 24 mỗi | `md_fetch` -> `gxgy` | Mỗi bus ghép 3 mẫu 8-bit dùng trong phép gradient 3x3 | VERIFIED width/route; diễn giải 3 mẫu là INFERRED |

## 7. `gxgy`

Tính hai đáp ứng gradient có dấu (`gxgy.v:18-39`).

| Cổng | Hướng | Rộng | Nguồn -> Đích | Mô tả | Status |
|---|---:|---:|---|---|---|
| `clk`, `rstn` | in | 1 mỗi | `md_top` -> `gxgy` | Clock/reset | VERIFIED |
| `gxgyrun` | in | 1 | `control.gxgyrun` -> `gxgy` | Enable cập nhật `gx/gy` | VERIFIED |
| `x1`, `x2`, `x3` | in | 24 mỗi | `md_fetch` -> `gxgy` | Ba hàng/cửa sổ, mỗi hàng gồm 3 trường 8-bit | VERIFIED slicing; “hàng” INFERRED |
| `gx` | out signed | 11 | `gxgy` -> `counter`, `dc_planar` | Đáp ứng ngang theo tổng trọng số `1,2,1` và hiệu hai biên | VERIFIED expression; tên chức năng INFERRED |
| `gy` | out signed | 11 | `gxgy` -> `counter`, `dc_planar` | Đáp ứng dọc theo tổng trọng số `1,2,1` và hiệu hai biên | VERIFIED expression; tên chức năng INFERRED |

## 8. `counter`

Tạo và tích lũy metric cho các mode góc 2..33 (`counter.v:46-330`).

| Cổng | Hướng | Rộng | Nguồn -> Đích | Mô tả | Status |
|---|---:|---:|---|---|---|
| `clk`, `rstn` | in | 1 mỗi | `md_top` -> `counter` | Clock/reset | VERIFIED |
| `counterrun1` | in | 1 | `control.counterrun1` -> `counter` | Tính các tổ hợp tuyến tính tạm từ `gx/gy`; đồng thời bắt đầu lượt tích lũy mới | VERIFIED |
| `counterrun2` | in | 1 | `control.counterrun2` -> `counter` | Cộng trị tuyệt đối của metric tạm vào accumulator từng mode | VERIFIED |
| `gx`, `gy` | in signed | 11 mỗi | `gxgy` -> `counter` | Thành phần gradient dùng cho 32 hướng | VERIFIED |
| `mode2` ... `mode33` | out | 22 mỗi | `counter` -> `compare` | 32 accumulator metric ứng viên cho mode 2..33 | VERIFIED width/route; “metric” INFERRED |

## 9. `compare`

So sánh các metric mode góc và gộp cost theo cấp 8/16/32/64 (`compare.v:59-102`, logic đến dòng 1175).

| Cổng | Hướng | Rộng | Nguồn -> Đích | Mô tả | Status |
|---|---:|---:|---|---|---|
| `clk`, `rstn` | in | 1 mỗi | `md_top` -> `compare` | Clock/reset | VERIFIED |
| `cnt` | in | 6 | `control.cyclecnt` -> `compare` | Lập lịch load/compare cho mode 2..33 | VERIFIED |
| `blockcnt` | in | 7 | `control.blockcnt` -> `compare` | Chọn thời điểm gộp cấp 16/32/64 | VERIFIED |
| `mode2` ... `mode33` | in | 22 mỗi | `counter` -> `compare` | Metric của 32 mode góc | VERIFIED |
| `bestmode` | out | 6 | `compare` -> `dc_planar.bestmode` | Chỉ số mode góc tốt nhất cấp 8x8 | VERIFIED |
| `bestmode16` | out | 6 | `compare` -> `dc_planar.bestmode16` | Chỉ số mode góc tốt nhất cấp 16x16 | VERIFIED |
| `bestmode32` | out | 6 | `compare` -> `dc_planar.bestmode32` | Chỉ số mode góc tốt nhất cấp 32x32 | VERIFIED |
| `bestmode64` | out | 6 | `compare` -> `dc_planar.bestmode64` | Chỉ số mode góc tốt nhất cấp 64x64 | VERIFIED |
| `modebest` | out | 22 | `compare` -> `dc_planar.modebest` | Metric nhỏ nhất cấp 8x8 | VERIFIED width/route; “nhỏ nhất” VERIFIED từ phép so sánh |
| `modebest16` | out | 24 | `compare` -> `dc_planar.modebest16` | Metric gộp nhỏ nhất cấp 16x16 | VERIFIED |
| `modebest32` | out | 26 | `compare` -> `dc_planar.modebest32` | Metric gộp nhỏ nhất cấp 32x32 | VERIFIED |
| `modebest64` | out | 28 | `compare` -> `dc_planar.modebest64`, đồng thời ra `md_top`/RC | Metric gộp nhỏ nhất cấp 64x64 | VERIFIED |

## 10. `dc_planar`

So sánh ứng viên DC/planar với mode góc và phát mode cuối (`DC_Plannar.v:41-72`, `DC_Plannar.v:75-159`). Tên file có chữ `Plannar`, tên module thực tế là `dc_planar`.

| Cổng | Hướng | Rộng | Nguồn -> Đích | Mô tả | Status |
|---|---:|---:|---|---|---|
| `clk`, `rstn` | in | 1 mỗi | `md_top` -> `dc_planar` | Clock/reset | VERIFIED |
| `counterrun1`, `counterrun2` | in | 1 mỗi | `control` -> `dc_planar` | Điều khiển tính/tích lũy `abs(gx)+abs(gy)` | VERIFIED |
| `gx`, `gy` | in signed | 11 mỗi | `gxgy` -> `dc_planar` | Gradient để tạo `modedata` | VERIFIED |
| `cnt` | in | 6 | `control.cyclecnt` -> `dc_planar` | Lập lịch chốt/gộp/quyết định mode | VERIFIED |
| `blockcnt` | in | 7 | `control.blockcnt` -> `dc_planar` | Chọn biên nhóm 8/16/32/64 | VERIFIED |
| `bestmode`, `bestmode16`, `bestmode32`, `bestmode64` | in | 6 mỗi | `compare` -> `dc_planar` | Mode góc tốt nhất từng cấp | VERIFIED |
| `modebest` | in | 22 | `compare` -> `dc_planar` | Cost góc cấp 8 | VERIFIED |
| `modebest16` | in | 24 | `compare` -> `dc_planar` | Cost góc cấp 16 | VERIFIED |
| `modebest32` | in | 26 | `compare` -> `dc_planar` | Cost góc cấp 32 | VERIFIED |
| `modebest64` | in | 28 | `compare` -> `dc_planar` | Cost góc cấp 64 | VERIFIED |
| `bestmode_o` | out | 6 | `dc_planar` -> `md_top` -> `mode_write` | Mode cuối 8x8: `1` nếu dưới ngưỡng DC, `0` nếu planar thắng, ngược lại mode góc | VERIFIED |
| `bestmode16_o` | out | 6 | `dc_planar` -> `md_top` -> `mode_write` | Mode cuối 16x16 | VERIFIED |
| `bestmode32_o` | out | 6 | `dc_planar` -> `md_top` -> `mode_write` | Mode cuối 32x32 | VERIFIED |
| `bestmode64_o` | out | 6 | `dc_planar` -> `md_top` -> `mode_write` | Mode cuối 64x64 | VERIFIED |

## 11. `mode_write`

Đóng gói mode được chọn thành giao tiếp write-back (`mode_write.v:24-75`).

| Cổng | Hướng | Rộng | Nguồn -> Đích | Mô tả | Status |
|---|---:|---:|---|---|---|
| `clk`, `rstn` | in | 1 mỗi | `hevc_md_top` -> `mode_write` | Clock/reset | VERIFIED |
| `cnt` | in | 6 | `fetch8x8.cnt` -> `mode_write` | Chọn thời điểm và loại mode write-back | VERIFIED |
| `blockcnt` | in | 7 | `fetch8x8.blockcnt` -> `mode_write` | Tạo địa chỉ và điều kiện write | VERIFIED |
| `bestmode` | in | 6 | `md_top.bestmode_o` -> `mode_write` | Mode cuối 8x8 | VERIFIED |
| `bestmode16` | in | 6 | `md_top.bestmode16_o` -> `mode_write` | Mode cuối 16x16 | VERIFIED |
| `bestmode32` | in | 6 | `md_top.bestmode32_o` -> `mode_write` | Mode cuối 32x32 | VERIFIED |
| `bestmode64` | in | 6 | `md_top.bestmode64_o` -> `mode_write` | Mode cuối 64x64 | VERIFIED |
| `finish` | in | 1 | `md_top.finish` -> `mode_write` | Cổng được nối nhưng không được đọc trong thân module | VERIFIED unused |
| `md_we` | out | 1 | `mode_write` -> `hevc_md_top/prei_top` -> khối ngoài | Xung write cho mode 8/16/32/64 theo `cnt/blockcnt` | VERIFIED |
| `md_waddr` | out | 7 | `mode_write` -> khối ngoài | Địa chỉ: công thức khác nhau tại `cnt=11..14` | VERIFIED |
| `md_wdata` | out | 6 | `mode_write` -> khối ngoài | Chọn một trong bốn `bestmode*` | VERIFIED |

## 12. `rate_control`

Rate control mức LCU/CTU (`rate_control.v:54-242`).

| Cổng | Hướng | Rộng | Nguồn -> Đích | Mô tả | Status |
|---|---:|---:|---|---|---|
| `clk`, `rstn` | in | 1 mỗi | `prei_top` -> `rate_control` | Clock/reset | VERIFIED |
| `rc_start_i` | in | 1 | `hevc_md_top.finish` -> `rate_control` | Khởi động chuỗi RC 10 cycle | VERIFIED |
| `rc_done_o` | out | 1 | `rate_control` -> `prei_top.prei_done` | Xung done khi `cnt=10` | VERIFIED |
| `rc_ctu_x_i`, `rc_ctu_y_i` | in | 6 mỗi | `prei_top` -> `rate_control` | Tọa độ CTU cho ROI và xử lý hàng đầu | VERIFIED |
| `actual_bitnum_i` | in | 16 | `prei_top` -> accumulator `frame_bit` | Số bit thực tế được cộng tại RC cycle 1 | VERIFIED |
| `modebest64_i` | in | 28 | `compare.modebest64` -> `rate_control` | Metric mode 64x64 được tích lũy/delay qua `modebest_1..7` | VERIFIED |
| `reg_k` | in | 16 | `prei_top` -> multiplier | Hệ số trong `modebest_7 * reg_k` | VERIFIED |
| `reg_bitnum_i` | in | 32 | `prei_top` -> không consumer nội bộ | Không được dùng | VERIFIED unused |
| `reg_ROI_height` | in | 6 | `prei_top` -> logic ROI | Biên chiều cao ROI | VERIFIED |
| `reg_ROI_width`, `reg_ROI_x`, `reg_ROI_y` | in | 7 mỗi | `prei_top` -> logic ROI | Kích thước X và tọa độ ROI | VERIFIED |
| `reg_ROI_enable` | in | 1 | `prei_top` -> logic QP ROI | Enable trừ `reg_delta_qp` | VERIFIED |
| `reg_L1_frame_byte`, `reg_L2_frame_byte` | in | 10 mỗi | `prei_top` -> so sánh `diff_abs` | Hai ngưỡng chọn `diff_level=0/1/2` | VERIFIED |
| `reg_lcu_rc_en` | in | 1 | `prei_top` -> logic `qp_tmp` | Nếu 0 thì dùng `reg_initial_qp` | VERIFIED |
| `reg_initial_qp`, `reg_max_qp`, `reg_min_qp`, `reg_delta_qp` | in | 6 mỗi | `prei_top` -> datapath QP | QP cơ sở, clamp và delta ROI | VERIFIED |
| `rc_qp_o` | out | 6 | `rate_control` -> `prei_top` -> khối ngoài | QP kết quả tại RC cycle 9 | VERIFIED |
| `mod64_sum_o` | out | 32 | `modebest_1[31:0]` -> `prei_top` -> khối ngoài | 32 bit thấp của accumulator 39-bit; RTL có comment cảnh báo overflow | VERIFIED |

## 13. `md_ram` (không nằm trong hierarchy hoạt động ở trên)

`md_ram.v` không được instantiate bởi bất kỳ module nào trong thư mục `prei`; do đó endpoint thật của wrapper này là **UNKNOWN**. Bên trong nó instantiate `rf_2p #(.Addr_Width(4), .Word_Width(32))`.

| Cổng | Hướng | Rộng | Nguồn -> Đích | Mô tả | Status |
|---|---:|---:|---|---|---|
| `clk` | in | 1 | Khối ngoài UNKNOWN -> `md_ram` -> cả hai port `rf_2p` | Clock chung cho read/write port | VERIFIED port/internal route; external UNKNOWN |
| `wdata` | in | 32 | Khối ngoài UNKNOWN -> `rf_2p.datab_i` | Dữ liệu write | VERIFIED |
| `waddr` | in | 4 | Khối ngoài UNKNOWN -> `rf_2p.addrb_i` | Địa chỉ write, 16 entry có thể đánh địa chỉ | VERIFIED |
| `we` | in | 1 | Khối ngoài UNKNOWN -> `~we` -> `cenb_i`, `wenb_i` | Write enable active-high ở wrapper, đảo thành active-low cho macro | VERIFIED |
| `rd` | in | 1 | Khối ngoài UNKNOWN -> `~rd` -> `cena_i` | Read enable active-high ở wrapper, đảo thành active-low cho macro | VERIFIED |
| `raddr` | in | 4 | Khối ngoài UNKNOWN -> `rf_2p.addra_i` | Địa chỉ read | VERIFIED |
| `rdata` | out | 32 | `rf_2p.dataa_o` -> khối ngoài UNKNOWN | Dữ liệu read | VERIFIED route; timing read UNKNOWN vì source `rf_2p` chưa khảo sát |

Phân loại memory: **explicit dual-port memory wrapper**, rộng 32 bit, sâu 16 địa chỉ theo parameter. Reset behavior và synchronous/asynchronous read: **UNKNOWN** vì implementation `rf_2p` không nằm trong phạm vi file đã đọc.

## Các điểm dễ nhầm và UNKNOWN

1. `modebest64[27:0]` là một giá trị metric/cost; mode cuối thực sự là `bestmode64_o[5:0]`.
2. `fetch8x8.cnt/blockcnt` cấp cho `mode_write`; `control.cyclecnt/blockcnt` chỉ tồn tại bên trong `md_top`. Hai bộ counter chạy theo logic tương tự nhưng là các register riêng.
3. `md_fetch.enable`, `mode_write.finish`, `rate_control.reg_bitnum_i` được khai báo và nối nhưng không được sử dụng trong thân module.
4. `control.newblock` được tạo nhưng chỉ nối tới wire không có consumer trong `md_top`.
5. `md_ram` không được instantiate trong hierarchy `prei_top`; mối liên hệ giữa nó với các tín hiệu `md_we/md_waddr/md_wdata` không có bằng chứng kết nối trực tiếp trong các file khảo sát.
6. Producer/consumer ngoài `prei_top` của giao tiếp pixel, mode RAM, cấu hình RC, QP và done là **UNKNOWN** trong phạm vi tài liệu này.
7. Ý nghĩa encoding của `md_sel_o`, `md_size_o`, `md_idx_o` ngoài các giá trị RTL cụ thể là **UNKNOWN**.
8. Đơn vị chính xác của `reg_L1_frame_byte`, `reg_L2_frame_byte` và scaling/fixed-point của `reg_k` là **UNKNOWN**.
9. Timing read của `rf_2p` bên dưới `md_ram` là **UNKNOWN**.

## Checklist xác minh cho phạm vi I/O

- [x] Module hierarchy verified
- [x] Module instantiations verified
- [x] Port directions verified
- [x] Signal widths verified, gồm resolve parameter/macro mặc định
- [x] Major signal connections verified
- [x] Memories/buffers verified trong phạm vi I/O; timing nội bộ `rf_2p` UNKNOWN
- [x] FSMs checked: không có FSM state-encoded riêng trong các controller khảo sát; điều khiển dựa trên counter/flag
- [x] Sequential registers liên quan trực tiếp I/O verified
- [ ] Pipeline boundaries: ngoài phạm vi bảng I/O; latency đầy đủ giữa mọi stage chưa được chứng minh
- [x] Datapath chính giữa các module verified
- [x] Control path chính giữa các module verified
- [x] Clock/reset domains verified: một `clk`; reset active-low bất đồng bộ cho các module có `rstn`; `md_ram` không có reset

Các mục bắt buộc cuối báo cáo:

1. **RTL blocks missing from diagram:** `rf_2p` chỉ biểu diễn như child ngoài thư mục; implementation chưa có trong phạm vi. Không có module `prei` nào khác bị bỏ khỏi hierarchy; `md_ram` được tách standalone.
2. **Diagram blocks without direct RTL evidence:** không có.
3. **Missing or uncertain connections:** endpoints ngoài `prei_top`; connection sử dụng thực tế của `md_ram`; consumer của `newblock` không tồn tại.
4. **Uncertain signal directions:** không có tại boundary các module đã khảo sát.
5. **Uncertain signal widths:** không có với cấu hình macro/parameter hiện tại; nếu override `MODE/DIGIT`, các bus metric thay đổi theo biểu thức đã nêu.
6. **Unresolved hierarchy:** implementation của `rf_2p`; hierarchy bên ngoài `prei_top`.
7. **Unresolved pipeline stages:** latency end-to-end đầy đủ chưa lập trong tài liệu I/O này.
8. **Unresolved memory implementation:** primitive/behavior của `rf_2p`, nhất là read timing/reset.
9. **Unresolved FSM behavior:** không có FSM mã hóa riêng được nhận diện; sequence counter đã xác minh nhưng chưa lập biểu đồ thời gian đầy đủ.
10. **Other UNKNOWN items:** encoding giao tiếp fetch, scaling `reg_k`, đơn vị threshold RC, endpoint hệ thống ngoài PREI.
