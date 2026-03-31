# Tutorial - Fully Developed Laminar Flow (FDLF)

## Goal
Build an incompressible channel-flow + constant wall heat-flux case with analytic reference profiles for validation.

## Main steps
1. Generate mesh (`genbox`) from `fdlf.box` and produce `fdlf.re2`.
2. Create `fdlf.par` with fluid/scalar properties and case data.
3. Implement `fdlf.udf`:
   - read user parameters,
   - set ICs,
   - apply inlet/flux BCs in OKL callbacks.
4. Run with `nrsbmpi fdlf 4`.
5. Compare velocity/temperature line plots against analytic solutions.

## What this tutorial teaches
- End-to-end case assembly (`.re2 + .par + .udf`).
- Parameter handoff from `.par` user sections to host/device code.
- Basic postprocessing with checkpoint files + ParaView/Visit.

## Prerequisites
- [[Quickstart & Environment Setup]]
- [[Case Files - par udf re2 usr sess upd]]
