# Physical Properties

## Constant properties
Set in field cards:
- Fluid density/viscosity in `[FLUID VELOCITY]`
- Scalar transport/diffusion coefficients in `[SCALAR ...]`

## Variable properties
Use `variableProperties = true` and implement `uservp` in `.udf`.
Populate contiguous property arrays (or helper kernels) for fluid/scalars.

## CHT-specific property setup
For conjugate heat transfer:
- Set scalar mesh to `fluid+solid`.
- Provide both fluid and solid coefficients (`transportCoeffSolid`, `diffusionCoeffSolid`).
- Keep indexing consistent with element phase and scalar layout.

## Related notes
- [[Models & Source Terms]]
- [[Tutorial - Conjugate Heat Transfer (CHT)]]
- [[Theory - Low-Mach Model]]
