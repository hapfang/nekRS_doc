> Source: `source/problem_setup/debugging.rst` (converted to markdown; retains documentation detail)

# Debugging Tips

This page contains a few topics that, although aren't necessary to know about to
run nekRS be default, can be the source of issues while trying to get nekRS
running in more complex environments or while making multiple changes.

## Just-in-time Compilation

nekRS uses just-in-time compilation to build the functions in the ``.udf`` and ``.oudf``
case files, as well as for compiling certain fixed-size arrays based on the order of
the polynomial approximation or other problem settings.

For most cases, no special actions need to be taken by the user for this
process to work correctly. However, a high-level understanding of the just-in-time
compilation is useful to know what steps need to be taken to fully clear the cached
build files, as well as how to perform the pre-compilation separately from a full run
to obtain more accurate runtime measurements.

When nekRS performs just-in-time compilation, object files are created in the
``.cache`` directory within the current case directory. To completely clear the
cached state of the case, simply delete the ``.cache`` directory:

```
```

  user$ rm -rf .cache/

> **Tip:**
> If you experience strange behavior when running your case during the precompilation
> step (such as failures to build in COMMON blocks or other parts of the code that you
> are not touching in the ``.udf`` and ``.oudf`` files), try deleting the ``.cache``
> directory and trying again. It is not uncommon for the precompilation process to miss
> the need to build new versions of object files if you are making frequent changes to
> the nekRS source. This is also sometimes encountered if you are using multiple nekRS
> versions in different projects (such as standalone nekRS or nekRS wrapped within
> a multiphysics coupling application such as :term:`ENRICO`), but don't have your
> environment completely self-consistent.

The precompilation process usually takes on the order of one minute. Depending on
the use case, it may be advantageous to force the precompilation separately from the run itself.
To precompile the case, use the ``--build-only`` option. See the

As an example, the following commands first precompile a case named
``my_case`` for a later run with at least 4 GPUs. After the precompilation
step, you can run as usual with the ``nrsmpi`` script; the precompiled case
will be reused and the build step skipped:

```bash
```

  # precompilation
  user$ nrsmpi my_case 4 --build-only 4

  # actual run
  user$ nrsmpi my_case 32

## Verbose and Debug Mode

In the ``.par`` file, you can enable verbose output by setting
``verbose = true`` in the ``[GENERAL]`` section. This prints additional
informational messages during the run, such as detailed option settings and
norm/residual information from the linear solvers.

*NekRS* also supports a ``--debug`` mode (see Run NekRS<running>), which
enables verbose output and activates extra runtime checks, including additional
floating-point exception trapping (via ``FE_ALL_EXCEPT``). This can help detect
NaNs, overflows, and other numerical issues early in the run.

## Check Device Arrays

You can quickly inspect device data by printing minima, maxima, and norms using

Alternatively, you can dump a field to a checkpoint file with
preferred post-processing tool.

## Kernel in Serial Mode

When developing a new kernel, it is often useful to test it with the CPU
backend using the command-line option ``--backend serial``. In this mode,
OCCA translates the OKL kernel to C++, so you can use standard host I/O
(e.g., ``printf`` or ``std::cout``) inside the kernel for debugging. The only
extra step is to include the C/C++ I/O headers inside the same ``__okl__``
block so that the translated C++ code can see them:

```cpp
#ifdef __okl__
#include <cstdio>
#include <iostream>

@kernel void fillProp(const dlong Nelements,
                      const dfloat Re,
                      const dfloat Pe,
                      @ restrict dfloat* MUE,
                      @ restrict dfloat* RHO,
                      @ restrict dfloat* K,
                      @ restrict dfloat* RHOCP)
{
  for (dlong e = 0; e < Nelements; ++e; @outer(0)) {
    for (int n = 0; n < p_Np; ++n; @inner(0)) {
      const int id = e * p_Np + n;

      MUE[id]   = 1.0 / Re;
      K[id]     = 1.0 / Pe;

      RHO[id]   = 1.0;
      RHOCP[id] = 1.0;

      printf("debug: id = %d, RHO = %g\n", id, RHO[id]);
      fflush(stdout);

      std::cout << "debug: id = " << id
                << ", MUE = " << MUE[id] << std::endl;

    }
  }
}
#endif
```

Explicitly flushing (``fflush(stdout)`` and using ``std::endl``) helps ensure
that debug output appears immediately, which is useful when diagnosing hangs
or crashes.

## Synchronization

In parallel programming, it is sometimes helpful to insert explicit barriers to
isolate or block sections of code while debugging.

- In legacy *Nek5000*, you can force a global synchronization in ``userchk``
  using ``nekgsync()``:

```fortran
  call nekgsync()
```

- In a *NekRS* ``udf`` file, you can insert an MPI barrier explicitly:

```cpp
  MPI_Barrier(platform->comm.mpiComm());
```

- For host–device synchronization, you can ensure all device work is completed
  after a kernel launch by synchronizing the device:

```cpp
  platform->device.finish();
```

  For CUDA or HIP backends, setting the following environment variables forces
  all kernel launches to be synchronous (mainly useful for debugging, as it can
  significantly reduce performance):

```bash
  export CUDA_LAUNCH_BLOCKING=1
  export HIP_LAUNCH_BLOCKING=1
```

## Using ``gdb``

Using a debugger helps locate segmentation faults, set breakpoints, detect
problematic conditions, and obtain backtraces. Here we demonstrate some
typical workflows with `gdb <https://www.sourceware.org/gdb/>`__. Other
debugging tools can be used in a similar manner.

### Attaching to a Process

After starting *NekRS*, you can attach ``gdb`` to the running process.
First, find the process ID (PID), for example using ``top`` or ``ps``:

```bash
top
# or
ps aux | grep nekrs
```

Then attach ``gdb`` to the PID:

```bash
gdb -p <PID>
```

Once attached, you can obtain a backtrace:

```console
(gdb) bt
```

This is particularly useful when the code appears to hang and you still have
a chance to inspect where a process is stopped. Note that, because the code
runs in parallel, it is common for some ranks to be waiting (e.g., in an MPI
call) for a problematic rank to reach a synchronization point. Attaching to a
single process may therefore only show that it is blocked in MPI, rather than
the original source location of the error.

You can also inspect variables, move between stack frames, monitor threads,
set breakpoints, and so on. See the `gdb <https://www.sourceware.org/gdb/>`__
documentation for details.

> **Tip:**
> On modern Linux, ``gdb -p`` can be blocked by ptrace policy (ptrace_scope),
> so sometimes you need sudo.

### Attach Mode

For an alternative workflow that attaches *NekRS* right at startup, you can use
the attach mode by Run NekRS<running> with the ``--attach`` option.
This pauses *NekRS* and prints the PID of each rank. For example:

```none
$HOME/bin/nrsmpi ethier.par 8 --backend serial --attach

rank 0 on pop-os: pid<3255824>
Attach debugger, then send <SIGCONT> to rank0 to continue
rank 1 on pop-os: pid<3255825>
rank 2 on pop-os: pid<3255826>
rank 3 on pop-os: pid<3255827>
rank 4 on pop-os: pid<3255828>
rank 5 on pop-os: pid<3255829>
rank 6 on pop-os: pid<3255830>
rank 7 on pop-os: pid<3255831>
```

In another terminal, attach ``gdb`` to rank 0, set breakpoints, and then
continue execution:

```bash
gdb -p 3255824
```

```console
(gdb) break some_function   # optional
(gdb) continue
```

```bash
kill -CONT 3255824
```

Finally, from a third terminal (or after detaching from ``gdb``), send
``SIGCONT`` to rank 0 so that *NekRS* leaves the pause state and proceeds under
debugger control:

```bash
kill -CONT 3255824
```

This allows you to attach ``gdb`` right from the beginning of the run, which is
especially useful when the breakpoint is in the early setup stage.

### Launch NekRS under ``gdb``

You can also run *NekRS* directly under ``gdb`` on all ranks. For example, the
following command starts *NekRS* under ``gdb`` and prints a backtrace when an
error is detected:

```bash
mpirun -np 8 gdb -ex "run" -ex "bt" --args $NEKRS_HOME/bin/nekrs --setup ethier.par
```

> **Note:**
> This makes every process print its backtrace, which can be overwhelming.
> You can manually configure ``gdb`` logging (for example, directing each
> rank’s output to a separate file), but in many cases it is simpler to use
> ``NEKRS_SIGNUM_BACKTRACE``. See :ref:`nekrs_signal`.

> **Tip:**
> *NekRS* monitors OCCA errors and assertions and will print an OCCA error
> message and abort when a fatal condition is detected (e.g., ``OUT_OF_MEMORY``,
> ``DEVICE_NOT_FOUND``). When this happens, the code may terminate via
> ``MPI_Abort`` or an internal error handler, so a simple backtrace on exit is
> not always enough to see where the error originated.
>
> To catch such failures more reliably, you can set breakpoints on common error
> paths before running:
>
> .. code-block:: bash
>
>    mpirun -np 8 gdb \
>      -ex "break __cxa_throw" \
>      -ex "catch catch" \
>      -ex "break 'occa::exception'" \
>      -ex "break 'occa::error'" \
>      -ex "break 'occa::memory::assertInitialized'" \
>      -ex "break MPI_Abort" \
>      -ex "run" \
>      --args $NEKRS_HOME/bin/nekrs --setup ethier.par
>
> With these breakpoints in place, ``gdb`` will stop as soon as one of these
> error paths is triggered, allowing you to inspect the call stack and local
> variables at the point where the failure occurs.

## Related knowledge-base notes
- [[Running & Runtime Controls]]
