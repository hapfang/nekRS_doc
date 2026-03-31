# Theory - Low-Mach Model

## Concept
Low-Mach formulation filters acoustic dynamics while retaining compressibility effects needed for thermal/density coupling.

## Setup implications in nekRS
- Activate low-Mach routines in `.udf`.
- Provide EOS-related coefficients/arrays (`beta`, `kappa`) and property updates.
- Include divergence and thermodynamic-pressure derivative contributions when required (e.g., closed systems).

## Related implementation notes
- [[Models & Source Terms]]
- [[Physical Properties]]
