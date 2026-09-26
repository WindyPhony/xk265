# cyclecnt[5:0]

### cyclecnt[5:0]

**Block:** `cyclecnt[5:0]` update path. **Status:** register and behavior VERIFIED; mux/gate realization INFERRED. **RTL evidence:** [control.v:55](/home/windyphony/projects/xk265/rtl/prei/control.v:55), lines 55–61. **Evidence:** `if (!rstn) 0; else if (cyclecnt == 40 || finish) 0; else if (enable) cyclecnt + 1; else hold.`

![cyclecnt[5:0]](cyclecnt.svg)

[SVG](cyclecnt.svg) · [PNG](cyclecnt.png) · [Mermaid](cyclecnt.mmd)

<details>
<summary>Mermaid source and preview</summary>

```mermaid
flowchart LR
  M1[/"MUX [I]"\] -->|0| M2[/"MUX [I]"\]
  M2 -->|D| Q["cyclecnt[5:0]<br/>DFF reset 0 [V]"]
  Q --> O(["cyclecnt[5:0]"])
  Q -->|"0: hold"| M1
  Q -->|"1: cyclecnt + 1"| M1
  Z2["6'd0"] -->|1| M2
  S1(["enable"]) -.->|select| M1
  C0(["cyclecnt == 40"]) -.-> G["OR [I]"]
  C1(["finish"]) -.-> G["OR [I]"]
  G -.->|select| M2
  CLK["clk"] -.->|posedge| Q
  RST["rstn"] -.->|"async active-low reset"| Q
  classDef default fill:#fff,stroke:#555,color:#111;
```

</details>
