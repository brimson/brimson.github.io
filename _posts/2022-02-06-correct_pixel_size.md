---
layout: post
date: 2022-02-06
title: Correctly Calculating Pixel Sizes
category: Shaders
tags: [Post-Processing, Tutorial]
---

It is important to calculate correct pixel-sizes when writing shaders such as discrete convolutions.

## Scenario

1. Your backbuffer's resolution is `1920*1080`, something the user can change anytime
    + Your application provides your the backbuffer's resolution in the form of `vec2(BUFFER_WIDTH, BUFFER_HEIGHT)`
    + You do not have a function that retrieves the screen size from any other textures
2. You are writing a shader that does the following
    + Applies Gaussian convolution by offsetting textures
        + Requires knowledge of the render-target's pixel-size
    + Writes the result to a `7*4` render-target. 256 times smaller than the backbuffer

## Naive Method

You may calculate the render-target's pixel size by applying a reciprocal on the divided backbuffer resolution.

```glsl
vec2 PixelSize = 1.0 / (vec2(BUFFER_WIDTH, BUFFER_HEIGHT) / 256.0);
```

This is an incorrect way to calculate pixel size because the hypothetical screen-size becomes `vec2(7.5, 4.21875)`. Textures almost never have decimal resolutions.

Performing a reiprocal on the screen-size gives pixel sizes of `vec2(0.13333334, 0.237037033)`.

## Correct Method

The correct method is to divide the resolution, cast the quotient into an integer, and reciprocal the quotient.

```glsl
vec2 PixelSize = 1.0 / ivec2(vec2(BUFFER_WIDTH, BUFFER_HEIGHT) / 256.0);
```

The hypothetical screen-size ends up being `vec2(7, 4)`, giving a pixel size of `vec2(0.142857149, 0.25)`.
