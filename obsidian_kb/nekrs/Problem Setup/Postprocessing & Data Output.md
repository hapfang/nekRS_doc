> Source: `source/problem_setup/postprocessing.rst` (converted to markdown; retains documentation detail)

# Postprocessing

Once a case has been setup correctly so it can be run without errors, you may want
to modify the postprocessing to of the simulation output. By default, NekRS will
output a basic set of data according to frequency set in the ``writeInterval`` of
the par_file which can subsequently be viewed through a visualization
tool such as Paraview or Visit. However, additional data or derived values can
be extracted by setting up User Defined outputs using the ``UDF_ExecuteStep``
function of the ``.udf`` file.

> **Tip:**
> Most postprocessing routines can be placed in ``UDF_ExecuteStep``.  Unless
> they must be updated every timestep, it is recommended to evaluate them only
> occasionally to save computational resources.
>
> The following flags and counters are often used in ``if``-statements to
> control when postprocessing is executed:
>
> .. code-block:: cpp
>
>    // Every 100 timesteps
>    if (tstep % 100 == 0)
>
>    // For outer iterations (e.g., neknek); only when the outer step converged
>    if (nrs->timeStepConverged)
>
>    // Only on timesteps when checkpoints are written
>    if (nrs->checkpointStep)
>
>    // Only on the final timestep
>    if (nrs->lastStep)
>
>    // Only on rank 0 to reduce printing
>    if (platform->comm.mpiRank() == 0)

## Checkpointing & Visualization

Standard *NekRS* field output files have the form ``<case>0.fXXXXX``, where
``<case>`` is the case name and ``XXXXX`` is a five-digit zero-based index for
the output file. Each file corresponds to a single output time step and is
written according to the ``checkPointControl`` and ``checkPointInterval`` in the
``.par`` file (See tab:generalparams).
These files use in binary ``nek`` format that requires a header file,
``<case>.nek5000``, to be viewed in ParaView (or similar tools). This header is
generated automatically, but it can also be created manually with the
``nrsvis`` script.

*NekRS* can also write *ADIOS2* BP5 output by setting
``checkPointEngine = adios``. See postproc_adios for details.

> **Note:**
> To open ``<case>.nek5000`` in ParaView (5.12+), you can use either
> "VisIt Nek5000 Reader" or "Nek5000 Reader". The latter provides improved
> parallel I/O performance, while the former has better support for Nek5000
> features and is recommended for moving-mesh cases or 64-bit ``nek`` files.

### Adding Custom Fields

To append extra fields to the checkpoint output, register them with
``nrs->addUserCheckpointField``. This call associates a user-defined field name
with a list of device buffers:

```cpp
// examples/turbPipe/turbPipe.udf
nrs->addUserCheckpointField("scalar01", std::vector<deviceMemory<dfloat>>{o_nuAVM});
```

Other than ``checkPointInterval``, you can also trigger a default checkpoint
manually in ``UDF_ExecuteStep`` using

```cpp
nrs->writeCheckpoint(time);
```

> **Note:**
> In the ``nek`` format, field names are fixed: ``velocity``, ``pressure``,
> and ``scalarXX``. Fields must be either three-component vectors (e.g.,
> velocity) or scalars. For ADIOS2 output (``checkPointEngine = adios``),
> arbitrary field names are allowed, including vector fields.

> **Tip:**
> In the ``.par`` file, the option ``checkpointing = true`` controls whether a
> given field is written to the checkpoint files. Combined with
> ``solver = none``, this allows you to reserve fields that are written only
> for post-processing.

### Adding Custom Output File

A more flexible approach is to write all desired fields to a separate output
file. This lets you keep the default checkpoint files for restart, while using
the additional file for post-processing or other quantities of interest.
This functionality is provided by the ``iofldFactory`` class
(``src/core/iofld/iofldFactory.cpp``).

The example below writes the density field to ``scalar00`` (which corresponds to
``temperature`` in the ``nek`` format). The setup is done in ``UDF_Setup``,
where we choose the ``nek`` engine, specify the output file name (``density``),
single precision, and request interpolation to a uniformly spaced grid of order
controls whether the mesh is written to every file; if it is ``false``, only the
first file contains the mesh to reduce storage. In ``UDF_ExecuteStep``, we then
write the output at time step :math:`1000`.

```cpp
// UDF global variable
std::unique_ptr<iofld> io;

// UDF_Setup
auto mesh = nrs->meshV;
io = iofldFactory::create("nek");              // "nek" (default) or "adios"
io->open(mesh, iofld::mode::write, "density");
io->writeAttribute("precision", "32");         // "32" (default) or "64"
io->writeAttribute("uniform", "true");         // default = false
io->writeAttribute("polynomialOrder", std::to_string(mesh->N + 2)); // default = mesh->N
io->writeAttribute("outputMesh", "false");     // default = false

io->addVariable("scalar00", std::vector<deviceMemory<dfloat>>{nrs->o_rho});

// UDF_ExecuteStep
if (tstep == 1000) {
  io->addVariable("time", time);
  io->process();
}
if (nrs->lastStep) io->close();
```

### Element Filter

Often, post-processing does not require all elements. To reduce storage and I/O
cost, you can write only a selected subset of elements using an
``elementFilter``. You simply provide the local element indices to be written in
a ``std::vector<int>``. In ``examples/turbPipe/turbPipe.udf``, for example, only
elements that intersect or lie above ``zRecycLayer`` are written. Similarly, you
might select elements that contain geometric objects of interest for
visualization, or elements where a locally computed CFL number exceeds
a given threshold.

```cpp
:emphasize-lines: 13

auto elementFilter = [&]()
{
  std::vector<int> elementFilter;
  for(int e = 0; e < mesh->Nelements; e++) {
     auto zmaxLocal = std::numeric_limits<dfloat>::lowest();
     for(int i = 0; i < mesh->Np; i++) {
       zmaxLocal = std::max(z[i + e * mesh->Np], zmaxLocal);
     }
     if (zmaxLocal > zRecycLayer) elementFilter.push_back(e);
  }
  return elementFilter;
}();
io->writeElementFilter(elementFilter);
```

## Compute Derived Quantity

Additional control of the simulation to compute additional/derived quantities
or output custom fields can be achieved by utilising the ``UDF_ExecuteStep``
function of the ``.udf`` file. Here we demonstrate how this can be used to
compute a derived quantity.

Common operators are already provided in opsem and linalg. In
addition, helper routines such as ``nrs->strainRate``,
``nrs->strainRotationRate``, ``nrs->aeroForces`` and ``nrs->Qcriterion``
are available. Through the legacy ``.usr`` file, you can also reuse existing
*Nek5000* subroutines and post-processing routines. As a starting point, it is
recommended to combine these existing APIs before implementing your own

### Array Operators

This section demonstrates common array operations. For the full list of
available routines, see linalg. In the examples below, ``N`` is the
length of the local arrays, ``a`` and ``b`` are constants, and ``o_x``, ``o_y``,
and ``o_z`` are OCCA device arrays.

- Fill

```cpp
  // o_u[i] = a
  platform->linAlg->fill(N, a, o_u);
```

- Scale and addition

```cpp
  // o_x[i] = a*o_x[i]
  platform->linAlg->scale(N, a, o_x);

  // o_y[i] = a*o_x[i] + b*o_y[i]
  platform->linAlg->axpby(N, a, o_x, b, o_y);

  // o_z[i] = a*o_x[i] + b*o_y[i]
  platform->linAlg->axpbyz(N, a, o_x, b, o_y, o_z);
```

- Multiply and division

```cpp
  // o_y[i] = a * o_x[i] * o_y[i]
  platform->linAlg->axmy(N, a, o_y);

  // o_z[i] = a / o_y[i]
  platform->linAlg->adyz(N, a, o_y, o_z);

  // o_z[i] = a * o_x[n] / o_y[i]
  platform->linAlg->axdyz(N, a, o_x, o_y, o_z);
```

- Reduction (min, max and sum)

```cpp
  // sum_i x[i]
  auto s = platform->linAlg->sum(N, o_x, platform->comm.mpiComm);

  // helper function to print min/max of an OCCA array
  auto printMinMax = [&](std::string tag, const occa::memory& o_u)
  {
    if (o_u.isInitialized()) {
      const auto N = o_u.length();
      const auto umin = platform->linAlg->min(N, o_u, platform->comm.mpiComm);
      const auto umax = platform->linAlg->max(N, o_u, platform->comm.mpiComm);
      if (platform->comm.mpiRank == 0) {
        printf("chk min/max %d %2.4e %6s %13.6e %13.6e\n", tstep, time, tag.c_str(), umin, umax);
      }
    }
  };

  printMinMax("UX", nrs->o_U.slice(0 * nrs->fieldOffset, mesh->Nlocal));
  printMinMax("UY", nrs->o_U.slice(1 * nrs->fieldOffset, mesh->Nlocal));
  printMinMax("UZ", nrs->o_U.slice(2 * nrs->fieldOffset, mesh->Nlocal));
  printMinMax("PR", nrs->o_P.slice(0 * nrs->fieldOffset, mesh->Nlocal));
```

- Inner product, outer product and norms

```cpp
  // sum_i (o_x[i] * o_y[i])
  auto i1 = platform->linAlg->innerProd(N, o_x, o_y, platform->comm.mpiComm);

  // sum_i (o_w[i] * o_x[i] * o_y[i])
  auto i2 = platform->linAlg->weightedInnerProd(N, o_w, o_x, o_y, platform->comm.mpiComm);

  // o_vec3 = o_vec1 cross o_vec2, o_vecX are of size 3 * fieldOffset.
  platform->linAlg->crossProduct(N, nrs->fieldOffset, o_vec1, o_vec2, o_vec3);

  // sum_i | o_x[i] |
  auto n1 = platform->linAlg->norm1(N, o_x, platform->comm.mpiComm);

  // sqrt( sum_i (o_x[i] * o_x[i]) )
  auto n2 = platform->linAlg->norm2(N, o_x, platform->comm.mpiComm);

  // sqrt( sum_i (o_w[i] * o_x[i] * o_x[i]) )
  auto wn2 = platform->linAlg->weightedNorm2(N, o_w, o_x, platform->comm.mpiComm);
```

### Spatial Derivatives

First-order derivatives of a scalar (e.g., ``temperature``) can be computed via
``opSEM::strongGradVec``. For other differential operators, see opsem.

```cpp
auto mesh = scalar->mesh("temperature");
auto o_S = nrs->scalar->o_solution("temperature");
auto o_gradS = opSEM::strongGrad(mesh, nrs->scalar->fieldOffset(), o_S);
```

### Spatial Integrals

- Volumetric integral

  The spectral element method uses Gauss quadrature for numerical integration.
  With mass lumping, the mass matrix is diagonal, and *nekRS* stores the
  diagonal entries (including the Jacobian scaling) in ``mesh->o_Jw`` (or
  ``mesh->o_LMM``). Volume integrals can then be computed via an inner product.
  The example below computes the domain average of :math:`v_z` :

```cpp
  auto mesh = nrs->meshV;
  auto o_UZ = nrs->fluid->o_U + 2 * nrs->fluid->fieldOffset;
  const dfloat ubar =
        platform->linAlg->innerProd(mesh->Nlocal,
                                    o_UZ,
                                    mesh->o_Jw,
                                    platform->comm.mpiComm())
        / mesh->volume;
```

- Surface integral

  Surface integrals over selected boundary IDs can be computed using the mesh
  helper functions. In this example, we first compute the surface area by
  integrating :math:`1` over the surface, and then compute the flux
  vector :math:`{\vec n}` is oriented outward from the domain.

```cpp
  // tag surface boundary ID(s)
  std::vector<int> bIdWall{2};

  deviceMemory<int> o_bid(bIdWall.size());
  o_bid.copyFrom(bIdWall.data());

  // compute surfArea = int_surface 1 ds
  poolDeviceMemory<dfloat> o_one(mesh->Nlocal);
  platform->linAlg->fill(o_one.size(), 1.0, o_one);
  dfloat surfArea = mesh->surfaceAreaMultiplyIntegrate(o_bid, o_one);

  // compute Uflux = int_surface (U dot n) ds
  dfloat Uflux =
    mesh->surfaceAreaNormalMultiplyVectorIntegrate(nrs->fluid->fieldOffset,
                                                   o_bid,
                                                   nrs->fluid->o_U);

  dfloat avgUflux = Uflux / surfArea;
```

### Strain \& Rotation Rate

The first-order derivatives of the velocity field :math:`\vec u = (u_x, u_y, u_z)`
can be collected into the velocity gradient tensor

```math
 \nabla \vec{u} =
    \begin{pmatrix}
       \dfrac{\partial u_x}{\partial x} & \dfrac{\partial u_x}{\partial y} & \dfrac{\partial u_x}{\partial z} \\
       \dfrac{\partial u_y}{\partial x} & \dfrac{\partial u_y}{\partial y} & \dfrac{\partial u_y}{\partial z} \\
       \dfrac{\partial u_z}{\partial x} & \dfrac{\partial u_z}{\partial y} & \dfrac{\partial u_z}{\partial z}
    \end{pmatrix}.

```

The strain-rate tensor :math:`\boldsymbol{\varepsilon}` and the rotation-rate
tensor :math:`\boldsymbol{\omega}` are the symmetric and skew-symmetric parts of

```math
 \boldsymbol{\varepsilon}
 = \frac{1}{2}\left( \nabla \vec{u} + (\nabla \vec{u})^{\mathsf{T}} \right),
 \qquad
 \boldsymbol{\omega}
 = \frac{1}{2}\left( \nabla \vec{u} - (\nabla \vec{u})^{\mathsf{T}} \right).

```

In index notation this reads

```math
 \varepsilon_{ij} = \frac{1}{2}\left(\partial_i u_j + \partial_j u_i\right),
 \qquad
 \omega_{ij} = \frac{1}{2}\left(\partial_i u_j - \partial_j u_i\right).

```

Because of symmetry and skew-symmetry, only the upper-triangular entries of
independent components, respectively. To compute the strain rate, or both strain
rate and rotation rate, based on the current velocity, use

```cpp
// Strain-rate tensor: 6 components
// SO[id + 0 * offset] = 0.5 * 2 * dudx;          // ε_xx
// SO[id + 1 * offset] = 0.5 * (dudy + dvdx);     // ε_xy
// SO[id + 2 * offset] = 0.5 * (dudz + dwdx);     // ε_xz
// SO[id + 3 * offset] = 0.5 * 2 * dvdy;          // ε_yy
// SO[id + 4 * offset] = 0.5 * (dvdz + dwdy);     // ε_yz
// SO[id + 5 * offset] = 0.5 * 2 * dwdz;          // ε_zz
auto o_Sij = nrs->strainRate();

// Strain + Rotation-rate tensor: 9 components
// SO[id + 6 * offset] = 0.5 * (dvdx - dudy);     // ω_xy
// SO[id + 7 * offset] = 0.5 * (dudz - dwdx);     // ω_xz
// SO[id + 8 * offset] = 0.5 * (dwdy - dvdz);     // ω_yz
auto o_SijOij = nrs->strainRotationRate();
```

### Aero-Forces

The hydrodynamic force on a surface :math:`S` with unit normal
Cauchy stress,

```math
 \vec F = \int_S \boldsymbol{\sigma} \cdot \vec n_S \, dS,

```

where the stress tensor is decomposed as
Accordingly,

```math
 \vec F = \vec F_v + \vec F_p, \qquad
 \vec F_v = \int_S \boldsymbol{\tau} \cdot \vec n_S \, dS, \qquad
 \vec F_p = - \int_S p \, \vec n_S \, dS.

```

For a Newtonian fluid,

```math
 \tau_{ij}
 = 2 \mu \varepsilon_{ij}
   + \lambda \, (\nabla \cdot \vec{u}) \, \delta_{ij},

```

where :math:`\mu` is the dynamic viscosity, :math:`\lambda` is the bulk
viscosity, :math:`\delta_{ij}` is the Kronecker delta, and
For incompressible flow :math:`\nabla \cdot \vec{u} = 0`, so the second
term vanishes and :math:`\tau_{ij} = 2 \mu \varepsilon_{ij}`.
In the numerical implementation, the computational domain is the fluid
region outside the surface, and :math:`\vec n` denotes the outward
normal of this domain. On :math:`S` we then have

```math
 \vec F_v = - \int_S \boldsymbol{\tau}\cdot\vec n\, dS,
 \qquad
 \vec F_p = \int_S p\, \vec n\, dS.

```

The helper class ``AeroForce`` collects these contributions as Cartesian
3-vectors and exposes simple accessors for post-processing.
An instance is obtained via ``nrs->aeroForces``:

```cpp
auto mesh = nrs->meshV;

std::vector<int> bidWall{1};
auto o_bidWall = platform->device.malloc<int>(bidWall.size(), bidWall.data());

auto o_Sij   = nrs->strainRate();
auto forces  = nrs->aeroForces(o_bidWall, o_Sij);

// Each is a std::array<dfloat, 3>
auto viscousForce  = forces->viscous();
auto pressureForce = forces->pressure();
auto totalForce    = forces->total(); // pressureForce + viscousForce
```

> **Note:**
> **Drag and lift**
>
> Let :math:`\hat{\mathbf e}_D` and :math:`\hat{\mathbf e}_L` be unit vectors
> defining drag and lift directions. Their scalar values follow from projections
>
> .. math::
>
>    D = \vec F \cdot \hat{\boldsymbol{e}}_D,
>    \qquad
>    L = \vec F \cdot \hat{\boldsymbol{e}}_L,
>
> Here :math:`\vec F` can be taken as the total force, the viscous part
> :math:`\vec F_v`, or the pressure part :math:`\vec F_p`, depending on
> whether total, viscous, or pressure contributions are desired.
>
> **Drag and lift coefficients**
>
> Given a reference area :math:`A_{\text{ref}}`, a reference velocity
> (typically freestream) :math:`U_\infty` and a suitable reference density,
> the corresponding drag and lift coefficients are
>
> .. math::
>
>    C_D = \frac{D}{\tfrac{1}{2}\,\rho\,U_\infty^2\,A_{\text{ref}}},
>    \qquad
>    C_L = \frac{L}{\tfrac{1}{2}\,\rho\,U_\infty^2\,A_{\text{ref}}},

> **Tip:**
> **Skin friction and friction velocity**
>
> The viscous force can be further decomposed into its tangential component,
> represented by the skin-friction traction :math:`\boldsymbol{t}_f`. In
> continuous form,
>
> .. math::
>
>    \boldsymbol{t}_f
>      = (\boldsymbol{\tau}\cdot\vec n)
>        - \big[(\boldsymbol{\tau}\cdot\vec n)\cdot\vec n\big]\vec n,
>
> where :math:`\vec n` is the wall-normal unit vector (its sign does not matter
> for the tangential projection).  
> The helper ``forces->friction()`` returns the integrated tangential viscous
> force over the selected boundary.
>
> For example, in the ``ktauChannel`` case, assuming the first component of the
> returned vector corresponds to the streamwise (wall-parallel) direction and
> using a reference density :math:`\rho_{\text{ref}}`:
>
> .. code-block:: cpp
>
>    auto viscousForce = forces->friction(); // std::array<dfloat, 3>
>    auto utau = std::sqrt(std::abs(viscousForce[0]) / (rhoRef * areaWall));
>
> For nondimensional simulations with :math:`\rho_{\text{ref}} = 1`:
>
> .. code-block:: cpp
>
>    auto utau = std::sqrt(std::abs(forces->friction()[0]) / areaWall);
>
> The wall area ``areaWall`` can be obtained via :ref:`postproc_qoi_integrals`.

### Q criterion

The Q-criterion [Hunt1988]_ is an easy-to-compute vortex indicator, often used
for visualizing vortical structures (for example, iso-surfaces at
introduced earlier, the Q-criterion is defined a

```math
 Q = \frac{1}{2}\left( \lvert \boldsymbol{\omega} \rvert^2
                      - \lvert \boldsymbol{\varepsilon} \rvert^2 \right)
   = \frac{1}{2}\left( \omega_{ij}\,\omega_{ij}
                      - \varepsilon_{ij}\,\varepsilon_{ij} \right).

```

Regions with :math:`Q > 0` are rotation-dominated and are typically associated
with vortical structures. In *NekRS*, the Q-criterion based on the current
velocity field can be computed as follows. There are various API defined in
``app/nrs/nrs.hpp``. One can either use pooled device memory, but a user-supplied
persistent buffer can be provided for checkpointing or coupling to other tools.

```cpp
// Compute Q-criterion on pooled device memory
auto o_qcriterion = nrs->Qcriterion();

// Or compute into a user-supplied persistent buffer
static deviceMemory<dfloat> o_qcriterion(mesh->Nlocal);
nrs->Qcriterion(o_qcriterion);
```

## Averaging

For turbulent flows, individual realizations are too sensitive to initial
conditions to be exactly reproducible. In high-fidelity simulations such as DNS
or LES, it is therefore common to analyze statistical quantities, using time
averaging of the solution fields or spatial averaging over selected
cross-sections. This can sometimes be useful even in RANS simulations.
In the following sections we consider two basic types of averaging used in
post-processing: time averaging and spatial averaging.

### Time Averaging

*NekRS* provides the ``tavg`` class for on-the-fly time averaging of custom
fields.
To create custom averaging fields, declare a global ``std::unique_ptr<tavg> avg``
at the top of the ``.udf`` file. In ``UDF_Setup()`` you then define the fields
to be averaged. Each ``tavg::field`` is defined as a pair consisting of a string
name and a ``std::vector`` of device fields. At runtime, the pointwise product
of all fields in the vector is formed and time-averaged. The number of fields in
this vector can range from 1 to 4, allowing first- through fourth-order moments
or correlations. For a generic quantity :math:`X(t)`, the time average
corresponds to

```math
 \overline{X} := \mathbb{E}[X]
 = \frac{1}{T_1 - T_0} \int_{t_0}^{t_1} X(t)\, dt.

```

In the example below, the registered fields include

- first-order moments: :math:`\mathbb{E}[U]`, :math:`\mathbb{E}[V]`, :math:`\mathbb{E}[T]`,
- second-order moments: :math:`\mathbb{E}[U^2]`, :math:`\mathbb{E}[V^2]`,
- a mixed correlation: :math:`\mathbb{E}[U^2 T]`.

After appending all entries with ``tavgFields.push_back``,
``std::make_unique<tavg>`` constructs the averaging object, using the field
offset ``nrs->fluid->fieldOffset`` and the accumulated ``tavgFields`` container
as arguments.

```
#include "tavg.hpp"

std::unique_ptr<tavg> avg;

void UDF_Setup()
{
   std::vector< tavg::field > tavgFields;

   deviceMemory<dfloat> o_U(nrs->fluid->o_solution("x"));
   deviceMemory<dfloat> o_V(nrs->fluid->o_solution("y"));
   deviceMemory<dfloat> o_T(nrs->scalar->o_solution("temperature"));

   // First-order moments
   tavgFields.push_back({"U", std::vector{o_U}}); // E[U]
   tavgFields.push_back({"V", std::vector{o_V}}); // E[V]
   tavgFields.push_back({"T", std::vector{o_T}}); // E[T]

   // Second moments and correlations
   tavgFields.push_back({"UU", std::vector{o_U, o_U}}); // E[UU]
   tavgFields.push_back({"VV", std::vector{o_V, o_V}}); // E[UV]
   tavgFields.push_back({"UUT", std::vector{o_U, o_U, o_T}}); // E[UUT]

   avg = std::make_unique<tavg>(nrs->fluid->fieldOffset, tavgFields);
}

void UDF_ExecuteStep(double time, int tstep)
{
   auto mesh = nrs->meshV;

   // Accumulate the running averages
   if(nrs->timeStepConverged) {
     avg->run(time);
   }

   // Output at checkpoint steps
   if(nrs->checkpointStep) {
     avg->writeToFile(mesh);
   }
}
```

The time averaging is advanced in ``UDF_ExecuteStep()`` via ``avg->run(time)``,
typically guarded by ``nrs->timeStepConverged``  (e.g., for outer steps in
neknek). To write the averaged fields to a file, ``avg->writeToFile(mesh)``
is called when ``nrs->checkpointStep`` is true. Each registered entry is written
as a scalar field to files ``tavg0.fXXXXX``. This call also resets the
averaging window.

> **Note:**
> By default, ``writeToFile`` resets the start time :math:`t_0`. In other
> words, each output file contains the average over a window
> :math:`[t_{i-1}, t_i]`. If the user does *not* want to reset :math:`t_0` and
> instead continue accumulating from the beginning of the ``avg::run``, a
> second ``bool`` argument can be used:
>
> .. code-block:: cpp
>
>    bool resetAveragingTime = false;
>    avg->writeToFile(mesh, resetAveragingTime);
>
> On large HPC runs it is generally recommended to keep the default reset
> behavior and combine files in post-processing mode, especially if ``dt`` may
> change or the final I/O might not complete within the runtime.

> **Note:**
> Notably, the ``time`` variable stored in ``tavg`` records the averaging
> interval :math:`t_1 - t_0`. Each registered entry is written as a scalar
> field. For the example above, the output file ``tavg0.fXXXXX`` contains
>
> .. csv-table::
>    :header: "Variable","Scalar index"
>    :widths: 50, 50
>
>    :math:`\overline{U}`,scalar 0
>    :math:`\overline{V}`,scalar 1
>    :math:`\overline{T}`,scalar 2
>    :math:`\overline{U^2}`,scalar 3
>    :math:`\overline{V^2}`,scalar 4
>    :math:`\overline{U^2 T}`,scalar 5

### Time Averaging (Legacy Mode)

On the other hand, *NekRS* also supports a legacy time-averaging interface
that mirrors the Nek5000 routine
`avg_all <https://nek5000.github.io/NekDoc/problem_setup/features.html#averaging>`_,
which computes run-time averages of all primitive variables, i.e.
terms :math:`u^2`, :math:`v^2`, :math:`w^2`, :math:`T^2`, :math:`uv`,
interface shown above. In this legacy mode, averaging is managed through a
dedicated C++ helper object ``std::unique_ptr<nrs_t::tavgLegacy_t> avg``. The
call ``avg->writeToFile(mesh)`` writes three checkpoint files,
``avg0.fXXXXX``, ``rms0.fXXXXX``, and ``rm20.fXXXXX``, following the
*Nek5000*-style naming convention. See the example below and the subsequent
table for the mapping between variables and output files.

```c++
std::unique_ptr<nrs_t::tavgLegacy_t> avg;

void UDF_Setup()
{
   avg = std::make_unique<nrs_t::tavgLegacy_t>();
}

void UDF_ExecuteStep(double time, int tstep)
{
   auto mesh = nrs->meshV;

   if (nrs->timeStepConverged) {
     avg->run(time);
   }

   if (nrs->checkpointStep) {
     avg->writeToFile(mesh);
     avg->reset(); // reset time window
   }
}
```

### Planar Averaging

Planar averaging reduces three-dimensional fields to one- or two-dimensional
profiles by averaging over user-specified planes (typically aligned with
geometric or homogeneous directions, such as channel or pipe cross-sections).
For turbulence statistics, this not only yields mean and fluctuation profiles,
but in statistically homogeneous flows (e.g., isotropic turbulence) the added
spatial averaging also accelerates convergence by increasing the effective
sample count at each time step.

*NekRS* provides a built-in function ``planarAvg`` (in ``core/mesh/planarAvg.cpp``)
for meshes that are ordered lexicographically and are at least extruded in the
``NUMBER_ELEMENTS_X``, ``NUMBER_ELEMENTS_Y``, and ``NUMBER_ELEMENTS_Z`` in each
direction is used to compute planar averages over :math:`x`\–:math:`z` planes,
i.e. :math:`\langle X \rangle(y) = \int\!\!\int X(x,y,z)\,\mathrm{d}x\,\mathrm{d}z`,
for six scalar fields: :math:`u`, :math:`w`, :math:`T`, :math:`\partial_y u`,
specified as a string and can be one-dimensional (``"x"``, ``"y"``, ``"z"``) or
two-dimensional (e.g., ``"xy"``, ``"xz"``, ``"yz"``).

The code first fills the scratch array ``o_work`` with the values to be
averaged; ``planarAvg`` then computes the requested averages and broadcasts
them to all grid points on each averaging plane. In this example, after the
call we have :math:`X(x,y,z) = \langle X \rangle(y)` for all points on a given

```cpp
auto planarAverage()
{
  auto mesh = nrs->meshV;
  poolDeviceMemory<dfloat> o_work(6 * nrs->fieldOffset); // scratch space for 6 scalars

  // <u>(y)
  auto o_ux = nrs->scalar->o_U.slice(0 * nrs->fieldOffset, nrs->fieldOffset);
  o_work.copyFrom(o_ux, nrs->fieldOffset, 0 * nrs->fieldOffset);

  // <w>(y)
  auto o_uz = nrs->scalar->o_U.slice(2 * nrs->fieldOffset, nrs->fieldOffset);
  o_work.copyFrom(o_uz, nrs->fieldOffset, 1 * nrs->fieldOffset);

  // <temp>(y)
  auto o_temp = nrs->scalar->o_S.slice(0 * nrs->fieldOffset, nrs->fieldOffset);
  o_work.copyFrom(o_temp, nrs->fieldOffset, 2 * nrs->fieldOffset);

  // d<u,w,temp>/dy(y)
  auto o_ddyAvg = o_work.slice(3 * nrs->fieldOffset, 3 * nrs->fieldOffset);
  vecGradY(mesh->Nelements, mesh->o_vgeo, mesh->o_D, nrs->fieldOffset, mesh->o_invAJw, o_work, o_ddyAvg);
  nrs->qqt->startFinish("+", o_ddyAvg, nrs->fieldOffset);

  planarAvg(mesh, "xz", NUMBER_ELEMENTS_X, NUMBER_ELEMENTS_Y, NUMBER_ELEMENTS_Z, 6, nrs->fieldOffset, o_work);

  return o_work;
}
```

> **Tip:**
> For a non-box mesh that is still extruded in :math:`z` (e.g., a pipe), you
> can set ``NUMBER_ELEMENTS_X = nelx * nely``, ``NUMBER_ELEMENTS_Y = 1``, and
> ``NUMBER_ELEMENTS_Z = nelz`` and still use ``"z"`` for averaging along the
> axial direction, or ``"x"`` for averaging over cross-sectional planes.

## ADIOS2 Format (.bp/)

*NekRS* also supports writing checkpoints with the
`ADIOS2 <https://adios2.readthedocs.io>`__ BPFile engine using version 5 (BP5)
(`engine documentation <https://adios2.readthedocs.io/en/v2.10.2/engines/engines.html#bp5>`__).
ADIOS2 is bundled with the *NekRS* third-party libraries and is built
automatically when ``ENABLE_ADIOS=ON`` (the default in ``CMakeLists.txt``). To
enable ADIOS2-based checkpoints, set ``checkpointEngine = adios`` in the
``.par`` file.

> **Tip:**
> At configure/compilation stage, ADIOS support can be disabled with
>
> .. code-block:: bash
>
>    ./build.sh -DENABLE_ADIOS=OFF
>
> If you have a preferred (e.g., optimized) ADIOS2 installation, you can also
> point *NekRS* to it via
>
> .. code-block:: bash
>
>    ./build.sh -DADIOS2_INSTALL_DIR=<path-to-adios-installation>

The checkpoint is written as a directory with a ``.bp`` suffix, e.g.
``turbPipe.bp/``, which can contain multiple time steps. The
`ADIOS2 command-line utilities <https://adios2.readthedocs.io/en/latest/ecosystem/utilities.html#bpls-inspecting-data>`__
are also installed under ``$NEKRS_HOME/bin/`` and provide convenient tools to
inspect and manipulate the data. The checkpoint data are exposed in a
VTK-like layout and can be read directly in ParaView using the
``ADIOS2VTXReader``.

Here are some sample usages to inspect the file:

- Check metadata with ``$NEKRS_HOME/bin/bpls turbPipe.bp/``. In this case,
  there are 5 time steps.

```bash
   uint64_t  connectivity      [2]*{1344560, 9}
   uint64_t  globalElementIds  [2]*{3920}
   uint32_t  numOfCells        scalar
   uint32_t  numOfPoints       scalar
   uint32_t  polynomialOrder   scalar
   float     pressure          5*[2]*{2007040}
   double    time              5*scalar
   uint32_t  types             scalar
   float     velocity          5*[2]*{2007040, 3}
   float     vertices          [2]*{2007040, 3}
```

- Dump specific variables with ``-d``. For example,

  ``$NEKRS_HOME/bin/bpls turbPipe.bp/ time polynomialOrder numOfCells numOfPoints -d``

```bash
   uint32_t  numOfCells        scalar
 1344560

   uint32_t  numOfPoints       scalar
 2007040

   uint32_t  polynomialOrder   scalar
 7

   double    time              5*scalar
     (0)    0.003 0.006 0.0135 0.0255 0.0375
```

By default, the VTK cell type ``types = 12`` (hexahedra) is used, which
converts each element to :math:`343 = (N-1)^3` cells. In this example,
3920 elements with 7th-degree polynomials give a total of
``connectivity`` array, whose coordinates are stored in ``vertices``.
All scalar and vector fields are then represented on the
``numOfPoints = 2007040`` mesh vertices.

> **Note:**
> ADIOS checkpoints are more flexible and allow you to name variables freely
> and add extra fields. For example, you can write a file that contains a
> customized ``vorticity`` vector and an additional scalar field
> ``q_criterion``.
>
> ADIOS2 output also supports interpolation to a different polynomial order
> (or to a uniform grid). See :ref:`ic_restart` for details.

## Data Extraction

*TODO* New plugin and hpts

## Insitu Visualization

*TODO*

## Legacy Support (userchk)

See usr_file.

## Related knowledge-base notes
- [[Running & Runtime Controls]]
- [[Debugging Guide]]
