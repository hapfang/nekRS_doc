# Quickstart & Environment Setup

## Objective
Install nekRS, set runtime environment, and run a first example.

## Requirements (high level)
- Linux/macOS shell familiarity
- C/C++/Fortran compilers + MPI + CMake
- Optional GPU backend (CUDA/HIP/SYCL), CPU-only also possible

## Installation flow
1. Get source (release tarball or git clone).
2. Build with `build.sh` (optional `CC/CXX/FC` and CMake flags).
3. Set `NEKRS_HOME` and prepend `$NEKRS_HOME/bin` to `PATH`.
4. Run a sample case with `nrsmpi` or `nrsbmpi`.

## Operational checks
- `nrsman env` for environment variables.
- `nrsman par` for `.par` key reference.

## Link-outs
- Setup details: [[Problem Setup MOC]]
- Running options: [[Running & Runtime Controls]]
- First tutorial: [[Tutorial - Fully Developed Laminar Flow (FDLF)]]
