# Models & Source Terms

## Included model categories
- LES regularization/filtering controls via `.par`.
- RANS (`k-tau`, `k-tau SST`) via `RANSktau` setup in `.udf`.
- Low-Mach model via `lowMach` setup and thermodynamic coupling arrays.

## Typical setup pattern
1. Declare needed scalars in `.par`.
2. Include model headers in `.udf`.
3. Initialize model in `UDF_Setup`.
4. Provide `uservp` / `userq` / custom kernels as needed.
5. Enforce model-specific boundary conditions in OKL callbacks.

## Custom source terms
- Momentum explicit source: assign `nrs->userSource`, fill `o_EXT`.
- Implicit linearized source: set `userImplicitLinearTerm`.
- Scalar source: fill `nrs->scalar->o_EXT` with per-scalar offsets.

## Theory links
- [[Theory - RANS Models]]
- [[Theory - Low-Mach Model]]
- [[Theory - Incompressible & Thermal Equations]]
