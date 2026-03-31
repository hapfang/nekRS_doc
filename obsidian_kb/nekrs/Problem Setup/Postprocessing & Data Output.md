# Postprocessing & Data Output

## Output fundamentals
- Standard checkpoints: `<case>0.fXXXXX` + metadata for visualization.
- `nrsvis` can regenerate metadata for Visit/ParaView when needed.

## Advanced output patterns
- Append custom fields to checkpoints.
- Write dedicated custom output files (io factory workflow).
- Element filtering to reduce I/O volume.
- ADIOS2 `.bp/` output for parallel data workflows.

## Derived quantities
- Array operations and norms.
- Spatial derivatives/integrals.
- Strain/rotation metrics.
- Aero-force extraction.
- Q-criterion.
- Time and planar averaging.

## Related notes
- [[Running & Runtime Controls]]
- [[Debugging Guide]]
- [[Tutorial - RANS Channel (k-tau SST)|Tutorial - RANS Channel (k-tau / SST)]]
