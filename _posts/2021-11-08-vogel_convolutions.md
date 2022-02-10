---
layout: post
date: 2021-11-08
title: Convolutions with Vogel Spirals
category: Shaders
tags: [Convolutions, Post-Processing]
---

You do not need much to approximate convolutions. You only need a texture with mipmaps to blur a static image. In this post, we repurpose [Wojciech Sterna's shadow sampling](http://maxest.gct-game.net/content/chss.pdf) for screen convolutions.

## How Vogel Convolutions Work

1. Sample textures in a spiral
2. Use mipmaps or noise to fill the gaps between the sampled textures
    1. [Calculate LOD though the area the sample covers](https://john-chapman.github.io/2019/03/29/convolution.html) if you use mipmaps
3. Average all of the samples

## Source Code

**Note:** If you are writing HLSL, you can use `sincos()` to calculate both sine and cosine of a value in one instruction.

```glsl
void VogelSample(int Index, int SamplesCount, float Phi, out vec2 OutputValue)
{
    const float GoldenAngle = 2.4;
    float Radius = sqrt(float(Index) + 0.5) * inversesqrt(float(SamplesCount));
    float Theta = float(Index) * GoldenAngle + Phi;

    vec2 SineCosine;
    SineCosine[0] = sin(Theta);
    SineCosine[1] = cos(Theta);
    OutputValue = Radius * SineCosine;
}

void VogelConvolution(sampler2D Source, vec2 TexCoord, vec2 ScreenSize, float Radius, int Samples, float Phi, out vec4 OutputColor)
{
    // Initialize variables we need to accumulate samples and calculate offsets
    vec4 Output;
    vec2 Offset;

    // LOD calculation to fill in the gaps between samples
    const float Pi = 3.1415926535897932384626433832795;
    float SampleArea = Pi * (Radius * Radius) / float(Samples);
    float LOD = 0.5 * log2(SampleArea);
    
    // Offset and weighting attributes
    vec2 PixelSize = 1.0 / (ScreenSize / exp2(LOD));
    float Weight = 1.0 / (float(Samples) + 1.0);

    for(int i = 0; i < Samples; i++)
    {
        VogelSample(i, Samples, Phi, Offset);
        OutputColor += texture(Source, TexCoord + (Offset * PixelSize), LOD) * Weight;
    }
}

```

## Notes

+ `Phi` is an optional offset (you can use a noise function here)
+ Mipmapping is **optional**. You can use mipmapping or disable it and use a noise instead

## References

[Contact-hardening Soft Shadows Made Fast](http://maxest.gct-game.net/content/chss.pdf)

[Optimizing Convolutions](https://john-chapman.github.io/2019/03/29/convolution.html)
