# Review ASICON 2013 và so sánh với Intra hiện tại của xk265

## 1. Phạm vi

**Paper:** *A Highly Pipelined VLSI Architecture for All Modes and Block Sizes Intra Prediction in HEVC Encoder* (Cong Liu et al., ASICON 2013).

Paper mô tả một kiến trúc VLSI intra-prediction HEVC. Báo cáo này đối chiếu paper với RTL hiện tại trong rtl/rec/rec_intra/. Các con số synthesis trong paper không được dùng để suy ra PPA của RTL hiện tại nếu chưa chạy synthesis tương ứng.

## 2. Tóm tắt paper

Paper giải quyết data dependency do reconstructed neighbor samples, 35 modes và quad-tree partition. Kiến trúc đề xuất có:

- 16-pixel parallel processing;
- hai prediction engines;
- universal predictor cho 32x32, 16x16, 8x8 và 4x4;
- post-order traversal để giảm intermediate buffers;
- predictor pipeline ba stage;
- 8967 cycles cho một 32x32 treeblock;
- TSMC 65 nm, 600 MHz, 77K gates trong kết quả báo cáo.

HEVC modes được chia thành:

| Mode | Chức năng |
|---|---|
| 0 | Planar |
| 1 | DC |
| 2..34 | Angular |

Angular prediction được mô tả bằng:

~~~text
iIdx  = ((x + 1) * Angle) >> 5
iFact = ((x + 1) * Angle) & 31
P     = ((32 - iFact) * A + iFact * B + 16) >> 5
~~~

Với modes 18..34, trục x/y được hoán đổi. Reference samples được lấy từ top, left, top-left, top-right và bottom-left; trước prediction có substitution, filtering và projection.

### 2.1 Top-level paper

~~~mermaid
flowchart LR
    ORIG[Original pixel buffer] --> A[Prediction Engine A\nfull mode + size scan]
    RECON[Reconstructed pixel buffer] --> B[Prediction Engine B\nbest mode only]
    A --> DEC[Mode and split decision]
    A --> RES[Residual calculation]
    B --> RES
    A --> PBUF[Predicted pixel buffer]
    B --> PBUF
    RES --> TQ[IDCT/IDST + IQ + DCT/DST + Q]
    TQ --> LOOP[Reconstruction loop]
    LOOP --> RECON
    CTRL[Controller] -.-> A
    CTRL -.-> B
    CTRL -.-> DEC
~~~

Engine A quét full mode/full size trên original pixels để quyết định mode và partition. Engine B dùng reconstructed samples nhưng chỉ dự đoán mode tốt nhất.

### 2.2 Universal predictor

Paper chia datapath thành ba stage:

1. Stage 0: tính iIdx, iFact và projection.
2. Stage 1: chọn reference pixels.
3. Stage 2: interpolation để tạo 16 predicted pixels.

Paper cũng dùng post-order traversal:

~~~text
leaf 4x4 -> parent 8x8 -> parent 16x16 -> root 32x32
~~~

## 3. Algorithm hiện tại của xk265

### 3.1 Phân ranh giới chức năng

Trong xk265, mode decision đã được tách khỏi final predictor:

~~~text
PREI  -> pre-intra estimation / mode shortlist
POSI  -> mode cost + partition decision
REC   -> final intra prediction + transform/reconstruction
~~~

intra_top nhận partition_i[84:0] và mode qua md_rd_*; nó không tự quét toàn bộ 35 modes để quyết định mode. Đây là khác biệt cấp kiến trúc lớn so với Engine A trong paper.

### 3.2 Datapath trong intra_pred

RTL hiện tại giữ nguyên các ý tưởng chính của universal predictor:

- mode 0 chọn Planar;
- mode 1 chọn DC;
- mode 2..34 dùng angular prediction;
- bảng pred_angle;
- idx và fact dùng nhân với angle và dịch phải 5 bit;
- mode từ 18 trở lên đổi trục xử lý;
- interpolation dùng hai reference samples và fractional factor;
- output 4x4 được register.

Trong [intra_pred.v](../rtl/rec/rec_intra/intra_pred.v), các logic tương ứng là:

~~~text
pred_angle = mode-to-angle lookup
idx*       = ((coordinate + 1) * pred_angle) >>> 5
fact*      = ((coordinate + 1) * pred_angle)
ref_idx*   = delta_idx + idx*
~~~

Vì vậy, angular algorithm của xk265 tương đồng trực tiếp với paper ở mức công thức và hướng xử lý.

### 3.3 Block sizes

Paper và xk265 đều hỗ trợ effective prediction sizes 32x32, 16x16, 8x8 và 4x4. xk265 có CTU hệ thống 64x64, nhưng intra_ctrl duyệt prediction theo partition tree xuống bốn size này.

## 4. So sánh chi tiết

| Chủ đề | ASICON 2013 | xk265 hiện tại | Kết luận |
|---|---|---|---|
| Mode set | Planar, DC, 33 angular | Planar, DC, 33 angular | Tương đồng |
| Angular formula | iIdx/iFact, 1/32 interpolation | idx/fact, >>>5, interpolation | Tương đồng |
| Effective sizes | 32/16/8/4 | 32/16/8/4 | Tương đồng |
| Parallelism | 16 pixels/engine | Hai engine, mỗi engine 16 pixels; ghép thành 256-bit | Cùng granularity, xk rộng hơn ở output |
| Engine A | Original pixels + full scan + decision | intra_pred A nhận mode đã chọn | Khác lớn |
| Engine B | Reconstructed pixels + best mode | intra_pred B dùng cùng reference window với A | Khác lớn |
| Mode decision | Tích hợp trong Engine A | Tách thành PREI/POSI | Khác kiến trúc |
| Reference ownership | Original buffer cho A, reconstructed buffer cho B | intra_ref và memory trong REC; PREI/POSI là path riêng | Khác ownership |
| Predictor pipeline | Stage 0/1/2 được mô tả rõ | Có stage comments/registers tương tự; latency cần đo | Tương đồng một phần |
| Traversal | Post-order quad-tree | Position/size cập nhật theo partition bitfields | Chưa chứng minh cùng thứ tự |
| Cost algorithm | Full scan + distortion trong A | PREI gradient; POSI cost/SATD | Khác algorithm-level |
| Chroma | Có hỗ trợ | ENC_Y -> ENC_U -> ENC_V | Tương thích |
| Throughput | 8967 cycles/32x32 | Không có cycle cố định từ static RTL | Không so sánh trực tiếp |
| PPA | 77K gates, 600 MHz, 65 nm | Chưa có synthesis evidence tương ứng | Không được suy diễn |

## 5. Hai engine trong paper và hai instance trong xk265

Đây là điểm dễ gây nhầm lẫn nhất.

### Paper

~~~text
Engine A = original reference + full mode/size scan + mode decision
Engine B = reconstructed reference + best mode prediction
~~~

### xk265

~~~text
u_intra_pred_a = prediction lane 0
u_intra_pred_b = prediction lane 1
~~~

Bằng chứng từ [intra_top.v](../rtl/rec/rec_intra/intra_top.v):

- A và B đều nhận cùng ref_tl, ref_t, ref_r, ref_l, ref_d từ intra_ref;
- A và B đều nhận cùng mode_i, size_i, pre_start_w;
- B chỉ thay đổi i4x4_x_i thành pre_4x4_x_w + 1;
- done_o của B không được dùng;
- output A/B được ghép trực tiếp thành pre_dat_o[255:0].

Do đó, không nên mô tả hai instance hiện tại là original-pixel engine và reconstructed-pixel engine. Chúng là hai lane song song tạo 32 pixel trong một transfer.

## 6. Reference và buffer architecture

Paper nói đến original pixel buffer, reconstructed pixel buffer và predicted pixel buffer, đồng thời nhấn mạnh substitution/filtering/projection.

Trong xk265:

- intra_ref tạo cửa sổ reference;
- intra_buf_wrapper remap địa chỉ theo Y/U/V;
- có row buffer ram_sp_384x32;
- có column buffer ram_sp_384x32;
- có frame buffer ram_sp_1536x32;
- reconstructed samples quay lại qua rec_bgn_i, rec_sel_i, rec_pos_i, rec_siz_i, rec_val_i, rec_idx_i, rec_dat_i.

Điều này cho thấy xk265 vẫn giữ tư tưởng reference preprocessing của paper, nhưng ownership và timing đã được tích hợp vào REC/CTU pipeline.

## 7. Traversal và mode decision

intra_ctrl có FSM:

~~~text
IDLE -> ENC_Y -> ENC_U -> ENC_V -> IDLE
~~~

Với luma, vị trí được cập nhật theo kích thước:

~~~text
SIZE_04: +1
SIZE_08: +4
SIZE_16: +16
SIZE_32: +64
~~~

Partition input được tách thành split fields 32/16/8. Đây là partition-aware traversal, nhưng static RTL chưa đủ để kết luận thứ tự hoàn toàn giống post-order của paper. Cần waveform với các partition tree khác nhau để xác nhận.

Mode decision hiện tại cũng khác paper:

- PREI thực hiện gradient-based pre-estimation;
- POSI thực hiện intra mode/cost/partition decision;
- intra_top chỉ đọc mode và phát prediction cuối.

## 8. Kết luận kỹ thuật

xk265 hiện tại kế thừa rõ ràng universal intra predictor của paper:

- 35-mode organization;
- angular prediction với 1/32 fractional interpolation;
- effective sizes 32/16/8/4;
- 16-pixel prediction granularity;
- reference preprocessing và register pipeline.

Nhưng xk265 không còn là top-level architecture y hệt paper. Khác biệt chính là:

1. mode decision được tách thành PREI/POSI;
2. hai intra_pred trong intra_top là hai lane song song, không phải Engine A/Engine B khác loại reference;
3. traversal và buffer được tích hợp vào CTU pipeline;
4. không thể gán trực tiếp 8967 cycles, 77K gates hoặc 600 MHz của paper cho RTL hiện tại.

Mô tả phù hợp trong microarchitecture specification là:

~~~text
The current xk265 implementation retains the universal 16-pixel intra
prediction datapath and angular interpolation method described in the
ASICON 2013 architecture, but integrates it as a two-lane final
prediction datapath inside REC. Mode and partition decisions are
produced upstream by PREI/POSI, while reconstructed reference samples
are maintained by intra_ref and intra_buf_wrapper.
~~~

## 9. Verification plan

1. Trace cur_position_r, cur_size_r, ref_start_o và ref_done_i để xác nhận traversal order.
2. Đo latency từ intra_pred.start_i đến done_o.
3. Đếm transfer prediction theo size: 4x4 = 1, 8x8 = 2, 16x16 = 8, 32x32 = 32.
4. So sánh output RTL với golden software cho mode 0..34.
5. Đo timing ref_ready_i -> pre_start_o -> pre_val_o -> rec_done_o -> done_o.
6. Chạy synthesis riêng cho RTL hiện tại trước khi so sánh PPA với paper.

## 10. Evidence index

### Paper

- Algorithm/mode organization: pages 1-2, Figures 1-2.
- Two-engine top architecture: page 2, Figure 3.
- Timing/data dependency: page 2, Figure 4.
- Three-stage predictor: page 3, Figure 5.
- Post-order traversal: pages 3-4, Figure 6.
- 8967-cycle result: page 4, Figure 7 and Section 4.

### xk265 RTL

- [intra_top.v](../rtl/rec/rec_intra/intra_top.v)
- [intra_pred.v](../rtl/rec/rec_intra/intra_pred.v)
- [intra_ctrl.v](../rtl/rec/rec_intra/intra_ctrl.v)
- [intra_ref.v](../rtl/rec/rec_intra/intra_ref.v)
- [intra_buf_wrapper.v](../rtl/rec/rec_intra/intra_buf_wrapper.v)
- [rec_top.v](../rtl/rec/rec_top.v)
- [04_prei.md](../rtl/docs/architecture/04_prei.md)
- [07_posi.md](../rtl/docs/architecture/07_posi.md)
