---
layout: post
title: "Building a Custom Graphics Engine: From Triangles to PBR"
date: 2026-02-05
categories: [Graphics Programming, C++, OpenGL]
tags: [Deferred Shading, PBR, IBL, Instancing, SSAO, Post-Processing]
---

Developing a graphics engine is a journey of layering complexity. What starts as a simple `glDrawArrays` call eventually evolves into a complex pipeline managing gigabytes of texture data and millions of polygons.

In this post, I’ll walk through the technical implementation of my engine, built with C++ and OpenGL ES 3.0, detailing how each system was constructed.

## 1. The Basics: Primitives & Geometry
Every graphics engine begins with the "Hello World" of rendering: a triangle. This seemingly simple task establishes the core infrastructure for communicating with the GPU.

### Rendering a Triangle
In my `Triangle` sample, the process involves the raw basics:

![Triangle Image](/images/triangle.png)

* **Shader Compilation:** Loading, compiling, and linking `.vert` and `.frag` files into a program.
* **Buffer Management:** I define vertices containing positions and colors, upload them to a **VBO (Vertex Buffer Object)**, and configure the layout using a **VAO (Vertex Array Object)**.
* **The Draw Call:** The render loop binds the shader and VAO, then calls `glDrawArrays` to push the geometry to the screen.

### Scaling Up: Squares & Indices
A square isn't a native primitive in OpenGL; it's constructed from two triangles. To render this efficiently, I introduced the **EBO (Element Buffer Object)** in the `Square` class.

Instead of duplicating vertices for the shared edge, I store unique vertices in the VBO and a list of indices (e.g., `0, 1, 3, 1, 2, 3`) in the EBO. The draw call changes to `glDrawElements`, which tells the GPU to assemble triangles by looking up these indices, saving memory.

## 2. Handling Assets: Textures & Models
Before loading complex 3D meshes, the engine needs to handle the most fundamental asset: the texture.

### Texture Loading
To bring images into OpenGL, I integrated the **stb_image** library. My `TextureFromFile` helper function handles the raw byte management:

```cpp
inline unsigned int TextureFromFile(const char *path, const std::string &directory) {
    std::string filename = std::string(path);

    filename = directory + '/' + filename;



    unsigned int textureID;

    glGenTextures(1, &textureID);



    int w, h, c;

    if (unsigned char *data = stbi_load(filename.c_str(), &w, &h, &c, 0)) {

        GLenum format = (c == 3) ? GL_RGB : GL_RGBA;

        glBindTexture(GL_TEXTURE_2D, textureID);
        glTexImage2D(GL_TEXTURE_2D, 0, format, w, h, 0, format, GL_UNSIGNED_BYTE, data);
        glGenerateMipmap(GL_TEXTURE_2D);

        glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_REPEAT);
        glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_REPEAT);
        glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR_MIPMAP_LINEAR);
        glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);

        stbi_image_free(data);
    } else {

        std::cout << "Texture failed to load at path: " << filename << std::endl;

        stbi_image_free(data);
    }
    return textureID;
}
```

1.  **Loading:** `stbi_load` reads the image file (JPG/PNG) into an array of unsigned chars.
2.  **Generation:** I generate an OpenGL texture ID with `glGenTextures`.
3.  **Parameters:** Crucial for quality. I set `GL_REPEAT` for wrapping and `GL_LINEAR_MIPMAP_LINEAR` for minification filtering to prevent aliasing artifacts at a distance.
4.  **Upload:** `glTexImage2D` pushes the pixel data to the GPU, followed by `glGenerateMipmap` to create the mip chain.

### Model Loading & Mesh Architecture
To support complex 3D assets, I built a robust loading system using **Assimp**. In this engine, a `Model` is a high-level container that manages a hierarchy of `Mesh` objects.

#### The Recursive Scene Graph
My `Model::Load` function initiates an `Assimp::Importer`, which parses the file into a scene graph. Because 3D models are often composed of many separate parts, I implemented `processNode` to traverse the `aiNode` tree recursively. This ensures that parent-child transformations and multiple sub-meshes are handled correctly. During this process, I also calculate the model's bounding box and radius by tracking the minimum and maximum vertex positions across all meshes.

#### Anatomy of a Mesh
Each `Mesh` object represents a single drawable entity. I defined a `Vertex` structure to pack all necessary data into a single buffer:

```cpp
    struct Vertex {
        glm::vec3 Position;
        glm::vec3 Normal;
        glm::vec2 TexCoords;
    };
```

The `Mesh::setupMesh()` function handles the heavy lifting of GPU memory allocation. It generates a **VAO**, **VBO**, and **EBO**, then uses `glVertexAttribPointer` with the `offsetof` macro to tell OpenGL exactly how to read the `Vertex` struct:
* **Location 0:** Positions ($x, y, z$).
* **Location 1:** Normals.
* **Location 2:** Texture Coordinates ($u, v$).

#### The Render Pipeline & Material Mapping
When drawing, the `Mesh::Draw` function iterates through its assigned textures and binds them to the GPU using `glActiveTexture`. To make the shaders flexible, I map internal texture types (like `texture_diffuse`) to specific sampler names (like `material_diffuse`) required by the shader pipeline. This abstraction allows the engine to handle different material properties without hardcoding texture units.

```glsl
uniform sampler2D material_diffuse;
uniform sampler2D material_specular;
uniform sampler2D material_normal;
```

## 3. Lighting, Shadows & Depth
Once geometry is on screen, the next step is making it look 3D.

### The Lighting Model
I implemented a standard **Blinn-Phong** lighting model, managed by a `LightManager`. The engine supports three distinct light types:
1.  **Directional Light:** Simulates the sun with parallel rays.
2.  **Point Lights:** Radiate in all directions with **attenuation** (calculated using constant, linear, and quadratic terms).
3.  **Spotlights:** Defined by a direction and two cones (inner and outer cutoff) for soft edges.

### Visualizing Lights (Gizmos)
Debugging invisible lights is difficult, so I created a `LightGizmo` class. It renders a cube at the light's position, but with a visual trick: inside `gizmo.frag`, I check the brightness of the color. If it's a pure white light, I output it to a separate buffer. This allows the light source to physically "glow" when Bloom is applied later.

### Advanced Shadows
To prevent the scene from looking flat, I implemented two shadow techniques:
* **Directional Shadows:** Uses an orthographic projection to render a depth map from the "Sun's" perspective.
* **Omni-directional Point Shadows:** Since point lights shine everywhere, I render the scene *six times* into a **Depth Cubemap**. The fragment shader calculates the linear distance from the light to determine occlusion.

### SSAO (Screen Space Ambient Occlusion)
To ground objects in the scene, I implemented SSAO. This technique estimates how exposed a point is to ambient light.
1.  **Kernel:** I generate a hemisphere of 64 random samples.
2.  **Noise:** A 4x4 rotation noise texture jitters the samples to reduce banding.
3.  **Blur:** A final pass blurs the result to create smooth contact shadows in crevices.

## 4. The Main Scene: Architecture & Pipeline
The main scene brings everything together using a high-performance **Deferred Rendering** pipeline.

### Deferred Rendering & The G-Buffer
Forward rendering struggles with many lights (M*N complexity). I solved this using a **Deferred** approach.
1.  **Geometry Pass:** I render the scene into a **G-Buffer** with multiple render targets (MRT) using `GL_RGBA16F` precision:
    * `gPosition`: World space positions.
    * `gNormal`: World space normals.
    * `gAlbedoSpec`: Color and specular intensity.
2.  **Lighting Pass:** A full-screen quad samples these textures to calculate lighting for hundreds of lights cheaply.



**Debug Modes:**
To diagnose the G-Buffer, I added debug modes in `deferred.frag` to visualize the raw textures:
* **Mode 1/2:** Visualize Position/Normal textures.
* **Mode 3/4:** Visualize Albedo/Specular data.

### Optimization: GPU Instancing
Rendering thousands of cubes individually is a CPU bottleneck. I implemented **GPU Instancing**.
* I store model matrices in a separate VBO.
* I use `glVertexAttribDivisor` to update the matrix only once per instance.
* The vertex shader applies the instance matrix to the positions.
This allows drawing thousands of meshes in a single API call.

## 5. The Post-Processing Stack
The `Framebuffer` class manages the final polish of the image.

* **Convolution Kernels:** Support for Sharpen, Blur, and Edge Detection using 3x3 kernel matrices.
* **Bloom:** Bright areas are extracted, blurred via Gaussian blur, and added back to the scene.
* **HDR Tone Mapping:** Exposure-based mapping from HDR to LDR.
* **Retro Dithering:** To achieve a 1-bit retro look, I implemented **Ordered Dithering**. I defined a 4x4 Bayer Matrix in the shader and mapped screen coordinates to this grid. By comparing the pixel's brightness against the threshold in the matrix, the engine quantizes the image into a cross-hatch pattern.

## 6. The PBR Extension
While the main scene relies on Deferred Blinn-Phong, I built a separate "Testbed" scene to implement **Physically Based Rendering (PBR)**.

This scene uses the Cook-Torrance BRDF and a runtime **Image Based Lighting (IBL)** generator. On startup, the engine processes an HDR equirectangular map into:
1.  **Irradiance Map:** For diffuse lighting.
2.  **Prefilter Map:** For specular reflections at varying roughness.
3.  **BRDF LUT:** A pre-computed lookup table.

## Conclusion
Building this engine has been an exhaustive exercise in modern graphics programming. From the initial struggle of getting a triangle on screen to the satisfaction of seeing PBR materials reacting to an HDR environment, every step revealed the intricate dance between CPU data management and GPU parallelism.