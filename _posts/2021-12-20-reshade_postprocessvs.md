---
layout: post
date: 2021-12-20
title: How It's Shade - PostProcessVS
category: Shaders
tags: [Post-Processing]
---

You may encounter this function called `PostProcessVS` and have no idea what it does.

```glsl
void PostProcessVS(in uint ID : SV_VERTEXID, out float4 Position : SV_POSITION, out float2 TexCoord : TEXCOORD0)
{
    TexCoord.x = (ID == 2) ? 2.0 : 0.0;
    TexCoord.y = (ID == 1) ? 2.0 : 0.0;
    Position = float4(TexCoord * float2(2.0, -2.0) + float2(-1.0, 1.0), 0.0, 1.0);
}
```

This post should explain how `PostProcessVS` works.

## "What Does PostProcessVS Do?"

`PostProcessVS` is a vertex shader that renders a triangle *twice* the program's [-1.0, 1.0] clip-space. The GPU will discard any information that exceeds the [-1.0, 1.0] range, turning the triangle into a quad.

## "What is `SV_VERTEXID`?

The `SV_VERTEXID` is an identifier assigned to each vertex. The system generates these IDs from 0 to `VertexCount`. For example, the `SV_VERTEXID` in a shader with `VertexCount = 3` will be [0, 1, 2]. Here is an example of a triangle with its respective vertex IDs

```md
1
.
...
.....
0    2
```

## Step 1. Call in Input and Output Semantics

Input Semantics | Output Semantics
:-------------: | :--------------:
`SV_VertexID`   | `SV_POSITION`
None            | `TEXCOORD0`

```glsl
void PostProcessVS(in uint ID : SV_VERTEXID, out float4 Position : SV_POSITION, out float2 TexCoord : TEXCOORD0)
```

## Step 2. Calculate Texture Coordinates with Vertex IDs

```glsl
/*
        1
    (0.0, 2.0)
        .
        . .
        .   .
        .     .
        ---------
        --------- .
        ---------   .
        ---------     .
        0             2
    (0.0, 0.0)    (2.0, 0.0)

    ID = 0 -> TexCoord = (0.0, 0.0)
    ID = 1 -> TexCoord = (0.0, 2.0)
    ID = 2 -> TexCoord = (2.0, 0.0)
*/

void PostProcessVS(in uint ID : SV_VERTEXID, out float4 Position : SV_POSITION, out float2 TexCoord : TEXCOORD0)
{
    TexCoord.x = (ID == 2) ? 2.0 : 0.0;
    TexCoord.y = (ID == 1) ? 2.0 : 0.0;
}
```

## Step 3. Calculate Vertex Positions with Texture Coordinates

```glsl
/*
        1
     (0.0, 2.0)
    [-1.0, 3.0]   [3.0, 3.0]
        .
        . .
        .   .
        .     .
        ---------
        --------- .
        ---------   .
        ---------     .
        0             2
     (0.0, 0.0)   (2.0, 0.0)
    [-1.0,-1.0]   [3.0,-1.0]

    ID = 0 -> Position = [-1.0, -1.0], TexCoord = (0.0, 0.0)
    ID = 1 -> Position = [-1.0,  3.0], TexCoord = (0.0, 2.0)
    ID = 2 -> Position = [ 3.0, -1.0], TexCoord = (2.0, 0.0)
*/

void PostProcessVS(in uint ID : SV_VERTEXID, out float4 Position : SV_POSITION, out float2 TexCoord : TEXCOORD0)
{
    TexCoord.x = (ID == 2) ? 2.0 : 0.0;
    TexCoord.y = (ID == 1) ? 2.0 : 0.0;
    Position = float4(TexCoord * float2(2.0, -2.0) + float2(-1.0, 1.0), 0.0, 1.0);
}
```

## References

[An interesting vertex shader trick](https://web.archive.org/web/20140719063725/http://www.altdev.co/2011/08/08/interesting-vertex-shader-trick/)
