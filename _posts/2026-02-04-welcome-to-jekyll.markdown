---
layout: post
title: "Architecting a High-Performance GLES 3.0 Renderer: A Deep Dive"
date: 2026-02-05
categories: [Graphics Programming, C++, OpenGL]
tags: [Deferred Shading, PBR, IBL, Instancing, SSAO, Post-Processing]
---

Developing a custom graphics engine is an exercise in managing the intricate dance between CPU data orchestration and GPU parallelism. What began as a foundational exploration of the graphics pipeline has evolved into a feature-complete OpenGL ES 3.0 renderer capable of deferred shading, physically-based effects, and high-density geometry instancing.

In this post, I break down the core architecture and the technical hurdles overcome during the development of this engine.

## 🛠 Engine Overview & Features
* **Core API:** OpenGL ES 3.0 / C++.
* **Rendering Path:** Multi-pass Deferred Shading.
* **Lighting Model:** Blinn-Phong & PBR (Cook-Torrance BRDF).
* **Optimization:** Hardware Instancing and Texture Batching.

---

## 🏗 Asset Architecture: The Model-Mesh Hierarchy

A major challenge in engine design is translating complex 3D file formats into a format the GPU can consume efficiently. I utilized a hierarchical structure where a `Model` serves as a manager for multiple `Mesh` units.

### Recursive Scene Parsing
Using **Assimp**, the engine recursively traverses the node tree of a 3D file. This is critical for maintaining parent-child transformations—ensuring that if a character's arm moves, the hand follows. During this pass, I also compute the **Axis-Aligned Bounding Box (AABB)** by tracking the `minBounds` and `maxBounds` of every vertex to facilitate future frustum culling.



### The Mesh Data Payload
To minimize draw call overhead, each `Mesh` encapsulates its own Vertex Array Object (VAO). I defined a packed `Vertex` structure to ensure high cache locality during vertex fetching.

```cpp
struct Vertex {
    glm::vec3 Position;
    glm::vec3 Normal;
    glm::vec2 TexCoords;
};
```

```cpp
class Mesh {
public:
    std::vector<Vertex>       vertices;
    std::vector<unsigned int> indices;
    std::vector<Texture>      textures;
    // ... Methods
};
```

The `setupMesh()` function is where the CPU-to-GPU handoff occurs. By using `glVertexAttribPointer` with the `offsetof` macro, the engine tells the GPU exactly how to stride through the memory buffer to find positions, normals, and UVs.



```cpp
// Vertex Positions
glEnableVertexAttribArray(0);
glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, sizeof(Vertex), nullptr);

// Vertex Normals
glEnableVertexAttribArray(1);
glVertexAttribPointer(1, 3, GL_FLOAT, GL_FALSE, sizeof(Vertex), reinterpret_cast<void *>(offsetof(Vertex, Normal)));

// Vertex Texture Coords
glEnableVertexAttribArray(2);
glVertexAttribPointer(2, 2, GL_FLOAT, GL_FALSE, sizeof(Vertex), reinterpret_cast<void *>(offsetof(Vertex, TexCoords)));
```

---

## 🎨 The Rendering Pipeline: Deferred Shading

To support hundreds of dynamic lights without a massive performance hit, I implemented a **Deferred Shading** path. This splits rendering into two distinct phases:

### Phase 1: The Geometry Pass (G-Buffer)
Instead of calculating lighting immediately, the engine renders geometric properties into a series of high-precision textures known as the **G-Buffer**.

* **gPosition**: World-space coordinates using `GL_RGBA16F` to prevent precision-loss artifacts at a distance.
* **gNormal**: Unit vectors representing the surface orientation, used for calculating light reflection.
* **gAlbedoSpec**: A composite texture where RGB channels store the base color and the Alpha channel stores specular intensity.

#### G-Buffer Visualizations
Below is the raw data captured by the G-Buffer before the lighting pass is applied:

<table style="width: 100%; border-collapse: collapse;">
  <tr>
    <td style="width: 50%; padding: 5px; text-align: center;">
      <img src="/images/position.png" alt="Position" style="width: 100%; border-radius: 4px;">
      <br><em>Position (World Space)</em>
    </td>
    <td style="width: 50%; padding: 5px; text-align: center;">
      <img src="/images/normal.png" alt="Normal" style="width: 100%; border-radius: 4px;">
      <br><em>Normal (Surface Vectors)</em>
    </td>
  </tr>
  <tr>
    <td style="width: 50%; padding: 5px; text-align: center;">
      <img src="/images/albedo.png" alt="Albedo" style="width: 100%; border-radius: 4px;">
      <br><em>Albedo (Base Color)</em>
    </td>
    <td style="width: 50%; padding: 5px; text-align: center;">
      <img src="/images/specular.png" alt="Specular" style="width: 100%; border-radius: 4px;">
      <br><em>Specular (Material Shine)</em>
    </td>
  </tr>
</table>

### Phase 2: The Lighting Pass
A full-screen quad is rendered using the G-Buffer textures as inputs. Lighting is calculated only for the pixels that are actually visible on screen, effectively decoupling lighting complexity from geometric complexity.

---

## 💡 Lighting & Shadow Implementation

### The Blinn-Phong Model
While standard Phong lighting calculates specular highlights based on a reflection vector, I opted for the **Blinn-Phong** model. It is more computationally efficient and avoids the specular "cutoff" artifacts that occur when the angle between the view and reflection vectors exceeds 90 degrees.

The core of this model is the **Halfway Vector (H)**. This represents the direction exactly halfway between the light source and the viewer. 

**Calculation Logic:**

1.  **The Halfway Vector**: We sum the **Light Direction (L)** and the **View Direction (V)**, then normalize the resulting vector.
    * **H = Normalize(L + V)**
2.  **Specular Intensity**: We calculate the dot product between the surface **Normal (N)** and the **Halfway Vector (H)**, then raise it to the power of the material's **Shininess (s)**.
    * **Specular = pow(max(dot(N, H), 0.0), s)**



This approach is faster because it avoids the expensive calculation of a reflection vector per fragment and provides a more consistent highlight even when the camera moves at sharp angles.

### Dynamic Shadow Mapping
To ground objects in the scene, the engine utilizes two distinct shadow techniques depending on the light source:

**1. Directional Shadows (The Sun)**
These use an **Orthographic projection** to render a 2D depth map from the sun's perspective. Because the sun is infinitely far away, we treat all light rays as parallel. We "record" the distance of every object from the sun into a texture; if a fragment is further away than the value in the texture, it is in shadow.

**2. Omnidirectional Point Shadows (Lamps/Torches)**
Point lights radiate in all directions, so a single 2D map is insufficient. I implemented **Depth Cubemaps**. The scene is rendered six times per point light—once for each face of a cube (Up, Down, Left, Right, Forward, Backward). 

In the lighting shader, we calculate the vector from the light to the fragment to "sample" the correct face of the cubemap:

```cpp
// Calculating distance for Omni-directional shadows
float closestDepth = texture(shadowMap, fragToLight).r;
closestDepth *= far_plane; // Convert from [0,1] back to world distance
float currentDepth = length(fragToLight);

// Apply a small bias to prevent "shadow acne"
float shadow = currentDepth - bias > closestDepth ? 1.0 : 0.0;
```

---

## 🚀 Performance Optimizations: GPU Instancing

Rendering thousands of individual objects creates a massive bottleneck due to repeated draw calls. I solved this by implementing **Hardware Instancing**.

```cpp
 //Link the Instance VBO to this Mesh's VAO
void SetupInstancingAttributes(const GLuint instanceVBO) const {
    glBindVertexArray(VAO);
    glBindBuffer(GL_ARRAY_BUFFER, instanceVBO);

    for (int i = 0; i < 4; i++) {
        constexpr std::size_t vec4Size = sizeof(glm::vec4);
        glEnableVertexAttribArray(5 + i);
        glVertexAttribPointer(5 + i, 4, GL_FLOAT, GL_FALSE, 4 * vec4Size, reinterpret_cast<void *>(i * vec4Size));
        glVertexAttribDivisor(5 + i, 1);
    }

    glBindVertexArray(0);
    glBindBuffer(GL_ARRAY_BUFFER, 0);
}
```

By storing model matrices in a separate VBO and using `glVertexAttribDivisor`, the GPU can render thousands of instances in a single call, updating the transformation matrix only once per instance.

```cpp
glBindVertexArray(VAO);
//Instanced Draw Call
glDrawElementsInstanced(GL_TRIANGLES, indices.size(), GL_UNSIGNED_INT, nullptr, instanceCount);
```

---

## 🖼 Post-Processing Stack

The final image is passed through a custom framebuffer stack. I implemented several filters to enhance the visual fidelity or create specific styles:

* **Bloom**: Extracting bright areas via a threshold and applying a separable Gaussian blur to create light "bleeding".
* **HDR & Tone Mapping**: Rendering to a floating-point framebuffer (`GL_RGBA16F`) and using Reinhard tone mapping to preserve details in high-intensity areas.
* **Ordered Dithering**: A retro 1-bit aesthetic achieved by comparing pixel brightness against a 4x4 Bayer Matrix.

---

## 🧪 The PBR Testbed & Image-Based Lighting

While the main engine uses a Blinn-Phong model for performance, I implemented a separate "PBR Testbed" scene to explore **Physically Based Rendering**.

This implementation relies on the **Cook-Torrance BRDF**, focusing on microfacet theory and energy conservation. To ground the objects, I added **Image-Based Lighting (IBL)**, which pre-computes environment maps to provide realistic diffuse and specular reflections.

* **Irradiance Map**: Pre-convoluted map for ambient diffuse lighting.
* **Prefilter Map**: Captures environment reflections at varying roughness levels.



Currently, this system exists in a standalone scene. Merging it with the main Deferred pipeline remains a technical challenge, primarily due to the increased complexity of the G-Buffer (requiring additional channels for Metallic, Roughness, and Ambient Occlusion) and the overhead of real-time IBL sampling.

---

## 🔍 Technical Challenges & Lessons
* **Memory Management**: Ensuring textures are shared via a `textures_loaded` vector to prevent redundant VRAM usage.
* **Precision Issues**: Moving to 16-bit floating-point buffers for the G-Buffer to avoid "banding" in lighting gradients.
* **Coordinate Systems**: Handling the UV-flip and winding order differences between Assimp's default and OpenGL's expectations.

## Conclusion
Building this engine provided a deep understanding of how modern renderers handle data at scale. Moving from a single triangle to a deferred system was a challenge in both math and architecture, resulting in a flexible platform for further graphics experimentation.