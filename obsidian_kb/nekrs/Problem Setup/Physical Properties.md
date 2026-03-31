> Source: `source/problem_setup/properties.rst` (converted to markdown; retains documentation detail)

# Physical properties

## Constant Properties

Constant physical properties, including transport and diffusion coefficients for
all equations, are specified in the Parameter file under the
respective field sections. Consider the following
template for a non-dimensional case setup:

```
[FLUID VELOCITY]
density = 1.0           # or rho = 1.0

viscosity = 1./1000.0   # or mu = 1e-3
#mu = -1000.0           # Reynolds number (Re = 1000)

[SCALAR FOO]
transportCoeff = 1.0
diffusionCoeff = 1.0/500.0   # or -500.0 for Peclet number
```

The keys ``density`` or ``rho`` assign the fluid density :math:`\rho`, and
``viscosity`` or ``mu`` assign the fluid dynamic viscosity :math:`\mu`.
Similarly, ``transportCoeff`` and ``diffusionCoeff`` set the transport
coefficient and diffusion coefficient, :math:`\lambda`, for a general passive
scalar equation. When ``FOO`` represents temperature, the transport coefficient
corresponds to the volumetric heat capacity :math:`\rho c_p`, and
``diffusionCoeff`` corresponds to the thermal conductivity :math:`k`.

> **Note:**
> A **negative** value for ``mu`` or ``diffusionCoeff`` means “take the
> reciprocal of the magnitude.” That is, if you specify a parameter as
> ``x = -A``, *NekRS* internally uses :math:`x = 1/A`.
> In the standard non-dimensional setup with unit density, length, and
> velocity scales, this lets you write, for example, ``mu = -Re`` to request
> :math:`\mu = 1/Re`. Similarly, ``diffusionCoeff = -Pe`` is interpreted as
> :math:`\lambda = 1/Pe` for the scalar equation.
> See :ref:`non-dimensional scalar equation <intro_energy_nondim>`.
>
> If your problem is not scaled so that :math:`\rho = 1` (or :math:`L = U = 1`),
> you should specify :math:`\mu` and :math:`\lambda` directly. Using ``-Re`` or
> ``-Pe`` will **not** correspond to the usual definitions
> :math:`Re = \rho U L / \mu` and :math:`Pe = \rho c_p U L / \lambda`.

## Variable Properties

Spatially and/or temporally varying transport and diffusion properties are
specified in the ``.udf`` file. To enable this, first assign a user function
pointer to the internal *NekRS* member pointer ``nrs->userProperties``:

```cpp
void UDF_Setup()
{
  nrs->userProperties = &uservp;
}
```

This instructs *NekRS* to call the function ``uservp`` in the ``.udf`` file for
property specification on each time step. Inside ``uservp``, properties must be
written into the corresponding ``o_prop`` arrays, which store both diffusion and
transport coefficients at all GLL points. Typically, you do this with a
custom kernel. A template example is:

```c++
#ifdef __okl__
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
    }
  }
}
#endif

void uservp(double time)
{
  auto& fluid = nrs->fluid;
  auto& scalar = nrs->scalar;

  static int updateProperties = 1;

  if (updateProperties) {
    const dfloat Re = 1000.0;     // Reynolds number
    const dfloat Pe = 500.0;      // Peclet number

    auto o_mue = fluid->o_prop.slice(0 * fluid->fieldOffset);
    auto o_rho = fluid->o_prop.slice(1 * fluid->fieldOffset);

    auto o_k = scalar->o_prop.slice(0 * scalar->fieldOffsetSum);
    auto o_rhocp = scalar->o_prop.slice(1 * scalar->fieldOffsetSum);

    fillProp(fluid->mesh->Nelements,
             Re,
             Pe,
             o_mue,
             o_rho,
             o_k,
             o_rhocp);

    // Properties are constant in this example. Update only once.
    updateProperties = 0;
  }
}
```

This example shows how to fill the ``o_prop`` arrays for fluid and scalar
(temperature) properties. The fluid property array is accessed via the
``nrs->fluid`` object, and the scalar property array via ``nrs->scalar``.

Diffusion and transport coefficients are stored contiguously in the respective
pointers to individual coefficient fields. For the fluid, a single field is
present, so ``fieldOffset`` is sufficient. For scalars, there may be multiple
fields, so it is safer to use ``fieldOffsetSum``. The following excerpts from
``solver/fluid/fluidSolver.cpp`` and ``solver/scalar/scalarSolver.cpp`` show how
these arrays are allocated:

```cpp
// fluid
o_prop = platform->device.malloc<dfloat>(2 * nrs->fluid->fieldOffset);
o_mue = o_prop.slice(0 * nrs->fluid->fieldOffset, nrs->fluid->mesh->Nlocal);
o_rho = o_prop.slice(1 * nrs->fluid->fieldOffset, nrs->fluid->mesh->Nlocal);

// scalars (possibly multiple scalar fields)
o_prop = platform->device.malloc<dfloat>(2 * nrs->scalar->fieldOffsetSum);
o_diff = o_prop.slice(0 * nrs->scalar->fieldOffsetSum, nrs->scalar->fieldOffsetSum);
o_rho = o_prop.slice(1 * nrs->scalar->fieldOffsetSum, nrs->scalar->ieldOffsetSum);
```

In many cases it is more convenient (and less error-prone) to use the helper
accessors for the corresponding coefficients:

```cpp
// fluid properties
auto o_mue  = nrs->fluid->o_diffusionCoeff();
auto o_rho  = nrs->fluid->o_transportCoeff();

// scalar "temperature" properties
auto o_k     = nrs->scalar->o_diffusionCoeff("temperature");
auto o_rhocp = nrs->scalar->o_transportCoeff("temperature");
```

The ``fillProp`` kernel above assigns constant coefficients, but in practice it
can be generalized to depend on space (and, if desired, time) based on the
target application. Because the example uses constant properties, the call is
guarded by ``if (updateProperties)`` so that the properties are filled only
once. *NekRS* will still call ``uservp`` at the beginning of each time step,
and the ``time`` argument is available if temporally varying properties are
required.

> **Note:**
> If variable properties are assigned in the ``.udf`` file, *NekRS* will first
> initialize them from the ``.par`` file and then overwrite them with values
> provided by ``uservp``. In other words, constants in ``.par`` serve only as
> defaults.

> **Warning:**
> If the fluid dynamic viscosity is spatially varying, the ``equation`` key
> must be set in the ``[PROBLEMTYPE]`` section of the ``.par`` file to enable
> the full stress formulation:
>
> .. code-block::
>
>    [PROBLEMTYPE]
>    equation = navierStokes + variableViscosity

## Conjugate Heat Transfer

For CHT cases (See conjugate heat transfer turorial)
*NekRS* provides a convenient way to define both fluid and solid properties for
the temperature equation in the Parameter file:

```
[FLUID VELOCITY]
density   = 1.0
viscosity = 1./1000.0

[SCALAR TEMPERATURE]
mesh = fluid+solid

transportCoeff      = 1.0         # fluid:  rho c_p
diffusionCoeff      = 1.0 / 500.0 # fluid:  k

transportCoeffSolid = 1.0         # solid:  rho c_p
diffusionCoeffSolid = 1.0 / 5.0   # solid:  k
```

The ``mesh`` key value ``fluid+solid`` informs *NekRS* that this is a conjugate
heat-transfer case and that the temperature equation is solved in both the
fluid and solid regions of the domain. For each scalar, the user can choose
whether it is solved only in the fluid (default) or in both fluid and solid via
the corresponding ``mesh`` key in its field section. Keys that end with
``Solid`` (e.g., ``transportCoeffSolid``, ``diffusionCoeffSolid``) specify the
corresponding material properties in the solid region.

As for variable properties, they can be set in ``uservp`` in a similar way. For
example:

```c++
#ifdef __okl__
@kernel void cFill(const dlong Nelements,
                   const dfloat CONST1,
                   const dfloat CONST2,
                   @ restrict const dlong *eInfo,
                   @ restrict dfloat *QVOL)
{
  for (dlong e = 0; e < Nelements; ++e; @outer(0)) {
    const dlong solid = eInfo[e];
    for (int n = 0; n < p_Np; ++n; @inner(0)) {
      const int id = e * p_Np + n;

      QVOL[id] = CONST1;
      if (solid) {
        QVOL[id] = CONST2;
      }
    }
  }
}
#endif

void uservp(double time)
{
  auto& fluid = nrs->fluid;
  auto& scalar = nrs->scalar;

  // Properties are constant in this example. Fill them only once.
  static int updateProperties = 1;

  if (updateProperties) {

    const dfloat rho = 1.0;
    const dfloat mue = 1.0 / 1000.0;

    { //fluid
      auto mesh = fluid->mesh;
      const auto o_mue = fluid->o_prop.slice(0 * fluid->fieldOffset);
      const auto o_rho = fluid->o_prop.slice(1 * fluid->fieldOffset);

      // For the fluid mesh there are no solid elements, so CONST2 is unused (0.0).
      cFill(mesh->Nelements, mue, 0.0, mesh->o_elementInfo, o_mue);
      cFill(mesh->Nelements, rho, 0.0, mesh->o_elementInfo, o_rho);
    }

    { //temperature
      auto mesh = scalar->mesh("temperature");
      const dfloat rhoCpFluid = rho * 1.0;
      const dfloat conFluid = mue;
      const dfloat rhoCpSolid = rhoCpFluid * 0.1;
      const dfloat conSolid = 10.0 * conFluid;

      const auto o_con = scalar->o_diffusionCoeff("temperature");
      const auto o_rhoCp = scalar->o_transportCoeff("temperature");

      // Use mesh->o_elementInfo to distinguish fluid (0) vs solid (1) elements
      cFill(mesh->Nelements, conFluid, conSolid, mesh->o_elementInfo, o_con);
      cFill(mesh->Nelements, rhoCpFluid, rhoCpSolid, mesh->o_elementInfo, o_rhoCp);
    }

    updateProperties = 0;
  }
}

void UDF_Setup()
{
  nrs->userProperties = &uservp;
}
```

In this example, a single custom kernel ``cFill`` populates all property arrays
for the fluid and temperature fields. The key ingredient in the CHT
setup is ``mesh->o_elementInfo``, which flags solid elements in a given mesh.
For the fluid mesh (``fluid->mesh``), all entries are zero, so ``cFill`` always
uses the "fluid" values. For the conjugate temperature mesh
(e.g., ``scalar->mesh("temperature")``), ``o_elementInfo`` distinguishes fluid
and solid elements, allowing the same kernel to assign different coefficients in
the two regions.

## Related knowledge-base notes
- [[Models & Source Terms]]
- [[Theory - Low-Mach Model]]
