---
layout: post
date: 2021-10-30
title: Efficient Gaussian Blur with Linear Sampling in GLSL
category: Shaders
tags: [Convolutions, Optimizations, Post-Processing]
---

You are probably familiar with [RasterGrid's efficient gaussian blur][0].

However, the article did not provide shader code for linear Gaussian blur. So, you end up using RasterGrid's pre-computed 7-taps or [port a shader from a seperate repository][1].

I wrote a GLSL snippet for us to use. Feel free to write your own calculations for Gaussian weight distribution, kernel size, and Sigma value.

## Source Code

### Shader (GLSL)

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

### C# Kernel Generator (.NET 6.0+)

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

// Create list of offsets and weights to print
var offsetList = new List<double>();
var weightList = new List<double>();

// Calculate center tap first
offsetList.Add(0.0);
weightList.Add(Gaussian(0, kernelTaps));

// Initialize loop values
int pixelIndex = 1;
int valueIndex = 0;

// Remaining taps (negate offsets for left-side taps)
while(pixelIndex < kernelTaps)
{
    int offset1 = pixelIndex;
    int offset2 = pixelIndex + 1;
    double weight1 = Gaussian(offset1, kernelTaps);
    double weight2 = Gaussian(offset2, kernelTaps);

    double linearWeight = weight1 + weight2;
    double linearOffset = ((offset1 * weight1) + (offset2 * weight2)) / linearWeight;

    offsetList.Add(linearOffset);
    weightList.Add(linearWeight);

    pixelIndex += 2;
    valueIndex += 1;
}

string totalOffsets = String.Join(", ", offsetList);
string totalWeights = String.Join(", ", weightList);
Console.WriteLine($"Offsets: {totalOffsets}\nWeights: {totalWeights}");
```

## References

[Efficient Gaussian blur with linear sampling][0]

[Optimized single-pass blur shaders for GLSL][1]

[0]: https://www.rastergrid.com/blog/2010/09/efficient-Gaussian-blur-with-linear-sampling/
[1]: https://github.com/Jam3/glsl-fast-gaussian-blur
