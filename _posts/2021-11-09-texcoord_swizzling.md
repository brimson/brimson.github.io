---
layout: post
date: 2021-11-09
title: Swizzling Texture Coordinates for Box Filters
category: Shaders
tags: [Convolutions, Optimizations, Post-Processing]
---

## Introduction

Programmers use box filters to blur an image. For example, [Blender uses box filters to render its Eevee engine bloom][0].

This is what simple box filter looks like:

```glsl
// (x, y) are sample locations
A(x - 0.5, y + 0.5) C(x + 0.5, y + 0.5)
B(x - 0.5, y - 0.5) D(x + 0.5, y - 0.5)
...
(A + B + C + D) / 4.0
```

The GPU can cache this filter because it is small and local. "Local" means that the shader samples textures not far from its origin.

## Vertex Shader Optimization

You calculate texture coordinates in the vertex shader to prevent per-pixel calculations.

Unfortunately, Shader Model 3.0 [limits us to calculate 8 texture coordinates in the vertex shader][2]. However, you can pack multiple offsets into one `TexCoord` and swizzle them in the pixel shader ["for free"][3].

> [Read more about the optimization here][1]

## Source Code

### 4x4 Downsample Box Filter

```glsl
void Box_Filter_VS(in uint ID : SV_VERTEXID, inout float4 Position : SV_POSITION, inout float4 TexCoord : TEXCOORD0)
{
    float2 VSTexCoord = 0.0;
    VSTexCoord.x = (ID == 2) ? 2.0 : 0.0;
    VSTexCoord.y = (ID == 1) ? 2.0 : 0.0;
    Position = float4(VSTexCoord * float2(2.0, -2.0) + float2(-1.0, 1.0), 0.0, 1.0);
    TexCoord = VSTexCoord.xyxy + float4(-1.0, -1.0, 1.0, 1.0) * SourcePixelSize.xyxy;
}
```

### 6x6 Downsample Tent Filter

```glsl
void Tent_Filter_VS(in uint ID : SV_VERTEXID, inout float4 Position : SV_POSITION, inout float4 TexCoords[3] : TEXCOORD0)
{
    float2 VSTexCoord = 0.0;
    VSTexCoord.x = (ID == 2) ? 2.0 : 0.0;
    VSTexCoord.y = (ID == 1) ? 2.0 : 0.0;
    Position = float4(VSTexCoord * float2(2.0, -2.0) + float2(-1.0, 1.0), 0.0, 1.0);
    // Left column
    TexCoords[0] = VSTexCoord.xyyy + float4(-2.0, 2.0, 0.0, -2.0) * SourcePixelSize.xyyy;
    // Center column
    TexCoords[1] = VSTexCoord.xyyy + float4(0.0, 2.0, 0.0, -2.0) * SourcePixelSize.xyyy;
    // Right column
    TexCoords[2] = VSTexCoord.xyyy + float4(2.0, 2.0, 0.0, -2.0) * SourcePixelSize.xyyy;
}
```

## References

[Blender Source Code - Eevee Bloom][0]

[Optimizing Gaussian blurs on a mobile GPU][1]

[Shader model 3 (HLSL reference)][2]

[texld - ps_2_0 and up][3]

[0]: https://github.com/blender/blender/blob/master/source/blender/draw/engines/eevee/eevee_bloom.c

[1]: http://www.sunsetlakesoftware.com/2013/10/21/optimizing-gaussian-blurs-mobile-gpu

[2]: https://docs.microsoft.com/en-us/windows/win32/direct3dhlsl/shader-model-3

[3]: https://docs.microsoft.com/en-us/windows/win32/direct3dhlsl/texld---ps-2-0#ps_3_0
