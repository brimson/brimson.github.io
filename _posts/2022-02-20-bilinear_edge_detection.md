---
layout: post
date: 2022-02-20
title: Bilinear Directional Edge Detection
category: Shaders
tags: [Computer Vision, Optimization]
---

I needed to approximate a large, 5x5 kernels for derivative detection. I did not want to sample 25 textures due to the buffer being half-resolution.

With the help of bilinear interpolation and CeeJayDK, we were able to approximate 5x5 edge detectors in 8 samples. The trick is to sample diagonally between 4 pixel neighbors. This post gives 2 8-tap, 5x5 kernels for the operation

+ Non-rotational (10-pixel neighborhood)
+ One rotational (8-pixel neighborhood)

## Non-Rotational Kernel

```glsl
void BilinearEdgeDetection(in sampler2D Source, in vec2 TexCoord, in vec2 PixelSize, out vec4 Ix, out vec4 Iy)
{
    // Custom 5x5 bilinear derivatives, normalized to [-1, 1]
    // A0 B0 C0
    // A1    C1
    // A2 B2 C2
    vec4 A0 = texture(Source, TexCoord + vec2(-1.5, 1.5) * PixelSize);
    vec4 A1 = texture(Source, TexCoord + vec2(-1.5, 0.0) * PixelSize);
    vec4 A2 = texture(Source, TexCoord + vec2(-1.5, -1.5) * PixelSize);

    vec4 B0 = texture(Source, TexCoord + vec2(0.0, 1.5) * PixelSize);
    vec4 B2 = texture(Source, TexCoord + vec2(0.0, -1.5) * PixelSize);

    vec4 C0 = texture(Source, TexCoord + vec2(1.5, 1.5) * PixelSize);
    vec4 C1 = texture(Source, TexCoord + vec2(1.5, 0.0) * PixelSize);
    vec4 C2 = texture(Source, TexCoord + vec2(1.5, -1.5) * PixelSize);

    // -1 -1  0  +1 +1
    // -1 -1  0  +1 +1
    // -1 -1  0  +1 +1
    // -1 -1  0  +1 +1
    // -1 -1  0  +1 +1
    Ix  = (((C0 * 4.0) + (C1 * 2.0) + (C2 * 4.0)) - ((A0 * 4.0) + (A1 * 2.0) + (A2 * 4.0))) / 10.0;

    // +1 +1 +1 +1 +1
    // +1 +1 +1 +1 +1
    //  0  0  0  0  0
    // -1 -1 -1 -1 -1
    // -1 -1 -1 -1 -1
    Iy = (((A0 * 4.0) + (B0 * 2.0) + (C0 * 4.0)) - ((A2 * 4.0) + (B2 * 2.0) + (C2 * 4.0))) / 10.0;
}
```

## Rotational Kernel

```glsl
void BilinearEdgeDetection(in sampler2D Source, in vec2 TexCoord, in vec2 PixelSize, out vec4 Ix, out vec4 Iy)
{
    // Custom 5x5 bilinear edge-detection by CeeJayDK
    //   B0 B1
    // A0     A1
    //     C
    // A2     A3
    //   B2 B3
    vec4 A0 = texture(Source, TexCoord + vec2(-1.5, 0.5) * PixelSize) * 4.0;
    vec4 A1 = texture(Source, TexCoord + vec2(1.5, 0.5) * PixelSize) * 4.0;
    vec4 A2 = texture(Source, TexCoord + vec2(-1.5, -0.5) * PixelSize) * 4.0;
    vec4 A3 = texture(Source, TexCoord + vec2(1.5, -0.5) * PixelSize) * 4.0;

    vec4 B0 = texture(Source, TexCoord + vec2(-0.5, 1.5) * PixelSize) * 4.0;
    vec4 B1 = texture(Source, TexCoord + vec2(0.5, 1.5) * PixelSize) * 4.0;
    vec4 B2 = texture(Source, TexCoord + vec2(-0.5, -1.5) * PixelSize) * 4.0;
    vec4 B3 = texture(Source, TexCoord + vec2(0.5, -1.5) * PixelSize) * 4.0;

    //    -1 0 +1
    // -1 -2 0 +2 +1
    // -2 -2 0 +2 +2
    // -1 -2 0 +2 +1
    //    -1 0 +1
    Ix = ((B1 + A1 + A3 + B3) - (B0 + A0 + A2 + B2)) / 12.0;

    //    +1 +2 +1
    // +1 +2 +2 +2 +1
    //  0  0  0  0  0
    // -1 -2 -2 -2 -1
    //    -1 -2 -1
    Iy = ((B0 + B1 + A0 + A1) - (A2 + A3 + B2 + B3)) / 12.0;
}
```
