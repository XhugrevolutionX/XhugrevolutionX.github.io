---
layout: post
title: "Building a Custom Graphics Engine: From Triangles to PBR"
date: 2026-02-05
categories: [Graphics Programming, C++, OpenGL]
tags: [Deferred Shading, PBR, IBL, Instancing, SSAO, Post-Processing]
---

Developing a graphics engine is a journey of layering complexity. What starts as a simple `glDrawArrays` call eventually evolves into a complex pipeline managing gigabytes of texture data and millions of polygons.

In this post, I’ll walk through the technical implementation of my engine, built with C++ and OpenGL ES 3.0, detailing how each system was constructed.

## 1. The Starting Point: Triangles and Squares
Every graphics engine begins with the "Hello World" of rendering: a triangle. This seemingly simple task establishes the core infrastructure for communicating with the GPU.

### Rendering a Triangle
In my `Triangle` sample, the process involves the raw basics:

![Local Image](/images/triangle.png)

* **Shader Compilation:** Loading, compiling, and linking `.vert` and `.frag` files into a program.
* **Buffer Management:** I define vertices containing positions and colors, upload them to a **VBO (Vertex Buffer Object)**, and configure the layout using a **VAO (Vertex Array Object)**.
* **The Draw Call:** The render loop binds the shader and VAO, then calls `glDrawArrays` to push the geometry to the screen.

### Scaling Up: Rendering a Square
A square isn't a native primitive in OpenGL; it's constructed from two triangles. To render this efficiently, I introduced the **EBO (Element Buffer Object)** in the `Square` class.

Instead of duplicating vertices for the shared edge, I store unique vertices in the VBO and a list of indices (e.g., `0, 1, 3, 1, 2, 3`) in the EBO. The draw call changes to `glDrawElements`, which tells the GPU to assemble triangles by looking up these indices, saving memory and processing power.

## 2. Model Loading & Geometry
Moving from hardcoded arrays to real assets requires a robust loading system. I integrated **Assimp** to handle standard formats like `.obj` and `.fbx`.

The `Model` class recursively processes the scene graph, converting Assimp's data into my internal `Mesh` structure. Each vertex is a rich data structure containing:
* **Position:** The 3D coordinate.
* **Normal:** For lighting calculations.
* **TexCoords:** For UV mapping.
* **Tangent/Bitangent:** Calculated for Normal Mapping.

## 3. The Lighting System & Visualizers
To illuminate these models, I implemented a standard **Blinn-Phong** lighting model, managed by a `LightManager`. The engine supports three distinct light types, processed in a single Uber-Shader:

* **Directional Light:** Simulates the sun with parallel rays.
* **Point Lights:** Radiate in all directions. I calculate **attenuation** using constant, linear, and quadratic terms to simulate realistic light falloff over distance.
* **Spotlights:** Defined by a direction and two cones (inner and outer cutoff) to create soft-edged beams.

### Visualizing Lights (Light Gizmos)
Debugging invisible lights is difficult, so I created a `LightGizmo` class to render physical representations of light sources in the scene.

It renders a simple cube at the light's position, but with a twist: in the fragment shader (`gizmo.frag`), I check the brightness of the object color. If it's bright enough (like a pure white light), I output it to a separate "Bloom" buffer. This makes the light gizmos appear to physically glow on screen, distinguishing them from standard geometry.

## 4. The Post-Processing Stack
The final frame is never the raw render. The `Framebuffer` class manages a post-processing chain that adds style and polish to the image.

### Convolution Kernels
I implemented a generalized kernel processor that samples 3x3 neighboring pixels to apply effects:
* **Sharpen:** Highlights edges by subtracting neighboring pixel values from the center.
* **Blur:** Averages neighboring pixels to soften the image.
* **Edge Detection:** Highlights areas of high contrast, useful for "toon" outlines or debugging geometry.

### Color Filters & Retro Effects
Simple math operations allow for rapid stylistic changes:
* **Inversion:** `1.0 - color`. Creates a negative film look.
* **Grayscale:** Converts the RGB image to black and white by dotting the color with human eye sensitivity weights (`vec3(0.21, 0.71, 0.07)`).
* **Dithering:** To achieve a retro, 1-bit aesthetic, I implemented **Ordered Dithering**. I defined a 4x4 Bayer Matrix in the shader and mapped screen coordinates (`gl_FragCoord`) to this grid. By comparing the pixel's brightness against the threshold in the matrix, the engine quantizes the image into a cross-hatch pattern.

### Bloom & HDR
Finally, the engine applies **HDR Tone Mapping** (converting high-precision floating point colors to LDR) and **Bloom**. The Bloom effect works by extracting bright regions of the image, blurring them with a Gaussian filter, and additively blending them back on top of the original scene.

## 5. Deferred Rendering Pipeline
Forward rendering struggles when you have many lights because every object calculates lighting for every light. I solved this by implementing a **Deferred Rendering** pipeline.

### The G-Buffer
Instead of calculating lighting immediately, the geometry pass renders surface properties into a **G-Buffer** (Geometry Buffer). My G-Buffer uses multiple render targets (MRT):
* `gPosition`: World space positions (`GL_RGBA16F`).
* `gNormal`: World space normals (`GL_RGBA16F`).
* `gAlbedoSpec`: Diffuse color and specular intensity (`GL_RGBA16F`).

In the **Lighting Pass**, a full-screen quad samples these textures. This decouples geometry complexity from lighting complexity, allowing for hundreds of lights with minimal performance cost.

### Debug Modes
A complex G-Buffer can be hard to debug. If lighting looks wrong, is it the normal map? The position data? To diagnose this, I implemented 5 distinct debug modes in the fragment shader that allow me to dump raw texture data directly to the screen:
* **Mode 1 (Position):** Visualizes the `gPosition` texture. Useful for checking if world coordinates are being written correctly.
* **Mode 2 (Normal):** Visualizes `gNormal`. Crucial for verifying that Normal Mapping is correctly perturbing surface normals.
* **Mode 3 (Albedo):** Shows the raw color texture without any lighting applied.
* **Mode 4 (Specular):** Visualizes the specular intensity map (often stored in the alpha channel of Albedo).
* **Mode 0 (Full Render):** The standard composed lighting pass.

## 6. Optimization: GPU Instancing
Rendering thousands of objects individually (like in my Cube scenes) is a CPU bottleneck. I implemented **GPU Instancing** to solve this.

* **Instance VBO:** I store model matrices for all instances in a separate buffer.
* **Vertex Attrib Divisor:** I tell OpenGL to update these matrix attributes only once per instance.
* **The Shader:** The vertex shader receives the matrix as `layout (location = 5) in mat4 aInstanceMatrix` and applies it to the vertex.

This allows drawing thousands of asteroids or debris chunks in a single API call.

## 7. Advanced Shadows & SSAO
Lighting without shadows feels flat. I implemented three specific techniques to add depth, applied during the lighting passes.

### Directional & Point Shadows
* **Directional Lights** use a standard shadow map approach, rendering the scene from the light's view into a 4096x4096 depth texture.
* **Point Lights** are trickier. Since they shine in all directions, I render the scene six times into a **Depth Cubemap**. The fragment shader then calculates the linear distance from the light to the fragment to determine occlusion.

### SSAO (Screen Space Ambient Occlusion)
To ground objects in the scene, I implemented SSAO. This technique estimates how exposed a point is to ambient light.
1.  **Kernel Generation:** I generate a hemisphere of 64 random samples oriented around the surface normal.
2.  **Noise:** A 4x4 rotation noise texture jitters the samples to trade banding for noise.
3.  **Occlusion Check:** I check if these samples fall behind the geometry in the depth buffer.
4.  **Blur:** A final pass blurs the noise to create smooth contact shadows.

## 8. The Crown Jewel: PBR & IBL
While the main engine uses Blinn-Phong, I built a specialized scene to implement **Physically Based Rendering (PBR)** using the Cook-Torrance BRDF.

To achieve photorealism, I implemented a runtime **Image Based Lighting (IBL)** generator. When the scene loads, it processes an HDR equirectangular environment map into three critical textures:
1.  **Irradiance Map:** Solves the diffuse integral by convolving the environment map.
2.  **Prefilter Map:** Stores specular reflections at different roughness levels in the cubemap mipmaps.
3.  **BRDF LUT:** A pre-computed 2D lookup table for the Fresnel response.

The result is a material system where metal and plastic look physically correct under any lighting condition, without tweaking magic numbers.

## Conclusion
Building this engine has been an exhaustive exercise in modern graphics programming. From the initial struggle of getting a triangle on screen to the satisfaction of seeing PBR materials reacting to an HDR environment, every step revealed the intricate dance between CPU data management and GPU parallelism.

There is still plenty to explore—cascaded shadow maps, temporal anti-aliasing, and compute shaders—but the foundation built here offers a robust platform for any future rendering experiments.