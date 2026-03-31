# Meshing & Mesh Workflows

## Core facts
- nekRS consumes binary `.re2` meshes (hex-based workflows).
- Boundary conditions depend on side-set/tag consistency.
- Connectivity is reconstructed and can depend on tolerance settings.

## Mesh creation paths
- Native Nek5000 tools (`genbox`, `n2to3`, `reatore2`).
- External conversion:
  - `gmsh2nek` for Gmsh `.msh`
  - `exo2nek` for Exodus `.exo` (with supported conversions)

## Special workflows
- CHT: fluid + solid meshes and boundary map discipline.
- Moving mesh: ALE options and coordinate updates.
- h-refinement: on-the-fly global refinement and restart-aware schedules.

## Boundary-tag guidance
- Prefer numeric boundary IDs with `boundaryIDMap` + `boundaryTypeMap`.
- Keep ordering synchronized between IDs and BC type lists.
- Use `none` where faces are periodic/internal and not explicitly imposed.

## Related notes
- [[Boundary Conditions]]
- [[Case Files - par udf re2 usr sess upd]]
- [[Tutorial - Conjugate Heat Transfer (CHT)]]
