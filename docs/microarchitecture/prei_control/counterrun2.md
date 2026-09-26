# counterrun2

### counterrun2

**Block:** `counterrun2` update path. **Status:** register and behavior VERIFIED; mux/gate realization INFERRED. **RTL evidence:** [control.v:85](/home/windyphony/projects/xk265/rtl/prei/control.v:85), lines 85–89. **Evidence:** `if (!rstn) 0; else counterrun1.`

![counterrun2](counterrun2.svg)

[SVG](counterrun2.svg) · [PNG](counterrun2.png) · [Mermaid](counterrun2.mmd)

<details>
<summary>Mermaid source and preview</summary>

```mermaid
flowchart LR
  S["counterrun1<br/>shared registered source [V]"] -->|D| Q["counterrun2<br/>DFF reset 0 [V]"]
  Q --> O(["counterrun2"])
  CLK["clk"] -.->|posedge| Q
  RST["rstn"] -.->|"async active-low reset"| Q
  classDef default fill:#fff,stroke:#555,color:#111;
```

</details>
