# Case Files (.par/.udf/.re2/.usr/.sess/.upd)

## Minimum required files
- `.par`: simulation and solver parameters
- `.udf`: user customization hooks
- `.re2`: hexahedral mesh + boundary tags

## Optional files
- `.oudf`: separated OKL kernels
- `.usr`: legacy Nek5000 Fortran routines
- `.sess`: NekNek multi-domain session file
- `nekrs.upd`: runtime trigger updates

## `.par` anatomy (mental model)
- `[GENERAL]`: global controls and scalar names
- `[MESH]`: mesh/boundary ID maps
- `[FLUID VELOCITY]`, `[FLUID PRESSURE]`, `[SCALAR ...]`: field solver and physics settings
- User sections via `userSections` for case constants

## `.udf` anatomy (most-used callbacks)
- `UDF_Setup0`: pre-state setup, option extraction
- `UDF_LoadKernels`: define device macros
- `UDF_Setup`: initialize fields, hooks, custom routines
- `UDF_ExecuteStep`: per-step postprocessing/control logic
- `#ifdef __okl__`: boundary kernels and custom device kernels

## Important cross-links
- BC implementation: [[Boundary Conditions]]
- IC implementation: [[Initial Conditions]]
- Runtime controls and signals: [[Running & Runtime Controls]]
- Legacy bridge patterns: [[Debugging Guide]]
