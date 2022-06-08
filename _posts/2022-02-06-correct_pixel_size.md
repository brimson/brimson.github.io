---
layout: post
date: 2022-02-06
title: Correctly Calculating Pixel Sizes
category: Shaders
tags: [Post-Processing, Tutorial]
---

It is essential to calculate correct pixel-sizes when writing blur shaders to a lower resolution.

We will address a scenario where the variable `ScreenSize.xy` is `vec2(1920.0, 1080.0)`, but the target resolution is **7x4** (1/256th of **1920x1080**).

## Naive Method

You may calculate pixel size by applying a reciprocal on the target resolution.

```glsl
vec2 PixelSize = 1.0 / (vec2(ScreenSize.xy) / 256.0);
```

This is an incorrect way to calculate pixel size because the result becomes `vec2(0.13333334, 0.237037033)`. The hypothetical screen-size becomes `vec2(7.5, 4.21875)`. Textures do not have decimal resolutions.

## Correct Method

You need to divide the resolution, cast the quotient into an integer, and apply the reciprocal.

```glsl
vec2 PixelSize = 1.0 / ivec2(vec2(ScreenSize.xy) / 256.0);
```

The pixel size becomes `vec2(0.142857149, 0.25)`, with the hypothetical screen-size becoming `vec2(7, 4)`.
