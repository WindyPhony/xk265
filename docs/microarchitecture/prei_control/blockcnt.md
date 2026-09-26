# blockcnt[6:0]

### blockcnt[6:0]

**Block:** `blockcnt[6:0]` update path. **Status:** register and behavior VERIFIED; mux/gate realization INFERRED. **RTL evidence:** [control.v:63](/home/windyphony/projects/xk265/rtl/prei/control.v:63), lines 63–69. **Evidence:** `if (!rstn) 0; else if (enable && cyclecnt == 40) blockcnt + 1; else if (finish) 0; else hold.`

![blockcnt[6:0]](blockcnt.svg)

[SVG](blockcnt.svg) · [PNG](blockcnt.png) · [Mermaid](blockcnt.mmd)

<details>
<summary>Mermaid source and preview</summary>

```mermaid
flowchart LR
  M1[/"MUX [I]"\] -->|0| M2[/"MUX [I]"\]
  M2 -->|D| Q["blockcnt[6:0]<br/>DFF reset 0 [V]"]
  Q --> O(["blockcnt[6:0]"])
  Q -->|"0: hold"| M1
  Z1["7'd0"] -->|1| M1
  Q -->|"1: blockcnt + 1"| M2
  S1(["finish"]) -.->|select| M1
  C0(["cyclecnt == 40"]) -.-> G["AND [I]"]
  C1(["enable"]) -.-> G["AND [I]"]
  G -.->|select| M2
  CLK["clk"] -.->|posedge| Q
  RST["rstn"] -.->|"async active-low reset"| Q
  classDef default fill:#fff,stroke:#555,color:#111;
```

</details>
