---
layout: post
title:  "Physically Based Rendering"
date:   2025-06-11
---
Physically Based Rendering (PBR) is a rendering technique which aims to simulate the real world behavior of light to produce more realistic scene. It not only considers the light behavior but also the material properties like its metalness and roughness.

# Core Principles of PBR

## Microfacet theory
Surfaces at a microscopic level are made of many tiny reflective mirrors called as microfacets. The alignments of these tiny mirrors aka microfacets defines the roughness of a surface.

So, on rougher surfaces, the incoming light rays will scatter in different random directions, resulting in widespread reflection. While for smooth surfaces, it will be a small and sharp reflection. 

It can also be understood using halfway vector, *h*, which is just in between light *l* and view *v* vector.

<!-- ![Halfway Vector](/assets/images/halfwayVector.png) -->
<div style="text-align: center;">
  <img src="/assets/images/halfwayVector.png" alt="Halfway Vector" style="width: 100%; max-width: 500px; height: auto;"/>
</div>
<br/>
So, more the number of microfacets aligned to the halfway vector, the sharper and stronger reflection will be there and vice versa.

## Energy conservation 
If we think light as a form of energy here and when it gets pointed to a surface, the amount of light energy getting reflected should never exceed the amount of incoming light energy. Because the some amount of energy will get conserved by the surface.

The moment a light ray hits the surface, it gets divided into 2 parts, one which gets reflected known as specular light and the other which gets inside the surface (refracts) known as diffuse light. 

Even though in real world, the refracted part of light ray not remains totally inside the object. It keeps getting collided with different parts of the object material until it looses its all energy. In this process, some part also gets reflected from different point of surface. For our case, we are assuming that all of the refracted amount of light will get absorbed completely. 
As the reflected amount of energy will never enter again the surface unless reflected and direct back to the same surface, the energy left for refracted light is total minus the reflected light.

## Surface Interaction Type or BRDF
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

### Bidirectional Reflectance Distribution Function (BRDF)
Bidirectional Reflectance Distribution Function (BRDF) is used for simulating the appearances of surfaces under lighting. For this application, we are going to use Cock-Torrance BRDF, which is well known and used in all real-time PBR pipelines. It incorporates specular reflection, diffuse reflection and fresnel effects:

$$
\begin{align*}
f_r = k_d.f_{lambert} + k_s.f_{cookTorrance}
\end{align*}
$$

Here $k_d$ is ratio of light that gets refracted while $k_s$ is the ratio that gets reflected. $f_{lambert}$ is the **Lambertian Diffuse** which is a constant denoted as:

$$
\begin{align*}
f_{lambert} = \frac{c}{\pi}
\end{align*}
$$

where $c$ is the surface Albedo (color). We divide it by $\pi$ to normalize as we already did integral over the hemisphere's solid angle.

The specular part of BRDF, $f_{cookTorrance}$ is defined as following:

$$
\begin{align*}
f_{cookTorrance} = \frac{D.F.G}{4.(\omega_o \cdot n).(\omega_i \cdot n)}
\end{align*}
$$

Here,

$D$ is a <ins>**Normal Distribution function**</ins> that calculates the amount of microfacets which are aligned to halfway vector. We will use a very standard NDF used by mutiple game engines for PBR, Trowbridge-Reitz GGX. It is expressed as:

$$
\begin{align*}
NDF_{GGXTR}(n, h, \alpha) = \frac{\alpha^2}{\pi ((\mathbf{n} \cdot \mathbf{h})^2 (\alpha^2 - 1) + 1)^2}
\end{align*}
$$

where $h$ is halfway vector and $\alpha$ is surface roughness.

In GLSL, we can convert this function as follows:
```GLSL
// Distribution function: GGX
float DistributionGGX(vec3 N, vec3 H, float ROUGHNESS)
{
    float a = ROUGHNESS * ROUGHNESS;
    float a2 = a * a;
    float NdotH = max(dot(N, H), 0.0);
    float NdotH2 = NdotH * NdotH;

    float num = a2;
    float denom = (NdotH2 * (a2 - 1.0) + 1.0);
    denom = PI * denom * denom;

    return num / denom;
}
```
<br/>

$G$ is <ins>**Geometry function**</ins>, which calculates how many microfacets are visible from both the view and light directions as sometimes due to roughness some microfacets blocks other microfacets to reflect the light. We will again use a very standard function, Smith's Schlick-GGX to calculate it.

$$
\begin{align*}
G_{SchlickGGX}(n, v, k) = \frac{n \cdot v}{(n \cdot v) (1 - k)+k}
\end{align*}
$$

where k is kinda similar to $\alpha$ (ROUGHNESS) but with some standard changes depending on direct or IBL based lighting:

$$
\begin{align*}
k_{direct} = \frac{(\alpha +1)^2}{8}
\\
k_{IBL} = \frac{\alpha^2}{2}
\end{align*}
$$


Similarly, we will calculate Geometry function for light direction, $G_{SchlickGGX}(n, l, k)$ and then combine it with $G_{SchlickGGX}(n, v, k)$.

$$
\begin{align*}
G(n, v, l, k) = G_{SchlickGGX}(n, v, k) * G_{SchlickGGX}(n, l, k)
\end{align*}
$$

In GLSL, we will convert it to code like as follows:
```GLSL
// Geometry function: Smith
float GeometrySchlickGGX(float NdotV, float ROUGHNESS)
{
    float r = (ROUGHNESS + 1.0);
    float k = (r * r) / 8.0;

    float num = NdotV;
    float denom = NdotV * (1.0 - k) + k;

    return num / denom;
}
float GeometrySmith(vec3 N, vec3 V, vec3 L, float ROUGHNESS)
{
    float ggx2 = GeometrySchlickGGX(max(dot(N, V), 0.0), ROUGHNESS);
    float ggx1 = GeometrySchlickGGX(max(dot(N, L), 0.0), ROUGHNESS);

    return ggx1 * ggx2;
}
```
<br/>

$F$ is <ins>**Fresnel equation**</ins>, which tells the ratio of amount of light which gets reflected versus the amount that gets refracted. It also varies over the angle we are looking at that surface. We can denote it using Fresnel-Schlick approximation:

$$
\begin{align*}
F_{Schlick}(h,v,F_0) = F_0 + (1 - F_0)(1-(h\cdot v))^5
\end{align*}
$$

where $F_0$ is base reflectivity. As many game engines have adopted 0.04 as a standard for dielectrics, we will also take $F_0$ as 0.04.

In GLSL, we will implement it as following:
```GLSL
// Calculate reflectance at normal incidence
vec3 F0 = vec3(0.04);
F0 = mix(F0, Albedo, METALNESS);
...
vec3 fresnelTerm = fresnelSchlick(max(dot(H, V), 0.0), F0);
```
```GLSL
// Fresnel-Schlick approximation
vec3 fresnelSchlick(float cosTheta, vec3 F0)
{
    return F0 + (1.0 - F0) * pow(1.0 - cosTheta, 5.0);
}
```
### Radiance
Now we have to calculate $L_i(p, \omega_i)$, which is the incoming radiance at point $p$ from direction $\omega_i$. It is determined by the light source's position, intensity and direction.

In case of a direct lighting, we can say radiance function, $L_i(p, \omega_i)$ calculates light contribution at point $p$, taking into account light attenuation due to distance and relative angle between surface normal, $\mathbf{n}$ and incoming light direction $\omega_i$.

We can integrate it in our fragment shader code as following:
```GLSL
//currPos is world Position here
void main(){
  vec3 N = normalize(Normal);
  vec3 V = normalize(camPos - currPos);
  ...
  vec3 L = normalize(lightPositions[i] - currPos); // light direction
  vec3 H = normalize(V + L); // halfway vector
  float distance = distance(lightPositions[i], currPos);

  float attenuation = 1.0 / distance; // its correct to use 1 / (distance * distance), but for more visual effect I'm using (1 / distance)
  vec3 radiance = attenuation * lightColors[i]; // we will add angular dependency in final lighting calculation
  ...
}
```
Finally we will combine all calculations of Cook-Torrance BRDF and add with *directLighting* as following:
```GLSL
// Cook-Torrance BRDF
float NDF = DistributionGGX(N, H, ROUGHNESS);
float G = GeometrySmith(N, V, L, ROUGHNESS);
vec3 fresnelTerm = fresnelSchlick(max(dot(H, V), 0.0), F0);

vec3 numerator = NDF * G * fresnelTerm;
float denominator = 4.0 * max(dot(N, V), 0.0) * max(dot(N, L), 0.0) + 0.001; // adding 0.001 so we will not divide numerator by 0 
vec3 specular = numerator / denominator;

vec3 kS = fresnelTerm; // specular reflection
vec3 kD = vec3(1.0) - kS; // diffuse reflection
kD *= 1.0 - METALNESS;

float NdotL = max(dot(N, L), 0.0);

directLighting += (kD * Albedo / PI + specular) * radiance * NdotL;
```
We can return FragColor with this *directLighting* or mix it with **Albedo**.
```GLSL
vec3 ambient = vec3(0.03) * Albedo * ao;
vec3 color = directLighting + ambient;
...
FragColor = vec4(color, 1.0);
```
Now you will be able to see the spheres with direct lighting and PBR enabled. It will look more realistic and physically based. As, I have 9 lights and they are moving in a circluar path with variable speed, you define the number of lights, their positions, colors and everything as you want. I'll just share the output I got.

![PBR with direct lighting](/assets/images/PBR_direct_lighting.png)

So, we are done with PBR with direct lighting here. In next blog we will cover our last topic Image Based Lighting (IBL).

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