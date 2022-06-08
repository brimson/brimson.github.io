---
layout: post
date: 2021-11-12
title: How It's Shade - Void Functions
category: Shaders
tags: [Post-Processing, Tutorial]
---

**void()** functions are essential for vertex and pixel shaders. Void functions allow you to output multiple values.

Here is an example of a `void` function.

```glsl
// The following void function outputs 0.0
void Function_PS(out float4 OutputValue)
{
    OutputValue = 0.0;
}
```

Do this to call a void function within a function

```glsl
float4 Function()
{
    // Initialize variable
    float4 Value = 0.0;

    // Run Function_PS(x), which outputs the result to the "Value" variable
    Function_PS(Value);
    return Value;
}
```

## "What are semantics?"

Semantics represent data passed between stages in the graphics pipeline. Here is an example with the `TEXCOORD0` semantic:

Function | Description
-------- | -----------
`out float2 TexCoord : TEXCOORD0` | A vertex shader outputs the `TEXCOORD0` semantic for texture coordinates
`in float2 TexCoord : TEXCOORD0`  | The pixel shader grabs the `TEXCOORD0` data from the vertex shader's output semantic

First, let's dissect `out float2 TexCoord : TEXCOORD0` step-by-step

Step | Function | Description
:--: | -------- | -----------
1    | `out`                             | The variable will be an output
2    | `out float2`                      | The variable will have a `float2` data type
3    | `out float2 TexCoord`             | The variable will have a name *TexCoord*
4    | `out float2 TexCoord : TEXCOORD0` | The variable will output data to the `TEXCOORD0` semantic

What about `in float2 TexCoord : TEXCOORD0`?

Step | Function | Description
:--: | -------- | -----------
1    | `in`                             | The variable will be an input
2    | `in float2`                      | The variable will have a `float2` data type
3    | `in float2 TexCoord`             | The variable will have a name *TexCoord*
4    | `in float2 TexCoord : TEXCOORD0` | The variable will receive data from the `TEXCOORD0` semantic

## Step 1. Name Your Function

```glsl
// Call a void function called **Function_PS**
void Function_PS()
```

## Step 2. Write the input semantics

```glsl
// "in" tells the compiler that you want to input two semantics: SV_POSITION and TEXCOORD0
void Function_PS(in float4 Position : SV_POSITION, in float2 TexCoord : TEXCOORD0)
```

## Step 3. Write the output semantics

```glsl
// "out" tell the compiler that you want to output one semantic: SV_TARGET0 
void Function_PS(in float4 Position : SV_POSITION, in float2 TexCoord : TEXCOORD0, out float4 OutputColor0 : SV_TARGET0)
```

## Step 4. Write the value the output semantic should have

```glsl
void Function_PS(in float4 Position : SV_Position, in float2 TexCoord : TEXCOORD0, out float4 OutputColor0 : SV_TARGET0)
{
    // Output texture coordinate value to red|green components
    OutputColor0 = float4(TexCoord, 0.0, 1.0);
}
```

## Source Code

```glsl
void PostProcessVS(in uint ID : SV_VERTEXID, out float4 Position : SV_POSITION, out float2 TexCoord : TEXCOORD0)
{
    TexCoord.x = (ID == 2) ? 2.0 : 0.0;
    TexCoord.y = (ID == 1) ? 2.0 : 0.0;
    Position = float4(TexCoord * float2(2.0, -2.0) + float2(-1.0, 1.0), 0.0, 1.0);
}

void Function_PS(in float4 Position : SV_Position, in float2 TexCoord : TEXCOORD0, out float4 OutputColor0 : SV_TARGET0)
{
    // Output texture coordinate value to red|green components
    OutputColor0 = float4(TexCoord, 0.0, 1.0);
}

technique Function
{
    pass
    {
        VertexShader = PostProcessVS;
        PixelShader = Function_PS;
    }
}
```

## References

[Semantics](https://docs.microsoft.com/en-us/windows/win32/direct3dhlsl/dx-graphics-hlsl-semantics)

[Return Statement](https://docs.microsoft.com/en-us/windows/win32/direct3dhlsl/dx-graphics-hlsl-return)
