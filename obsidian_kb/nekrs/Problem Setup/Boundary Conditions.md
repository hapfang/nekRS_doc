> Source: `source/problem_setup/boundary_conditions.rst` (converted to markdown; retains documentation detail)

# Boundary conditions

Boundary surfaces should already be named or tagged by the meshing software and
stored in the mesh. See mesh_setup_sidesets for details on how *NekRS*
handles surface tagging and how users can configure these mappings manually.

Boundary conditions are specified per tag for each field in the par_file
using ``boundaryTypeMap``. This chapter focuses on the use of
``boundaryTypeMap``, which is referenced in the ``FLUID VELOCITY`` and
``SCALAR FOO`` sections to define the boundary conditions for the corresponding
fields. Nek5000-style string-based ``cbc`` tags are handled automatically, and
boundary values can be prescribed directly in ``udfDirichlet`` or ``udfNeumann``
as needed. See boundary_conditions_user_bc for details.

> **Tip:**
>  Some example cases from the `NekRS Github repo <https://github.com/Nek5000/nekRS/tree/next>`_ are listed below to show the usage of different user defined boundary condition types:
>
>  +------------------+-------------------------------------+------------------------------------------------------------------------------------+
>  | BC type          | Field                               | Example case                                                                       |
>  +------------------+-------------------------------------+------------------------------------------------------------------------------------+
>  | ``udfDirichlet`` | ``"fluid velocity"``                | `ethier <https://github.com/Nek5000/nekRS/tree/next/examples/ethier>`_             |
>  |                  +-------------------------------------+------------------------------------------------------------------------------------+
>  |                  | ``"fluid velocity"`` (recycling)    | `turbPipe <https://github.com/Nek5000/nekRS/tree/next/examples/turbPipe>`_         |
>  |                  +-------------------------------------+------------------------------------------------------------------------------------+
>  |                  | ``"fluid velocity"`` (interpolated) | `eddyNekNek <https://github.com/Nek5000/nekRS/tree/next/examples/eddyNekNek>`_     |
>  |                  +-------------------------------------+------------------------------------------------------------------------------------+
>  |                  | ``"fluid pressure"``                | `hemi <https://github.com/Nek5000/nekRS/tree/next/examples/hemi>`_                 |
>  |                  +-------------------------------------+------------------------------------------------------------------------------------+
>  |                  | ``"scalar XXX"``                    | `conj_ht <https://github.com/Nek5000/nekRS/tree/next/examples/conj_ht>`_           |
>  |                  +-------------------------------------+------------------------------------------------------------------------------------+
>  |                  | ``"scalar XXX"`` (interpolated)     | `eddyNekNek <https://github.com/Nek5000/nekRS/tree/next/examples/eddyNekNek>`_     |
>  |                  +-------------------------------------+------------------------------------------------------------------------------------+
>  |                  | ``"geom"``                          | `mv_cyl <https://github.com/Nek5000/nekRS/tree/next/examples/mv_cyl>`_             |
>  +------------------+-------------------------------------+------------------------------------------------------------------------------------+
>  | ``udfNeumann``   | ``"fluid velocity"``                | `gabls1 <https://github.com/Nek5000/nekRS/tree/next/examples/gabls1>`_             |
>  |                  +-------------------------------------+------------------------------------------------------------------------------------+
>  |                  | ``"scalar XXX"``                    | `ethier <https://github.com/Nek5000/nekRS/tree/next/examples/ethier>`_             |
>  +------------------+-------------------------------------+------------------------------------------------------------------------------------+

## Basic boundary conditions

All available boundary condition types are summarized in the *NekRS*
command-line help. Users can find the complete list in the *NekRS* manual
using:

```bash
nrsman par
```

   Consider an inlet--outlet pipe flow with three boundary IDs:
   389 (inlet, ``udfDirichlet``), 231 (outlet, ``zeroNeumann``), and
   4 (walls, ``zeroDirichlet``). The corresponding ``.par`` file entries are:

```ini
   [MESH]
   boundaryIDMap = 389, 231, 4  # boundary IDs from the mesher

   [FLUID VELOCITY]
   boundaryTypeMap = udfDirichlet, zeroNeumann, zeroDirichlet
```

   For conjugate heat transfer (CHT) cases, fluid velocity boundary IDs
   must also be specified explicitly using ``boundaryIDMapFluid`` to identify
   conformal fluid--solid interfaces.

   Extending the example above, assume the pipe is surrounded by a solid annular
   jacket. Boundary ID 4 corresponds to the conformal fluid--solid wall, while
   the remaining insulated solid walls have boundary IDs 10, 20, and 30. The
   corresponding configuration is:

```ini
    [MESH]
    boundaryIDMap = 389, 231 ,4
    boundaryIDMapFluid = 10, 20, 30, 389, 231, 4

    [FLUID VELOCITY]
    boundaryTypeMap = udfDirichlet, zeroNeumann, zeroDirichlet

    [SCALAR TEMPERATURE]
    boundaryTypeMap = zeroNeumann, zeroNeumann, zeroNeumann, udfDirichlet, zeroNeumann, none
```

From an implementation standpoint, the available boundary conditions in *NekRS*
can be broadly classified into three major categories, which are discussed in the
following sections:

- **Zero-value BCs**: homogeneous boundary conditions that do not require any
  user input.
- **User-value BCs**: inhomogeneous boundary conditions whose values are
  prescribed by the user.
- **Internal / periodic BCs**: boundary conditions used for internal interfaces
  or periodic domain connectivity.

#### Zero-value BC

Boundary conditions of type ``zeroDirichlet`` and/or ``zeroNeumann`` are handled
entirely within *NekRS* and do not require any explicit user definitions in the
``.udf`` file. As the names imply, these boundary conditions enforce homogeneous
Dirichlet or Neumann values. The available zero-value boundary conditions are
summarized in the table below, using the following notation:

- :math:`\mathbf{\hat e_n}` is the unit vector normal to the boundary face.
- :math:`\mathbf{\hat e_x}`, :math:`\mathbf{\hat e_y}`, and
- :math:`\mathbf{\hat e_t}` and :math:`\mathbf{\hat e_b}` are the local tangential
  and bitangential unit vectors, respectively.

   Fluid, ``zeroDirichlet``,``w`` / ``wall``, :math:`\mathbf u = 0`
   Fluid, ``zeroNeumann``,"``o`` / ``O`` / ``outlet`` / ``outflow``", :math:`\boldsymbol{\underline \tau} \cdot \mathbf{\hat{e}_n} = 0`
   Fluid, ``zeroDirichletX/zeroNeumann``,``slipx`` / ``symx``,":math:`\mathbf{u} \cdot \mathbf{\hat{e}_x} = 0` |br| :math:`(\boldsymbol{\underline \tau} \cdot \mathbf{\hat{e}_n}) \cdot \mathbf{\hat{e}_y} = 0` |br| :math:`(\boldsymbol{\underline \tau} \cdot \mathbf{\hat{e}_n}) \cdot \mathbf{\hat{e}_z} = 0` "
   Fluid, ``zeroDirichletY/zeroNeumann``,``slipy`` / ``symy``,":math:`\mathbf{u} \cdot \mathbf{\hat{e}_y} = 0` |br| :math:`(\boldsymbol{\underline \tau} \cdot \mathbf{\hat{e}_n}) \cdot \mathbf{\hat{e}_x} = 0` |br| :math:`(\boldsymbol{\underline \tau} \cdot \mathbf{\hat{e}_n}) \cdot \mathbf{\hat{e}_z} = 0` "
   Fluid, ``zeroDirichletZ/zeroNeumann``,``slipz`` / ``symz``,":math:`\mathbf{u} \cdot \mathbf{\hat{e}_z} = 0` |br| :math:`(\boldsymbol{\underline \tau} \cdot \mathbf{\hat{e}_n}) \cdot \mathbf{\hat{e}_x} = 0` |br| :math:`(\boldsymbol{\underline \tau} \cdot \mathbf{\hat{e}_n}) \cdot \mathbf{\hat{e}_y} = 0` "
   Fluid, ``zeroDirichletN/zeroNeumann``,``slip`` / ``sym``,":math:`\mathbf{u} \cdot \mathbf{\hat{e}_n} = 0` |br| :math:`(\boldsymbol{\underline \tau} \cdot \mathbf{\hat{e}_n}) \cdot \mathbf{\hat{e}_{t}} = 0` |br| :math:`(\boldsymbol{\underline \tau} \cdot \mathbf{\hat{e}_n}) \cdot \mathbf{\hat{e}_{b}} = 0` "
   Fluid, ``zeroDirichletYZ/zeroNeumann``,``onx``,":math:`\mathbf{u} \cdot \mathbf{\hat{e}_y} = 0` |br|  :math:`\mathbf{u} \cdot \mathbf{\hat{e}_z} = 0` |br| :math:`(\boldsymbol{\underline \tau} \cdot \mathbf{\hat{e}_n}) \cdot \mathbf{\hat{e}_x} = 0` "
   Fluid, ``zeroDirichletXZ/zeroNeumann``,``ony``,":math:`\mathbf{u} \cdot \mathbf{\hat{e}_x} = 0` |br|  :math:`\mathbf{u} \cdot \mathbf{\hat{e}_z} = 0` |br| :math:`(\boldsymbol{\underline \tau} \cdot \mathbf{\hat{e}_n}) \cdot \mathbf{\hat{e}_y} = 0` "
   Fluid, ``zeroDirichletXY/zeroNeumann``,``onz``,":math:`\mathbf{u} \cdot \mathbf{\hat{e}_x} = 0` |br|  :math:`\mathbf{u} \cdot \mathbf{\hat{e}_y} = 0` |br| :math:`(\boldsymbol{\underline \tau} \cdot \mathbf{\hat{e}_n}) \cdot \mathbf{\hat{e}_z} = 0` "

   Scalar, ``zeroNeumann``,"``I`` / ``insulated``", :math:`\lambda \nabla s \cdot \mathbf{\hat{e}_n} = 0`

   Geom, ``zerodirichlet``, "``w`` / ``v`` / ``wall`` / ``inlet``", zero movement

#### User-value BC

Boundary conditions that require prescribed values are implemented through user
callbacks in the ``.udf`` file. These values are supplied via the **UDF boundary
condition functions** ``udfDirichlet``, ``udfNeumann``, and ``udfRobin``, defined
inside the okl block, corresponding to Dirichlet, Neumann, and
Robin (mixed) boundary condition types.

During the simulation, each solver iterates over boundary surface points with
user-value BCs and invokes the appropriate UDF callback. The required boundary
values are then assigned through the provided ``bcData`` structure.

The table below summarizes the supported user defined boundary condition keys,
their legacy aliases, and the corresponding ``bcData`` variables to be set.

.. _tab:udfbcs:

   Fluid, ``udfDirichlet``, ``v`` / ``inlet``, "``bc->uxFluid`` |br| ``bc->uyFluid`` |br| ``bc->uzFluid``"
   Fluid, ``zeroDirichletN / udfNeumann``, ``shl`` / ``traction``, "``bc->tr1`` :math:`((\boldsymbol{\underline{\tau}} \cdot \mathbf{\hat e_n}) \cdot \mathbf{\hat e_t})` |br| ``bc->tr2`` :math:`((\boldsymbol{\underline{\tau}} \cdot \mathbf{\hat e_n}) \cdot \mathbf{\hat e_b})` |br| :math:`\mathbf{u} \cdot \mathbf{\hat e_n} = 0` (enforced internally)"
   Fluid, ``interpolation``, ``int``, "``bc->uxFluid`` |br| ``bc->uyFluid`` |br| ``bc->uzFluid``"

   Scalar, ``udfDirichlet``, ``t`` / ``inlet``, ``bc->sScalar``
   Scalar, ``udfNeumann``, ``f`` / ``flux``, "``bc->fluxScalar`` :math:`(\lambda \nabla s \cdot \mathbf{\hat e_n})`"
   Scalar, ``udfRobin``, ``c`` (Newton's cooling), ``bc->h`` ``bc->sInfScalar`` |br| :math:`(\lambda \nabla s \cdot \mathbf{\hat e_n}) = h (s - s_{\infty})`
   Scalar, ``interpolation``, ``int``, ``bc->sScalar``

   Geom, ``udfDirichlet`` |br| ``udfDirichlet+moving``, ``mv``, "``bc->uxGeom`` |br| ``bc->uyGeom`` |br| ``bc->uzGeom``"

   The table below summarizes the variables in the ``bcData`` structure to which
   boundary values are assigned in the ``udfDirichlet``, ``udfNeumann``, and
   ``udfRobin`` callbacks.

   .. _tab:bcdata:output:

      "``uxFluid`` , ``uyFluid`` , ``uzFluid``", ``dfloat``, Dirichlet, fluid velocity components
      ``pFluid``,``dfloat``, Outflow*, fluid pressure
      "``tr1``",``dfloat``, Neumann, fluid traction along tangent
      "``tr2``",``dfloat``, Neumann, fluid traction along bi-tangent
      ``sScalar``, ``dfloat``, Dirichlet, scalar value
      ``fluxScalar``,``dfloat``, Neumann, scalar flux
      ``h``, ``dfloat``, Robin, heat transfer coefficient
      ``sInfScalar``, ``dfloat``, Robin, ambient temperature :math:`(T_\infty)`
      "``uxGeom`` , ``uyGeom`` , ``uzGeom``", ``dfloat``, Dirichlet, mesh velocity components

> **Note:**
>    For **outflow**, pressure uses a Dirichlet condition and velocity uses a
>    Neumann condition.

   These values may depend on space, time, material properties or coefficients, or
   other user defined variables. To support variable boundary values, related input
   data supplied by the solver are also passed to the ``bcData`` structure. This
   includes:

   .. _tab:bcdata:input:

      ``fieldName``,``char*``,character array to store field name
      ``fieldOffset``,``int``,array offset
      ``idxVol``,``int``,volume index of boundary GLL
      ``id``,``int``,boundary ID tag
      ``time``,``double``,current simulation time
      "``x`` , ``y`` , ``z``",``dfloat``,location coordinates of boundary GLL
      "``nx`` , ``ny`` , ``nz``",``dfloat``,local normal vector components
      "``t1x`` , ``t1y`` , ``t1z``",``dfloat``,local tangent vector components
      "``t2x`` , ``t2y`` , ``t2z``",``dfloat``,local bi-tangent vector components
      ``idScalar``,``int``,scalar index
      ``transCoeff``,``dfloat``,field transport coefficient
      ``diffCoeff``,``dfloat``,field diffusion coefficient
      ``usrwrk``,``@globalPtr const dfloat*``, pointer to user work array
      "``uxFluidInt`` , ``uyFluidInt`` , ``uzFluidInt``", ``dfloat``, interpolated velocity components (NekNek)
      ``sScalarint``, ``dfloat``, interpolated scalar value (NekNek)

> **Warning:**
>    Only a subset of solution fields is provided as input through ``bcData``
>    in a given boundary callback.
>
>    For example, when setting a fluid Dirichlet BC, scalar values are not
>    accessible through ``bc->sScalar``. If a boundary condition requires
>    coupling to other fields, use ``bc->usrwrk`` to pass the required data
>    and populate it explicitly in advance.
>
>    Only the following inputs are available:
>
>    - ``uxFluid``, ``uyFluid``, ``uzFluid`` for fluid pressure and scalar
>      related BCs
>    - ``sScalar`` for scalar Neumann BC

   Multiple fields may use the same type of boundary condition. To associate a
   user defined boundary condition with a specific field, the target field is
   identified inside the UDF callbacks using the ``isField()`` function. The
   ``isField()`` macro is defined in ``bcData.h`` and takes a string field
   identifier as its argument.

   The supported string field identifiers are listed below:

   +-----------------------------------+-------------------------------------------+
   | String Identifier                 | Field                                     |
   +===================================+===========================================+
   | ``"fluid velocity"``              | velocity                                  |
   +-----------------------------------+-------------------------------------------+
   | ``"fluid pressure"``              | pressure                                  |
   +-----------------------------------+-------------------------------------------+
   | ``"scalar foo"``                  | field corresponding to scalar ``"FOO"``   |
   +-----------------------------------+-------------------------------------------+
   | ``"geom"``                        | (moving) mesh                             |
   +-----------------------------------+-------------------------------------------+

> **Note:**
>    The scalar field string identifiers are declared in :ref:`Parameter file <par_file>` under the :ref:`General Section<sec:generalpars>`.

   A generic example showing how user defined Dirichlet and Neumann boundary
   conditions are mapped using the *Minimal par file example* above and
   implemented in the okl block is shown below.

```c++
   #ifdef __okl__

   void udfDirichlet(bcData *bc)
   {
      if(isField("fluid velocity")) {
       if(bc->id == 1) {                // inlet surface id
        bc->uxFluid = 1.0;
        bc->uyFluid = 0.0;
        bc->uzFluid = 0.0;
       }
      }
      else if (isField("fluid pressure")) {
        bc->pFluid = 0.0;
      }
      else if (isField("scalar foo")) {
        if(bc->id == 1) bc->sScalar = 1.0;  // inlet id
        if(bc->id == 3) bc->sScalar = 0.0;  // wall id
      }
   }

   void udfNeumann(bcData *bc)
   {
    if(isField("fluid velocity")) {
      bc->tr1 = 0.0;
      bc->tr2 = 0.0;
    }
    else if (isField("scalar foo")) {
      bc->fluxScalar = 0.0;
    }
   }
   #endif
```

#### Internal / periodic BC

Since every boundary ID must be assigned in ``boundaryTypeMap``, internal
interfaces or surfaces that do not require a physical boundary condition should
be assigned ``none``.

Periodic boundary conditions are defined by the mesh connectivity and handled by
the meshing tool. Tagging a boundary as ``periodic`` in ``boundaryTypeMap`` only
indicates intent and the actual periodic pairing is determined by the mesh.

> **Note:**
> Periodic pairs are part of the mesh connectivity stored in the ``.re2`` file.
> Simply changing a boundary condition from ``wall`` to ``periodic`` in
> ``.par`` will likely not work. See :ref:`mesh_setup_connectivity` for details.

To sustain a prescribed flow rate in an axis-aligned periodic channel, a
pressure gradient can be applied either through forcing in ``userf`` or by using
the built-in flow-rate control mechanism (see the ``turbPipePeriodic`` example).
The latter is configured in the ``[GENERAL]`` section of the ``.par`` file using
``constFlowRate = meanVelocity=1.0 + direction=Z``.
The applied pressure-gradient scaling is reported in the log file, for example
in the final column shown below:

```bash
$ grep flowrate logfile

step=10       flowrate            : uBulk0 1.00e+00  uBulk 1.00e+00  err 2.44e-15  scale 3.13313e-02
```

## Special cases

#### NekNek coupling

NekNek couples two (or more) Nek instances using an overlapping Schwarz
approach. Overlapping meshes do not need to be conformal, which simplifies
meshing and allows flexible placement of resolution as well as distribution of
computational resources. For example, different instances may use different
polynomial orders or time step sizes.

Behind the scenes, NekNek is implemented as a Dirichlet boundary condition for
velocity and scalar fields. Solution values are interpolated from other sessions
and applied as boundary values through an outer predictor-corrector loop. Users
only need to set the ``interpolation`` boundary condition to enable this
mechanism, and then assign the interpolated values in ``udfDirichlet`` as shown
below:

```c++
void udfDirichlet(bcData * bc)
{
  if (isField("fluid velocity")) {
    bc->uxFluid = bc->uxFluidInt;
    bc->uyFluid = bc->uyFluidInt;
    bc->uzFluid = bc->uzFluidInt;
  } else if (isField("scalar foo")) {
    bc->sScalar = bc->sScalarInt;
  }
}
```

With careful planning, this coupling can also be made one directional, for
example to generate a turbulent inlet using a smaller upstream domain.

#### Turbulent inflow

A simple way to trip turbulence for an initial condition or inflow boundary is
to add a small, reproducible random perturbation. *NekRS* provides a helper in
``platform/utils/randomVector.hpp``:

```c++
template <typename T>
std::vector<T> randomVector(int N,
                            T min = 0,
                            T max = 1,
                            bool deterministic = false);
```

For example, the following generates a deterministic uniform random vector in
``turbPipePeriodic``.

```c++
auto rand = randomVector<dfloat>(mesh->Nlocal, -1.0, 1.0, true);

std::vector<dfloat> U(mesh->dim * nrs->fluid->fieldOffset, 0.0);
for (int n = 0; n < mesh->Nlocal; n++) {
  const auto R = 0.5;
  const auto xr = x[n] / R;
  const auto yr = y[n] / R;

  auto rr = xr * xr + yr * yr;
  rr = (rr > 0) ? sqrt(rr) : 0.0;

  auto uz = 6/5. * (1 - pow(rr, 6));
  U[n + 2 * nrs->fluid->fieldOffset] = uz + 0.01 * rand[n];
}

 nrs->fluid->o_U.copyFrom(U.data(), U.size());
```

The generated random values can be used as a tripping signal by scaling and
adding them to the inlet velocity profile or to the initial condition.

#### Velocity recycling

NekRS offers an in-built technique to impose fully developed turbulent inflow conditions.
It comprises recycling the velocity from an offset surface in the domain interior to the inlet boundary resulting in a quasi-periodic domain.
The required routines are available through the ``planarCopy.hpp`` header file.
Sample ``.udf`` code to use velocity recycling is as follows,

```cpp
```

  #include "planarCopy.hpp"

  deviceMemory<int> o_bID;
  planarCopy* recyc = nullptr;
  dfloat area, uBulk;

  #ifdef __okl__
   void udfDirichlet(bcData * bc)
   {
    if (isField("fluid velocity")) {
      bc->uxFluid = bc->usrwrk[bc->idxVol + 0 * bc->fieldOffset];
      bc->uyFluid = bc->usrwrk[bc->idxVol + 1 * bc->fieldOffset];
      bc->uzFluid = bc->usrwrk[bc->idxVol + 2 * bc->fieldOffset];
    }
   }
  #endif

  void UDF_Setup()
  {
    auto mesh = nrs->meshV;
    const auto zmax = platform->linAlg->max(mesh->Nlocal, mesh->o_z, platform->comm.mpiComm());
    const auto zmin = platform->linAlg->min(mesh->Nlocal, mesh->o_z, platform->comm.mpiComm());
    const auto zLen = abs(zmax - zmin);

    platform->app->bc->o_usrwrk.resize(mesh->dim * nrs->fluid->fieldOffset);

    uBulk = 1.0;                            // mean bulk velocity
    const dfloat Dh = 1.0;                  // hydraulic diameter
    const dfloat xOffset = 0.0;             // x-offset of recycling surface
    const dfloat yOffset = 0.0;             // y-offset of recycling surface
    const dfloat zOffset = 10.0 * Dh;       // z-offset of recycling surface

    std::vector<int> bID;
    bID.push_back(1);
    o_bID.resize(bID.size());
    o_bID.copyFrom(bID);

    deviceMemory<dfloat> o_tmp(mesh->Nlocal);
    platform->linAlg->fill(o_tmp.size(), 1.0, o_tmp);
    area = mesh->surfaceAreaMultiplyIntegrate(o_bID, o_tmp);

    recyc = new planarCopy(mesh,
                           nrs->fluid->o_solution(),
                           mesh->dim,
                           nrs->fluid->fieldOffset,
                           xOffset,
                           yOffset,
                           zOffset,
                           bID[0],
                           platform->app->bc->o_usrwrk);
  }

  void UDF_ExecuteStep(double time, int tstep)
  {
    recyc->execute();

    const auto flux = mesh->surfaceAreaNormalMultiplyVectorIntegrate(nrs->fluid->fieldOffset, o_bID, platform->app->bc->o_usrwrk);
    platform->linAlg->scale(nrs->fluid->fieldOffsetSum, -uBulk * area / flux, platform->app->bc->o_usrwrk);
  }

The above example assumes that the inlet surface, with boundary ID (``bID``) equal to 1, is perpendicular to the *x-y* plane.
The recycling surface is located at an offset distance equal to :math:`20 Dh` from the inlet surface, where :math:`Dh` is the hydraulic diameter of the inlet cross-section.
``uBulk`` is the desired mean velocity at the inlet boundary.

> **Note:**

  The recommended recycling length is :math:`8-10 Dh` [Lund1998]_ to allow the resolution of largest eddies. The user must ensure that the boundary layer is not acted upon by significant pressure gradients or geometrical changes between the inlet and recycle plane.

The interpolated velocities from the recycling surface are stored in the ``o_usrwrk`` array which is accessible in the ``udfDirichlet`` kernel.
Therefore, ``o_usrwrk`` vector must be resized to accommodate the velocity data to ``mesh->dim * nrs->fluid->fieldOffset`` length.
The ``planarCopy()`` call setups up the necessary interpolation procedures to map the velocity from domain interior to the inlet surface.
Note the parameters passed as arguments to the setup routine.
The ``execute()`` routine is called from the ``UDF_ExecuteStep`` routine at every time step.
It, internally, populates the ``o_usrwrk`` array with the scaled instantaneous velocity at the recycle plane based on specified ``ubulk``.
Finally, the captured velocity is specified as inlet Dirichlet boundary condition in ``udfDirichlet`` in the okl block section, as shown above.
Note the proper indexing and offset specified using ``bc->idxVol`` and ``bc->fieldOffset`` for the three velocity components.

#### Stabilized outflow [Dong]

One way to mitigate numerical instabilities caused by backflow at outflow
boundaries, for example due to large vortical structures, is to apply a
stabilized outflow boundary condition such as that proposed in [Dong2014]_.
This approach introduces a pressure correction that locally damps backflow by
penalizing negative normal velocity.

An example implementation is shown below:

```cpp
void udfDirichlet(bcData *bc)
{
  if (isField("fluid pressure")) {
    const dfloat iU0delta = 20.0;
    const dfloat un = bc->uxFluid * bc->nx + bc->uyFluid * bc->ny + bc->uzFluid * bc->nz;
    const dfloat s0 = 0.5 * (1.0 - tanh(un * iU0delta));
    bc->pFluid = -0.5 * (bc->uxFluid * bc->uxFluid + bc->uyFluid * bc->uyFluid + bc->uzFluid * bc->uzFluid) * s0;
  }
}
```

## Related knowledge-base notes
- [[Meshing & Mesh Workflows]]
- [[Case Files - par udf re2 usr sess upd]]
