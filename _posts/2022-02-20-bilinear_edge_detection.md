---
layout: post
date: 2022-02-20
title: Bilinear, Directional Edge Detection
category: Shaders
tags: [Computer Vision, Optimization]
---

Recently, I needed a large, 5x5 kernels for derivative detection. I did not want to sample 25 textures due to the buffer being half-resolution.

With the help of bilinear interpolation and CeeJayDK, we were able to approximate 5x5 edge detectors in 8 samples. This post gives 2, 5x5 kernels for the operation

+ One rotational (8-pixel neighborhood)
+ Non-rotational (10-pixel neighborhood)
