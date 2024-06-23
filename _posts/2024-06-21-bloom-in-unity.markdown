---
layout: post
title: Bloom in Unity URP 
categories: Graphics - Unity
---

# Bloom in Unity URP

Unity SRP offers a flexible way to add render features, and comes with a number
of pre-implemented postprocessing effects. Bloom is one of them. The physical basis for bloom is considered 
to occur due to the fact that lenses never focus perfectly. Photons emerging from 
point-light source passing though a circular aperture, will result in diffraction 
pattern, known as Airy disk. The aperture in question can be for example retina in human eye. 
When the light intensity is low enough, the lower intensity components contributing 
to the pattern are not noticable. When the intensity is high enough, these components 
start to be visible. Another could be an Cherenkov Radiation, where a charged particles 
pass through a dielctric medium, such as water, with high enough velocity. 
Another place where bloom, or glow, can be withnessed is when high power lasers interact 
with air (or some other gas phase media), and the gas molecules ionize and then again recombine.
This effect can result in emission of electromagnetic raditation, where the wavelenght depends
on the media.

In games, bloom is a nice visual effect used in 3D for pronouncing lit surfaces, light sources and reflections 
but also in 2D games to add glowing and similar effects. It adds tons to the visual experience,
but is also quite hardware intensive.

The implementation of bloom in computer graphics attemps to approximate the Airy Disk, with some blur filter.
For bloom, the blurring is done by alternating vertical and horizontal blurring to ~2 x downscaled textures,
and then upscaling the textures one by one, finally combining the results with the final resolution image.

Let's look at RenderDoc capture of bloom to figure out hot it's implemented
in Unity. 

As we can see, the effect takes 17 subsequent draw calls to render, for each call the color buffer of the previous
pass is passed, with some additional textures, and in total involves using 7 mips. With the current setup, with 1920 x 1080 resolution, the mips are:
- 1920 x 1080
- 960 x 540
- 480 x 270
- 240 x 135
- 120 x 67
- 60 x 33
- 30 x 16

Conceptually, this approach is fine. Let's see how it performs:

[HERE]

In SIGGRAPH 2015 Marius Bjørge presented another way to do bloom. The solution involves
linear sampling [X], Kawase blur pattern, or a variation of it, "Dual Filtering". Let's implement it as
a rendering feature in Unity.

[HERE]

Compute shader approach.

## Setup

### Hardware 
* ASUS ROG Strix 
* 32 GB RAM
* Gfx 3080 Ti (laptop gpu)

### Software
* Unity 2022.3.8f1
    * URP High Fidelity PR (Quality settings: Ultra)
* RenderDoc 1.33
* NVidia NSight Graphics 2024.1.0
 
