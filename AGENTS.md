# Repository Guidelines

This repository is an open-source HEVC/H.265 hardware encoder from VIP Lab, Fudan University. It is a pure Verilog RTL project; there is no software build system beyond simulator and reference-vector utilities.

## Project Structure & Module Organization

- `rtl/` contains encoder RTL by block: `top/`, `prei/`, `posi/`, `ime/`, `fme/`, `fetch/`, `rec/`, `cabac/`, `db/`, and `mem/`.
- `rtl/rec/` is split into `rec_tq/`, `rec_intra/`, `rec_mc/`, and `rec_wrapper/`.
- `rtl/enc_defines.v` holds shared macros and is included as `` `include "enc_defines.v" ``.
- `rtl/mem/*.v` wraps behavioral memories from `lib/behave/mem/`; keep simulation models portable.
- `sim/top_testbench/` contains `tb_enc_top.v`, `file_list.f`, the makefile, and `tv/` input/golden vectors.
- `sw/` contains prebuilt reference encoder binaries and outputs under `sw/testVector/`.

## Build, Test, and Development Commands

Run simulator commands from `sim/top_testbench/`:

- `make vlog`: compile RTL with Questa/ModelSim; this is the fastest local check.
- `make vsim`: compile and run the top testbench with ModelSim.
- `make nclog` / `make ncsim`: compile or run with ncverilog, if installed.
- `make vcs`: run the VCS flow and generate FSDB output, if VCS/Verdi are installed.
- `make clean` / `make cleanall`: remove simulator work directories, logs, dumps, and waveforms.

`file_list.f` is the compile source of truth. Add every new RTL file there; do not rely on globbing.

## Testing Guidelines

The top-level testbench is `sim/top_testbench/tb_enc_top.v`. Default checks compare reconstructed YUV and bitstream output against `sim/top_testbench/tv/` goldens. Keep input sequence, resolution, QP, and golden vectors aligned. Use `+define+SMOKE_NO_GOLDEN` for smoke runs that intentionally skip golden checks.

Local note: the makefile references `/eda/synopsys/verdi201412/`, which may not exist. For local Questa runs without Verdi, remove `-pli $(NOVAS_LIB)` from `vsim` and compile with `+define+NO_DUMP`.

## Coding Style & Naming Conventions

Use the existing Verilog style: two-space indentation, aligned port declarations, and one module per file where practical. Prefer `lower_snake_case` filenames matching module names. Keep suffixes consistent: `_i` inputs, `_o` outputs, `_w` wires, and `_r` registers. The global active-low reset is `rstn`.

## Commit & Pull Request Guidelines

Recent commit messages are very short, such as `tb` or `b-d`; prefer concise but descriptive messages like `ime: fix mv buffer timing`. Do not commit generated simulator files such as `work/`, `INCA_libs/`, logs, FSDBs, dumps, or `dump/*`. PRs should identify the affected RTL block, simulator command used, and any test-vector or golden-output changes.
