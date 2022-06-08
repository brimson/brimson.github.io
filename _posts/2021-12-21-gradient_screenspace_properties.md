---
layout: post
date: 2021-12-22
title: Using Gradient Instructions to Retrieve Screen-space Properties 
category: Shaders
tags: [Post-Processing]
---

Sometimes you have to ride the tiger.

## The Problem

Here's a 2x2 screen

```glsl
A B
C D
```

The following code calculates the example's pixel size

```glsl
// ScreenSize = ivec2(2, 2)
// This should return 0.5 for a 2x2 screen
// 0.5 == Each pixel occupies 50% of X and Y axis
float2 PixelSize = 1.0 / ScreenSize.xy;
```

However, what if you do not have access to your screen's resolution? There is a solution to this issue if you have the right resources.

## A Solution: Gradient Instructions

You can calculate screenspace properties by using normalized texture coordinates, `ddx()`, and `ddy()`.

Lets use a 2x2 screen example again

```glsl
A B
C D
```

Now, display the screen example as normalized texture coordinates

```glsl
1.0                           1.0
   A(0.25, 0.75) C(0.75, 0.75)
   B(0.25, 0.25) D(0.75, 0.25)
0.0                           1.0
```

We have access to derivative instructions in Shader Model 3.0+. Therefore, we can do following screen-space functions:

- `ddx(x)` -> `D(0.75, 0.25) - B(0.25, 0.25)` -> `(0.5, 0.0)`
- `ddy(x)` -> `A(0.25, 0.75) - B(0.25, 0.25)` -> `(0.0, 0.5)`

The functions tell us that the 2D texture coordinates' rate-of-change is 0.5. In other words, it takes 2 pixels for the texture coordinates to go from 0.0 to 1.0.

Therefore, we obtain the same results as the formula in the problem section.

## Source Code

### Pixel Size

```glsl
// Assume TexCoord.xy are normalized screenspace texture coordinates
float2 PixelSize = float2(ddx(TexCoord.x), ddy(TexCoord.y));
```

### Resolution

```glsl
float2 ScreenSize = 1.0 / float2(ddx(TexCoord.x), ddy(TexCoord.y));
```

### Aspect Ratio

```glsl
float2 ScreenSize = 1.0 / float2(ddx(TexCoord.x), ddy(TexCoord.y));
float AspectRatio = ScreenSize.x / ScreenSize.y;
```
