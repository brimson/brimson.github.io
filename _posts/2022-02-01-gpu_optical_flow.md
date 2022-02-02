---
layout: post
date: 2022-02-01
title: Pyramidal Horn Schunck on the GPU
category: Shaders
tags: [Computer Vision, Optimization]
---

Implementing basic motion estimation on pixel shaders was not trivial. We need an algorithm that satisfies the following conditions:

+ Able to estimate larger movements
+ Bandwidth and fillrate efficient
+ Global method that applies to every pixel
+ Not too swayed by illuminance and noise
+ Benefits from hardware interpolation

In this post, I construct a shader implementation of Horn-Schunck's algorithm in 24 draw-calls with the following modifications:

+ Bilinear convolutions and sampling
+ Chromaticity estimation
+ Coarse-to-fine scheme
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

> Note: You do not need `BufferIxy` if you are calculating discrete derivatives in each pyramid pass

## Preprocessing

### Normalization

This optical flow implemenation uses a 2-dimensional color space (RG Chromaticity) instead of conventional 1-dimensional intenisity (luminance). On Direct3D 9.0c, this operation is as cheap as 3 instructions (DP3-RCP-MUL):

`saturate(Color.xy / dot(Color.rgb, 1.0))`

Pass | Type | Sample | RenderTarget
:--: | :--: | :----: | :----------:
1 | Normalize | Backbuffer | Buffer0

### Preprocess blur

In this implementation, we rely on Jorge Jimenez's pyramid convolution scheme instead of seperated Gaussian blurs like Pete Warden's GPU implementation

Pass | Type | Sample | RenderTarget
:--: | :--: | :----: | :----------:
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

## References

[Pete Warden - GPU Optical Flow](http://web.archive.org/web/20081020065947/http://www.petewarden.com:80/notes/archives/2005/05/gpu_optical_flo.html)

[Jorge Jimenez - Next Generation Post Processing in Call of Duty: Advanced Warfare](http://www.iryoku.com/next-generation-post-processing-in-call-of-duty-advanced-warfare)
