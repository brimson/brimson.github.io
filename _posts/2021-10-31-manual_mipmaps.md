---
layout: post
date: 2021-10-31
title: Manually Calculating Level of Detail (LOD)
category: Shaders
tags: [Optimizations]
---

There are situations where you need to calculate a texture's LOD by hand.

First, you must manually calculate your texture's LOD if you are looping texture sampling. You may encounter errors or warnings if you loop gradient instructions like `tex2D()`:

Error / Warning | Description
--------------- | -----------
ERR_GRADIENT_WITH_BREAK 3526  | Gradient instructions can't be used in loops with breaks.
WAR_GRADIENT_WITH_BREAK 3553  | Can't use gradient instructions in loops with break.
WAR_GRADIENT_MUST_UNROLL 3570 | A gradient instruction is used in a loop with varying iteration, which forces the loop to unroll.
ERR_GRADIENT_FLOW 4014        | Gradient operations can't occur inside loops with divergent flow control.
WARN_HOISTING_GRADIENT 4121   | Gradient-based operations must be moved out of flow control to prevent divergence. Performance might improve by using a non-gradient operation.

> [Learn more about HLSL errors and warnings](https://docs.microsoft.com/en-us/windows/win32/direct3dhlsl/hlsl-errors-and-warnings)

Second, you may manually calculate have artistic intentions. Some examples

+ Dynamic LOD bias without sampler states (MipLODBias, MinLOD, and MaxLOD)
+ Convolutions such as [Poisson disk with mipmaps](https://john-chapman.github.io/2019/03/29/convolution.html) in a loop (especially if you are writing to a downsampled framebuffer)

## Source Code: Calculating LOD in 2D Post-Processing (ReShade)

```glsl
// Function inspired by http://web.cse.ohio-state.edu/~crawfis.3/cse781/Readings/MipMapLevels-Blog.html

float ComputeLOD(vec2 TextureSize, vec2 OutputSize, float Bias)
{
    // Calculate the total number of texels in each attribute
    float Texels = TextureSize.x * TextureSize.y;
    float Pixels = OutputSize.x * OutputSize.y;

    // Calculate the LOD value the input texture needs to reach 1:1 texel:pixel ratio
    // Halve step 2's result because each LOD level quarters texels
    float MipLevel = 0.5 * log2(Texels / Pixels);

    // Append optional bias and clamp the result to prevent negative values
    // You can subtract the result by 1 if you are short on miplevels (The GPU will bilinearly interpolate to the last miplevel)
    return max(MipLevel + Bias, 0.0)
}
```

## Source Code: Calculating LOD in [The DirectX 11 Method](https://microsoft.github.io/DirectX-Specs/d3d/archive/D3D11_3_FunctionalSpec.htm#7.18.10%20Mipmap%20Selection)

```glsl
// Optimized version by Ned Plays Games
float ComputeLOD(vec2 TexCoord, vec2 InputTextureSize, float Bias)
{
    // Calculate how many texels are in each TexCoord (UV) pixel index
    vec2 TexelIndex = TexCoord * InputTextureSize;

    // Calculate how many texels change in UV pixels' <x, y> directions in their respective dimensions
    vec2 Ix = dFdx(TexelIndex);
    vec2 Iy = dFdy(TexelIndex);

    // Calculate the derivatives' square product and compare which dimension needs more LOD coverage
    // We need to calculate 4 derivatives <IxU, IyU, IxV, IyV> to account for cases such as rotated UV maps
    float SquareProduct = max(dot(Ix, Ix), dot(Iy, Iy));

    // Calculate the square-root of log base 2 and bias to find the LOD level
    return max(0.5 * log2(SquareProduct) + Bias, 0.0);
}
```

## References

[Basics of Mipmaps in Unity Part 2](https://www.youtube.com/watch?v=2G0Sime3OH0)

[Direct3D 11.3  Functional Specification - Mipmap Selection](https://microsoft.github.io/DirectX-Specs/d3d/archive/D3D11_3_FunctionalSpec.htm#7.18.10%20Mipmap%20Selection)

[HLSL errors and warnings](https://docs.microsoft.com/en-us/windows/win32/direct3dhlsl/hlsl-errors-and-warnings)

[Knowing which mipmap levels are needed](http://web.cse.ohio-state.edu/~crawfis.3/cse781/Readings/MipMapLevels-Blog.html)

[Optimizing Convolutions](https://john-chapman.github.io/2019/03/29/convolution.html)
