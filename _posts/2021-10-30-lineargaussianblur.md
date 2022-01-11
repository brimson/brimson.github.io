---
layout: post
date: 2021-10-30
title: Efficient Gaussian Blur with Linear Sampling in GLSL
category: Shaders
tags: [Convolutions, Optimizations, Post-Processing]
---

You are probably familiar with [RasterGrid's efficient gaussian blur](https://www.rastergrid.com/blog/2010/09/efficient-Gaussian-blur-with-linear-sampling/).

However, the article did not provide shader code for linear Gaussian blur. So, you end up using RasterGrid's pre-computed 7-taps or [port a shader from a seperate repository](https://github.com/Jam3/glsl-fast-gaussian-blur).

I wrote a GLSL snippet for us to use. Feel free to write your own calculations for Gaussian weight distribution, kernel size, and Sigma value.

## Source Code

### Shader

```glsl
float Gaussian(float PixelIndex, float Sigma)
{
    const float Pi = 3.1415926535897932384626433832795f;
    float Output = inversesqrt(2.0 * Pi * (Sigma * Sigma));
    return Output * exp(-(PixelIndex * PixelIndex) / (2.0 * Sigma * Sigma));
}

vec4 GaussianBlur(sampler2D Source, vec2 TexCoord, vec2 BufferSize, vec2 Direction, float Sigma)
{
    vec2 PixelSize = (1.0 / BufferSize) * Direction;
    float KernelSize = Sigma * 3.0;

    // Sample and weight center first to get even number sides
    float TotalWeight = Gaussian(0.0, Sigma);
    vec4 OutputColor = texture(Source, TexCoord) * TotalWeight;

    // May replace texture() with textureLod() if needed!
    for(float i = 1.0; i < KernelSize; i += 2.0)
    {
        float PixelOffset1 = i;
        float PixelOffset2 = i + 1.0;
        float PixelWeight1 = Gaussian(PixelOffset1, Sigma);
        float PixelWeight2 = Gaussian(PixelOffset2, Sigma);
        float PixelWeightLinear = PixelWeight1 + PixelWeight2;
        float PixelOffsetLinear = ((PixelOffset1 * PixelWeight1) + (PixelOffset2 * PixelWeight2)) / PixelWeightLinear;

        OutputColor += texture(Source, TexCoord - PixelOffsetLinear * PixelSize) * PixelWeightLinear;
        OutputColor += texture(Source, TexCoord + PixelOffsetLinear * PixelSize) * PixelWeightLinear;
        TotalWeight += PixelWeightLinear * 2.0;
    }

    // Normalize intensity to prevent altered output
    return OutputColor / TotalWeight;
}
```

### C# Kernel Generator

```csharp
static double Gaussian(int pixelIndex, double kernelSize)
{
    double pi = 3.141592653589793238462643;
    double sigma = kernelSize / 3.0;
    double output = 1.0 / Math.Sqrt(2.0 * pi * (sigma * sigma));
    return output * Math.Exp(-(pixelIndex * pixelIndex) / (2.0 * (sigma * sigma)));
}

Console.WriteLine("Enter kernel size:");
int kernelTaps = Convert.ToInt32(Console.ReadLine());
int pixelIndex = 1;
int valueIndex = 0;

var linearOffsetList = new List<double>();
var linearWeightList = new List<double>();

// Calculate center tap first
linearOffsetList.Add(0.0);
linearWeightList.Add(Gaussian(0, kernelTaps));

// Remaining taps (negate offsets for left-sided taps)
while(pixelIndex < kernelTaps)
{
    int pixelOffset1 = pixelIndex;
    int pixelOffset2 = pixelIndex + 1;
    double pixelWeight1 = Gaussian(pixelOffset1, kernelTaps);
    double pixelWeight2 = Gaussian(pixelOffset2, kernelTaps);

    double pixelWeightLinear = pixelWeight1 + pixelWeight2;
    double pixelOffsetLinear = ((pixelOffset1 * pixelWeight1) + (pixelOffset2 * pixelWeight2)) / pixelWeightLinear;

    linearOffsetList.Add(pixelOffsetLinear);
    linearWeightList.Add(pixelWeightLinear);

    pixelIndex += 2;
    valueIndex += 1;
}

Console.WriteLine($"Offsets: {string.Join(", ", linearOffsetList)}");
Console.WriteLine($"Weights: {string.Join(", ", linearWeightList)}");
```

## References

[Efficient Gaussian blur with linear sampling](https://www.rastergrid.com/blog/2010/09/efficient-Gaussian-blur-with-linear-sampling/)

[Optimized single-pass blur shaders for GLSL](https://github.com/Jam3/glsl-fast-gaussian-blur)
