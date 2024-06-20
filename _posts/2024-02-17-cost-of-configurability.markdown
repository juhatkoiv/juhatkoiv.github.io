---
layout: post
title:  "Gaussian blur shader in Unity - The cost of configurability"
categories: Graphics - Unity
---

# Gaussian blur shader in Unity - The cost of configurability

> ABSTRACT - Here, two versions of the same Gaussian blur shader were implemented in Unity as Render Features. From both implementations, separate builds were made, and run with NVidia NSight Graphics. Subsequently, their captures were explored. First shader supported client side-configurability for kernel size and sigma, and the gaussian weights were computed on a shader. For the other the Gaussian weights were precomputed and added to the shader as static constants. On both implementations, the kernel size was 21 x 21. Both shaders exhibited similarities in stalls and cache misses possibly due to similar requirements for texture reads. Unremarkably, the precomputed version executed less than half the ALU and FP operations compared to the configurable version. The precomputed version also proved to have higher SM occupancy, and on furhter optimizations, also higher SM PS throughput. The conclusion is that although the texture reads are the most demanding part of both shaders, to fully optimize blur shaders, the demand for configurability should be questioned. Best results in optimizations are often gained by targeting use-case specific implementations over generic ones.
  
## Motivation
After working in the gaming industry for a while, it's come to my attention that performance related technical debt is often more welcomed than technical debt accumulating due to faulty programming practices. Issues with performance rarely arise from skill issues related to profiling and optimizing. They tend to accumulate over time because small things are neglected or not cared for. This is detrimental, and will eventually lead to unfixable situation, as the issues usually present themselves on later stages on development. At that point they are a result of tons of small things, as the accumulated computational overhead is everywhere, and impossible to fix. I have been passionte about computer grpahics for a long time and, I wanted to start investigating graphics implementations both on CPU and GPU, and how they could be optimized.


## Configurability Bias
In case of shaders, configurability can be useful for iteration speed, so artists can tune the arguments to achieve the required visual outcome. However, after that outcome has been achieved, vast majority (almost all) of the configurations will remain the same forever. This, unsurprisingly, leads to higher computational overhead as well, and gets even worse if multiple textures are used to create an effect, where the same visual effect could be created by combining some visual elements in different textures or made with shader computations. 

For gaussian blur, investigated here, the difference in the outcome is not that dramatic, as the bottleneck appears to be the texture reads, which are always required, but still noticeable. The GPUs on low end mobile devices are not that great compared to high end development machines. In order to provide our players the best possible experience, performance in shader implementations should be better taken into account.


## Setup

### Hardware 
* ASUS ROG Strix 
* 32 GB RAM
* Gfx 3080 Ti

### Software
* Unity 2022.3.8f1
* URP High Fidelity PR (Quality settings: Ultra)

The scene contained one cube and a Blur render feature. Blur was implemented using two passes: first one for vertical, second for horizontal blurring. Three fields - Sigma, BlurVertical and BlurHorizontal signifying FWHM/2.355, vertical and horizontal kernel sizes, respectively. Kernel size was 21 pixels per dimension, and sigma 5. 

Implemented several versions of the same blur shader. 
1. One with constant buffer with three the fields configurable from the UI. Gaussian weights were calculated in the pixel shader based on the offset form the processed pixel. 
2. Using the same values, but gaussian weights were precomputed and added to the shader as static const array of floats.

### Implementation for (1).

	int _VerticalBlur;
	int _HorizontalBlur;
	int _Sigma;
	float4 _BlitTexture_TexelSize;

	float gauss(int x0, int x, float sigma)
	{
		return exp(-(x - x0) * (x - x0) / (2 * sigma * sigma)) / (sqrt(2 * 3.14159) * sigma);
	}

	float4 BlurVertical(Varyings input) : SV_Target
	{
		float3 color = float3(0, 0, 0);
		float unitOffset = _BlitTexture_TexelSize.y;

		float sum = 0;
		for(int i = -_VerticalBlur; i <= _VerticalBlur; i++)
		{
			float2 offset = float2(0, unitOffset * i);
			
			float coeff = gauss(0, i, _Sigma);
			sum += coeff;
			color += coeff * SAMPLE_TEXTURE2D(_BlitTexture, sampler_LinearClamp, input.texcoord + offset).rgb;
		}
		return float4(color.rbg / sum, 1);
	}

	float4 BlurHorizontal(Varyings input) : SV_Target
	{   
		float3 color = float3(0, 0, 0);
		float unitOffset = _BlitTexture_TexelSize.x;

		float sum = 0;
		for(int i = -_HorizontalBlur; i <= _HorizontalBlur; i++)
		{
			float2 offset = float2(unitOffset * i, 0);

			float coeff = gauss(0, i, _Sigma);
			sum += coeff;
			color +=coeff * SAMPLE_TEXTURE2D(_BlitTexture, sampler_LinearClamp, input.texcoord + offset).rgb;
		}
		return float4(color.rbg / sum, 1);
	}

### Implementation for (2).

	float4 _BlitTexture_TexelSize;
	
	static const float kernel[21] = { 0.2699548326, 0.3947507915, 0.5546041734, 0.7486373282, 0.9709302749, 1.209853623, 1.448457764, 1.666123014, 1.841350702, 1.95521347, 1.994711402, 1.95521347, 1.841350702, 1.666123014, 1.448457764, 1.209853623, 0.9709302749, 0.7486373282, 0.5546041734, 0.3947507915, 0.2699548326 };

	static const float sum = 24.11446335;
	static const int kernelSize = 10;

	float4 BlurVertical(Varyings input) : SV_Target
	{
		float unitOffset = _BlitTexture_TexelSize.y;

		float3 color = float3(0, 0, 0);
		
		for(int i = 0; i <= 20; i++)
		{
			color += kernel[i] * SAMPLE_TEXTURE2D(_BlitTexture, sampler_LinearClamp, input.texcoord + float2(0, unitOffset * (i - kernelSize))).rgb;
		}

		return float4((color.rbg) / sum, 1);
	}

	float4 BlurHorizontal(Varyings input) : SV_Target
	{   
		float unitOffset = _BlitTexture_TexelSize.x;
		
		float3 color = float3(0, 0, 0);
		
		for(int i = 0; i <= 20; i++)
		{
			color += kernel[i] * SAMPLE_TEXTURE2D(_BlitTexture, sampler_LinearClamp, input.texcoord + float2(unitOffset * (i - kernelSize), 0)).rgb;
		}

		return float4((color.rbg) / sum, 1);
	}


### Measurement and Diagnostics
* Initial analysis was done with RenderDoc by capturing individual frames and inspecting the data in Performance Counter Viewer.
	* DirectX11 builds.
* More deeper analysis was done with NVidia Nsight Graphics.
	* Vulkan builds.

#### RenderDoc
The variations in the execution times of the both passes, for both (1) and (2) varied from ~1100us to ~1700us depending on when the frame was captured. Without further data and statistics, it's not enough to make any statements to why this happened. The only conclusion that can be made from the RenderDoc data was that the blur postprocessing passes were the bottleneck.

#### NVidia NSight Graphics
// TODO 

