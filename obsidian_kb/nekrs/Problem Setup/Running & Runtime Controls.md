# Running & Runtime Controls

## Core commands
- Foreground MPI run: `nrsmpi <case> <nranks>`
- Background run with logfile handling: `nrsbmpi <case> <nranks>`
- Native executable path supports additional CLI options.

## HPC scripts
- Additional launcher scripts can be installed into `$NEKRS_HOME/bin`.
- Platform-specific wrappers are available through `nekRS_HPCsupport`.

## Runtime signal controls
nekRS supports signal-triggered runtime actions via environment variables (see `nrsman env`), including:
- backtrace triggers,
- controlled shutdown/update hooks,
- reload patterns for trigger files (e.g., `nekrs.upd`).

## Related notes
- [[Case Files - par udf re2 usr sess upd]]
- [[Postprocessing & Data Output]]
- [[Debugging Guide]]
