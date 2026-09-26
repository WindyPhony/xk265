# fetch8x8 — one diagram per output

SVG schematics follow the approved vertical-mux style. Each output also has a Mermaid source; Mermaid uses approximate mux geometry and automatic layout. Registers use the latest rectangular DFF style. Shared counter references do not add storage.

## cnt

Clear on cnt == 40 or finish has priority over increment. Otherwise enable selects increment versus hold.

![cnt](fetch8x8_outputs/cnt.svg)

[SVG](fetch8x8_outputs/cnt.svg) · [Mermaid](fetch8x8_outputs/cnt.mmd)

<details>
<summary>Mermaid source and preview</summary>

```mermaid
%% Main spine follows the user's schematic: update mux -> clear mux -> DFF -> output.
%% cnt + 1 on a feedback edge denotes 6-bit combinational arithmetic, not a wire alias.
%% Native Mermaid does not guarantee vertical mux geometry or fixed pin placement.
flowchart LR
  M1[/"MUX"\] -->|0| M2[/"MUX"\]
  M2 -->|D| Q["cnt[5:0]<br/>DFF / reset = 0"]
  Q --> OUT(["cnt[5:0]"])
  Q -->|"0: cnt"| M1
  Q -->|"1: cnt + 6'd1"| M1
  EN["enable"] -.->|select| M1
  ZERO["6'd0"] -->|1| M2
  Q -.-> CMP(["cnt == 6'd40"])
  CMP --> OR["OR"]
  FIN["finish"] --> OR
  OR -.->|select| M2
  CLK["clk"] -.->|posedge| Q
  RST["rstn"] -.->|"async active-low"| Q
  classDef default fill:#fff,stroke:#555,color:#111;
```

</details>

## blockcnt

Increment on enable AND cnt == 32 has priority over finish. Arithmetic wraps at 7 bits.

![blockcnt](fetch8x8_outputs/blockcnt.svg)

[SVG](fetch8x8_outputs/blockcnt.svg) · [Mermaid](fetch8x8_outputs/blockcnt.mmd)

<details>
<summary>Mermaid source and preview</summary>

```mermaid
%% blockcnt: clear/hold mux -> increment-priority mux -> register -> output.
%% blockcnt + 7'd1 denotes 7-bit combinational arithmetic on the feedback path.
%% Native Mermaid approximates mux geometry and automatically places nodes.
flowchart LR
  M1[/"MUX"\] -->|0| M2[/"MUX"\]
  M2 -->|D| Q["blockcnt[6:0]<br/>DFF / reset = 0"]
  Q --> OUT(["blockcnt[6:0]"])
  Q -->|"0: blockcnt"| M1
  ZERO["7'd0"] -->|1| M1
  FIN["finish"] -.->|select| M1
  Q -->|"1: blockcnt + 7'd1"| M2
  CNT["cnt[5:0]<br/>shared counter output"] --> CMP(["cnt == 6'd32"])
  CMP --> AND["AND"]
  EN["enable"] --> AND
  AND -.->|select| M2
  CLK["clk"] -.->|posedge| Q
  RST["rstn"] -.->|"async active-low"| Q
  classDef default fill:#fff,stroke:#555,color:#111;
```

</details>

## rf_512bit

Upper lane captures at pre-edge cnt == 2; lower lane at cnt == 10. Capture is independent of enable and finish.

![rf_512bit](fetch8x8_outputs/rf_512bit.svg)

[SVG](fetch8x8_outputs/rf_512bit.svg) · [Mermaid](fetch8x8_outputs/rf_512bit.mmd)

<details>
<summary>Mermaid source and preview</summary>

```mermaid
flowchart LR
C["cnt[5:0]<br/>shared counter output"] --> E2["== 6'd2"]
  C --> E10["== 6'd10"]
  DATA["md_data_i[255:0]"] --> RDATA["rdata[255:0]<br/>wire alias"]
  RDATA -->|1| HM[/"MUX"\]
  RDATA -->|1| LM[/"MUX"\]
  E2 -.->|S| HM
  E10 -.->|S| LM
  H["rf_512bit[511:256]<br/>256-bit DFF bank / reset = 0"] -->|"0: hold"| HM
  L["rf_512bit[255:0]<br/>256-bit DFF bank / reset = 0"] -->|"0: hold"| LM
  HM -->|D| H
  LM -->|D| L
  H -->|"bits 511:256"| PACK["Concatenate {upper, lower}<br/>wiring only"]
  L -->|"bits 255:0"| PACK
  PACK --> O(["rf_512bit[511:0]"])
  CLK["clk"] -.->|posedge| H
  CLK -.->|posedge| L
  RST["rstn"] -.->|"async active-low reset"| H
  RST -.->|"async active-low reset"| L
  classDef default fill:#fff,stroke:#555,color:#111;
```

</details>

## md_ren_o

Set on cnt == 0 AND enable; else clear on cnt == 17; else hold.

![md_ren_o](fetch8x8_outputs/md_ren_o.svg)

[SVG](fetch8x8_outputs/md_ren_o.svg) · [Mermaid](fetch8x8_outputs/md_ren_o.mmd)

<details>
<summary>Mermaid source and preview</summary>

```mermaid
flowchart LR
C["cnt[5:0]<br/>shared counter output"] --> E0["== 6'd0"]
  C --> E17["== 6'd17"]
  E0 --> AND["AND"]
  EN["enable"] --> AND
  Q["md_ren_o register / 1-bit<br/>reset = 0"] -->|"0: hold"| CM[/"MUX"\]
  ZERO["1'b0"] -->|1| CM
  E17 -.->|S| CM
  CM -->|0| SM[/"MUX"\]
  ONE["1'b1"] -->|1| SM
  AND -.->|S| SM
  SM -->|D| Q
  Q --> O(["md_ren_o"])
  CLK["clk"] -.->|posedge| Q
  RST["rstn"] -.->|"async active-low reset"| Q
  classDef default fill:#fff,stroke:#555,color:#111;
```

</details>

## md_sel_o

Constant 1'b0.

![md_sel_o](fetch8x8_outputs/md_sel_o.svg)

[SVG](fetch8x8_outputs/md_sel_o.svg) · [Mermaid](fetch8x8_outputs/md_sel_o.mmd)

<details>
<summary>Mermaid source and preview</summary>

```mermaid
flowchart LR
CONST["1'b0"] --> O(["md_sel_o"])
  classDef default fill:#fff,stroke:#555,color:#111;
```

</details>

## md_size_o

Constant 2'b01.

![md_size_o](fetch8x8_outputs/md_size_o.svg)

[SVG](fetch8x8_outputs/md_size_o.svg) · [Mermaid](fetch8x8_outputs/md_size_o.mmd)

<details>
<summary>Mermaid source and preview</summary>

```mermaid
flowchart LR
CONST["2'b01"] --> O(["md_size_o[1:0]"])
  classDef default fill:#fff,stroke:#555,color:#111;
```

</details>

## md_4x4_x_o

{blockcnt[4], blockcnt[2], blockcnt[0], 1'b0}; wiring only.

![md_4x4_x_o](fetch8x8_outputs/md_4x4_x_o.svg)

[SVG](fetch8x8_outputs/md_4x4_x_o.svg) · [Mermaid](fetch8x8_outputs/md_4x4_x_o.mmd)

<details>
<summary>Mermaid source and preview</summary>

```mermaid
flowchart LR
B["blockcnt[6:0]<br/>shared counter output"] -->|"blockcnt[4]"| PACK["Concatenate / wiring only<br/>{blockcnt[4], blockcnt[2], blockcnt[0], 1'b0}"]
  B -->|"blockcnt[2]"| PACK
  B -->|"blockcnt[0]"| PACK
  ZERO["1'b0"] -->|"bit 0"| PACK
  PACK --> O(["md_4x4_x_o[3:0]"])
  classDef default fill:#fff,stroke:#555,color:#111;
```

</details>

## md_4x4_y_o

{blockcnt[5], blockcnt[3], blockcnt[1], 1'b0}; wiring only.

![md_4x4_y_o](fetch8x8_outputs/md_4x4_y_o.svg)

[SVG](fetch8x8_outputs/md_4x4_y_o.svg) · [Mermaid](fetch8x8_outputs/md_4x4_y_o.mmd)

<details>
<summary>Mermaid source and preview</summary>

```mermaid
flowchart LR
B["blockcnt[6:0]<br/>shared counter output"] -->|"blockcnt[5]"| PACK["Concatenate / wiring only<br/>{blockcnt[5], blockcnt[3], blockcnt[1], 1'b0}"]
  B -->|"blockcnt[3]"| PACK
  B -->|"blockcnt[1]"| PACK
  ZERO["1'b0"] -->|"bit 0"| PACK
  PACK --> O(["md_4x4_y_o[3:0]"])
  classDef default fill:#fff,stroke:#555,color:#111;
```

</details>

## md_idx_o

{2'b00, flag, 2'b00}; flag samples pre-edge cnt[3] every clock.

![md_idx_o](fetch8x8_outputs/md_idx_o.svg)

[SVG](fetch8x8_outputs/md_idx_o.svg) · [Mermaid](fetch8x8_outputs/md_idx_o.mmd)

<details>
<summary>Mermaid source and preview</summary>

```mermaid
flowchart LR
C["cnt[5:0]<br/>shared counter output"] --> SLICE["Select bit 3"]
  SLICE -->|D| F["flag / 1-bit DFF<br/>reset = 0"]
  F -->|"bit 2"| PACK["Concatenate / wiring only<br/>{2'b00, flag, 2'b00}"]
  ZERO["2'b00"] -->|"bits 4:3 and 1:0"| PACK
  PACK --> O(["md_idx_o[4:0]"])
  CLK["clk"] -.->|posedge| F
  RST["rstn"] -.->|"async active-low reset"| F
  classDef default fill:#fff,stroke:#555,color:#111;
```

</details>

## Verification

All nine RTL outputs are covered. Mux branches, update priorities, pre-edge semantics, bit mappings, reset values and constants were checked against the supplied RTL. SVG files passed XML parsing.

The seven newly completed output schematics were rendered to PNG and visually reviewed; overlapping coordinate-pack labels were corrected. Mermaid source is provided without a rendered-layout validation.
