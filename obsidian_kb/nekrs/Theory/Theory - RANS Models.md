# Theory - RANS Models

## Implemented family in docs
The documentation emphasizes `k-tau` and `k-tau SST` closures and their integration through plugin setup routines.

## Setup implications
- Requires additional scalar equations (`k`, `tau`).
- Momentum diffusion uses eddy-viscosity contribution.
- Source terms and property updates happen per step through model routines.
- Wall BCs for `k` and `tau` are zero-valued in the standard wall-resolved tutorial.

## Practical links
- [[Tutorial - RANS Channel (k-tau SST)]]
- [[Models & Source Terms]]
- [[Postprocessing & Data Output]]
