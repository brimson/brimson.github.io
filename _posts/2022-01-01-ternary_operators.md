---
layout: post
date: 2022-01-01
title: Compare Instruction Appreciation Post
category: Shaders
tags: [Post-Processing, Optimizations]
---

I was writing a red-black checkerboard shader but feared that I would trigger flow control. 

```glsl
void main(void)
{
    float RedBlack = fract(dot(gl_FragCoord.xy, vec2(0.5))) * 2.0;
    gl_FragColor = RedBlack == 1.0 ? vec4(1.0, 0.0, 0.0, 0.0) : vec4(0.0);
}
```

I was wrong. In best case scenario, the shader would lead us to the following output:

```asm
dp3
frc
mul
cmp
```

Shader programs have conditional instructions that are arithmetic, not flow-control. Pixel shaders have `cmp`, an instruction that uses `src0` to choose between `src1` and `src2`. The vertex shader have counterparts to `cmp` in the form of `slt` and `sge`.

## References

[ARB_fragment_program](https://www.khronos.org/registry/OpenGL/extensions/ARB/ARB_fragment_program.txt)

[ARB_vertex_program](https://www.khronos.org/registry/OpenGL/extensions/ARB/ARB_vertex_program.txt)

[ps_3_0 Instructions](https://docs.microsoft.com/en-us/windows/win32/direct3dhlsl/dx9-graphics-reference-asm-ps-instructions-ps-3-0)

[Instructions - vs_3_0](https://docs.microsoft.com/en-us/windows/win32/direct3dhlsl/dx9-graphics-reference-asm-vs-instructions-vs-3-0)
