# Debugging Guide

## Common debug levers
- JIT precompile with build-only mode before launching full runs.
- Increase verbosity/debug level in `.par`.
- Inspect device arrays (min/max/norm) and checkpoint dumps.

## Kernel development workflow
- Start with serial backend for easier print-based debugging.
- Flush output explicitly for deterministic logs.
- Add synchronization/barriers carefully during diagnosis.

## gdb workflows
- Attach to a running rank by PID.
- Use attach mode to pause at startup, then continue once breakpoints are set.
- Launch directly under `gdb` for all-rank debugging when needed.

## Related notes
- [[Running & Runtime Controls]]
- [[Case Files - par udf re2 usr sess upd]]
