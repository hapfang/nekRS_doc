# Boundary Conditions

## Where BCs are declared
- Boundary surfaces come from mesh tags/IDs.
- Runtime mapping is primarily done in `.par` via `boundaryTypeMap`.
- Value assignment for user BCs is done in OKL device callbacks (`udfDirichlet`, `udfNeumann`, `udfRobin`).

## Working model
1. Map mesh IDs using `[MESH] boundaryIDMap` (and `boundaryIDMapFluid` for CHT).
2. In each field section (`[FLUID VELOCITY]`, `[SCALAR ...]`), set `boundaryTypeMap` in the same ordering.
3. For `udf*` entries, implement value logic in the OKL block.

## BC families
- Zero-value BCs: `zeroDirichlet`, `zeroNeumann`, and directional/symmetry variants.
- User-value BCs: `udfDirichlet`, `udfNeumann`, `udfRobin`, `interpolation`.
- Internal/periodic: often set to `none` in `boundaryTypeMap`, connectivity/periodicity handled in mesh.

## Key references
- [[Meshing & Mesh Workflows]] for side-set and numeric tag behavior.
- [[Case Files - par udf re2 usr sess upd]] for field-card context.
- [[Tutorial - Conjugate Heat Transfer (CHT)]] for multi-domain BC ordering.
