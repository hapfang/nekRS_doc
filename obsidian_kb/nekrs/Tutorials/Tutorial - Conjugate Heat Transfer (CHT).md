# Tutorial - Conjugate Heat Transfer (CHT)

## Goal
Solve a quasi-2D Rayleigh-Bénard style conjugate heat-transfer case with fluid + solid thermal coupling.

## Main steps
1. Build fluid and solid meshes in Gmsh (`fluid.geo`, `solid.geo`).
2. Convert to nekRS mesh via `gmsh2nek`, including periodic pair setup.
3. Configure `cht.par`:
   - `boundaryIDMap` + `boundaryIDMapFluid`,
   - scalar mesh as `fluid+solid`,
   - fluid/solid thermal properties.
4. Implement `cht.udf`:
   - initialize velocity + temperature on separate mesh objects,
   - set thermal Dirichlet BCs for solid top/bottom,
   - add buoyancy source.
5. Run and inspect checkpointed fields.

## What this tutorial teaches
- CHT boundary-ID ordering discipline.
- Multi-mesh initialization patterns.
- Coupled thermal physics setup with explicit source terms.

## Related notes
- [[Meshing & Mesh Workflows]]
- [[Physical Properties]]
- [[Boundary Conditions]]
