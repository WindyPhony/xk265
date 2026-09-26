# counterrun1

### counterrun1

**Block:** `counterrun1` update path. **Status:** register and behavior VERIFIED; mux/gate realization INFERRED. **RTL evidence:** [control.v:79](/home/windyphony/projects/xk265/rtl/prei/control.v:79), lines 79–83. **Evidence:** `if (!rstn) 0; else gxgyrun.`

![counterrun1](counterrun1.svg)

[SVG](counterrun1.svg) · [PNG](counterrun1.png) · [Mermaid](counterrun1.mmd)

<details>
<summary>Mermaid source and preview</summary>

```mermaid
flowchart LR
  S["gxgyrun<br/>shared registered source [V]"] -->|D| Q["counterrun1<br/>DFF reset 0 [V]"]
  Q --> O(["counterrun1"])
  CLK["clk"] -.->|posedge| Q
  RST["rstn"] -.->|"async active-low reset"| Q
  classDef default fill:#fff,stroke:#555,color:#111;
```

</details>
