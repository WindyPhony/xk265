# gxgyrun

### gxgyrun

**Block:** `gxgyrun` update path. **Status:** register and behavior VERIFIED; mux/gate realization INFERRED. **RTL evidence:** [control.v:71](/home/windyphony/projects/xk265/rtl/prei/control.v:71), lines 71–77. **Evidence:** `if (!rstn) 0; else if (cyclecnt == 5 && blockcnt != 64) 1; else if (cyclecnt == 1) 0; else hold.`

![gxgyrun](gxgyrun.svg)

[SVG](gxgyrun.svg) · [PNG](gxgyrun.png) · [Mermaid](gxgyrun.mmd)

<details>
<summary>Mermaid source and preview</summary>

```mermaid
flowchart LR
  M1[/"MUX [I]"\] -->|0| M2[/"MUX [I]"\]
  M2 -->|D| Q["gxgyrun<br/>DFF reset 0 [V]"]
  Q --> O(["gxgyrun"])
  Q -->|"0: hold"| M1
  Z1["1'b0"] -->|1| M1
  Z2["1'b1"] -->|1| M2
  S1(["cyclecnt == 1"]) -.->|select| M1
  C0(["cyclecnt == 5"]) -.-> G["AND [I]"]
  C1(["blockcnt != 64"]) -.-> G["AND [I]"]
  G -.->|select| M2
  CLK["clk"] -.->|posedge| Q
  RST["rstn"] -.->|"async active-low reset"| Q
  classDef default fill:#fff,stroke:#555,color:#111;
```

</details>
