---
layout: post
date: 2021-11-04
title: Efficient Sobel Operator with Linear Sampling
category: Shaders
tags: [Convolutions, Optimizations, Post-Processing]
---

Shaders such as [edge detection][1] and [optical flow][0] require calculating derivatives.

The [Sobel operation][2] is a seperable, 3x3 operator that approximates horizontal and vertical derivatives. I will show you how to a single-pass version of this operation in 4 texture fetches.

## Discrete Sobel Filter

The first assumption is that sobel requires 8 texture fetches, each **on** the pixel index. Here we represent sampler locations, horizontal kernel, and vertical kernel.

```glsl
// Sampler locations
[A D F]
[B   G]
[C E H]

// Horizontal kernel
[-1 0 1]
[-2 0 2]
[-1 0 1]

// Vertical kernel
[ 1  2  1]
[ 0  0  0]
[-1 -2 -1]
```

## Linear Sobel Filter

What if we sample adjacently, **in-between** 4 pixels? In a pixel index, the GPU will preform a 2x2 bilinear interpolation between sampled texels.

```glsl
// Sampler locations (A, B, C, D)
// Let numbers be pixel indices
[1   2   3]
[  A   C  ]
[4   5   6]
[  B   D  ]
[7   8   9]

// Horizontal kernel
[-0.25 0.00 0.25]
[-0.50 0.00 0.50]
[-0.25 0.00 0.25]

// Vertical kernel
[ 0.25  0.50  0.25]
[ 0.00  0.00  0.00]
[-0.25 -0.50 -0.25]
```

## Source Code

```glsl
void BilinearSobel(sampler2D Source, vec2 TexCoord, vec2 PixelSize, out vec4 Ix, out vec4 Iy)
{
    vec4 A0 = texture(Source, TexCoord + vec2(-0.5, 0.5) * PixelSize);
    vec4 A2 = texture(Source, TexCoord + vec2( 0.5, 0.5) * PixelSize);
    vec4 A3 = texture(Source, TexCoord + vec2(-0.5, -0.5) * PixelSize);
    vec4 A4 = texture(Source, TexCoord + vec2( 0.5, -0.5) * PixelSize);

    Ix = (A2 + A4) - (A0 + A3);
    Iy = (A0 + A2) - (A3 + A4);
}
```

## Notes

+ You need to set the sampler's minification and magnification filtering to `LINEAR`
+ You can use `SobelFetch()` for a 3x3 Gaussian blur by writing a shader that sums the 4 samples and divide by 4

## References

[Implementation and Analysis of Real Time Optical Flow Solutions for GPU architectures][0]

[KinoContour][1]

[Sobel Edge Detector][2]

[0]: https://oa.upm.es/47692/
[1]: https://github.com/keijiro/KinoContour
[2]: https://homepages.inf.ed.ac.uk/rbf/HIPR2/sobel.htm
