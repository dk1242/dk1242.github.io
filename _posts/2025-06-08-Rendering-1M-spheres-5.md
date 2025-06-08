---
layout: post
title:  "Rendering 1 Million spheres: Part 5 (Physically Based Rendering)"
date:   2025-05-27
---
Physically Based Rendering (PBR) is a rendering technique which aims to simulate the real world behavior of light to produce more realistic scene. It not only considers the light behavior but also the material properties like its metalness and roughness.

## Core Principles of PBR

### Microfacet theory
Surfaces at a microscopic level are made of many tiny reflective mirrors called as microfacets. The alignments of these tiny mirrors aka microfacets defines the roughness of a surface.

So, on rougher surfaces, the incoming light rays will scatter in different random directions, resulting in widespread reflection. While for smooth surfaces, it will be a small and sharp reflection.

### Energy conservation 
If we think light as a form of energy here and when it gets pointed to a surface, the amount of light energy getting reflected should never exceed the amount of incoming light energy. Because the some amount of energy will get conserved by the surface.

The moment a light ray hits the surface, it gets divided into 2 parts, one which gets reflected known as specular light and the other which gets inside the surface (refracts) known as diffuse light. 

Even though in real world, the refracted part of light ray not remains totally inside the object. It keeps getting collided with different parts of the object material until it looses its all energy. In this process, some part also gets reflected from different point of surface. For our case, we are assuming that all of the refracted amount of light will get absorbed completely. 
As the reflected amount of energy will never enter again the surface unless reflected and direct back to the same surface, the energy left for refracted light is total minus the reflected light.

### Surface Interaction Type or BRDF
Now we need to consider the type of surface which will decide how much each individual light ray will contribute to the final reflected light of the surface given its material properties. So if its a smooth surface, light will reflect based on viewing angle and for a rough surface it will scatter in different directions.

But for its calculation, we need to know some standard PBR equation which we kinda use in the real world. This is known as **reflectance equation**:
$$
\begin{align*}
L_o(p, \omega_o) = \int_{\Omega} f_r(p, \omega_i, \omega_o)\, L_i(p, \omega_i)\, (\mathbf{n} \cdot \omega_i)\, d\omega_i
\end{align*}
$$
To really understand PBR, we have to understand this equation truly.

This equation calculates the outgoing radiance (magnitude or strength of light) at a point $p$ in direction $\omega_o$. It integrates all the incoming light from all direction $\omega_i$ over the hemisphere $\Omega$ above the point $p$, including material's reflectance and surface orientations.
* $f_r(p, \omega_i, \omega_o)$ is Bidirectional Reflectance Distribution Function (BRDF) which tells how much of the incoming light from direction $\omega_i$ is reflected toward $\omega_o$, depending on the material at $p$.
* $L_i(p, \omega_i)$ is the incoming radiance at point $p$ from direction $\omega_i$.
* $(\mathbf{n} \cdot \omega_i)$ is the dot product and cosine of the angle between surface normal $\mathbf{n}$ and incoming direction $\omega_i$. We consider it to follow the law that light is weaker the less it directly radiates onto the surface, and strongest when it is directly perpendicular to the surface. It scales the energy based on the light's incidence angle to the surface.
* $d\omega_i$ is the very small solid angle, which will get integrated over the hemisphere.



<script>
window.MathJax = {
  tex: {
    inlineMath: [['$', '$'], ['\\(', '\\)']],
    displayMath: [['$$', '$$'], ['\\[', '\\]']]
  }
};
</script>
<script type="text/javascript" id="MathJax-script" async
  src="https://cdn.jsdelivr.net/npm/mathjax@3/es5/tex-mml-chtml.js">
</script>