---
layout: post
title:  "Shader System"
date:   2025-10-20 15:38:00 +0200
categories: jekyll update
---

## Introduction
TODO

## Disambiguation

"Shader" is a term that is quite overloaded and can mean different things depending on the context/who you ask:
- A single compiled program that can run on the GPU (e.g. a "Vertex Shader")
- A single compiled pipeline with possibly multiple programs, one for each stage (e.g. Vertex Shader + Fragment Shader)
- A collection of pipelines, that are needed to render an object in an engine (e.g. Depth Prepass, GBuffer, Forward, etc.)
This is what Unity calls a shader. And this is what you may be familiar with as a Material in Unreal (which is quite a confusing term on top of another confusing term :) )

And for all of the above it can be either the source code form or the compiled form that can run on your GPU. So here's how I disambiguated this overloading in my engine:
- ShaderProgram: A single compiled program for a single stage
- ShaderVariant: A set of ShaderPrograms (up to one for each possible stage) that can form a full functional pipeline. They are called ShaderVariant because you can have multiple similar variations of your
shader based on some configuration. e.g. one that has Alpha Clipping support and one that does not.
- ShaderPass: A set of ShaderVariants that perform a similar function. e.g. there is one ShaderPass for the DepthPrepass and one for the ForwardPass, etc.
- Shader: A set of ShaderPasses. This is what you will see in the editor as an asset and is also what can be assigned to materials

TODO diagram?

## The anatomy of a shader

Lets look at a simple shader example like this blit shader:

{% highlight hlsl %}
#pragma Stage VS
#pragma Stage PS

struct VertexOutput {
    float2 uv : TEXCOORD0;
    float4 vertex : SV_POSITION;
};

VertexOutput vert(uint id : SV_VertexID) {
    VertexOutput output;
  
    output.uv = float2((id << 1) & 2, id & 2);
    output.vertex = float4(output.uv * float2(2, -2) + float2(-1, 1), 0, 1);
    
    return output;
}

Texture2D tex : register(t0, space0);
SamplerState texSampler : register(s0, space1);

struct FragmentOuput {
    float4 color : SV_TARGET;
};

FragmentOuput frag(VertexOutput input) {
    FragmentOuput output;
	
    output.color = tex.Sample(texSampler, input.uv);
    
    return output;
}
{% endhighlight %}

<br>

This is almost completely standard HLSL. I wanted my shaders to be compilable by standard tools like dxc. This can be very helpful when trying to debug my compilation pipeline.
I can compile the shader with the usual command line tools and compare the output with what my engine generated. Also, I can paste it into the great [Shader Playground](https://shader-playground.timjones.io) and look at the generated assembly
or even try to cross compile it right in the browser.

It also means that you have very few abstraction layers between you and the shader. Even resource binding is fully up to you. Buffers, Textures and Samplers can assigned bindfull or in a bindless. Parameters can be passed as RootConstants, ConstantBuffers, ByteAddressBuffers, etc.

There are only 4 additions to the language itself in the form of #pragma directives: Stage, Feature, Option and OptionAdd.

# #pragma Stage
#pragma Stage is there to inform the compiler of each pipeline stage that is present in this source file. The supported stages in my engine are:
- Vertex
- Amplification
- Mesh
- Fragment
- Compute
- RayTracing: RayGeneration
- RayTracing: AnyHit
- RayTracing: ClosestHit
- RayTracing: Miss
- RayTracing: Callable

Early versions of the engine supported geometry shaders but I dropped them to keep the code as simple as possible and I just don't have the need for geometry shaders in my engine. The tesselation shader stages were never supported for the same reason.

It is possible (and often even required) to have multiple stages in the same source file. But each stage can only be present once. Also each stages entry point name is fixed, so if the shader compiler sees #pragma Stage VS it will always try to compile with "vert" as
the main function.

# #pragma Feature/Option/OptionAdd
#pragma Feature, #pragma Option and #pragma OptionAdd allow you to create ShaderVariants. They will produce preprocessor defines that you can check for in the code.

If the compiler finds a feature like
{% highlight hlsl %}
#pragma Feature ENABLE_ALPHA_CLIPPING
{% endhighlight %}
it will compile the shader twice. Once with the define and once without it. In the shader you can then check which variant is currently running like this:
{% highlight hlsl %}
#if defined(ENABLE_ALPHA_CLIPPING)
    if (shaderOutput.alpha < shaderOutput.alphaClipThreshold) {
        discard;
    }
#endif
{% endhighlight %}

Similarly, #pragma Option lets you define multiple states:
{% highlight hlsl %}
#pragma Option BLUR_DIRECTION HORIZONTAL VERTICAL

...

#if BLUR_DIRECTION == BLUR_DIRECTION_HORIZONTAL
    //Blur horizontal
#elif BLUR_DIRECTION == BLUR_DIRECTION_VERTICAL
    //Blur vertical
#else
#error Unknown Blur Direction
#endif
{% endhighlight %}

#pragma OptionAdd is just a convenience helper so you don't have to specify all posible values in one line. I added it, when I created the shader for my NosesisGUI implementation which has 48 different modes :D

Note that there is no pragma to add multiple ShaderPasses. When working with plain shader files, the compiler will always create a single pass called "main".
It would have complicated the file format quite a bit and probably would have required something more like [Unitys ShaderLab format](https://docs.unity3d.com/6000.2/Documentation/Manual/writing-shader-create-shader-pass.html).

The one part I sometimes miss is the possibility to have multiple compute shaders in a single file. I may add support for that at some point but who knows.<br>
For now ShaderPasses are reserved for the more complex shader types like Materials shaders which we will get to eventually.

## Compilation pipeline

TODO

TODO diagram?

TODO ShaderCompiler vs BackendCompiler

TODO editor vs build compilation, import vs actual compilation, code stripping

## Complex Shader Types?
TODO

TODO Spellcheck

TODO note problem of pass->variant->program hierarchy

TODO comment section