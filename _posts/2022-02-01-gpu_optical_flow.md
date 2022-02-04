---
layout: post
date: 2022-02-01
title: Multi-scale Horn Schunck on the GPU
category: Shaders
tags: [Computer Vision, Optimization]
---

Implementing basic motion estimation on pixel shaders was not trivial. We need an algorithm that satisfies multiple assumptions:

+ Constant lighting
+ Small temporal changes

In this post, I construct a shader implementation of Horn-Schunck's algorithm in 24 draw-calls to accomodate these assumptions:

+ Bilinear convolutions and sampling
+ Chromaticity estimation
+ Multi-scale estimation
+ Symmetric Gauss-Seidel solver

## Resource Requirements

Texture | Format | Resolution | MipLevels
:-----: | :----: | :--------: | :-------:
Buffer0 | RG16F   | BUFFER_SIZE / 2 | 8
Buffer1 | RG16F   | BUFFER_SIZE / 2 | 8
BufferIxy | RGBA16F | BUFFER_SIZE / 2 | 8
Temporary7 | RG16F | BUFFER_SIZE / 256 | 0
Temporary6 | RG16F | BUFFER_SIZE / 128 | 0
Temporary5 | RG16F | BUFFER_SIZE / 64 | 0
Temporary4 | RG16F | BUFFER_SIZE / 32 | 0
Temporary3 | RG16F | BUFFER_SIZE / 16 | 0
Temporary2 | RG16F | BUFFER_SIZE / 8 | 0
Temporary1 | RG16F | BUFFER_SIZE / 4 | 0
Temporary0 | RG16F | BUFFER_SIZE / 2 | 0

> Note: You do not need `BufferIxy` if you are calculating discrete derivatives in each optical flow pass

## Pre-processing

### Normalization

This optical flow implemenation uses a 2-dimensional color space (RG Chromaticity) instead of conventional 1-dimensional intenisity (luminance). On Direct3D 9.0c, this operation is as cheap as 3 instructions (DP3-RCP-MUL):

`saturate(Color.xy / dot(Color.rgb, 1.0))`

Pass | Shader | Input | Output
:--: | :----: | :---: | :----:
1 | Normalize | Backbuffer | Buffer0

### Convolution

In this implementation, we rely on Jorge Jimenez's pyramid convolution scheme instead of seperated Gaussian blurs like Pete Warden's GPU implementation. We save the prefiltered current frame to `Buffer0`

Pass | Shader | Input | Output
:--: | :----: | :---: | :----:
2 | Downsample | Buffer0 | Temporary1
3 | Downsample | Temporary1 | Temporary2
4 | Downsample | Temporary2 | Temporary3
5 | Upsample | Temporary3 | Temporary2
6 | Upsample | Temporary2 | Temporary1
7 | Upsample | Temporary1 | Buffer0

Reasons why this solutions is superior to conventional Gaussian blur:

+ Cache friendly due to small kernels and box sampling
+ Elimates temporal issues and prevents jumping on high-frequency areas
+ Takes advantage of multiple texels per fetch
  + 4 texels a fetch in downsampling
  + 9 texels a fetch in upsampling
+ Saves bandwidth because it does not a proxy texture of the same resolution
+ Easily to achieve wide-radii blur at expence of 2 draw-calls per level

## Derivatives

We create a pyramid of discrete derivates using a Sobel filter for this implementation. You can calculate the derivatives in each estimation pass to save bandwidth.

Pass | Shader | Input | Output
:--: | :----: | :---: | :----:
8 | Derivatives | Buffer0 | BufferIxy

## Optical Flow

We implement Horn-Schunck's optical flow algorithm with a set of modifications

+ Compute averages with a 7x7 low-pass tent filter
+ Estimate features in 2-dimensional chromaticity
+ Use multi-scale process to get initial values from neighboring pixels
+ Use symmetric Gauss-Seidel to solve linear equation at Page 8

### Low-pass Tent Filter

We use the same upsampling filter to get an average of the vector's neighborhood.

### Multi-scale Estimation

Computerphile has a video explaining how to solve for larger movements. The solution involved iterating optical flow through a mip-chain like the following:

1. Process the coarsest level `Temporary7`
2. Use results from the coarser level to initialize optical flow values at the finer level
3. Repeat until finest level `Temporary0`

### Iterative Solver

Horn-Schunck used a Jacobi solver with Cramer's Rule for optimization on a 1-dimensional source.

```glsl
void OpticalFlow(in vec2 UV, // Previous estimate
                 in vec2 Dxy, // Spatial derivatives
                 in float Dt, // Temporal derivative
                 in float Alpha, // Regularizer
                 out vec2 DUV)
{
    float Value = dot(Dxy, UV) + Dt;
    float Determinant = dot(Dxy, Dxy) + Alpha;
    DUV = UV.xy - ((Dxy * Value) / Determinant);
}
```

This implementation uses a custom solver on a 2-dimensional source.

```glsl
void OpticalFlowRG(in vec2 UV, // Previous estimate
                   in vec4 Dxy, // Spatial derivatives (Rx, Ry, Gx, Gy)
                   in vec2 Dt, // Temporal derivatives (Rt, Gt)
                   in float Alpha, // Regularizer
                   out vec2 DUV)
{
    // Compute diagonal
    vec2 Aii;
    Aii.x = dot(Dxy.xz, Dxy.xz) + Alpha;
    Aii.y = dot(Dxy.yw, Dxy.yw) + Alpha;
    Aii.xy = 1.0 / Aii.xy;

    // Compute right-hand side
    vec2 RHS;
    RHS.x = dot(Dxy.xz, Dt.rg);
    RHS.y = dot(Dxy.yw, Dt.rg);

    // Compute triangle
    float Aij = dot(Dxy.xz, Dxy.yw);

    // Symmetric Gauss-Seidel (forward sweep, from 1...N)
    DUV.x = Aii.x * ((Alpha * UV.x) - RHS.x - (UV.y * Aij));
    DUV.y = Aii.y * ((Alpha * UV.y) - RHS.y - (DUV.x * Aij));

    // Symmetric Gauss-Seidel (backward sweep, from N...1)
    DUV.y = Aii.y * ((Alpha * DUV.y) - RHS.y - (DUV.x * Aij));
    DUV.x = Aii.x * ((Alpha * DUV.x) - RHS.x - (DUV.y * Aij));
}
```

### Iterations

Pass | Shader | Input | Output
:--: | :----: | :---: | :----:
9 | OpticalFlow | Buffer0, Buffer1, BufferIxy | Temporary7
10 | OpticalFlow | Buffer0, Buffer1, BufferIxy | Temporary6
11 | OpticalFlow | Buffer0, Buffer1, BufferIxy | Temporary5
12 | OpticalFlow | Buffer0, Buffer1, BufferIxy | Temporary4
13 | OpticalFlow | Buffer0, Buffer1, BufferIxy | Temporary3
14 | OpticalFlow | Buffer0, Buffer1, BufferIxy | Temporary2
15 | OpticalFlow | Buffer0, Buffer1, BufferIxy | Temporary1
16 | OpticalFlow | Buffer0, Buffer1, BufferIxy | Temporary0

## Post-processing

We do the same convolutions like in the prefiltering phase. However, we use the optical flow texture as input and output the result to `Buffer1`. We do not write to `Buffer0` because we will use copy that texture and use it as the previous frame after all processing

Pass | Shader | Input | Output
:--: | :----: | :---: | :----:
17 | Downsample | Temporary0 | Temporary1
18 | Downsample | Temporary1 | Temporary2
19 | Downsample | Temporary2 | Temporary3
20 | Upsample | Temporary3 | Temporary2
21 | Upsample | Temporary2 | Temporary1
22 | Upsample | Temporary1 | Buffer1

## Output

This is the final pass, where we display the processed optical flow result in the form of xy or normalized RGB.

Pass | Shader | Input | Output
:--: | :----: | :---: | :----:
23 | Display | Buffer1 | BackBuffer

## Copy

We copy `Buffer0`, to `Buffer1` after all processing. This allows us to store information from the current frame `Buffer0` and the previous frame `Buffer1` for the optical flow passes.

Pass | Shader | Input | Output
:--: | :----: | :---: | :----:
24 | Copy | Buffer0 | Buffer1

## References

[Determining Optical Flow](https://dspace.mit.edu/handle/1721.1/6337)

[Jorge Jimenez - Next Generation Post Processing in Call of Duty: Advanced Warfare](http://www.iryoku.com/next-generation-post-processing-in-call-of-duty-advanced-warfare)

[Optical Flow - Computerphile](https://www.youtube.com/watch?v=5AUypv5BNbI)

[Optic Flow Solutions - Computerphile](https://www.youtube.com/watch?v=4v_keMNROv4)

[Pete Warden - GPU Optical Flow](http://web.archive.org/web/20081020065947/http://www.petewarden.com:80/notes/archives/2005/05/gpu_optical_flo.html)
