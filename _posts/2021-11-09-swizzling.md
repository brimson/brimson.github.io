---
layout: post
date: 2021-11-09
title: Swizzling Texture Coordinates for Box Filters
category: Shaders
tags: [Convolutions, Optimizations, Post-Processing]
---

Programmers use box filters, a convolution, to blur an image. For example, [Blender uses box filters for its Eevee engine bloom](https://github.com/blender/blender/blob/master/source/blender/draw/engines/eevee/eevee_bloom.c). The GPU can efficiently cache a box filter because it is a small kernel with predictable sample locations.

Here is what simple box filter looks like:

```glsl
[A(0.25) C(0.25)]
[B(0.25) D(0.25)]
```

## Vertex Shader Optimization

You can optimize convolutions by calculating texture coordinates in the vertex shader. This optimization encourages caching and prevents unnecessary per-pixel calculations.

> [Read more about the optimization here](http://www.sunsetlakesoftware.com/2013/10/21/optimizing-gaussian-blurs-mobile-gpu)

Unfortunately, we have have limitations if we use this optimization on Direct3D 9.0c. `vs_3_0` [limits us to 8 texture coordinate outputs](https://docs.microsoft.com/en-us/windows/win32/direct3dhlsl/shader-model-3).

However, you can pack multiple offsets into one attribute and swizzle them at lookup. Fortunately, Direct3D [allows you swizzle texture coordinates before texture lookup](https://docs.microsoft.com/en-us/windows/win32/direct3dhlsl/texld---ps-2-0#ps_3_0).

## Source Code

### 2x2 Box Filter

```glsl
void BoxFilterVS(in uint ID : SV_VERTEXID, inout float4 Position : SV_POSITION, inout float4 TexCoord : TEXCOORD0)
{
    float2 VTexCoord;
    VTexCoord.x = (ID == 2) ? 2.0 : 0.0;
    VTexCoord.y = (ID == 1) ? 2.0 : 0.0;
    Position = float4(VTexCoord * float2(2.0, -2.0) + float2(-1.0, 1.0), 0.0, 1.0);
    TexCoord = VTexCoord.xyxy + float4(-1.0, -1.0, 1.0, 1.0) * PixelSize;
}

void BoxFilterPS(in float2 Position : SV_POSITION, in float4 TexCoord : TEXCOORD0, out float4 OutputColor0 : SV_TARGET0)
{
    OutputColor0 += tex2D(_Sampler, TexCoord.xw) * 0.25; // (-1.0, +1.0)
    OutputColor0 += tex2D(_Sampler, TexCoord.zw) * 0.25; // (+1.0, +1.0)
    OutputColor0 += tex2D(_Sampler, TexCoord.xy) * 0.25; // (-1.0, -1.0)
    OutputColor0 += tex2D(_Sampler, TexCoord.zy) * 0.25; // (+1.0, -1.0)
}
```

This is pretty epic. Not only we reduced overhead and allow the pipeline to "pre-fetch" samplers, but we reduced 4 `mad-ps` instructions to 1 `mad-vs` instruction.

## References

[Blender Source Code - Eevee Bloom](https://github.com/blender/blender/blob/master/source/blender/draw/engines/eevee/eevee_bloom.c)

[Optimizing Gaussian blurs on a mobile GPU](http://www.sunsetlakesoftware.com/2013/10/21/optimizing-gaussian-blurs-mobile-gpu)

[Shader model 3 (HLSL reference)](https://docs.microsoft.com/en-us/windows/win32/direct3dhlsl/shader-model-3)

[texld - ps_2_0 and up](https://docs.microsoft.com/en-us/windows/win32/direct3dhlsl/texld---ps-2-0#ps_3_0)
