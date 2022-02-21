---
layout: post
date: 2022-02-01
title: Multi-scale Horn Schunck on the GPU
category: Shaders
tags: [Computer Vision, Optimization]
---

In this post, I address a shader implementation of Horn-Schunck's optical flow in 23 draw-calls.

## Limitations

### Assumptions

The original optical flow algorithm assumes the scene has brightness constancy and small changes. Therefore, it breaks on large movement (i.e. low-fps footage) and scenes with varying illumination.

### Iterations

Pixel shaders are not known for being the best for traditional stationary iterative solvers like Jacobi. These solvers use small kernels in dozens of iterations to get good results. These traditional schemes are simple, can are slow on the GPU due to fillrate and draw-call limitations.

The shader addresses these GPU limitations in Horn-Schunck's optical flow

Limitation | Mitigation
:--------: | :------:
Small changes assumption | Bilinear convolutions and sampling
Brightness constancy assumption | Chromaticity estimation
Iteration constraint | Multi-scale, symmetric Gauss-Seidel solver

## Resource Requirements

Texture | Format | Resolution | MipLevels
:-----: | :----: | :--------: | :-------:
Buffer0`*` | RG16F | BUFFER_SIZE / 2 | 8
Buffer1`*` | RGBA16F | BUFFER_SIZE / 2 | 8
Buffer2 | RG16F | BUFFER_SIZE / 2 | 8
Temporary8`*` | RG16F | BUFFER_SIZE / 256 | 0
Temporary7`*` | RG16F | BUFFER_SIZE / 128 | 0
Temporary6`*` | RG16F | BUFFER_SIZE / 64 | 0
Temporary5`*` | RG16F | BUFFER_SIZE / 32 | 0
Temporary4`*` | RG16F | BUFFER_SIZE / 16 | 0
Temporary3`*` | RG16F | BUFFER_SIZE / 8 | 0
Temporary2`*` | RG16F | BUFFER_SIZE / 4 | 0
Temporary1 | RG16F | BUFFER_SIZE / 2 | 0

> `*` Reusable texture

## Pre-processing

### Color Normalization

We use a 2-dimensional color space (RG Chromaticity) instead of conventional 1-dimensional intensity (luminance). This removes the derivatives' dependency on illumination. We apply this conversion to **each** sample before applying our median filter.

```glsl
vec2 Chroma(sampler2D Source, vec2 TexCoord)
{
    vec4 Color = max(texture(Source, TexCoord), exp2(-10.0));
    return max(Color.xy / dot(Color.rgb, vec2(1.0)), 0.0);
}
```

### Median Filtering

We apply a median filter in a 3x3 neighborhood for this shader. Remember to normalize each sample, not the median-filtered result.

Pass | Shader | Input | Output
:--: | :----: | :---: | :----:
1 | Normalize_Median | Backbuffer | Buffer0

### Convolution

Pete Warden's GPU optical flow uses seperated Gaussian blurs. However, we will use Jorge Jimenez's pyramid convolution, which   is superior for multiple reasons:

+ Cache friendly due to small kernels and box sampling
+ Elimates temporal instabilities (important for Horn-Schunck)
+ Fetches and processes multiple texels at once
+ Does not require 2 textures of the same resolution
+ Can easily achieve wide-radii convolutions

Pass | Shader | Input | Output
:--: | :----: | :---: | :----:
2 | Downsample | Buffer0 | Temporary2
3 | Downsample | Temporary2 | Temporary3
4 | Downsample | Temporary3 | Temporary4
5 | Upsample | Temporary4 | Temporary3
6 | Upsample | Temporary3 | Temporary2
7 | Upsample | Temporary2 | Buffer0

## Derivatives

We create a pyramid of **spatial** derivates using a directional edge detection filter for this shader. This pyramid allows the GPU can compute weighted averages over larger spaces.

We pack this pyramid texture into an `RGBA16F` texture, which we can use for other shaders. We do not need to build a pyramid for **temporal** derivatives because it does not rely on spatial kernels.

Pass | Shader | Input | Output
:--: | :----: | :---: | :----:
8 | Derivatives | Buffer0 | Buffer1

> **NOTE:** Many edge filter kernels are for **integer** sources. Make sure to normalize the derivatives to `[-1, 1]` range because we are working on the neighborhood of `[0, 1]` chromaticity. I recommend using a 5x5 kernel so the shader is robust against noise.

## Optical Flow

We implement Horn-Schunck's optical flow algorithm with a set of modifications

+ Use multi-scale process to get initial values from neighboring pixels
+ Average initial values with a 7x7 low-pass tent filter
+ Estimate features in 2-dimensional chromaticity
+ Use symmetric Gauss-Seidel to solve linear equation

### Iterative Solver

```glsl
// Horn-Schunck used a Jacobi solver with Cramer's Rule
// This is a symmetric Gauss-Seidel solver on a 2-dimensional source

void OpticalFlowRG(in vec2 UV, // Estimate from coarser level
                   in vec4 Di, // Spatial derivatives <Rx, Gx, Ry, Gy>
                   in vec2 Dt, // Temporal derivatives <Rt, Gt>
                   in float Alpha, // Regularizer
                   out vec2 DUV)
{
    /*
        We solve for X[i] (UV)
        Matrix => Horn–Schunck Matrix => Horn–Schunck Equation => Solving Equation

        Matrix
            [A11 A12] [X1] = [B1]
            [A21 A22] [X2] = [B2]

        Horn–Schunck Matrix
            [(Ix^2 + a) (IxIy)] [U] = [aU - IxIt]
            [(IxIy) (Iy^2 + a)] [V] = [aV - IyIt]

        Horn–Schunck Equation
            (Ix^2 + a)U + IxIyV = aU - IxIt
            IxIyU + (Iy^2 + a)V = aV - IyIt

        Solving Equation
            U = ((aU - IxIt) - IxIyV) / (Ix^2 + a)
            V = ((aV - IxIt) - IxIyu) / (Iy^2 + a)
    */

    // A11 = 1.0 / (Rx^2 + Gx^2 + a)
    // A22 = 1.0 / (Ry^2 + Gy^2 + a)
    // Aij = Rxy + Gxy
    float A11 = 1.0 / (dot(Di.xy, Di.xy) + Alpha);
    float A22 = 1.0 / (dot(Di.zw, Di.zw) + Alpha);
    float Aij = dot(Di.xy, Di.zw);

    // B1 = Rxt + Gxt
    // B2 = Ryt + Gyt
    float B1 = dot(Di.xy, Dt);
    float B2 = dot(Di.zw, Dt);

    // Forward Gauss-Seidel (from 1...i)
    DUV.x = A11 * ((Alpha * UV.x) - B1 - (UV.y * Aij));
    DUV.y = A22 * ((Alpha * UV.y) - B2 - (DUV.x * Aij));

    // Backward Gauss-Seidel (from i...1)
    DUV.y = A22 * ((Alpha * DUV.y) - B2 - (DUV.x * Aij));
    DUV.x = A11 * ((Alpha * DUV.x) - B1 - (DUV.y * Aij));
}
```

### Multi-scale Estimation

*Computerphile* has a video explaining how to solve optical flow for larger movements. The solution involved iterating optical flow through a mip-chain like the following:

1. Process the coarsest level `Temporary8`
2. Use results from the coarser level to initialize optical flow values at the finer level
3. Repeat until finest level `Temporary1`

Pass | Shader | Input | Output
:--: | :----: | :---: | :----:
9 | OpticalFlow | Buffer0, Buffer1, Buffer2 | Temporary8
10 | OpticalFlow | Buffer0, Buffer1, Buffer2 | Temporary7
11 | OpticalFlow | Buffer0, Buffer1, Buffer2 | Temporary6
12 | OpticalFlow | Buffer0, Buffer1, Buffer2 | Temporary5
13 | OpticalFlow | Buffer0, Buffer1, Buffer2 | Temporary4
14 | OpticalFlow | Buffer0, Buffer1, Buffer2 | Temporary3
15 | OpticalFlow | Buffer0, Buffer1, Buffer2 | Temporary2
16 | OpticalFlow | Buffer0, Buffer1, Buffer2 | Temporary1

## Post-processing

We do the same convolutions like in the prefiltering phase. However, we use the optical flow texture as input and output the final upsample to `Buffer1`

On pass 22, we also copy `Buffer0`, to `Buffer2`. This allows us to store information from the current convolved frame, `Buffer0`, for the next frame.

Pass | Shader | Input | Output
:--: | :----: | :---: | :----:
17 | Downsample | Temporary1 | Temporary2
18 | Downsample | Temporary2 | Temporary3
19 | Downsample | Temporary3 | Temporary4
20 | Upsample | Temporary4 | Temporary3
21 | Upsample | Temporary3 | Temporary2
22 | Upsample | Temporary2, Buffer0 | Buffer1, Buffer2

## Output

This is the final pass, where we display the processed optical flow result in the form of xy or normalized RGB.

Pass | Shader | Input | Output
:--: | :----: | :---: | :----:
23 | Display | Buffer1 | BackBuffer

## References

[Determining Optical Flow](https://dspace.mit.edu/handle/1721.1/6337)

[Jorge Jimenez - Next Generation Post Processing in Call of Duty: Advanced Warfare](http://www.iryoku.com/next-generation-post-processing-in-call-of-duty-advanced-warfare)

[Optical Flow - Computerphile](https://www.youtube.com/watch?v=5AUypv5BNbI)

[Optic Flow Solutions - Computerphile](https://www.youtube.com/watch?v=4v_keMNROv4)

[Pete Warden - GPU Optical Flow](http://web.archive.org/web/20081020065947/http://www.petewarden.com:80/notes/archives/2005/05/gpu_optical_flo.html)
