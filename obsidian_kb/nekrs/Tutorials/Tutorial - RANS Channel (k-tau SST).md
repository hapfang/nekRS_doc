# Tutorial - RANS Channel (k-tau / SST)

## Goal
Set up wall-resolved turbulent channel flow using `k-tau` / `k-tau SST` RANS models.

## Main steps
1. Create baseline channel mesh (periodic streamwise/spanwise, wall/symmetry wall-normal).
2. Configure `.par` with scalars `k` and `tau` and variable-viscosity equation mode.
3. In `.udf`:
   - initialize RANS plugin,
   - set field ICs (`u`, `k`, `tau`),
   - apply wall BCs for `k` and `tau`,
   - optionally remap wall-normal mesh spacing.
4. Run and compare statistics/profile outputs.

## What this tutorial teaches
- Plugin-based turbulence model activation.
- Additional scalar-field lifecycle in nekRS.
- Case-specific postprocessing (profiles, friction velocity).

## Related notes
- [[Models & Source Terms]]
- [[Theory - RANS Models]]
- [[Postprocessing & Data Output]]
