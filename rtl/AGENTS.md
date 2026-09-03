# xk265 RTL Reverse Engineering Rules

## Mission

Reverse-engineer the existing xk265 HEVC encoder RTL and generate
RTL-level architecture documentation and block diagrams.

The purpose is to understand the existing hardware implementation.

This is NOT a redesign task.

---

## Source of Truth

The RTL source code under `rtl/` is the primary source of truth.

Never assume that the RTL follows a generic HEVC architecture.

Never invent:

- modules
- signals
- memories
- datapaths
- FSMs
- pipeline stages
- interfaces

If something cannot be verified from RTL, mark it as:

UNKNOWN

If something is inferred from RTL behavior, mark it as:

INFERRED

---

## RTL Must Not Be Modified

Do not modify files under:

rtl/

unless explicitly requested.

The default task is READ-ONLY reverse engineering.

---

## Analysis Procedure

For every subsystem:

1. Identify top module.
2. Identify instantiated modules.
3. Build module hierarchy.
4. Identify input/output ports.
5. Trace important signals.
6. Identify datapath.
7. Identify control path.
8. Identify FSMs.
9. Identify memories and buffers.
10. Identify arithmetic units.
11. Identify registers.
12. Identify pipeline stages.
13. Identify interfaces with other subsystems.
14. Generate architecture documentation.
15. Generate block diagram.
16. Cross-check diagram against RTL.

---

## Evidence Requirement

Every major block in a diagram must have RTL evidence.

For example:

Block:
IME SAD Engine

RTL module:
ime_sad

File:
rtl/ime/ime_sad.v

Evidence:
actual module instantiation and signal connectivity.

---

## Diagram Rules

Solid arrows:
DATA

Dashed arrows:
CONTROL

Important buses must show width.

Example:

pixel [7:0]

mv_x [15:0]

cost [31:0]

Registers must be explicitly represented.

Memories must be explicitly represented.

FSMs must be explicitly represented.

MUXes should be represented when they affect datapath selection.

---

## Hierarchy

Preserve RTL hierarchy.

Do not flatten modules unnecessarily.

Example:

IME
├── ime_top
├── ime_ctrl
├── ime_search
├── ime_sad
└── ime_mem

---
## Evidence Classification

Every item in the architecture documentation and block diagram must be assigned one of three evidence levels:

- VERIFIED: Directly supported by RTL syntax such as module declarations, module instantiations, ports, registers, memories, always blocks, assignments, or explicit signal connectivity.
- INFERRED: Not implemented as a separate named module/block, but its functional behavior can be derived from the RTL.
- UNKNOWN: Cannot be reliably determined from the available RTL.

Do not present INFERRED or UNKNOWN items as VERIFIED.

Every functional block, register, memory, FSM, arithmetic unit, MUX, and interface shown in the diagram must have traceable RTL evidence.

---

## Module vs Functional Block

Preserve the actual RTL module hierarchy.

Do not create a new RTL module merely because a group of logic appears to implement a recognizable function.

For example, if an `always` block inside `ime_top` implements SAD calculation, represent it as:

    SAD Calculation
    Status: INFERRED
    RTL: ime_top.v

Do not represent it as:

    ime_top
    └── ime_sad

unless an actual `ime_sad` module exists in the RTL.

Signal names, comments, HEVC terminology, or expected codec architecture must never be used as the sole evidence for creating a hardware block.

---

## Sequential Logic and Pipeline Identification

Do not classify every `always @(posedge clk)` block as a pipeline stage.

A pipeline stage should only be identified when sequential registers form a meaningful datapath boundary between two processing stages.

For every reported pipeline stage, provide:

- source register
- destination register
- combinational logic between them
- clock boundary
- latency if determinable

If latency cannot be determined reliably from the RTL, report:

    Latency: UNKNOWN

or:

    Latency: Variable / FSM-dependent

---

## Memory Classification

For every memory or buffer, determine its implementation from RTL:

- register
- register array
- inferred RAM
- explicit RAM module
- FIFO
- shift register
- buffer
- UNKNOWN

Do not call a register array a RAM unless the RTL implementation supports that classification.

Report:

- depth
- width
- read port(s)
- write port(s)
- address
- read enable
- write enable
- reset behavior
- synchronous/asynchronous read behavior when determinable

---

## Parameters and Width Resolution

Before documenting signal widths, memory depth, counters, and address ranges, resolve:

- parameters
- localparams
- `define` macros
- constant expressions
- generate conditions

If the final width or range depends on an unresolved compile-time configuration, report the dependency explicitly.

Do not guess widths.

---

## Clock and Reset Analysis

Identify:

- clock signal(s)
- clock domains
- reset signal(s)
- reset polarity
- synchronous/asynchronous reset
- registers affected by reset
- FSM reset state

Do not assume that all modules belong to the same clock/reset domain.

---

## Evidence Traceability

Every major architecture statement must be traceable to RTL.

Use the following format:

Block:
    <block name>

Status:
    VERIFIED / INFERRED / UNKNOWN

RTL Evidence:
    <file path>

Evidence:
    <module instantiation, always block, assignment,
     signal connection, or relevant RTL behavior>

This evidence must be sufficient for another engineer to locate and verify the claim in the RTL.

## Verification Checklist

Before declaring the analysis complete, perform the following checks and report the result explicitly:

- [ ] Module hierarchy verified
- [ ] Module instantiations verified
- [ ] Port directions verified
- [ ] Signal widths verified
- [ ] Major signal connections verified
- [ ] Memories/buffers verified
- [ ] FSMs verified
- [ ] Sequential registers verified
- [ ] Pipeline boundaries verified
- [ ] Datapath verified
- [ ] Control path verified
- [ ] Clock/reset domains verified

Any unchecked item must be reported with the reason.

The final report must explicitly list:

1. RTL blocks missing from the diagram.
2. Diagram blocks without direct RTL evidence.
3. Missing or uncertain connections.
4. Uncertain signal directions.
5. Uncertain signal widths.
6. Unresolved hierarchy.
7. Unresolved pipeline stages.
8. Unresolved memory implementation.
9. Unresolved FSM behavior.
10. Other UNKNOWN items.
