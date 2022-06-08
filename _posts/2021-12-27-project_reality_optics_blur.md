---
layout: post
date: 2021-12-22
title: Updating Optics Blur for Project Reality
category: Shaders
tags: [Post-Processing, Optimization]
---

I am working on shaders for Project Reality, a Battlefield 2 modification. The Refractor games allow users to modify the game's shaders.

Here is a snippet of Project Reality's optics blur shader as of `1.6.8.2`

```glsl
float highPassGate : HIGHPASSGATE; // 3d optics blur; xxxx.yyyy; x - aspect ratio(H/V), y - blur amount(0=no blur, 0.9=full blur)

struct VS2PS_tr_blit
{
    float4 Pos : POSITION;
    float2 TexCoord0 : TEXCOORD0;
};

VS2PS_tr_blit vsDx9_tr_blit(APP2VS_blit indata) // TODO: implement support for old shader versions. TODO: try to use fakeHDRWeights as variables
{
    VS2PS_tr_blit outdata;
    outdata.Pos = float4(indata.Pos.x, indata.Pos.y, 0, 1);
    outdata.TexCoord0 = indata.TexCoord0;
    return outdata;
}

const float tr_gauss[9] = { 0.087544737,0.085811235,0.080813978,0.073123511,0.063570527,0.053098567,0.042612598,0.032856512,0.024340702 };

float4 psDx9_tr_opticsBlurH(VS2PS_tr_blit indata) : COLOR
{
    float1 aspectRatio = highPassGate/1000.f; // floor() isn't used for perfomance reasons
    float1 blurSize = 0.0033333333/aspectRatio;
    float4 color = tex2D(sampler0point, indata.TexCoord0)*tr_gauss[0];
    for (int i=1;i<9;i++)
    {
        color += tex2D(sampler0bilin, float2(indata.TexCoord0.x + i*blurSize, indata.TexCoord0.y))*tr_gauss[i];
        color += tex2D(sampler0bilin, float2(indata.TexCoord0.x - i*blurSize, indata.TexCoord0.y))*tr_gauss[i];
    }
    return color;
}

float4 psDx9_tr_opticsBlurV(VS2PS_tr_blit indata) : COLOR
{
    float1 blurSize = 0.0033333333; // 1/300 - no ghosting for vertical resolutions up to 1200 pixels
    float4 color = tex2D(sampler0point, indata.TexCoord0)*tr_gauss[0];
    for (int i=1;i<9;i++)
    {
        color += tex2D(sampler0bilin, float2(indata.TexCoord0.x, indata.TexCoord0.y + i*blurSize))*tr_gauss[i];
        color += tex2D(sampler0bilin, float2(indata.TexCoord0.x, indata.TexCoord0.y - i*blurSize))*tr_gauss[i];
    }
    return color;
}
```

There are a couple of issues with this code that I will explain and provide solutions for.

## Problems

### Blur Mismatching

The blur offsets are independent from screen resolution, causing "holes" in-between the samples.

```glsl
// Calculating horizontal offset size
float1 aspectRatio = highPassGate/1000.f; // floor() isn't used for perfomance reasons
float1 blurSize = 0.0033333333/aspectRatio;
```

```glsl
// Calculating vertical offset size
float1 blurSize = 0.0033333333; // 1/300 - no ghosting for vertical resolutions up to 1200 pixels
```

Are these calculations accurate? Maybe not. The aspect ratio of a 16x9 image is the same as 1600x900. As a result, we may see ghosting on lower-resolution screens.

### Discrete Sampling

The blur samples textures **on each pixel**, which is slow.

```glsl
float2(indata.TexCoord0.x + i * blurSize, indata.TexCoord0.y)
```

## Limitations

+ No documentation
+ Cannot modify what the shader outputs to (RenderTargets)
+ No global, custom access to essential system data such as
  + Matrices
  + Pixel size
  + Screen size
  + Texel size
  + Textures

However, we do have a couple of silver linings

+ Access to normalized texture coordinates
+ Access to Shader Model 3.0 (ps_3_0 and vs_3_0)

## Solutions

The solutions for these two issues are simple

1. Use derivative instructions to dynamically calculate offsets

2. Use RasterGrid's Gaussian blur optimization to halve the blur's cost

### Blur Mismatching: Derivative Instructions

The pixel size is a value on how much a pixel occupies the whole screen. In a blog post, I explained how calculating the derivates of a [0, 1] texture coordinate gives us pixel size.

```glsl
float2 PixelSize2D = float2(ddx(TexCoord.x), ddy(TexCoord.y))
```

> [Read about the post here][1]

### Discrete Sampling: Linear Gaussian Blur

We will sample between pixels instead of on each pixel. I used the C# kernel generator to output offsets and weights for a 18-tap (9 + 9), 16x16 Gaussian blur.

```glsl
const float Offsets[5] =
{
    0.0,
    1.4584295167832,
    3.4039848066734835,
    5.351805780136256,
    7.302940716034593
};

const float Weights[5] =
{
    0.1329807601338109,
    0.2322770777384485,
    0.13532693306504567,
    0.05115603510197893,
    0.012539291705835646
};
```

> [Read about the post here][0]

## Source Code

```glsl
struct VS2PS_tr_blit
{
    float4 Pos : POSITION;
    float2 TexCoord0 : TEXCOORD0;
};

VS2PS_tr_blit vsDx9_tr_blit(APP2VS_blit indata) // TODO: implement support for old shader versions. TODO: try to use fakeHDRWeights as variables
{
    VS2PS_tr_blit outdata;
    outdata.Pos = float4(indata.Pos.xy, 0.0, 1.0);
    outdata.TexCoord0 = indata.TexCoord0;
    return outdata;
}

const float Offsets[5] =
{
    0.0,
    1.4584295167832,
    3.4039848066734835,
    5.351805780136256,
    7.302940716034593
};

const float Weights[5] =
{
    0.1329807601338109,
    0.2322770777384485,
    0.13532693306504567,
    0.05115603510197893,
    0.012539291705835646
};

float4 OpticsBlur(sampler2D Source, float2 TexCoord, float2 Direction)
{
    float4 OutputColor = 0.0;
    float4 TotalWeights = 0.0;
    float2 PixelSize2D = float2(ddx(TexCoord.x), ddy(TexCoord.y)) * Direction;

    OutputColor += tex2D(Source, TexCoord + (Offsets[0] * PixelSize2D)) * Weights[0];
    TotalWeights += Weights[0];

    for(int i = 1; i < 5; i++)
    {
        OutputColor += tex2D(Source, TexCoord + (Offsets[i] * PixelSize2D)) * Weights[i];
        OutputColor += tex2D(Source, TexCoord - (Offsets[i] * PixelSize2D)) * Weights[i];
        TotalWeights += (Weights[i] * 2.0);
    }

    return OutputColor / TotalWeights;
}

float4 psDx9_tr_opticsBlurH(VS2PS_tr_blit indata) : COLOR
{
    return OpticsBlur(sampler0bilin, indata.TexCoord0, float2(1.0, 0.0));
}

float4 psDx9_tr_opticsBlurV(VS2PS_tr_blit indata) : COLOR
{
    return OpticsBlur(sampler0bilin, indata.TexCoord0, float2(0.0, 1.0));
}
```

## References

[Efficient Gaussian Blur with Linear Sampling in GLSL][0]

[Using Gradient Instructions to Retrieve Screen-space Properties][1]

[0]: https://brimson.github.io/shaders/2021/10/30/lineargaussianblur.html

[1]: https://brimson.github.io/shaders/2021/12/22/gradientscreensize.html
