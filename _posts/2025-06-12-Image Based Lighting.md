---
layout: post
title:  "Image Based Lighting"
date:   2025-06-12
---
Image Based Lighting is a technique used in computer graphics and rendering to light objects using the information from the images forming a sorrounding environment as one big light source. IBL uses an image, usually a high dynamic range image (HDRI) or a cubemap environment image, captured from real world to simulate realistic lighting environment.

IBL considers whole environment lighting not just the direct lighting, which makes it possible for objects to look more physically accurate. But with this environment lighting when we have to do the calculation for radiance using reflectance equation, we need to calculate the integral with multiple incoming light directions becuase of potential radiance from all directions. While for direct lighting, we already knew the limited light positions and the respected directions which make it easier to calculate because of radiance from only one direction.
$$
\begin{align*}
L_o(p, \omega_o) = \int_{\Omega} \left( k_d \frac{c}{\pi} + k_s \frac{DFG}{4(\omega_o \cdot n)(\omega_i \cdot n)}  \right) L_i(p, \omega_i) (n \cdot \omega_i) \, d\omega_i
\end{align*}
$$
