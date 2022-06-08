---
layout: post
date: 2021-11-12
title: How It's Shade - Grayscale
category: Shaders
tags: [Post-Processing, Tutorial]
---

Welcome, today is the day you make your first ReShade shader in 5 Steps.

I recommend reading the previous post on `void()` functions before proceeding.

## Glossary

Variable | Definition
-------- | ----------
`texture` | A set of values
`sampler` | Interpretation of a texture
`VertexShader` | Program that does does per-vertex calculations and outputs data to the `PixelShader`
`PixelShader` | Program that does per-pixel calculations and outputs data to a `RenderTarget`
`RenderTarget` | A target texture

## Step 1: Grab ReShade's BackBuffer

```glsl
// This code uses COLOR to read the most recent screen output. You cannot write to COLOR because ReShade does that for us
texture2D _RenderColor : COLOR;
```

## Step 2: Call the Sampler

```glsl
// The sampler interprets and prepares the texture for the PixelShader to read
sampler2D _SampleColor
{
    Texture = _RenderColor;
};
```

## Step 3: Write the Vertex Shader

```glsl
// A vertex shader outputs the vertex positions and texture coordinates the `PixelShader` requires.
// Feel free to just copy and paste `PostProcessVS` for now, you do not need to completely understand this right now
void PostProcessVS(in uint ID : SV_VERTEXID, out float4 Position : SV_POSITION, out float2 TexCoord : TEXCOORD0)
{
    // Output [0, 1] texture coordinates for the PixelShader to construct
    TexCoord.x = (ID == 2) ? 2.0 : 0.0;
    TexCoord.y = (ID == 1) ? 2.0 : 0.0;

    // Output the position of each vertex
    Position = float4(TexCoord * float2(2.0, -2.0) + float2(-1.0, 1.0), 0.0, 1.0);
}
```

## Step 4: Write the Pixel Shader

```glsl
void LuminancePS(in float4 Position : SV_POSITION, in float2 TexCoord : TEXCOORD0, out float4 OutputColor0 : SV_TARGET0)
{
    // Sample color
    float4 Color = tex2D(_SampleColor, TexCoord);

    // Output the maximum intensity of red|green|blue (1 float)
    // Assign the maximum intensity to all of OutputColor0's components
    OutputColor0.rgba = max(max(Color.r, Color.g), Color.b);
}
```

## Step 5: Assemble the Shader

```glsl
// A "technique" is the shader itself, give it a name
technique cLuminance
{
    // A pass is an iteration of the graphics pipeline you want to execute
    pass
    {
        // Always place the VertexShader before the PixelShader
        VertexShader = PostProcessVS;
        PixelShader = LuminancePS;
    }
}
```

## Source Code

```glsl
texture2D _RenderColor : COLOR;

sampler2D _SampleColor
{
    Texture = _RenderColor;
};

void PostProcessVS(in uint ID : SV_VERTEXID, out float4 Position : SV_POSITION, out float2 TexCoord : TEXCOORD0)
{
    TexCoord.x = (ID == 2) ? 2.0 : 0.0;
    TexCoord.y = (ID == 1) ? 2.0 : 0.0;
    Position = float4(TexCoord * float2(2.0, -2.0) + float2(-1.0, 1.0), 0.0, 1.0);
}

void LuminancePS(in float4 Position : SV_POSITION, in float2 TexCoord : TEXCOORD0, out float4 OutputColor0 : SV_TARGET0)
{
    float4 Color = tex2D(_SampleColor, TexCoord);
    OutputColor0 = max(max(Color.r, Color.g), Color.b);
}

technique cLuminance
{
    pass
    {
        VertexShader = PostProcessVS;
        PixelShader = LuminancePS;
    }
}
```

## References

[A slightly faster buffer-less vertex shader trick](https://www.reddit.com/r/gamedev/comments/2j17wk/a_slightly_faster_bufferless_vertex_shader_trick/)

[Graphics Pipeline](https://docs.microsoft.com/en-us/windows/win32/direct3d11/overviews-direct3d-11-graphics-pipeline)

[ReShade Reference](https://github.com/crosire/reshade-shaders/blob/slim/REFERENCE.md)

[Textures](https://docs.microsoft.com/en-us/windows/win32/direct3d11/overviews-direct3d-11-resources-textures#related-topics)
