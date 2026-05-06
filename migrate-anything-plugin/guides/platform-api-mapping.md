# Platform API Migration Decision Guide

This guide helps agents **decide how to migrate** platform-specific APIs. It does NOT provide exhaustive API mappings — the agent already knows those. Instead, it provides decision frameworks, non-obvious pitfalls, and cross-platform library recommendations.

## Decision Framework

When you encounter a platform-specific API call, use this decision tree:

```
1. Is there a direct 1:1 equivalent on the target platform?
   → YES: Direct replacement (e.g., Sleep() → usleep())
   → NO: Continue to 2

2. Does a cross-platform library provide this functionality?
   → YES: Replace with cross-platform library (e.g., WinHTTP → libcurl)
   → NO: Continue to 3

3. Can the functionality be isolated behind an abstraction layer?
   → YES: Create platform/ directory with per-OS implementations
   → NO: Continue to 4

4. Is this a core feature that defines the application?
   → YES: Rewrite for target platform (e.g., Metal renderer → Vulkan)
   → NO: Stub it out and document as "not available on target"
```

## Non-Obvious Pitfalls by Category

### Filesystem

**Case sensitivity is the #1 silent killer.** Code that works on macOS/Windows will break on Linux because of wrong-case `#include` directives, file paths, or resource references. After any migration to Linux, scan every `#include` and `fopen` for case mismatches.

**XDG directories** — Linux conventions for config/data/cache dirs are:
- Config: `~/.config/appname/`
- Data: `~/.local/share/appname/`
- Cache: `~/.cache/appname/`
- Use `XDG_*` environment variables when set, fallback to these paths

**Symlinks on Windows** — Require developer mode or admin privileges. Use junctions or hard links as alternatives.

### Threading

**`std::thread` is the universal answer.** Don't waste time mapping pthreads → Win32 threads. Use C++11 `std::thread`, `std::mutex`, `std::condition_variable` — they work everywhere.

The one exception: if you need platform-specific features (affinity, realtime scheduling, fiber), wrap those in a `platform/` abstraction.

### Graphics/GPU

**Coordinate system trap**: Metal/DirectX use Y-down origin (top-left). OpenGL/Vulkan use Y-up origin (bottom-left). This affects viewport, scissor, texture sampling, and projection matrices. Handle it in your projection matrix or viewport setup — don't try to flip at every call site.

**Shader compilation pipeline**: Metal compiles at pipeline creation time. Vulkan requires SPIR-V. Plan your shader build pipeline early — use shaderc or glslangValidator for offline compilation, not runtime.

**The "use wgpu" shortcut**: For many projects, the fastest path is replacing Metal/DirectX entirely with wgpu (WebGPU). It provides a modern GPU API that works across Vulkan, Metal, DX12, and OpenGL. Not suitable for every project, but worth evaluating before committing to a raw Vulkan rewrite.

### UI Frameworks

**Paradigm shift matters more than widget mapping.** Don't try to map NSButton → GtkButton one-by-one. Instead:
- AppKit uses **MVC** + **delegate pattern**
- GTK uses **composition** + **signals**
- Qt uses **signals/slots** + **parent-child ownership**
- Win32 uses **message loop** + **window procedure**

Choose the target framework's paradigm. Don't fight it.

**The rewrite boundary**: For Hard migrations, the key is finding where the UI code interfaces with the application logic. Everything above that boundary is portable; everything below gets rewritten. Define a clean interface at that boundary before starting.

### Networking

**Winsock requires WSAStartup/WSACleanup** — unique to Windows. After calling these, the rest of the socket API is nearly identical to POSIX.

**I/O multiplexing is platform-specific**:
- Linux: `epoll` (best)
- macOS: `kqueue` (best)
- Windows: IOCP (best) or `WSAPoll`
- Cross-platform: **libuv** or **libev** (handles all of the above)

### Process Management

**There is no `fork()` on Windows.** This is a fundamental difference. Code that relies on `fork()` must be restructured to use `CreateProcess()` or a cross-platform library.

**Environment variables** — POSIX uses `getenv()`/`setenv()`. Windows uses `GetEnvironmentVariable()`/`SetEnvironmentVariable()`. For cross-platform code, just use `getenv()` — it works on Windows too via MSVC.

### IPC

**Unix domain sockets** don't exist on Windows. Use named pipes (Windows) or TCP localhost (cross-platform).

**D-Bus is Linux-only.** macOS uses XPC, Windows uses COM/RPC. For cross-platform IPC, use:
- ZeroMQ (message queue over TCP)
- gRPC (RPC framework)
- Simple TCP/UDP sockets

## Cross-Platform Library Recommendations

### Prefer cross-platform over platform-specific

When replacing platform APIs, prefer established cross-platform libraries:

| Category | Recommended Library | Language | Notes |
|----------|-------------------|----------|-------|
| UI | **Qt 6** | C++ | Best cross-platform UI, most complete |
| UI | **GTK 4** | C | Good Linux-native, works on macOS/Windows |
| UI | **SDL 3** | C | Games, media apps, minimal UI |
| UI | **Dear ImGui** | C++ | Dev tools, editors, immediate-mode |
| GPU | **wgpu** | Rust/C | Vulkan+Metal+DX12+GL backend |
| GPU | **bgfx** | C++ | 10+ rendering backends |
| 2D | **Cairo** | C | Vector 2D, many backends |
| 2D | **Skia** | C++ | Chrome's 2D engine |
| Audio | **miniaudio** | C | Single-header, playback+recording |
| Audio | **PortAudio** | C | Professional audio I/O |
| Networking | **libcurl** | C | HTTP/FTP/SSL |
| Networking | **Boost.Asio** | C++ | Async I/O, TCP/UDP |
| Async I/O | **libuv** | C | Event loop (Node.js runtime) |
| Filesystem | **std::filesystem** | C++17 | Built-in, no dependency |
| Threading | **std::thread** | C++11 | Built-in, no dependency |
| Logging | **spdlog** | C++ | Fast, async-capable |
| Config | **TOML** (tomlplusplus) | C++ | Simple, human-readable |
| Crypto | **libsodium** | C | Modern, easy to use |
| SSL/TLS | **OpenSSL** | C | Industry standard |
| IPC | **ZeroMQ** | C/C++ | Message queue |
| RPC | **gRPC** | C++ | HTTP/2 based RPC |
| Serialization | **protobuf** / **flatbuffers** | C++ | Binary serialization |
| Video | **FFmpeg** | C | Encode/decode/mux |
| Window/GL | **GLFW** | C | OpenGL/Vulkan context |
| Window/GL | **SDL 3** | C | Window + input + GPU context |

### When NOT to use a cross-platform library

- The project already uses a platform framework deeply (e.g., AppKit throughout) — switching to Qt would be a full rewrite anyway
- Performance requirements demand native APIs (e.g., real-time audio at low latency)
- The target platform has a superior native option that users expect (e.g., macOS users expect AppKit feel)

## Platform-Specific Conventions

### Linux Conventions (XDG)

Follow the [XDG Base Directory Specification](https://specifications.freedesktop.org/basedir-spec/basedir-spec-latest.html):
- `$XDG_CONFIG_HOME` (default `~/.config/`) — config files
- `$XDG_DATA_HOME` (default `~/.local/share/`) — data files
- `$XDG_CACHE_HOME` (default `~/.cache/`) — cache files
- `$XDG_STATE_HOME` (default `~/.local/state/`) — state/logs

### macOS Conventions

- Use `~/Library/Application Support/AppName/` for app data
- Use `~/Library/Caches/AppName/` for cache
- Use `~/Library/Preferences/` (or NSUserDefaults) for settings
- Bundle resources in `.app/Contents/Resources/`

### Windows Conventions

- Use `%APPDATA%\AppName\` for roaming config
- Use `%LOCALAPPDATA%\AppName\` for local data/cache
- Use Known Folder APIs (`FOLDERID_Documents`, etc.) for standard folders
- MSI or NSIS for installers
