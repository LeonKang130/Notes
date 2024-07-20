# Spatiotemporal Variance-Guided Filtering

# Introduction

We extend Dammertz et al.'s hierarchical wavelet-based reconstruction filter to output temporally stable global illumination, including diffuse and glossy interreflections, and soft shadows from a stream of one sample per pixel images.

## Reconstruction Pipeline

Our path tracer utilizes rasterization to efficiently generate primary rays, which provides a noise-free G-buffer containing additional surface attributes used to steer our reconstruction filter. Our path tracer outputs direct and indirect illumination separately.

We first demodulate surface albedo of directly visible surfaces from our sample colors, which avoids our filter having to prevent overblurring of high-frequency texture details.

Our reconstruction performs three main steps:

-   temporally accumulating our 1 spp path-traced inputs to increase effective sampling rate
-   using these temporally augmented color samples to estimate local luminance variance
-   using these variance estimates to drive a hierarchical a-trous wavelet filter

After reconstruction, we modulate the filter output with the surface albedo.

## Spatiotemporal Filter

Our G-buffer contains depth, object- and world-space normals, mesh ID, and screen-space motion vectors generated from a rasterization pass for primary visibility. Our history buffers include temporally integrated color and color moment data along with the prior frame's depths, normals and mesh IDs.

### Temporal Filtering

As in TAA, we require a 2D motion vector associated with each color sample $C_i$ for frame $i$. For each $C_i$ we backproject to $C_{i-1}$ from the color history buffer, and compare the two sample's depths, object-space normals and mesh IDs to determine if they are consistent. Consistent samples are merged using an exponential moving average:
$$
C_i'=\alpha\cdot C_i+(1-\alpha)\cdot C_{i-1}'
$$
We use $\alpha=0.2$. Our motion vector handles camera and rigid object motion.

To improve image quality, we resample $C_{i-1}$ by using a $2\times2$ tap bilinear filter. Each tap individually tests backprojected depths, normals and mesh IDs. If no taps remain consistent, we try a larger $3\times3$ filter. If we still fail, the sample represents a disocclusion, so we discard the temporal history.

### Variance Estimation

We estimate per-pixel luminance variance using $\mu_1$ and $\mu_2$, the first and second raw moments of color luminance.

