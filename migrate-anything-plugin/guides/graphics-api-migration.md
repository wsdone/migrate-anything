# Graphics API Migration Guide

Decision framework and pitfalls for migrating between GPU/graphics APIs. The agent already knows API-level mappings (Metal types, Vulkan types, etc.) — this guide focuses on strategic decisions and non-obvious traps.

## Decision Framework

```
1. Is the graphics code isolated behind an abstraction layer?
   → YES: Write a new backend for the target API (ideal case)
   → NO: Continue to 2

2. Can you use a cross-platform GPU abstraction?
   → YES: Use wgpu or bgfx (fastest path, works everywhere)
   → NO: Continue to 3

3. How deep is the platform integration?
   → Shallow (a few render calls): Direct API translation
   → Deep (custom pipeline, shaders, compute): Full rewrite of graphics layer
```

## Key Pitfalls

### Coordinate System Trap

This is the **#1 silent bug** in GPU migrations:

| API | Origin | Y direction |
|-----|--------|-------------|
| Metal | Top-left | Y-down |
| DirectX 9-12 | Top-left | Y-down |
| OpenGL | Bottom-left | Y-up |
| Vulkan | Top-left | Y-down (NDC), but textures are Y-up |

**Impact**: Viewport coordinates, scissor rectangles, texture UV sampling, render targets, and projection matrices are all affected.

**Solution**: Handle the flip in ONE place (projection matrix or viewport setup). Don't scatter Y-flip logic throughout the codebase.

### Shader Build Pipeline

Metal compiles shaders at pipeline creation time. Vulkan/GL require precompiled SPIR-V or runtime compilation. This is an architectural difference, not just a syntax difference.

**Plan early**: Decide whether to use:
- **Offline compilation**: Shader files → SPIR-V at build time (shaderc, glslangValidator)
- **Runtime compilation**: Ship GLSL/HLSL source, compile on load (slower startup, more flexible)
- **Tools**: SPIRV-Cross can translate between MSL, HLSL, and GLSL via SPIR-V

### Synchronization Model

Metal has mostly implicit synchronization. Vulkan/D3D12 require explicit fences, semaphores, and pipeline barriers. This is a fundamental architectural difference:

- **Metal**: "Submit work, it just works"
- **Vulkan/D3D12**: "You must explicitly synchronize EVERYTHING — GPU↔CPU, GPU↔GPU, buffer↔image transitions"

Getting synchronization wrong causes random GPU crashes that are hard to debug. Start with overly conservative barriers, then optimize.

### Memory Management

Metal and D3D11 manage GPU memory implicitly. Vulkan and D3D12 require explicit memory allocation:

- You choose memory types (host-visible, device-local, etc.)
- You manage memory heaps and suballocation
- You handle buffer-image transitions explicitly

**Use a memory allocator library**: Don't write your own GPU memory allocator. Use:
- **Vulkan Memory Allocator (VMA)** — industry standard for Vulkan
- **D3D12 Memory Allocator (D3D12MA)** — equivalent for D3D12

## Recommended Cross-Platform GPU Libraries

| Library | Language | Backends | Best For |
|---------|----------|----------|----------|
| **wgpu** | Rust, C, Python | Vulkan, Metal, DX12, OpenGL | New projects, compute+graphics |
| **bgfx** | C++ | Vulkan, Metal, DX9-12, OpenGL | Embedded rendering, 10+ backends |
| **SDL 3 GPU** | C | Vulkan, Metal, DX12 | Simple GPU access via SDL |
| **Filament** | C++ | Vulkan, Metal, OpenGL, DX12 | PBR rendering engine |
| **Diligent Engine** | C++ | Vulkan, Metal, DX11/12, OpenGL | Full-featured rendering |

For 2D graphics specifically:

| Library | Language | Best For |
|---------|----------|----------|
| **Cairo** | C | 2D vector graphics, PDF/SVG export |
| **Skia** | C++ | High-performance 2D (Chrome, Android) |
| **Blend2D** | C++ | Fast 2D with JIT compilation |

## The "Rewrite Boundary" Pattern

For Hard migrations (e.g., Ghostty's Metal renderer → Linux):

1. **Find the interface boundary** — the abstract API the rest of the code calls to render
2. **Define it explicitly** — create a `Renderer` interface/base class
3. **Implement for target platform** — write a `VulkanRenderer` from scratch
4. **Wire it in** — swap MetalRenderer for VulkanRenderer at startup

The key insight: you're NOT translating Metal calls to Vulkan calls one-by-one. You're building a NEW renderer that satisfies the same interface. Think of it as "reimplementing the contract, not translating the code."

### Example: Renderer Interface

```cpp
// The interface the application uses (portable)
class Renderer {
public:
    virtual ~Renderer() = default;
    virtual void begin_frame() = 0;
    virtual void draw_glyphs(const GlyphBatch& batch) = 0;
    virtual void present() = 0;
    virtual void resize(int width, int height) = 0;
};

// Factory function — each platform provides its own implementation
std::unique_ptr<Renderer> create_renderer(void* native_window);
```

The Metal implementation uses `MTLDevice`, `MTLCommandQueue`, etc. The Vulkan implementation uses `VkDevice`, `VkQueue`, etc. The application code never sees either — it only knows the `Renderer` interface.

## When to Use Vulkan vs OpenGL

For Linux migrations specifically:

- **Use Vulkan** when: The source uses Metal or DX12 (explicit API), performance matters, you need compute shaders
- **Use OpenGL** when: The source uses DX9/DX11 or older, simplicity matters more than peak performance, the project is small
- **Use wgpu** when: You want both Vulkan and OpenGL support from one codebase, and WebGPU compatibility

OpenGL is easier to learn but Vulkan gives you more control. For terminal emulators (like Ghostty), Vulkan or wgpu is the better choice because of the rendering model.
