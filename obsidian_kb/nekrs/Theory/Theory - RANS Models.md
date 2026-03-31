> Source: `source/theory.rst (RANS sections)` (converted to markdown; retains documentation detail)

## RANS Models

For turbulence modeling *nekRS* offers the two-equation :math:`k`-:math:`\tau` RANS model [Tombo2025]_ and its SST and DES variants [Kumar2024]_.
Linear two-equation RANS models rely on the Bousinessq approximation which relates the Reynolds stress tensor to the mean strain rate, :math:`\boldsymbol{\underline {S}}`, linearly through eddy viscosity.
The time-averaged momentum equation is given as,

```math
 \rho \left(\frac{\partial \mathbf u}{\partial t} + \mathbf u \cdot \nabla \mathbf u \right) &=
 - \nabla p + \nabla \cdot \left[ (\mu + \mu_t)
 \left( 2 \boldsymbol{\underline S} -
 \frac{2}{3} Q \boldsymbol{\underline I}\right) \right] \\
 \boldsymbol{\underline S} &= \frac{1}{2} \left( \nabla \mathbf u + \nabla\mathbf{u}^T \right)

```

where :math:`\mu_t` is the turbulent or eddy viscosity and :math:`\boldsymbol{\underline I}` is an identity tensor.
Currently, nekRS only supports incompressible flow where the divergence constraint, :math:`Q`, is zero,

```math
```

	Q = \nabla \cdot \mathbf u = 0

In two-equation models, the description of the local eddy viscosity is given by two additional transported variables, which provide the velocity and length (or time) scale of turbulence. 
The velocity scale is given by turbulent kinetic energy, :math:`k`, while the choice of the second variable, which provides the length or time scale, depends on the specific two-equation model used. In the :math:`k`-:math:`\tau` model, the second transported variable is :math:`\tau`, which is the inverse of the specific dissipation rate :math:`\omega`, and it provides the local time scale of turbulence.

The :math:`k-\tau` model offers certain favorable characteristics over the :math:`k-\omega` model [Wilcox2008]_, including bounded asymptotic behavior of :math:`\tau` and its source terms and favorable near-wall gradients.
These make it especially suited for high-order codes and complex geometries.
It is, therefore, the preferred two-equation RANS model in NekRS.
The :math:`k-\tau` transport equations are,

```math
\rho\left( \frac{\partial k}{\partial t} + \mathbf u \cdot \nabla k\right) & =
\nabla \cdot (\Gamma_k \nabla k) + P_k - \rho \beta^* \frac{k}{\tau} \\
\rho\left( \frac{\partial \tau}{\partial t} + \mathbf u \cdot \nabla\tau\right) & =
\nabla \cdot (\Gamma_\tau \nabla \tau) - \alpha \frac{\tau}{k}P_k + \rho \beta -
2\frac{\Gamma_\tau}{\tau} (\nabla \tau \cdot \nabla \tau) + C_{D_\tau}

```

The diffusion terms are given by

```math
\Gamma_k & = \mu + \frac{\mu_t}{\sigma_k} \\
\Gamma_\tau & = \mu + \frac{\mu_t}{\sigma_\tau}

```

where, in the :math:`k-\tau` model the eddy viscosity is given by,

```math
\mu_t = \rho k \tau

```

The production term is given by

```math
P_k = \mu_t\left( \boldsymbol{\underline S : \underline S} \right)

```

where ":math:`\boldsymbol :`" denotes the double tensor contraction operator.
The final term in the :math:`\tau` equation is the cross-diffusion term, introduced by [Kok2000]_,

```math
:label: ktau_cd

C_{D_\tau} =(\rho \sigma_d \tau) \text{min}(\nabla k \cdot \nabla \tau,0)

```

The above term is especially relevant for external flows.
It eliminates non-physical free-stream dependence of the near-wall :math:`\tau` field (see [Tombo2025]_ for details).

All coefficients in the :math:`k-\tau` model are identical to the standard :math:`k-\omega` model [Wilcox2008]_, given as,

```math
\beta = 0.0708; \,\, \beta^*=0.09; \,\, \alpha=0.52; \,\, \sigma_k= \frac{1}{0.6} \,\, \sigma_\tau=2.0; \,\, \sigma_d=\frac{1}{8}

```

Further theoretical and implementation details on the :math:`k`-:math:`\tau` model can be found in [Tombo2025]_.

> **Note:**

  NekRS currently offers only wall resolved RANS models. The boundary condition for both :math:`k` and :math:`\tau` transport equations for wall resolved RANS are of Dirichlet type and equal to zero.

## Related knowledge-base notes
- [[Models & Source Terms]]
- [[Tutorial - RANS Channel (k-tau SST)]]
