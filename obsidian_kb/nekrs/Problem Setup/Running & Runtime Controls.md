> Source: `source/problem_setup/running.rst` (converted to markdown; retains documentation detail)

# Running

This page gives information on how to run *NekRS* after it has been installed (see Installation instructions ) and appropriate input files have been generated 
(see case).

## Native Running

To run *NekRS* natively from the directory containing the case, use the following general command

```bash
 mpirun -np <number of MPI tasks> nekrs --setup <par|sess file> [options]
```

Below are the command line options/argument that can be used to further modifiy how *NekRS* is run.

.. _tab:runoptions:

   ``--help``,"None |br| ``par`` |br| ``env``", "Prints summary of available command line arguments |br| Prints summary of Parameter file sections and keys |br| Prints list of *NekRS* enviorenment variables","No"
   ``--setup``,``par`` file |br| ``sess`` file, "Name of ``.par`` file of the case |br| Name of ``.sess`` file (NekNek)","Yes"
   ``--build-only``,"None |br| ``#procs``","Runs JIT compilation for ``np`` procs |br| Runs JIT compilation for specified number of processors","No"
   ``--cimode``,"``<id>``","Runs *NekRS* in CI mode corresponding to specified ``<id>``","No"
   ``--debug``,"None", Run *NekRS* in debug mode |br| See also debug_verbose,"No"
   ``--attach``,"None", "Pause *NekRS* at startup to allow attaching gdb for interactive debugging |br| See debugging with attach mode<debug_gdb_attach_mode> for details","No"
   ``--backend``,"``CPU`` |br| ``CUDA`` |br| ``HIP`` |br| ``DPCPP`` |br| ``OPENCL``","Manually set the backend device for running","No"
   ``--output``,"``<local-path-to-logfile>``",Name of log file to print *NekRS* output,"No"
   ``--device-id``,"``<id>`` |br| ``LOCAL-RANK``",Manually set OCCA device ID for GPU |br| For CPU,"No"

### MPI launch scripts

A number of scripts ship with *NekRS* itself and are located in the ``$NEKRS_HOME/bin``. A brief summary of these scripts and their usage is as follows.

* ``nrsmpi <casename> <processes>``: run *NekRS* in parallel with ``<processes>`` parallel processes for the case files that are prefixed with ``casename``.
* ``nrsbmpi <casename> <processes>``: same as ``nrsmpi``, except that nekRS runs in the background and the output is printed to log file

## Running on HPC Systems

Specific scripts and instructions for running on specific HPC systems can be found in `nekRS_HPCsupport Github repository <https://github.com/Nek5000/nekRS_HPCsupport>`_.
After installing `nekRS`, clone this repository for the latest scripts:

```bash
git clone https://github.com/Nek5000/nekRS_HPCsupport
cd nekRS_HPCsupport
```

With ``NEKRS_HOME`` pointing to your nekRS installation, run:

```bash
./install.sh
```

The HPC scripts will be installed into ``$NEKRS_HOME/bin`` with a lower case machine-name suffix.
For example, ``$NEKRS_HOME/bin/nrsqsub_frontier``.
Typical usage is as follows:

```bash
PROJ_ID=<project> ./nrsqsub_frontier <case> <#nodes> <hh:mm>
```

- ``PROJ_ID``: your allocation/account
- ``case``: case name, `.par` or `.sess` file
- ``#nodes``: number of nodes
- ``hh:mm``: walltime (hours:minutes).

For example, this requests to run 1 node with 30 mins on Frontier:

```bash
PROJ_ID=ABC123 ./nrsqsub_frontier turbPipe.par 1 00:30
```

You can see the full list of environment variables via ``--help``:

```none
$ ./nrsqsub_frontier --help

Usage: [<env-var>] $0 <par or sess file> <number of compute nodes> <hh:mm>

env-var                 Values(s)   Description / Comment
------------------------------------------------------------------------------------
NEKRS_HOME              string      path to installation
PROJ_ID                 string
QUEUE                   string
CPUONLY                 0/1         backend=serial
RUN_ONLY                0/1         skip pre-compilation
BUILD_ONLY              0/1         run pre-compilation only
FP32                    0/1         run solver in single precision
OPT_ARGS                string      optional arguements e.g. "--cimode 1 --debug"
```

## Runtime Control with Signals

A signal is a small, asynchronous message the OS sends to a process to request an action.
Examples include ``SIGTERM`` (polite terminate), ``SIGSTOP`` (suspend), and ``SIGKILL`` (force kill).
*NekRS* installs handlers for specific signals (e.g., ``SIGUSR1``/``SIGUSR2``) to perform runtime tasks.
You can map which signals trigger which actions via environment variables (see ``nrsman env``):

   "``NEKRS_SIGNUM_BACKTRACE``","<int>","Signal number to trigger stack trace"
   "``NEKRS_SIGNUM_TERM``","<int>","Signal number to exit gracefully. |br| Treat the *next* timestep as the final step. If ``checkpointInterval`` > 0 in ``.par``, write a checkpoint"
   "``NEKRS_SIGNUM_UPD``","<int>","Signal number to process update (trigger) file"

> **Note:**
> Systems differ slightly. 
> Check which numbers correspond to which signals on your terminal with ``kill -l``, ``kill -l SIGUSR2``, or the manuals (``man 7 signal``).
> We suggest using catchable signals like ``SIGUSR1``/``SIGUSR2``.

Suppose ``SIGUSR2`` is 12 on your system.
Export the mapping before running *NekRS* with ``export NEKRS_SIGNUM_BACKTRACE=12``.
Then, at runtime, send the signal to produce per-rank backtraces (useful when a simulation hangs):

```bash
# by pid (process id, from "top" or "ps aux")
kill -12 <pid>

# by name
pkill -12 nekrs
pkill -USR2 nekrs

# On HPC using slurm scheduler
scancel -s USR2 <job id>
```

## Related knowledge-base notes
- [[Case Files - par udf re2 usr sess upd]]
- [[Debugging Guide]]
