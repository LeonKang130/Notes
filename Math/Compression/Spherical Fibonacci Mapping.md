# Spherical Fibonacci Mapping

## Introduction

Spherical Fibonacci point sets are a well-known approach to generate a very uniform sampling of the sphere. In contract to other spherical sample distributions, the means by which to find the closest sample point for a given point on the unit sphere remain cumbersome. In this paper, we present an analytic and efficient solution for this problem. We show, how unit vectors can be quantized very efficiently, how spherical functions can be stored with highly improved uniform cell sizes, and how we can apply the inverse mapping to the field of procedural modeling.

## Spherical Fibonacci Point Sets

The construction rules for these sets are straightforward when using spherical coordinates $(\phi,\theta)^T$, for which we use the common parameterization:
$$
P(\phi,\theta)=(\cos(\phi)\sin(\theta),\sin(\phi)\sin(\theta),\cos(\theta))^T
$$
We denominate the points on the sphere using parameters $(\phi,z=\cos\theta)^T$.

A point with index $i$ of an SF point set with $n$ samples is given as:
$$
\textbf{SF}_i^n=P(\phi_i,\cos^{-1}(z_i))\\
\phi_i=2\pi[\frac{i}{\Phi}],\,z_i=1-\frac{2i+1}{n},\,i\in\{0,\dots,n-1\}
$$
In parameter space, this grid is similar to a Rand-1-Lattice in 2D, which has a generative vector but forms a 2D lattice due to the warp-around.

We derive the zone number for our parameterization by expressing the basis vector in terms of local Cartesian coordinates:
$$
\hat{b}_k=(-(-1)^k2\pi\Phi^{-k}\sqrt{1-z^2},\frac{-2F_k}{n}\frac{1}{\sqrt{1-z^2}})^T
$$
We use the approximation $F_k\approx\Phi^k/\sqrt{5}$. The real-valued zone number $\hat{k}$ is:
$$
\hat{k}=\log_{\Phi^2}(\sqrt{5}n\pi\sin^2\theta)=\log_{\Phi^2}(\sqrt{5}n\pi(1-z^2))
$$

## Inverse Mapping

An GLSL implementation of the inverse mapping:

```glsl
#define madfrac(A, B) mad((A), (B), -floor((A) * (B)))
float inverseSF(vec3 p, float n) {
    float phi = min(atan2(p.y, p.x), PI), cosTheta = p.z;
    float k = max(2, floor(log(n * PI * sqrt(5) * (1 - cosTheta * cosTheta)) / log(PHI * PHI)));
    float Fk = pow(PHI, k) / sqrt(5);
    float F0 = round(Fk), F1 = round(Fk * PHI);
    mat2 B = mat2(2 * PI * madfrac(F0 + 1, PHI - 1) - 2 * PI * (PHI - 1),
                  2 * PI * madfrac(F1 + 1, PHI - 1) - 2 * PI * (PHI - 1),
                  -2 * F0 / n,
                  -2 * F1 / n);
    mat2 invB = inverse(B);
    vec2 c = floor(mul(invB, vec2(phi, cosTheta - (1 - 1 / n))));
    float d = INFINITY, j = 0;
    for (uint s = 0; s < 4; ++s) {
        float cosTheta = dot(B[1], vec2(s % 2, s / 2) + c) + (1 - 1 / n);
        cosTheta = clamp(cosTheta, -1, 1) * 2 - cosTheta;
        float i = floor(n * 0.5 - cosTheta * 0.5);
        float phi = 2 * PI * madfrac(i, PHI - 1);
        cosTheta = 1 - (2 * i + 1) * rcp(n);
        float sinTheta = sqrt(1 - cosTheta * cosTheta);
        vec3 q = vec3(cos(phi) * sinTheta, sin(phi) * sinTheta, cosTheta);
        float squaredDistance = dot(q - p, q - p);
        if (squaredDistance < d) {
            d = squaredDistance;
            j = i;
        }
    }
    return j;
}
```

