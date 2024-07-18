# Edge-Avoiding A-Trous Wavelet Transform for fast Global Illumination Filtering

## Overview

Our method takes a noising Monte Carlo path traced full resolution image and produces a smooth output image by taking into account the input image itself as well as a normal and a position buffer.

## Edge-Avoiding A-Trous Filtered Path Tracing

In the absence of edges, it applies an even smoothing with increasing filter size per iteration. In the presence of edges the influence of neighboring samples is diminished by the edge-stopping function.

Each iteration of the A-Trous algorithm corresponds to computing one more level of the wavelet analysis which also corresponds to doubling the filter size.

The algorithm of computing discrete wavelet transform using approximation of wide gaussian kernels via *generating kernels* is given as follows:

1.   At level $i=0$ we start with the input signal $c_0(p)$
2.   $c_{i+1}(p)=c_i(p)*h_i$, where $*$ is the discrete convolution. The distance between the entries in the filter $h_i$ is $2^i$.
3.   $d_i(p)=c_{i+1}(p)-c_i(p)$, where $d_i$ are the detail or wavelet coefficients of level $i$.
4.   if $i<N$: increment $i$, go to step 2
5.   $\{d_0,d_1,\dots,d_{N-1},c_N\}$ is the wavelet transform of $c$.

The reconstruction is given by
$$
c=c_N+\sum_{i=N-1}^0 d_i
$$
Filter $h$ is based on a $B_3$ spline interpolation $(\frac{1}{16},\frac{1}{4},\frac{3}{8},\frac{1}{4},\frac{1}{16})$. At each level $i>0$ the filter doubles its extent by filling in $2^{i-1}$ zeros between the initial entries. The number of non-zero entries remains constant.

Edge-avoiding filtering is achieved by introducing a data-dependent weighting function. The discrete convolution in step 2 becomes
$$
c_{i+1}(p)=\frac{1}{k}\sum_{q\in\Omega}h_i(q)\cdot w(p,q)\cdot c_i(p)
$$
where $k$ is the normalization factor.

$w$ are the combined edge-stopping functions from the ray-traced input image, normal buffer and position buffer.
$$
w(p,q)=w_{rt}\cdot w_n\cdot w_x
$$

$$
w_{rt}(p,q)=e^{(-\frac{||I_p-I_q||}{\sigma^2_{rt}})}
$$

$w_x$ and $w_n$ are computed in the same way with their own $\sigma_x$ and $\sigma_n$ respectively.

Introduction of edge-stopping functions leads to the presence of edges at the coarser levels of the transformation. Therefore we discard the finer levels of the wavelet transformation, and directly use level $N$ as output image.

## Implementation Details

After computing the input buffers, the filter is applied multiple times to the rt buffer. Each pass uses the previously smoothed results as input.

When the support for arbitrary-formed light is not required, the direct light computation can be done using shadow maps or shadow volumes.

```glsl
uniform sampler2D colorMap, normalMap, posMap;
uniform float c_phi, n_phi, p_phi, stepWidth;
uniform float kernel[25];
uniform vec2 offset[25];

void main(void) {
    vec4 sum = vec4(0.);
    vec2 step = vec2(1./512., 1./512.);
    vec4 cval = texture2D(colorMap, gl_TexCoord[0].st);
    vec4 nval = texture2D(normalMap, gl_TexCoord[0].st);
    vec4 pval = texture2D(posMap, gl_TexCoord[0].st);
    
    float cum_w = 0.;
    for(int i = 0; i < 25; i++) {
        vec2 uv = gl_TexCoord[0].st + offset[i] * step * stepWidth;
        vec4 ctmp = texture2D(colorMap, uv);
        vec4 t = cval - ctmp;
        float dist2 = dot(t, t);
        float c_w = min(exp(-dist2 / c_phi), 1.);
        vec4 ntmp = texture2D(normalMap, uv);
        t = nval - ntmp;
        dist2 = max(dot(t, t) / (stepWidth * stepWidth), 0.);
        float n_w = min(exp(-dist2 / n_phi), 1.);
        vec4 ptmp = texture2D(posMap, uv);
        t = pval - ptmp;
        dist2 = dot(t, t);
        float p_w = min(exp(-dist2 / p_phi), 1.);
        float weight = c_w * n_w * p_w;
        sum += ctmp * weight * kernel[i];
        cum_w += weight * kernel[i];
    }
    gl_FragData[0] = sum / cum_w;
}
```





