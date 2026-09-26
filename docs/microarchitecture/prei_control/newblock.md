# newblock

### newblock

**Block:** `newblock` update path. **Status:** register and behavior VERIFIED; mux/gate realization INFERRED. **RTL evidence:** [control.v:47](/home/windyphony/projects/xk265/rtl/prei/control.v:47), lines 47–53. **Evidence:** `if (!rstn) 0; else (cyclecnt == 40).`

![newblock](newblock.svg)

[SVG](newblock.svg) · [PNG](newblock.png) · [Mermaid](newblock.mmd)

<details>
<summary>Mermaid source and preview</summary>

```mermaid
flowchart LR
  C["cyclecnt[5:0]<br/>shared counter [V]"] --> CMP(["== 6'd40 [V]"])
  CMP -->|D| Q["newblock<br/>DFF reset 0 [V]"]
  Q --> O(["newblock"])
  CLK["clk"] -.->|posedge| Q
  RST["rstn"] -.->|"async active-low reset"| Q
  classDef default fill:#fff,stroke:#555,color:#111;
```

</details>
