> Source: `source/theory.rst (Computational Approach + Incompressible/Stokes/Energy sections)` (converted to markdown; retains documentation detail)

# Theory

This page provides an overview of the governing equations available in *NekRS*. *NekRS* includes solvers for incompressible Navier-Stokes equation, a partially compressible low-Mach formulation, the Stokes equations, the :math:`k`-:math:`\tau` RANS equations and for general passive scalar advection-diffusion equation, including temperature equation.

## Computational Approach

The spatial discretization in *NekRS* is based on the spectral element method (SEM) [Patera1984]_, which is a high-order weighted residual technique similar to the finite element method.
In the SEM, the solution and data are represented in terms of :math:`N^{th}`-order tensor-product polynomials within each of :math:`E` deformable hexahedral (brick) elements.
Typical discretizations involve elements of order upto :math:`N=10` (corresponding to maximum of 1331 GLL points per element).
Vectorization and cache efficiency derive from the local lexicographical ordering within each macro-element and from the fact that the action of discrete operators, which nominally have :math:`O(EN^6)` nonzeros, can be evaluated in only :math:`O(EN^4)` work and :math:`O(EN^3)` storage through the use of tensor-product-sum factorization [Orszag1980]_.
The SEM exhibits very little numerical dispersion and dissipation, which can be important, for example, in stability calculations, for long time integrations, and for high Reynolds number flows.
We refer to [Denville2002]_ for more details.

*NekRS* solves the unsteady incompressible two-dimensional, axisymmetric, or three-dimensional Stokes or Navier-Stokes equations with heat transfer in both stationary (fixed) or time-dependent geometry.
It also solves the compressible Navier-Stokes in the Low Mach regime, and the magnetohydrodynamic equation (MHD).
The solution variables are the fluid velocity :math:`\mathbf u=(u_{x},u_{y},u_{z})`, the pressure :math:`p`, the temperature :math:`T`.
All of the above field variables are functions of space :math:`{\bf x}=(x,y,z)` and time :math:`t` in domains :math:`\Omega_f` and/or :math:`\Omega_s` defined in fig-walls.
Additionally *NekRS* can handle conjugate heat transfer problems.

![figure](_static/img/walls.png)

    Computational domain showing respective fluid and solid subdomains, :math:`\Omega_f` and
    and the solid boundary which is not shared by fluid is :math:`\overline{\partial\Omega_s}`,
    while the fluid boundary not shared by solid :math:`\overline{\partial\Omega_f}`.

## Incompressible Navier-Stokes Equations

The governing equations of incompressible flow in dimensional form are

```math
  \rho\left(\frac{\partial\mathbf u}{\partial t} +\mathbf u \cdot \nabla \mathbf u\right) = - \nabla p + \nabla \cdot \boldsymbol{\underline\tau} + \rho {\bf f} \,\, \quad \text{  (Momentum)  }

```

where :math:`\boldsymbol{\underline\tau}=\mu[\nabla \mathbf u+\nabla \mathbf u^{T}]` and :math:`\mathbf f` is a user defined acceleration.

```math
  \nabla \cdot \mathbf u =0 \,\, \quad \text{  (Continuity)  }

```

If the fluid viscosity is constant in the entire domain, the viscous stress tensor can be contracted using the Laplace operator.
Therefore, one may solve the Navier--Stokes equations in either the full-stress formulation

.. _sec:fullstress:

```math
 \nabla \cdot \boldsymbol{\underline\tau}=\nabla \cdot \mu[\nabla \mathbf u+\nabla \mathbf u^{T}]

```

or the no-stress formulation

.. _sec:nostress:

```math
 \nabla \cdot \boldsymbol{\underline\tau}=\mu\Delta \mathbf u

```

- Variable viscosity and RANS models require the full-stress tensor.
- Constant viscosity leads to a simpler stress tensor, which we refer to as the 'no-stress' formulation.

#### Non-Dimensional Navier-Stokes

Let us introduce the following non-dimensional variables :math:`\mathbf x^* = \frac{\mathbf x}{L}`, :math:`\mathbf u^* = \frac{u}{U}`, :math:`t^* = \frac{tU}{L}`, and :math:`\mathbf f^* =\frac{\mathbf f L}{U^2}`.
Where :math:`L` and :math:`U` are the (constant) characteristic length and velocity scales, respectively.
For the pressure scale we have two options:

- Convective effects are dominant i.e. high velocity flows :math:`p^* = \frac{p}{\rho_0 U^2}`
- Viscous effects are dominant i.e. creeping flows (Stokes flow) :math:`p^* = \frac{p L}{\mu_0 U}`,

where :math:`\rho_0` and :math:`\mu_0` are constant reference values for density and molecular viscosity, respectively.
For highly convective flows we choose the first scaling of the pressure and obtain the non-dimensional Navier-Stokes in the no-stress formulation:

```math
  \frac{\partial \mathbf{u^*}}{\partial t^*} + \mathbf{u^*} \cdot \nabla \mathbf{u^*}\ = -\nabla p^* + \frac{1}{Re}\Delta\mathbf u^* + \mathbf f^*.

```

For the full-stress formulation, we further introduce the dimensionless viscosity, :math:`\mu^*=\frac{\mu}{\mu_0}`, and obtain:

```math
  \frac{\partial \mathbf{u^*}}{\partial t^*} + \mathbf{u^*} \cdot \nabla \mathbf{u^*}\ = -\nabla p^* + \frac{1}{Re}\nabla \cdot \left[ \mu^* \left(\nabla\mathbf u^* + \nabla\mathbf u^{* T}\right)\right] + \mathbf f^*,

```

where :math:`\mathbf f^*` is the dimensionless user defined forcing function.
The non-dimensional number here is the Reynolds number :math:`Re=\frac{\rho_0 U L}{\mu_0}`.

## Stokes Flow

In the case of flows dominated by viscous effects *NekRS* can solve the reduced Stokes equations

```math
  \rho\left(\frac{\partial \mathbf u}{\partial t} \right) = - \nabla p + \nabla \cdot \boldsymbol{\underline\tau} + \rho {\bf f} \,\, , \text{in } \Omega_f \text{  (Momentum)  }

```

where :math:`\boldsymbol{\underline\tau}=\mu[\nabla \mathbf u+\nabla \mathbf u^{T}]` and

```math
  \nabla \cdot \mathbf u =0 \,\, , \text{in } \Omega_f  \text{  (Continuity)  }

```

As described earlier, we can distinguish between the stress and non-stress formulation according to whether the viscosity is variable or not.
The non-dimensional form of these equations can be obtained using the viscous scaling of the pressure.

## Energy Equation

In addition to the fluid flow, NekRS offers the capability to solve the energy equation, where temperature is treated as a passive scalar for an incompressible flow.

```math
  \rho c_{p} \left( \frac{\partial T}{\partial t} + \mathbf u \cdot \nabla T \right) =
     \nabla \cdot (\lambda \nabla T) + \dot{q} \,\,  \text{  (Energy)  } 

```

where, :math:`\lambda` is the thermal conductivity and :math:`c_p` is the specific heat at constant pressure.

#### Non-Dimensional Energy / Passive Scalar Equation

A similar non-dimensionalization as for the flow equations using the non-dimensional variables :math:`\mathbf x^* = \frac{\mathbf x}{L}`,  :math:`\mathbf u^* = \frac{u}{U}`, :math:`t^* = \frac{tU}{L}`, :math:`T^*=\frac{T-T_0}{\delta T}`, and :math:`\lambda^*=\frac{\lambda}{\lambda_0}` leads to

```math
  \frac{\partial T^*}{\partial t^*} + \mathbf u^* \cdot \nabla T^* =
    \frac{1}{Pe} \nabla \cdot \lambda^* \nabla T^* + q^* \,\,  \text{  (Energy)  } 

```

where :math:`q^*=\frac{\dot{q} L}{\rho_0 c_{p_0} U \delta T}` is the dimensionless user defined source term.
The non-dimensional number here is the Peclet number, :math:`Pe=\frac{\rho_0 c_{p_0} U L}{\lambda_0}`.

## Related knowledge-base notes
- [[Theory - Low-Mach Model]]
- [[Theory - RANS Models]]
