> Source: `source/theory.rst (Low-Mach sections)` (converted to markdown; retains documentation detail)

## Low-Mach Compressible Flow Equations

The low-Mach compressible equations are derived from the fully compressible Navier-Stokes equations by filtering the acoustic waves, obtained by splitting the pressure into thermodynamic, :math:`p_t`, and hydrodynamic components :math:`p_1`. The resulting low-Mach compressible governing equations, in dimensional form, are (for complete derivation refer [Tombo1997]_ or [Paulucci1982]_)

```math
\nabla \cdot \mathbf{u} &= \beta_T \frac{D T}{D t} - \kappa \frac{D p_{t}}{D t} = Q\\
\rho \left(\frac{\partial \mathbf{u}}{\partial t} + \mathbf{u} \cdot \nabla \mathbf{u} \right) &= -\nabla p_1 + \nabla \cdot \mu \left(2 \boldsymbol{\underline{S}} - \frac{2}{3} Q \boldsymbol{\underline{I}} \right) + \rho \mathbf{f} \\
\rho c_p \frac{D T}{D t} &= \nabla \cdot \lambda \nabla T + \dot{q} + \frac{D p_t}{D t}

```

where, :math:`\boldsymbol{\underline{S}} = \frac{1}{2}(\nabla \mathbf{u} + \nabla \mathbf{u}^T)`, :math:`Q` is the divergence, :math:`\boldsymbol{\underline{I}}` is the identity tensor and :math:`\dot{q}` is the volumetric heat source term.
Thermodynamic pressure is the leading order, spatially invariant, term in pressure expansion while hydrodynamic pressure is the first order term. :math:`\beta_T` is the isobaric expansion coefficient and :math:`\kappa` is the isothermal expansion coefficient,

```math
\beta_T &= \frac{1}{\rho} \left.\frac{D \rho}{D t}\right|_p \\
\kappa &= \frac{1}{\rho} \left.\frac{D \rho}{D t}\right|_T

```

> **Note:**

> **Note:**

  For an open domain, the thermodynamic pressure is both spatially and temporally constant, i.e. :math:`dp_t/dt = 0`. This further simplifies the above equation system. However, for a closed system, the thermodynamic pressure, although uniform in space, is subject to changing temporally to enforce mass conservation.

The equation system above is not closed and an equation of state (EOS) is required to relate the density to the thermodynamic quantities, :math:`\rho = f(p_t,T)`. Further, dynamic viscosity and thermal conductivity also need to be provided by constitutive relations (e.g., Sutherland's law for gases [Sutherland1893]_).

Introducing the non-dimensional variables as follows,

```math
\mathbf{u}^* = \frac{\mathbf{u}}{U}; \,\, T^* = \frac{T}{T_0}; \,\, \vec{x}^* = \frac{\vec{x}}{L};\,\, p_1^* = \frac{p_1}{\rho U^2};\,\, p_t^* = \frac{p_t}{p_0};\,\, t^* = \frac{t U}{L}; \vec{f}^* = \frac{\vec{f}}{f_0} \\
\rho^* = \frac{\rho}{\rho_0}; \,\, c_p^* = \frac{c_p}{c_{p0}}; \,\, \lambda^* =\frac{\lambda}{\lambda_0}; \,\, \mu^* = \frac{\mu}{\mu_0}; \,\, \beta_T^* = \frac{\beta_T}{\beta_0}; \,\, \kappa^* = \frac{\kappa}{\kappa_0}; \,\, \dot{q}^* = \frac{\dot{q} L}{\rho_0 c_{p0} T_0 U} 

```

the low-Mach governing equations are obtained as follows. The continuity equation: 

```math
\nabla \cdot \mathbf{u}^* = \beta_0 T_0 \beta_t^* \frac{D T^*}{D t^*} - \kappa_0 p_0 \kappa^* \frac{d p_t^*}{dt^*} = Q^*

```

mometum equation,

```math
\rho^* \left(\frac{\partial \mathbf{u}^*}{\partial t^*} + \mathbf{u}^* \cdot \nabla \mathbf{u}^*\right) = - \nabla p_1^* + \nabla \cdot \frac{\mu^*}{Re} \left(2 \boldsymbol{\underline{S}}^* - \frac{2}{3} Q^* \boldsymbol{\underline{I}}\right) + \frac{1}{Fr} \rho^* \mathbf{f}^*

```

and energy equation,

```math
\rho^* c_p^* \frac{D T^*}{D t^*} = \nabla \cdot \frac{\lambda^*}{Re Pr} \nabla T^* + \dot{q}^* + \frac{p_0}{\rho_0 c_{p0} T_0} \frac{d p_t^*}{d t^*}

```

where :math:`U` and :math:`L` are the characteristic velocity and length scales. :math:`f_0` is reference magnitude of body force.

The equations are closed by corresponding EOS in non-dimensional form, :math:`\rho^* = f(p_t^*,T^*)`.
The above equations represent the lowMach equations in the most general form, applicable to real gases.
Depending on the target application and associated assumptions, several simplifications to the equations are possible.
In the subsequent section, we discuss the simplifications corresponding to the most commonly employed assumption, i.e., ideal gas assumption.

#### Low-Mach Equations with Ideal Gas Assumption

The EOS for an ideal gas is,

```math
p_t = \rho R T; \,\, c_p-c_v = R \equiv \frac{R}{c_p} = \frac{\gamma - 1}{\gamma}

```

where :math:`R` is the ideal gas constant, :math:`c_v` is the specific heat at constant volume and :math:`\gamma = c_p/c_v` is the isentropic expansion factor.
In non-dimensional form, considering the properties at reference conditions for non-dimensionalization (i.e., :math:`p_0 = \rho_0 R T_0` and :math:`\frac{R}{c_{p0}}= \frac{\gamma_0-1}{\gamma_0}`), the EOS is simply written,

```math
p_t^* = \rho^* T^*

```

The expansion coefficients, derived from the EOS, in non-dimensional form are,

```math
\beta_T^* = \frac{1}{T^*} \,\, \kappa^* = \frac{1}{p_t^*}

```

The resulting governing equations for ideal gas assumption, thus, are,

```math
\nabla \cdot \mathbf{u}^* &= \frac{1}{T^*} \frac{D T^*}{D t^*} - \frac{1}{p_t^*} \frac{d p_t^*}{dt^*} = Q^* \\
\rho^* \left(\frac{\partial \mathbf{u}^*}{\partial t^*} + \mathbf{u}^* \cdot \nabla \mathbf{u}^*\right) &= - \nabla p_1^* + \nabla \cdot \frac{\mu^*}{Re} \left(2 \boldsymbol{\underline{S}}^* - \frac{2}{3} Q^* \boldsymbol{\underline{I}}\right) + \frac{1}{Fr} \rho^* \mathbf{f}^* \\
\rho^* c_p^* \frac{D T^*}{D t^*} &= \nabla \cdot \frac{\lambda^*}{Re Pr} \nabla T^* + \dot{q}^* + \frac{\gamma_0-1}{\gamma_0} \frac{d p_t^*}{d t^*}

```

> **Note:**

  For a calorically perfect ideal gas, :math:`c_p` will be constant and non-dimensional :math:`c_p^* = 1`.

> **Note:**

  Another often used assumption is to consider dynamic viscosity and thermal conductivity independent of temperature (constant). Thus, :math:`\mu^*` and :math:`\lambda^*` will both be unity, further simplifying the above equations.

## Related knowledge-base notes
- [[Models & Source Terms]]
- [[Physical Properties]]
