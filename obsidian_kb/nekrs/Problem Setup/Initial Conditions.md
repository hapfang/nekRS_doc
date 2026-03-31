# Initial Conditions

## IC pathways
- Initialize directly in `UDF_Setup` (host vectors + copy to device fields).
- Restart from checkpoint files.
- Legacy `useric` in `.usr` for Nek5000-compatible workflows.

## Restart capabilities (high level)
- Different polynomial order between restart and target run.
- Different MPI partitioning.
- Built-in h-refinement-aware restart path.
- Multi-file/selective field restart and grid-to-grid interpolation.

## Practical pattern
1. Detect restart state in `.udf`.
2. If fresh run, initialize velocity/scalars explicitly.
3. Use scalar names that match `.par` (e.g., `temperature`, `k`, `tau`).

## Related notes
- [[Case Files - par udf re2 usr sess upd]]
- [[Meshing & Mesh Workflows]]
- [[Tutorial - Fully Developed Laminar Flow (FDLF)]]
