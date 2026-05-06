# Build System Migration Guide

Decision framework and pitfalls for migrating build systems between platforms. The agent already knows CMake syntax and can translate Xcode/VS settings — this guide focuses on strategic decisions and non-obvious traps.

## Decision Framework

```
1. Is the existing build system already cross-platform (CMake, Meson, Bazel)?
   → YES: Probably just needs new platform flags/dependencies. Minimal work.
   → NO: Continue to 2

2. Does the project use a platform-locked build system (Xcode, Visual Studio)?
   → YES: Must translate to CMake or Meson. Continue to 3.
   → NO: Continue to 4

3. How complex is the build?
   → Simple (few targets, no custom phases): Translate directly to CMake
   → Complex (custom build phases, code generation, script build steps): Plan carefully, see pitfalls below

4. Is it Makefile-based?
   → YES: Translate to CMake if cross-platform needed, otherwise just add target-platform targets
   → NO: Probably a language-specific build tool (Cargo, Go modules, npm) — these are already portable
```

## Build System Detection

| Indicator | Build System |
|-----------|-------------|
| `.xcodeproj/` or `.xcworkspace/` | Xcode |
| `Package.swift` | Swift Package Manager |
| `.sln` + `.vcxproj` | Visual Studio / MSBuild |
| `CMakeLists.txt` | CMake |
| `meson.build` | Meson |
| `configure.ac` + `Makefile.am` | Autotools |
| `SConstruct` or `SConscript` | SCons |
| `BUILD` + `WORKSPACE` | Bazel |
| `Cargo.toml` | Cargo (Rust) |
| `build.gradle` or `pom.xml` | Gradle / Maven (Java) |
| `Makefile` (without configure.ac) | Make |
| `go.mod` | Go modules |

## Key Pitfalls

### Xcode Custom Build Phases

Xcode projects often have custom build phases (shell scripts, file copy phases, code generation) that have no direct CMake equivalent. Each one must be manually translated to `add_custom_command()` or `add_custom_target()`.

**Common Xcode-specific build phases to watch for:**
- "Copy Bundle Resources" phase → `install(FILES ...)` or `add_custom_command()`
- "Run Script" phases → `add_custom_command(COMMAND ...)`
- "Compile Sources" filters (per-file flags) → `set_source_files_properties()`
- Pre/post-build scripts → `add_custom_command(POST_BUILD ...)`

### Visual Studio Per-Configuration Settings

VS projects can have different settings per configuration (Debug/Release) AND per platform (Win32/x64). CMake handles this with generator expressions. Don't try to replicate VS's UI-driven configuration — use CMake's approach:

```cmake
target_compile_definitions(myapp PRIVATE
    $<$<CONFIG:Debug>:_DEBUG>
    $<$<CONFIG:Release>:NDEBUG>
)
```

### Framework Linking on macOS

macOS uses `-framework AppKit` linking. CMake handles this with `find_library()`:
```cmake
if(APPLE)
    find_library(APPKIT AppKit)
    target_link_libraries(myapp ${APPKIT})
endif()
```
**Don't hardcode framework paths** — `find_library()` resolves them correctly across Xcode versions and macOS SDKs.

### pkg-config vs find_package

Linux libraries are typically found via pkg-config. macOS/Windows libraries may use CMake's `find_package()`. Handle both:

```cmake
# Try find_package first, fall back to pkg-config
find_package(GTK3 QUIET)
if(NOT GTK3_FOUND)
    find_package(PkgConfig REQUIRED)
    pkg_check_modules(GTK3 REQUIRED gtk+-3.0)
endif()
```

### The Source File Extraction Problem

When migrating from Xcode or VS, you need to extract source file lists from proprietary formats:

- **Xcode**: Sources are in `project.pbxproj` — grep for `.m"`, `.swift"`, `.cpp"` entries
- **Visual Studio**: Sources are in `.vcxproj` XML — grep for `Include="*.cpp"` attributes
- **Both**: Verify against actual files on disk — these lists are often stale

### Cross-Compilation Toolchain Files

If you're cross-compiling (e.g., building for Windows from Linux), use CMake toolchain files:

```cmake
# toolchain-mingw.cmake
set(CMAKE_SYSTEM_NAME Windows)
set(CMAKE_C_COMPILER x86_64-w64-mingw32-gcc)
set(CMAKE_CXX_COMPILER x86_64-w64-mingw32-g++)
```

This is a common need when migrating Windows apps to Linux but needing to verify the Windows build still works.

## The "Minimal Viable CMake" Pattern

When translating from Xcode or VS, don't try to replicate every setting. Start with the minimal CMake that compiles the project, then add platform-specific handling:

```cmake
cmake_minimum_required(VERSION 3.20)
project(MyApp LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)

# Collect sources (extract from Xcode/VS project)
file(GLOB_RECURSE SOURCES src/*.cpp src/*.c)
file(GLOB_RECURSE HEADERS src/*.h src/*.hpp)

add_executable(myapp ${SOURCES} ${HEADERS})
target_include_directories(myapp PRIVATE include/)

# Platform-specific dependencies ONLY
if(APPLE)
    find_library(APPKIT AppKit)
    target_link_libraries(myapp ${APPKIT})
elseif(WIN32)
    target_link_libraries(myapp user32 gdi32)
else()
    find_package(PkgConfig REQUIRED)
    pkg_check_modules(GTK3 REQUIRED gtk+-3.0)
    target_link_libraries(myapp ${GTK3_LIBRARIES})
    target_include_directories(myapp PRIVATE ${GTK3_INCLUDE_DIRS})
endif()
```

**Iterate from here.** Get this compiling first, then refine compiler flags, optimization settings, and packaging.

## Platform-Specific Resource Handling

Each platform has different packaging and resource requirements:

### macOS
- `Info.plist` — Application metadata (required)
- `.icns` — Application icon
- Code signing via `codesign`, notarization via `notarytool`
- Bundle structure: `MyApp.app/Contents/`

### Windows
- `.rc` files — Resource scripts (icons, version info, manifest)
- `.manifest` files — UAC and DPI settings
- MSI or NSIS for installers
- Code signing via `signtool` (requires paid certificate)

### Linux
- `.desktop` file — Desktop entry for app launcher (required)
- `.png` icons — Multiple sizes in hicolor theme
- AppImage / Flatpak / Snap for packaging
- No code signing (GPG signatures for packages)

Minimal `.desktop` template:
```ini
[Desktop Entry]
Type=Application
Name=MyApp
Comment=A great application
Exec=myapp
Icon=myapp
Terminal=false
Categories=Development;
```

## Code Signing

| Platform | Tool | Cost |
|----------|------|------|
| macOS | `codesign` + `notarytool` | Apple Developer Program ($99/year) |
| Windows | `signtool` | DigiCert, Sectigo, etc. ($100-500/year) |
| Linux | GPG signatures | Free (self-signed OK) |

**Plan signing early.** Obtaining certificates takes days to weeks. Windows users get scary warnings for unsigned executables. macOS blocks unsigned apps by default (Gatekeeper).

## Cross-Platform CMake Patterns

### Platform Detection
```cmake
if(APPLE)
    # macOS specific
elseif(WIN32)
    # Windows specific (includes MSYS/Cygwin)
elseif(UNIX)
    # Linux and other Unix
endif()
```

### Finding Libraries Across Platforms
```cmake
if(APPLE)
    find_library(COREGRAPHICS CoreGraphics)
    target_link_libraries(myapp ${COREGRAPHICS})
elseif(WIN32)
    # Windows SDK libs are auto-found
    target_link_libraries(myapp opengl32)
else()
    find_package(X11 REQUIRED)
    target_link_libraries(myapp X11::X11)
endif()
```

### Installing Platform Resources
```cmake
if(APPLE)
    install(FILES icon.icns DESTINATION MyApp.app/Contents/Resources)
elseif(WIN32)
    install(FILES myapp.ico DESTINATION bin)
else()
    install(FILES myapp.desktop DESTINATION share/applications)
    install(FILES myapp.png DESTINATION share/icons/hicolor/256x256/apps)
endif()
```
