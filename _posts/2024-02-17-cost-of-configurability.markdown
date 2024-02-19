---
layout: post
title:  "Gaussian blur shader in Unity - The cost of configurability"
categories: Graphics - Unity
---

# Gaussian blur shader in Unity - The cost of configurability

> ABSTRACT - Here, two versions of the same Gaussian blur shader were implemented in Unity as Render Features. From both implementations, separate builds were made, and run with NVidia NSight Graphics. Subsequently, their captures were explored. First shader supported client side-configurability for kernel size and sigma, and the gaussian weights were computed on a shader. For the other the Gaussian weights were precomputed and added to the shader as static constants. On both implementations, the kernel size was 21 x 21. Both shaders exhibited similarities in stalls and cache misses possibly due to similar requirements for texture reads. Unremarkably, the precomputed version executed less than half the ALU and FP operations compared to the configurable version. The precomputed version also proved to have higher SM occupancy. The conclusion is that although the texture reads are the most demanding part of both shaders, to fully optimize blur shaders, the demand for configurability should be questioned. Best results in optimizations are often gained by targeting use-case specific implementations over generic ones.
  
## Motivation
After working in the gaming industry for a while, it's come to my attention that performance related technical debt is often more welcomed than technical debt accumulating due to faulty programming practices. Issues with performance rarely arise from skill issues related to profiling and optimizing. They tend to accumulate over time because small things are neglected or not cared for. This is detrimental, and will eventually lead to unfixable situation, as the issues usually present themselves on later stages on development. At that point they are a result of tons of small things, as the accumulated computational overhead is everywhere, and impossible to fix. I wanted to start investigating different implementations both on CPU and GPU, and how they could be optimized.

## Configurability Bias
In case of shaders, configurability can be useful for iteration speed, so artists can tune the arguments to achieve the required visual outcome. However, after that outcome has been achieved, vast majority (almost all) of the configurations will remain the same forever. This, unsurprisingly, leads to higher computational overhead as well, and gets even worse if multiple textures are used to create an effect, where the same visual effect could be created by combining some visual elements in different textures or made with shader computations. 

For gaussian blur, investigated here, the difference in the outcome is not that dramatic, as the bottleneck appears to be the texture reads, which are always required, but still noticeable. The GPUs on low end mobile devices are not that great compared to high end development machines. In order to provide our players the best possible experience, performance in shader implementations should be better taken into account.

