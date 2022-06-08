---
layout: post
date: 2022-02-01
title: Multi-scale Horn Schunck on the GPU
category: Shaders
tags: [Computer Vision]
---

In this post, I address a shader implementation of Horn-Schunck's optical flow in 16 draw-calls.

## Limitations

### Assumptions

The original optical flow algorithm assumes the scene has brightness constancy and small changes. Therefore, it breaks on large movement (i.e. low-fps footage) and scenes with varying illumination.

### Iterations

Pixel shaders are not good for traditional stationary solvers like Jacobi do not translate. These solvers use small kernels in dozens of iterations (draw-calls) to get good results.

The shader addresses these GPU limitations in Horn-Schunck's optical flow

Limitation | Solution
:--------: | :------:
Small changes assumption | Bilinear blurs and sampling
Brightness constancy assumption | Multi-channel estimation
Iteration constraint | Multi-scale solver

## Resource Requirements

Texture | Format | Resolution | MipLevels
:-----: | :----: | :--------: | :-------:
Buffer0`*` | RG16F | BUFFER_SIZE / 2 | 8
Buffer1`*` | RG16F | BUFFER_SIZE / 2 | 8
Buffer2 | RG16F | BUFFER_SIZE / 2 | 8
Temporary1`*` | RG16F | BUFFER_SIZE / 2 | 0
Temporary2`*` | RG16F | BUFFER_SIZE / 4 | 0
Temporary3`*` | RG16F | BUFFER_SIZE / 8 | 0
Temporary4`*` | RG16F | BUFFER_SIZE / 16 | 0
Temporary5`*` | RG16F | BUFFER_SIZE / 32 | 0
Temporary6`*` | RG16F | BUFFER_SIZE / 64 | 0
Temporary7`*` | RG16F | BUFFER_SIZE / 128 | 0
Temporary8`*` | RG16F | BUFFER_SIZE / 256 | 0

> `*` Reusable texture

## Pre-processing

### Color Normalization

We use a 2-dimensional color space (RG Chromaticity) instead of conventional 1-dimensional intensity (luminance). This removes the derivatives' dependency on illumination.

```glsl
vec2 Chroma(sampler2D Source, vec2 TexCoord)
{
    vec4 Color = max(texture(Source, TexCoord), exp2(-10.0));
    return clamp(Color.xy / dot(Color.rgb, vec2(1.0)), 0.0, 1.0);
}
```

Pass | Shader | Input | Output
:--: | :----: | :---: | :----:
1 | Normalize | Backbuffer | Buffer0

### Pre-Process

We apply a seperable, linear gaussian blur on the image.

Pass | Shader | Input | Output
:--: | :----: | :---: | :----:
2 | BlurH | Buffer0 | Buffer1
3 | BlurV | Buffer1 | Buffer0

## Derivatives

### Temporal Derivative

First, we calculate the temporal derivative. We will only use one-channel for this shader as we are summing the changes of multiple channels (`Iz = DzR + DzG`).

Pass | Shader | Input | Output
:--: | :----: | :---: | :----:
4 | DerivativeZ | Buffer0, Buffer2 | Buffer1

### Spatial Derivatives

We create a pyramid of `(x, y)` derivates using an edge-detection filter on the current frame. This pyramid allows the GPU can compute weighted averages over larger spaces. Because we are doing chroma edge detection, we are just going to sum the derivatives of each channel into one (`Ix = DxR + DxG`, `Iy = DyR + DyG`).

Pass | Shader | Input | Output
:--: | :----: | :---: | :----:
5 | DerivativeXY | Buffer0 | Buffer2

## Optical Flow

We implement Horn-Schunck's optical flow algorithm with our modifications

```glsl
void OpticalFlow(in vec2 UV, // Estimate from coarser level (You can apply a blur here)
                 in vec2 Di, // Spatial derivatives
                 in float Dt, // Temporal derivatives
                 in float Alpha, // Regularizer
                 out vec2 DUV)
{
    /*
        We solve for X[i] (UV)
        Matrix => Horn–Schunck Matrix => Horn–Schunck Equation => Factor Equation
        Matrix
            [A11 A12] [X1] = [B1]
            [A21 A22] [X2] = [B2]
        Horn–Schunck Matrix
            [(Ix^2 + A) (IxIy)] [U] = [aU - IxIt]
            [(IxIy) (Iy^2 + A)] [V] = [aV - IyIt]
        Horn–Schunck Equation
            (Ix^2 + A)U + IxIyV = aU - IxIt
            IxIyU + (Iy^2 + A)V = aV - IyIt
        Factor Equation
            U = ((aU - IxIt) - IxIyV) / (Ix^2 + A)
            V = ((aV - IxIt) - IxIyu) / (Iy^2 + A)
    */

    vec2 Aii = 1.0 / ((Di.xy * Di.xy) + Alpha);
    vec2 Aij = (Di.xx * Di.yy) * UV.yx;
    vec2 Bi = Di.xy * Dt;
    DUV.xy = (((Alpha * UV.xy) - Bi.xy) - Aij) * Aii.xy;
}
```

### Multi-scale Estimation

*Computerphile* has a video explaining how to solve optical flow for larger movements. The solution involved iterating optical flow through a mip-chain like the following:

1. Process the coarsest level `Temporary8`
2. Use results from the coarser level to initialize optical flow values at the finer level
3. Repeat until finest level `Temporary1`

Pass | Shader | Input | Output
:--: | :----: | :---: | :----:
6 | OpticalFlow | Buffer1, Buffer2 | Temporary8
7 | OpticalFlow | Temporary8, Buffer1, Buffer2 | Temporary7
8 | OpticalFlow | Temporary7, Buffer1, Buffer2 | Temporary6
9 | OpticalFlow | Temporary6, Buffer1, Buffer2 | Temporary5
10 | OpticalFlow | Temporary5, Buffer1, Buffer2 | Temporary4
11 | OpticalFlow | Temporary4, Buffer1, Buffer2 | Temporary3
12 | OpticalFlow | Temporary3, Buffer1, Buffer2 | Temporary2
13 | OpticalFlow | Temporary2, Buffer1, Buffer2 | Temporary1

## Post-processing

We do the same convolutions like in the prefiltering phase.

Pass | Shader | Input | Output
:--: | :----: | :---: | :----:
14 | BlurH | Temporary1, Buffer0 | Buffer1, Buffer2
15 | BlurV | Buffer1 | Buffer0

On pass 14, we use MRTs to immediately copy `Buffer0`, to `Buffer2`. This allows us to store information from the current convolved frame, `Buffer0`, for the next frame.

## Output

This is the final pass, where we display the processed optical flow result in the form of xy or normalized RGB.

Pass | Shader | Input | Output
:--: | :----: | :---: | :----:
16 | Display | Buffer0 | BackBuffer

## References

[Determining Optical Flow](https://dspace.mit.edu/handle/1721.1/6337)

[Optical Flow - Computerphile](https://www.youtube.com/watch?v=5AUypv5BNbI)

[Optic Flow Solutions - Computerphile](https://www.youtube.com/watch?v=4v_keMNROv4)

[Pete Warden - GPU Optical Flow](http://web.archive.org/web/20081020065947/http://www.petewarden.com:80/notes/archives/2005/05/gpu_optical_flo.html)
