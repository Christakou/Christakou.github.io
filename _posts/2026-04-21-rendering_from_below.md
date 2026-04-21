
  ---
  title: "Godot's Rendering Pipelien From Below: A guided journey from Scene Tree to Frame Buffer"
  date: 2026-04-21 12:00:00 +0000
  categories: [Systems]
  tags: [tag1, tag2]
  ---

# Godot's Rendering Pipeline From Below: Scene Tree to Framebuffer
Most rendering pipeline explanations start from the top: "first you create a mesh, then the engine renders it." This one starts from the bottom -- from the actual source code that transforms your scene tree into pixels. I've been studying Godot's rendering internals as part of learning to contribute to the engine, and what follows are my annotated notes from reading the code that renders every frame.
This isn't a tutorial. It's a guided walk through the machinery.
---
## The Big Picture
Every frame, Godot executes roughly this sequence:
```
Scene tree cull → Shadow pass → Depth prepass → Opaque pass →
Sky → Transparent pass → Post-processing → Present
```
This isn't a design choice unique to Godot. It's what falls out of the physics constraints: shadows need depth before lighting, opaque objects must be drawn before transparent ones, post-processing needs the full scene rendered first. Every modern engine converges on roughly this order.
You can see it directly in the source. Godot's forward clustered renderer has `RENDER_TIMESTAMP` markers that trace the full pipeline:
```cpp
// render_forward_clustered.cpp — _render_scene()
// ~1,200 lines orchestrating every pass
RENDER_TIMESTAMP("Prepare 3D Scene");
RENDER_TIMESTAMP("Setup 3D Scene");
RENDER_TIMESTAMP("Render OmniLight Shadows");
RENDER_TIMESTAMP("Render Shadows");
RENDER_TIMESTAMP("Render GI");
RENDER_TIMESTAMP("Render SDFGI");
RENDER_TIMESTAMP("Render Depth Pre-Pass");
RENDER_TIMESTAMP("Process SSAO");
RENDER_TIMESTAMP("Process SSIL");
RENDER_TIMESTAMP("Render Opaque Pass");
RENDER_TIMESTAMP("Render Sky");
RENDER_TIMESTAMP("Render 3D Transparent Pass");
RENDER_TIMESTAMP("Process SSR");
RENDER_TIMESTAMP("FSR2");  // or TAA
RENDER_TIMESTAMP("Tonemap");
```
Each of these is a marker you can see in Godot's profiler. One function, one frame, every pass in order.
---
## Layer 1: Culling — What's Visible?
Before anything touches the GPU, the CPU needs to figure out what's visible. `RendererSceneCull::_render_scene()` builds frustum planes from the camera and tests every instance against them:
```cpp
// renderer_scene_cull.cpp
Vector<Plane> planes = p_camera_data->main_projection.get_projection_planes(
    p_camera_data->main_transform);
cull.frustum = Frustum(planes);
```
If there are many instances, this is parallelised across worker threads:
```cpp
if (cull_to > thread_cull_threshold) {
    WorkerThreadPool::get_singleton()->add_template_group_task(
        this, &RendererSceneCull::_scene_cull_threaded, &cull_data, ...);
    // Merge thread results
    for (InstanceCullResult &thread : scene_cull_result_threads) {
        scene_cull_result.append_from(thread);
    }
}
```
The output is a set of sorted lists: visible geometry, lights, reflection probes, decals, fog volumes. These get handed off to the renderer in one call:
```cpp
scene_render->render_scene(
    p_render_buffers, p_camera_data,
    scene_cull_result.geometry_instances,
    scene_cull_result.light_instances,
    scene_cull_result.reflections,
    scene_cull_result.decals,
    scene_cull_result.fog_volumes,
    ...);
```
This is the boundary between "what should we draw?" and "how do we draw it?"
---
## Layer 2: Instance Data — Getting Object Info to the GPU
Each visible object needs its data on the GPU: transform, flags, layer mask, lightmap info. Godot packs this into an `InstanceData` struct:
```cpp
// render_forward_clustered.h
struct InstanceData {
    float transform[12];
    float compressed_aabb_position[4];
    float compressed_aabb_size[4];
    float uv_scale[4];
    uint32_t flags;
    uint32_t instance_uniforms_ofs;
    uint32_t gi_offset;
    uint32_t layer_mask;
    float prev_transform[12];
    float lightmap_uv_scale[4];
};
```
A CPU-side loop fills one of these per visible object and writes them directly into a mapped GPU buffer:
```cpp
for (uint32_t i = 0; i < element_total; i++) {
    GeometryInstanceForwardClustered *inst = surface->owner;
    SceneState::InstanceData instance_data;
    store_transform_transposed_3x4(inst->transform, instance_data.transform);
    instance_data.flags = inst->flags_cache;
    instance_data.layer_mask = inst->layer_mask;
    instance_data.gi_offset = inst->gi_offset_cache;
    // Write directly into mapped GPU memory
    scene_state.curr_gpu_ptr[p_render_list][i] = instance_data;
}
```
On the shader side, a `flat uint instance_index_interp` varying carries the index through the rasteriser (flat because interpolating an integer ID is meaningless). The fragment shader uses it to look up everything about the current object:
```glsl
instances.data[instance_index].flags       // feature flags
instances.data[instance_index].layer_mask  // light filtering
instances.data[instance_index].transform   // model matrix
```
---
## Layer 3: The Vertex Shader — MVP Transform
For every vertex, the vertex shader transforms it from local object space to clip space. Godot splits this into two multiplies rather than one MVP:
```glsl
// scene_forward_clustered.glsl
// Step 1: model * view (combined)
mat4 modelview = read_view_matrix * model_matrix;
vertex = (modelview * vec4(vertex, 1.0)).xyz;
// Step 2: projection
gl_Position = projection_matrix * vec4(vertex_interp, 1.0);
```
Why not combine all three? Because `vertex_interp` -- the view-space position -- is needed by the fragment shader for lighting calculations. Distance from camera, specular highlights, fog -- all computed in view space. Collapsing everything into one multiply would lose that intermediate value.
Between these two steps sits `#CODE : VERTEX` -- the literal injection point where your Godot shader's `vertex()` function gets pasted:
```glsl
{
#CODE : VERTEX   // your code runs here
}
```
If you write an empty `vertex()` or don't write one at all, the MVP transform still happens. You only write custom vertex code to modify or add to the default behaviour.
---
## Layer 4: Rasterisation and Varyings
After vertex processing, the GPU groups vertices into triangles and rasterises them -- converting each triangle into fragments (pixel candidates). For each fragment, the rasteriser computes barycentric coordinates and uses them to interpolate all the values passed from the vertex shader.
Godot's shader declares these as varyings with `#ifdef` guards:
```glsl
// Vertex shader outputs
layout(location = 0) out vec3 vertex_interp;
#ifdef NORMAL_USED
layout(location = 1) out vec3 normal_interp;
#endif
#ifdef UV_USED
layout(location = 3) out vec2 uv_interp;
#endif
#ifdef TANGENT_USED
layout(location = 5) out vec3 tangent_interp;
layout(location = 6) out vec3 binormal_interp;
#endif
layout(location = 10) out flat uint instance_index_interp;
```
Those `#ifdef` guards are important. If your shader doesn't reference `NORMAL_MAP`, the `TANGENT_USED` define isn't set, and the tangent/binormal varyings are excluded entirely. This matters for mobile GPUs with tile-based architectures where varying count directly impacts whether data stays in fast on-chip memory or spills to slow main memory.
---
## Layer 5: The Fragment Shader — Materials and Lighting
The fragment shader is where most of the work happens. It runs per pixel and determines the final colour. Godot's approach splits this into two parts: your code sets material properties, then ~1,500 lines of generated code handles lighting.
Before your code runs, defaults are set:
```glsl
vec3 albedo = vec3(1.0);
float metallic = 0.0;
float roughness = 1.0;
vec3 emission = vec3(0.0);
float alpha = 1.0;
```
Then your fragment function is injected:
```glsl
{
#CODE : FRAGMENT    // ALBEDO = texture(tex, UV).rgb; etc.
}
```
After your code, the engine's lighting loop runs. For the forward clustered renderer, this means:
1. Ambient and indirect light (lightmaps, SDFGI, voxel GI, sky)
2. For each directional light: diffuse + specular + shadow
3. For each omni/spot light in this fragment's cluster: same
4. All using Cook-Torrance BRDF with your roughness/metallic values
The final output sums everything:
```glsl
frag_color = vec4(
    emission
    + ambient_light
    + diffuse_light
    + direct_specular_light
    + indirect_specular_light,
    alpha);
```
This is an HDR value -- it can exceed 1.0. The framebuffer stores it as 16-bit float. Tonemapping compresses it to displayable range in a later pass.
---
## Layer 6: Output Merging — Depth and Blending
Each fragment must pass the depth test before it's written to the framebuffer. Godot uses reversed-Z: near plane = 1.0, far plane = 0.0. This puts the best floating-point precision at far distances where z-fighting is worst. The compare operator flips accordingly:
```cpp
depth_stencil_state.depth_compare_operator = RD::COMPARE_OP_GREATER_OR_EQUAL;
```
With the depth prepass enabled, Godot switches the opaque pass to `COMPARE_OP_EQUAL`:
```cpp
if (depth_pre_pass_enabled) {
    // Depth already written. Only shade fragments at the exact closest depth.
    // Everything else is rejected BEFORE the fragment shader runs.
    depth_stencil_state.depth_compare_operator = RD::COMPARE_OP_EQUAL;
    depth_stencil_state.enable_depth_write = false;
}
```
This eliminates overdraw entirely -- no wasted fragment shader invocations on pixels that would be overwritten.
For transparent objects, the rules change: blending is enabled, depth write is disabled (so transparent objects don't block each other), and objects are drawn back-to-front:
```cpp
if (p_pipeline_key.color_pass_flags & PIPELINE_COLOR_PASS_FLAG_TRANSPARENT) {
    blend_state = blend_state_color_blend;      // blending ON
    depth_stencil_state.enable_depth_write = false;
}
```
The blend equation for standard transparency:
```cpp
// result = new * alpha + old * (1 - alpha)
attachment.src_color_blend_factor = RD::BLEND_FACTOR_SRC_ALPHA;
attachment.dst_color_blend_factor = RD::BLEND_FACTOR_ONE_MINUS_SRC_ALPHA;
```
---
## The Shader Compiler — A Pipeline Within the Pipeline
One of the most interesting things I found was how your `.gdshader` file becomes GPU-executable code. It's not a single compilation step -- it's a five-stage pipeline of its own:
**Stage 1: Parse.** `ShaderLanguage::compile()` tokenises your Godot shader and builds an AST. The AST is a tree of typed nodes -- `OperatorNode`, `VariableNode`, `ConstantNode`, `ControlFlowNode` -- with a separate flat linked list through `Node::next` for memory cleanup.
**Stage 2: Compile to GLSL.** `ShaderCompiler::compile()` walks the AST and emits GLSL source strings. During this walk, Godot variable names get renamed (`ALBEDO` → `albedo`, `UV` → `uv_interp`), and `render_mode` values get extracted via a clever pointer-write mechanism:
```cpp
// Each render_mode string maps to a pointer + value
// "cull_disabled" → write CULL_MODE_DISABLED into ShaderData::cull_mode
if (p_actions.render_mode_values.has(pnode->render_modes[i])) {
    Pair<int *, int> &p = p_actions.render_mode_values[pnode->render_modes[i]];
    *p.first = p.second;
}
```
The render_mode values never become GLSL. They bypass shader compilation entirely and flow into pipeline configuration.
**Stage 3: Template injection.** Godot has template `.glsl` files with placeholders like `#CODE : VERTEX`. Your compiled GLSL snippets get pasted in at these points.
**Stage 4: GLSL → SPIR-V.** The complete GLSL source is compiled to SPIR-V binary via glslang.
**Stage 5: SPIR-V → native.** Vulkan consumes SPIR-V directly. Metal cross-compiles to MSL via SPIRV-Cross. D3D12 cross-compiles to DXIL via Mesa/NIR.
The `render_mode` path is the surprising part. One string in your shader becomes a hardware configuration without ever touching the shader compiler's GLSL output:
```
"cull_disabled" (shader text)
  → parser (string in AST)
  → shader compiler (writes int via pointer into ShaderData::cull_mode)
  → pipeline key (raster_state.cull_mode)
  → RD::render_pipeline_create()
  → Vulkan/Metal/D3D12 rasteriser config
```
---
## The Multi-Backend Architecture
The rendering code we've been reading lives entirely above the backend split. Everything from `_render_scene()` down to `RD::draw_list_draw()` is backend-agnostic. Below that sits `RenderingDeviceDriver` -- an abstract base class with three implementations:
```
RenderingDevice (backend-agnostic API)
       │
  RenderingDeviceDriver (abstract)
       │
  ┌────┼────────────┐
  ▼    ▼            ▼
Vulkan  Metal      D3D12
```
For shaders, SPIR-V is the shared intermediate representation. For everything else -- buffer creation, texture allocation, command recording, synchronisation -- `RenderingDeviceDriver` defines the abstract interface, and each backend maps it to native API calls.
The engine could theoretically support multiple backends without SPIR-V by emitting different shader languages from the AST. SPIR-V makes the shader side easier, but the `RenderingDeviceDriver` abstraction is what makes the API side possible.
---
## What I Took Away
The rendering pipeline is less mysterious than it looks from the outside. The stages exist because of physics constraints, not arbitrary design. Shadows need depth. Opaque before transparent. Post-processing after everything. Every engine arrives at roughly the same structure.
The parts that are genuinely Godot-specific -- the shader compiler's dual-path design (code goes to GLSL, config goes to pipeline state), the `#ifdef` guard system that compiles out unused features, the `RenderingDeviceDriver` abstraction -- are well-designed solutions to real engineering problems. They're also the parts most worth understanding if you want to contribute.
The cheapest code is code that doesn't compile. The cheapest fragment is one that gets depth-rejected before the shader runs. The cheapest draw call is one that was culled on the CPU. Rendering optimisation is mostly about not doing work, and the pipeline is structured around exactly that principle.
---
*These are my study notes from working through Godot's rendering source code. All code references are from the Godot `master` branch. If you want to explore the source yourself, start with `render_forward_clustered.cpp` — the `_render_scene()` function and its `RENDER_TIMESTAMP` markers are the best map of the whole pipeline.*