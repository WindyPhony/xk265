# finish

### finish

**Block:** `finish` update path. **Status:** register and behavior VERIFIED; mux/gate realization INFERRED. **RTL evidence:** [control.v:91](/home/windyphony/projects/xk265/rtl/prei/control.v:91), lines 91–97. **Evidence:** `if (!rstn) 0; else if (blockcnt == 65 && cyclecnt == 15) 1; else if (enable) 0; else hold.`

![finish](finish.svg)

[SVG](finish.svg) · [PNG](finish.png) · [Mermaid](finish.mmd)

<details>
<summary>Mermaid source and preview</summary>

```mermaid
flowchart LR
  M1[/"MUX [I]"\] -->|0| M2[/"MUX [I]"\]
  M2 -->|D| Q["finish<br/>DFF reset 0 [V]"]
  Q --> O(["finish"])
  Q -->|"0: hold"| M1
  Z1["1'b0"] -->|1| M1
  Z2["1'b1"] -->|1| M2
  S1(["enable"]) -.->|select| M1
  C0(["blockcnt == 65"]) -.-> G["AND [I]"]
  C1(["cyclecnt == 15"]) -.-> G["AND [I]"]
  G -.->|select| M2
  CLK["clk"] -.->|posedge| Q
  RST["rstn"] -.->|"async active-low reset"| Q
  classDef default fill:#fff,stroke:#555,color:#111;
```

</details>
