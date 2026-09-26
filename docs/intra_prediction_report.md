# xk265 Intra-Prediction: Algorithm and Hardware Architecture

## 1. Mục tiêu và phạm vi

Report này mô tả khối intra-prediction được triển khai trong xk265. Nội dung được chia thành:

1. **Algorithm**: cách tạo mẫu dự đoán intra từ các mẫu tham chiếu phía trên, bên trái và góc trên-trái.
2. **Hardware architecture**: cách thuật toán được ánh xạ vào các module Verilog, FSM, buffer và interface trong `rtl/rec/rec_intra/`.

Nguồn sự thật là RTL hiện có; các nhận định không thể chứng minh trực tiếp từ RTL được đánh dấu **INFERRED** hoặc **UNKNOWN**.

Phạm vi của report là `intra_top` trong reconstruction path. Khối PREI/POSI thực hiện chọn mode trước đó và được xem là nguồn của `partition_i` và mode RAM, không phải lõi tạo prediction được mô tả ở đây.

## 2. Vị trí trong encoder

Trong `rec_top`, `intra_top` chỉ được kích hoạt khi loại block là INTRA. `intra_top` nhận partition tree và mode đã được quyết định, tạo prediction pixels, sau đó đưa prediction vào buffer reconstruction chung.

```mermaid
flowchart LR
    PREI[PREI/POSI\nmode candidates + partition] -->|partition_i[84:0]\nmd_rd_*| INTRA[intra_top]
    FETCH[Current pixel / reconstruction context] --> REC[rec_top]
    REC -->|start_i, CTU coordinates| INTRA
    INTRA -->|pre_dat_o[255:0]\npre_val_o| PREBUF[rec_buf_pre]
    INTRA -->|done_o| REC
    REC --> TQ[TQ / quantization / reconstruction]
    TQ -->|rec_bgn, rec_dat[255:0]| INTRA
```

## 3. Algorithm

### 3.1 Input block và reference samples

Một prediction block được xác định bởi:

- component `Y`, `U` hoặc `V`;
- kích thước `4x4`, `8x8`, `16x16` hoặc `32x32`;
- mode 6-bit (`mode_i`), dùng để biểu diễn các mode intra;
- tọa độ block 4x4 trong CTU;
- các mẫu tham chiếu đã lưu từ block phía trên, bên trái, bên phải và phía dưới.

Ở mức thuật toán, tập reference gồm:

```text
        T[0] ... T[31]       mẫu phía trên
L[0]    P[0][0] ...          block cần dự đoán
...
L[31]

TL                           mẫu góc trên-trái
```

Trong RTL, `intra_ref` phát ra `ref_tl`, `ref_t00..ref_t31`, `ref_l00..ref_l31`, `ref_r00..ref_r31` và `ref_d00..ref_d31`. Mỗi mẫu có độ rộng `PIXEL_WIDTH = 8` bit.

### 3.2 Reference availability, padding và filtering

Không phải mọi mẫu lân cận đều tồn tại ở biên CTU hoặc biên frame. Thuật toán thực hiện các bước logic sau:

1. Xác định component, kích thước và vị trí block.
2. Tính địa chỉ đọc từ row, column và frame reference buffers.
3. Đọc các mẫu lân cận khả dụng.
4. Pad hoặc giữ lại giá trị biên khi một hướng tham chiếu không khả dụng.
5. Xuất cửa sổ reference cho prediction engine.

Phần này được thực hiện trong `intra_ref`. Việc padding cụ thể phụ thuộc vào các điều kiện biên trong RTL; không nên thay thế bằng giả định chung của một encoder HEVC khác.

### 3.3 Mode prediction

`intra_pred` dùng `mode_i` để chọn một trong ba nhóm thuật toán chính.

#### Planar

Planar tạo mặt phẳng nội suy từ mẫu biên trên và trái. Với block kích thước `N`, dạng khái niệm là:

```text
pred(x,y) = ((N-1-x) * L[y] + (x+1) * T[N]
           + (N-1-y) * T[x] + (y+1) * L[N]
           + N) >> (log2(N)+1)
```

Các tín hiệu trung gian `ver_*` và `hor_*` trong `intra_pred.v` thể hiện phép nội suy theo hai hướng. Cách đóng gói và rounding phải lấy theo RTL.

#### DC

DC sử dụng trung bình các mẫu tham chiếu trên và trái, sau đó điền giá trị DC cho phần lớn block. Các pixel biên có thể dùng giá trị riêng theo logic trong `intra_pred.v`.

#### Angular

Angular prediction dùng bảng ánh xạ mode-to-angle. RTL có các giá trị `pred_angle` cho các mode 2..34. Với mỗi pixel, phần cứng tính vị trí tham chiếu phân số:

```text
fact  = (offset + 1) * pred_angle
index = ((offset + 1) * |pred_angle|) >>> 5
```

Sau đó chọn mẫu gần nhất hoặc nội suy giữa hai mẫu tham chiếu. RTL dùng các thanh ghi `idx*`, `fact*`, `ref_idx*` và các nhánh dọc/ngang để tạo prediction.

### 3.4 Prediction output

Mỗi instance `intra_pred` xuất 16 pixel 8-bit, tương ứng một nhóm 4x4. `intra_top` dùng hai instance chạy song song:

- `u_intra_pred_a`: nhóm thứ nhất;
- `u_intra_pred_b`: nhóm kế tiếp, với `i4x4_x_i + 1`.

Hai nhóm được ghép thành:

```text
pre_dat_o[255:0] = 32 pixels × 8 bits
```

`pre_val_o` báo prediction hợp lệ; `pre_sel_o`, `pre_siz_o`, `pre_4x4_x_o` và `pre_4x4_y_o` mang metadata đi kèm.

## 4. Hardware architecture

### 4.1 Module hierarchy

```text
intra_top                         rtl/rec/rec_intra/intra_top.v
├── intra_ctrl                    điều khiển thứ tự block/component
├── intra_ref                     tạo cửa sổ reference
├── intra_pred                    prediction engine A
├── intra_pred                    prediction engine B
└── intra_buf_wrapper             remap địa chỉ và memory wrapper
    ├── ram_sp_384x32             row reference buffer
    ├── ram_sp_384x32             column reference buffer
    └── ram_sp_1536x32            frame reference buffer
```

```mermaid
flowchart LR
    subgraph TOP[intra_top]
        CTRL[intra_ctrl\nFSM + mode/partition traversal]
        REF[intra_ref\nreference generation]
        PA[intra_pred A\n16 x 8-bit]
        PB[intra_pred B\n16 x 8-bit]
        PACK[Concatenation\n32 x 8-bit]
        BW[intra_buf_wrapper]
        ROW[ram_sp_384x32\nrow]
        COL[ram_sp_384x32\ncolumn]
        FRA[ram_sp_1536x32\nframe]
    end

    CTRL -->|sel[1:0]\nsize[1:0]\nmode[5:0]\nposition[7:0]| REF
    CTRL -->|pre_start\n4x4 x/y| PA
    CTRL -->|pre_start\n4x4 x/y + 1| PB
    REF -->|TL, top, left, right, down\n8-bit samples| PA
    REF -->|same reference window| PB
    PA -->|16 pixels| PACK
    PB -->|16 pixels| PACK
    PACK -->|pre_dat_o[255:0]| OUT[Prediction output]

    REF -->|row/col/frame read/write| BW
    BW --> ROW
    BW --> COL
    BW --> FRA
    ROW --> BW
    COL --> BW
    FRA --> BW
    BW --> REF

    CTRL -.->|md_rd_ena, md_rd_adr| MODE[Mode RAM]
    MODE -.->|md_rd_dat[5:0]| CTRL
    REC[rec_top] -.->|rec write interface| REF
```

### 4.2 Control path: `intra_ctrl`

`intra_ctrl` có FSM bốn trạng thái:

| State | Vai trò |
|---|---|
| `IDLE` | Chờ `start_i` |
| `ENC_Y` | Duyệt các block luma |
| `ENC_U` | Duyệt các block chroma U |
| `ENC_V` | Duyệt các block chroma V |

Các thanh ghi điều khiển chính:

| Signal | Width | Ý nghĩa |
|---|---:|---|
| `loop_sel_o` | 2 | Component Y/U/V |
| `loop_size_o` | 2 | Kích thước 4/8/16/32 |
| `loop_mode_o` | 6 | Intra mode |
| `loop_position_o` | 8 | Vị trí block luma |
| `md_addr_o` | 8 | Địa chỉ đọc mode RAM |
| `pre_start_o` | 1 | Bắt đầu tạo prediction |
| `ref_start_o` | 1 | Bắt đầu lấy reference |

Khi `ref_done_i` xuất hiện, controller cập nhật vị trí và kích thước tiếp theo. `done_o` chỉ được phát khi component V hoàn tất block cuối cùng.

### 4.3 Reference path: `intra_ref`

`intra_ref` là cầu nối giữa traversal control, reconstructed samples và prediction engines.

Nó thực hiện:

- tính địa chỉ theo component và block size;
- điều khiển read enable cho row/column/frame buffers;
- nhận mẫu reconstruction từ `rec_top` để cập nhật reference memory;
- tạo `ref_tl`, top, right, left và down reference arrays;
- phát `ref_ready_o` và `done_o`.

`intra_buf_wrapper` remap địa chỉ theo component:

```text
Y: address prefix 000
U: address prefix 100
V: address prefix 101
```

Ba memory wrapper dùng data width 32 bit, tương đương 4 pixel 8-bit mỗi word.

### 4.4 Prediction datapath: `intra_pred`

`intra_pred.v` là datapath kết hợp gồm:

1. mode-to-angle lookup;
2. tính index/fraction cho angular prediction;
3. reference selection;
4. arithmetic cho Planar, DC, horizontal, vertical và angular modes;
5. register output 4x4;
6. valid/done generation.

Output được register ở clock edge. Vì vậy `pre_val_o` là tín hiệu đồng bộ, không phải combinational valid.

Hai instance A/B dùng chung reference window nhưng phát hai nhóm pixel liên tiếp. Đây là tối ưu throughput ở cấp 4x4; không nên diễn giải thành hai prediction engine độc lập cho toàn bộ CTU.

### 4.5 Reconstruction interface

`intra_top` không tự thực hiện transform/quantization. TQ nằm ở `rec_top`. Khi inverse transform/reconstruction tạo samples, `rec_top` gửi chúng trở lại qua:

```text
rec_bgn_i
rec_sel_i
rec_pos_i
rec_siz_i
rec_val_i
rec_idx_i
rec_dat_i[255:0]
```

`intra_ref` ghi các samples này vào row, column và frame buffers để phục vụ các block tiếp theo.

## 5. End-to-end operation

```text
1. rec_top asserts intra_start_w.
2. intra_ctrl enters ENC_Y and reads partition/mode metadata.
3. intra_ref prepares the reference window.
4. intra_pred A/B generate 32 prediction pixels.
5. intra_top asserts pre_val_o and transfers pre_dat_o[255:0].
6. rec_top consumes prediction for residual/TQ/reconstruction.
7. Reconstructed samples return through rec_* interface.
8. intra_ref writes updated boundary/frame references.
9. intra_ctrl advances to the next block, then U/V.
10. After the final V block, intra_top asserts done_o.
```

## 6. Design parameters and widths

| Item | Implemented value | Evidence |
|---|---:|---|
| Pixel width | 8 bit | `enc_defines.v`, `PIXEL_WIDTH` |
| Mode input | 6 bit | `intra_pred.v`, `mode_i[5:0]` |
| Block size code | 2 bit | `SIZE_04`, `SIZE_08`, `SIZE_16`, `SIZE_32` |
| Prediction output per `intra_pred` | 16 × 8 bit | `pred_00_o..pred_33_o` |
| Combined output | 32 × 8 bit = 256 bit | `intra_top.v`, `pre_dat_o` |
| Partition input | 85 bit | `intra_top.v`, `partition_i` |
| Row/column memory word | 32 bit | `intra_buf_wrapper.v` |
| Row/column memory depth | 384 words | `ram_sp_384x32` |
| Frame memory depth | 1536 words | `ram_sp_1536x32` |

## 7. Algorithm-to-RTL mapping

| Algorithm function | RTL implementation |
|---|---|
| Select component/block/mode | `intra_ctrl` |
| Read mode decision | `md_rd_ena_o`, `md_rd_adr_o`, `md_rd_dat_i` |
| Build reference window | `intra_ref` |
| Store boundary/reconstructed samples | `intra_buf_wrapper` + three SP memories |
| Planar/DC/angular prediction | `intra_pred` |
| Parallel 32-pixel output | two `intra_pred` instances in `intra_top` |
| Transfer prediction to REC | `pre_val_o`, `pre_dat_o`, `pre_*` metadata |
| Signal completion | `ref_done_w`, `done_o` |

## 8. Verification points and limitations

Các điểm nên kiểm tra bằng simulation hoặc waveform:

1. `ref_ready_o` có đến trước `pre_start_o` cho mọi block size hay không.
2. Độ trễ từ `pre_start_o` đến `pre_val_o` của `intra_pred`.
3. Sự căn chỉnh giữa `pre_4x4_x/y`, `pre_dat_o` và `pre_siz_o`.
4. Padding tại top/left frame boundary.
5. Địa chỉ remap Y/U/V trong `intra_buf_wrapper`.
6. Tính đúng của việc dùng `i4x4_x_i + 1` cho prediction engine B.
7. Điều kiện chuyển `ENC_Y → ENC_U → ENC_V` và `done_o`.

Latency tổng cho một CTU không nên suy ra chỉ từ số pipeline stage. Nó phụ thuộc partition tree, component, block size, memory latency và điều kiện `ref_done_i` trong FSM.

## 9. RTL references

- [`intra_top.v`](../rtl/rec/rec_intra/intra_top.v)
- [`intra_ctrl.v`](../rtl/rec/rec_intra/intra_ctrl.v)
- [`intra_ref.v`](../rtl/rec/rec_intra/intra_ref.v)
- [`intra_pred.v`](../rtl/rec/rec_intra/intra_pred.v)
- [`intra_buf_wrapper.v`](../rtl/rec/rec_intra/intra_buf_wrapper.v)
- [`rec_top.v`](../rtl/rec/rec_top.v)
- [`08_rec.md`](../rtl/docs/architecture/08_rec.md)

