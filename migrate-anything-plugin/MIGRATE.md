# Agent Harness: Cross-Platform Software Migration

## Purpose

This harness provides a standard operating procedure (SOP) and toolkit for coding
agents (Claude Code, Codex, etc.) to migrate software source code between operating
system platforms. The goal: transform platform-specific code (macOS AppKit, Windows
Win32, etc.) into code that runs on the target platform — handling everything from
simple API swaps to complete platform-layer rewrites.

## General SOP: Migrating Software Between Operating Systems

### Phase 0: Platform Detection and Source Acquisition

1. **Detect the source platform** by scanning for platform indicators:
   - **macOS indicators**: `AppKit`, `Foundation`, `Cocoa`, `CoreGraphics`, `Metal`,
     `CoreAnimation`, `.xcodeproj/`, `Package.swift` with `platform: .macOS`,
     `#import <AppKit/>`, `NSApplication`, `NSWindow`, Swift files with `import Cocoa`
   - **Windows indicators**: `<windows.h>`, `WinMain`, `HWND`, `COM`, `WPF`, `WinUI`,
     `.sln`, `.vcxproj`, `MSVC`, `DirectX`, `GDI`, `registry`, `HRESULT`, `CoCreateInstance`
   - **Linux indicators**: `X11/Xlib.h`, `wayland-client.h`, `dbus/dbus.h`, `udev`,
     `systemd`, `/proc/`, `/sys/`, `gtk/gtk.h`, `QApplication` (Qt), `SDL`
   - **Cross-platform indicators**: `#ifdef _WIN32`, `#ifdef __APPLE__`, `#ifdef __linux__`,
     platform abstraction layers already present

2. **Detect the target platform**:
   - Always the current running OS — migration must be testable locally
   - If the source and target are the same platform, report an error

3. **Detect target language** (if `--lang` is specified):
   - The user has explicitly requested a language migration
   - Add "Language Runtime" as a dependency in the PDI
   - Language migration adds significant scope — warn the user
   - If `--lang` is NOT specified, do NOT attempt language migration

3. **Acquire source code**:
   - If a GitHub URL is provided, clone it to a local working directory
   - Verify the local path exists and contains source code
   - Derive the project name from the directory name

### Phase 1: Platform Dependency Analysis (PDI)

This is the most critical phase. The goal is to build a complete inventory of every
platform-specific dependency in the codebase, discover existing tests, and choose
the migration track.

**If MIGRATION-PLAN.md or analysis output already exists** (from a previous `/migrate-anything:analyze`
run), read it and skip directly to Phase 2. Do not re-analyze.

1. **Scan all source files** and build a **Platform Dependency Inventory (PDI)**:
   - Use file-level scanning (imports, includes, framework references)
   - Use build-system scanning (linked libraries, frameworks, packages)
   - Use configuration scanning (platform conditionals, feature flags)

2. **Classify each dependency** into one of these categories:

   | Category | Examples |
   |----------|----------|
   | **UI Framework** | AppKit, UIKit, Win32, WinForms, WPF, WinUI, GTK, Qt, Motif |
   | **Graphics API** | Metal, DirectX 9/11/12, OpenGL, Vulkan, CoreGraphics, GDI/GDI+ |
   | **System API** | File I/O, threading, processes, IPC, networking, permissions |
   | **Build System** | Xcode, MSBuild, CMake, Meson, Autotools, SCons, Bazel |
   | **Package Format** | .app, .dmg, .msi, .exe, .deb, .rpm, AppImage |
   | **Language Runtime** | Swift/ObjC runtime, .NET Framework, JVM, native C/C++ |
   | **Hardware Access** | GPU compute, camera, microphone, Bluetooth, USB, serial |
   | **Service Integration** | iCloud, Windows services, systemd, launchd, D-Bus |

3. **Rate complexity per dependency**:
   - **Simple (1-3)**: `#ifdef` swap, API rename, path separator change, simple wrapper
   - **Medium (4-7)**: Library replacement with similar API surface, moderate refactoring
   - **Hard (8-10)**: Complete rewrite (AppKit → GTK, Metal → Vulkan, Swift → C++)

4. **Calculate overall Complexity Score**:
   - Sum all dependency scores, weighted by lines of code affected
   - Classify the project: Simple (overall < 4), Medium (4-7), Hard (> 7)

5. **Produce the PDI report** as a structured table:
   ```
   | Dependency | Category | Files Affected | LoC | Complexity | Notes |
   ```

6. **Discover existing tests**. Scan for all test files, test frameworks, and test
   configurations in the source project:

   - Look for test directories: `tests/`, `test/`, `spec/`, `__tests__/`, `*_test.*`, `test_*.*`
   - Identify test frameworks: Google Test, pytest, XCTest, JUnit, etc.
   - Count total tests found
   - Classify each test:
     - **Portable**: Tests platform-independent logic, can run on any OS
     - **Platform-specific**: Tests that directly test platform APIs (e.g., Metal rendering tests)
     - **Integration**: Tests that exercise the full application on the source platform

   **Test Coverage Assessment**:
   - **High test coverage** (>50% of features have tests) → **TDD Track**
   - **Low or no tests** → **FI Track**

   Record the decision and rationale in the PDI report.

7. **Build a Feature Inventory (FI)** (always needed, regardless of track):

   Scan the source code to identify ALL user-facing features, not just platform dependencies:

   **Feature sources to scan:**
   - **CLI tools**: All command-line flags, subcommands, and their behaviors
   - **GUI apps**: All menus, toolbar actions, dialogs, keyboard shortcuts
   - **Libraries**: All public API functions, classes, and methods
   - **Daemons/services**: All configuration options, endpoints, protocols
   - **Config files**: All user-configurable options and their effects

   **For each feature, record:**
   ```
   | Feature | Category | Platform Deps | Source Location | Has Test? | Criticality |
   ```

   - **Category**: core / optional / cosmetic
   - **Platform Deps**: which PDI entries this feature depends on
   - **Source Location**: files/functions that implement this feature
   - **Has Test?**: whether an existing test covers this feature
   - **Criticality**: must-have (core) / nice-to-have (optional) / visual-only (cosmetic)

   **Coverage tracking**: After migration, each feature is marked:
   - **migrated**: Works on target platform (verified by testing)
   - **partial**: Compiles but not fully functional
   - **blocked**: Cannot migrate due to missing dependency
   - **skipped**: Intentionally not migrated (with reason)

   **Migration Coverage = (migrated features) / (total features) × 100%**

   **Coverage targets:**
   - **must-have**: 100% — all core features must be migrated
   - **nice-to-have**: >80% — most optional features should work
   - **cosmetic**: best effort — can be deferred

### Phase 1.5: Feasibility Assessment

**Before designing the migration architecture, assess whether migration is the right approach.**
This gate prevents wasting effort on projects that should be rewritten from scratch instead of migrated.

**Decision tree:**

```
1. Is the overall complexity Hard (8+)?
   → NO: Proceed to Phase 2. Migration is feasible.
   → YES: Continue to 2

2. Does the migration require a language change (Swift→C++, ObjC→Rust)?
   → YES: Continue to 3
   → NO: Continue to 4

3. Is the language change for the ENTIRE codebase (not just platform layer)?
   → YES: **STOP. Recommend full rewrite instead of migration.**
         The effort to translate languages across the whole codebase
         exceeds the effort to rebuild using the target platform's native
         tools and idioms. Suggest: "Build a new app using the source as
         a specification, don't try to mechanically translate the code."
   → NO (only platform layer needs language change): Continue to 4

4. Is more than 60% of the codebase platform-specific?
   → YES: **WARN. This is a major rewrite disguised as a migration.**
         Present honest assessment: "Only ~40% of this code is portable.
         The migration will involve writing mostly new code. Consider:
         (a) Extract the portable core as a library, build a new UI around it.
         (b) Use cross-platform frameworks (Qt, Electron) for a fresh build.
         (c) Proceed with migration, understanding it's essentially new development."
   → NO: Proceed to Phase 2 with Hard complexity flag.

5. Are there zero existing tests AND sparse documentation?
   → YES: Add +2 to complexity. Warn about reverse-engineering risk.
   → NO: Proceed to Phase 2.
```

**Action for the agent:** If the feasibility gate says STOP or WARN:
- Present the assessment to the user with clear reasoning
- Suggest concrete alternatives (which framework, which approach)
- If the user still wants to proceed, continue to Phase 2 but document the risk
- If the user agrees with the alternative, suggest a different approach (e.g., `/migrate-anything:analyze` to extract the portable core only)

### Phase 2: Migration Architecture Design

1. **Map each dependency** to the target platform equivalent:
   - Use the decision framework and library recommendations in `guides/platform-api-mapping.md`
   - Leverage the agent's built-in API knowledge for specific source → target mappings (the guide helps decide HOW to migrate, not what maps to what)
   - Document the mapping for each dependency: `source_api → target_api`

2. **Decide migration strategy per dependency**:

   | Strategy | When to Use | Effort |
   |----------|------------|--------|
   | **Direct Replacement** | 1:1 API mapping exists (e.g., `CreateFileW` → `open()`) | Low |
   | **Cross-Platform Abstraction** | Multiple platforms must be supported simultaneously | Medium |
   | **Library Migration** | Replace entire library with cross-platform equivalent | Medium |
   | **Rewrite** | No equivalent exists; component must be rebuilt from scratch | High |

3. **Design the cross-platform abstraction layer** if needed:
   - Create a `platform/` or `os/` directory with per-platform implementations
   - Define a common interface header or module
   - Use factory pattern or conditional compilation (`#ifdef`)
   - Example structure:
     ```
     platform/
     ├── platform.h          # Common interface
     ├── platform_linux.cpp  # Linux implementation
     ├── platform_macos.cpp  # macOS implementation
     └── platform_win32.cpp  # Windows implementation
     ```

4. **Decompose into sub-projects**. For Medium and Hard migrations, split the work into
   independently verifiable sub-projects. Each sub-project should be completable in a
   single refine iteration and produce a testable, committable state.

   **Decomposition guidelines:**
   - Group by subsystem/module (rendering, window management, input, config, etc.)
   - Each sub-project should have clear boundaries (specific files/directories)
   - Order by dependency: infrastructure first, then consumers
   - Keep sub-projects small enough to complete in one refine pass

   **Sub-project template:**
   ```markdown
   ### Sub-project N: {name}
   **Scope**: {files/directories}
   **Source module**: {original source files to cross-reference}
   **Priority**: must-have / nice-to-have
   **Status**: pending / partial / complete

   **Acceptance criteria** (ALL must be met for "complete"):
   - [ ] All tests for this module pass (actual execution output, not agent claim)
   - [ ] No source-platform API calls remaining in scope (`grep -c` = 0)
   - [ ] Binary compiles and runs without crash in this module's code paths
   - [ ] Source cross-reference: every public function/class in the original source module
         has a corresponding implementation in the migrated version
   - [ ] Source cross-reference: every feature/behavior in the original module is replicated
         (agent must list original features and verify each one exists in migrated code)
   ```

5. **Document the migration plan** in `MIGRATION-PLAN.md`:
   - Source platform and target platform
   - Per-dependency mapping and strategy
   - Proposed directory structure changes
   - Estimated effort and risk assessment
   - **Migration track** (TDD or FI) with rationale
   - **Sub-project list** with acceptance criteria and status
   - For TDD track: test migration plan (which tests to port, which to rewrite, which framework to use)
   - For FI track: feature implementation priority order

   **"complete" status rules** — a sub-project may ONLY be marked complete when:
   1. All acceptance criteria checkboxes are verified (not agent self-assessment, but objective evidence)
   2. Source cross-reference confirms no missing features (agent must enumerate original module's
      features and show each has a migrated equivalent)
   3. Test output shows actual pass/fail (not agent's paraphrase of results)
   4. If any criterion is unmet, status remains `partial` — never upgrade to `complete` prematurely

   **Sub-project status flow:**
   ```
   pending → partial → partial → ... → complete
                                  ↑
                           only when ALL acceptance
                           criteria are verified
   ```

### Phase 3: Build System + Test Framework Setup

1. **Analyze the existing build system** and select the target build system:
   - Prefer **CMake** as the universal cross-platform build system
   - Consider **Meson** for projects that value simplicity
   - Keep existing build system if it's already cross-platform

2. **Common build system transitions**:

   | Source | Target | Approach |
   |--------|--------|----------|
   | Xcode (.xcodeproj) | CMake | Parse `project.pbxproj`, generate `CMakeLists.txt` |
   | Visual Studio (.sln) | CMake | Parse `.sln`/`.vcxproj` XML, generate `CMakeLists.txt` |
   | MSBuild | CMake/Meson | Convert `.csproj`/`.vbproj` properties |
   | Makefile | CMake | Convert rules and variables |

3. **Handle during migration**:
   - Compiler flags and preprocessor defines
   - Link libraries and frameworks (translate `-framework AppKit` to `-lgtk-3`)
   - Include paths and system include directories
   - Platform conditionals (`WIN32`, `APPLE`, `LINUX`)
   - Resource file embedding (`.rc` on Windows, `.plist` on macOS, `.desktop` on Linux)
   - Code signing differences per platform

4. **Set up the test framework** on the target platform:
   - For TDD track: port tests to a cross-platform test framework (Google Test, pytest, etc.)
   - For FI track: set up a basic test runner for smoke tests
   - Configure the build system to build and run tests

5. **Verify the build system works** before proceeding to code migration

### Phase 4: Code Migration (Two Tracks)

**This phase requires actual execution, not just writing code.** You MUST compile and run
after each step. If a compiler or build tool is missing, install it.

For projects with sub-projects defined in Phase 2, work on one sub-project at a time.
Pick the highest priority `pending` or `partial` sub-project (must-have > nice-to-have,
partial > pending). After completing work on a sub-project, verify against its acceptance
criteria before updating its status.

**Source cross-reference verification** — before marking any sub-project as `complete`,
the agent MUST:
1. Read the original source module files listed in the sub-project scope
2. Enumerate every public function, class, method, and user-facing feature
3. For each one, verify a corresponding implementation exists in the migrated code
4. If any feature is missing, the sub-project stays `partial`
5. Record the cross-reference results in the sub-project's acceptance criteria

#### Track A: TDD (projects with good test coverage)

Use this track when Phase 1 determined the project has good test coverage (>50% of features
have tests). Tests drive the implementation — migrate tests first, then implement until all
tests pass.

1. **Migrate tests to target platform**:
   - Port portable tests directly (change imports/paths only)
   - Rewrite platform-specific tests to test the abstraction layer instead
   - All tests must COMPILE on the target platform
   - All tests should FAIL (RED) at this point — this is expected and correct

2. **Build skeleton with stubs**:
   - Create the platform abstraction interface with stub implementations
   - Stubs should compile but return empty/default values so tests can run
   - Verify: all tests compile and run (all RED)

3. **Implement features to turn tests GREEN**, one at a time:
   - Pick a failing test
   - Implement the feature that makes it pass
   - Run tests — verify it turns GREEN and no other tests break
   - Commit
   - **Green tests / total tests = coverage progress**

4. **Coverage metric**: Test pass rate drives the work.
   - 0 GREEN = skeleton only
   - 50% GREEN = half the features work
   - 100% GREEN = all tested features work
   - **Tests are the source of truth for "done"**

#### Track B: Feature Inventory Driven (projects with few/no tests)

Use this track when Phase 1 found low or no test coverage. The Feature Inventory drives
implementation. Tests are written as smoke tests after each feature is implemented.

1. **Build skeleton** — get the project compiling with stubs:
   - Create the platform abstraction interface
   - Stub out all platform-specific calls so the project compiles
   - Verify: project compiles and links (functionality won't work yet)

2. **Implement features by priority**, using the FI from Phase 1:
   - Work through the Feature Inventory in this order:
     1. must-have features first
     2. nice-to-have features second
     3. cosmetic features last
   - For each feature:
     1. Implement the platform-specific code
     2. Compile and verify it builds
     3. Write a basic smoke test (does it not crash? does it produce output?)
     4. Run the smoke test
     5. Commit
   - **FI coverage = migrated features / total features** drives the work

3. **Smoke tests for each implemented feature**:
   - **CLI tools**: Run with sample inputs, verify stdout/exit codes/output files
   - **Libraries**: Write test programs that call the public API, verify behavior
   - **Daemons**: Start/stop test, port binding, request/response verification
   - **GUI apps**: Launch test only (headless mode if supported)
   - See `guides/testing-strategy.md` for per-type test patterns

4. **Differential testing (optional)**: If SSH access to the source platform is available,
   compare migrated behavior against the original. See `guides/testing-strategy.md`
   for the differential testing methodology and output normalization rules.

#### Shared Implementation Guidance (both tracks)

1. **Apply direct replacements** (Simple):
   - API renames: `Sleep()` → `usleep()`, `CreateFileW()` → `open()`
   - Path separators: `\\` → `/` (or use cross-platform path library)
   - Type changes: `DWORD` → `uint32_t`, `HANDLE` → `int`
   - Add `#ifdef` guards where both platforms must be supported

   **Example direct replacement** (Win32 → POSIX):
   ```cpp
   // Before (Windows)
   HANDLE hFile = CreateFileW(path, GENERIC_READ, FILE_SHARE_READ,
                              NULL, OPEN_EXISTING, 0, NULL);
   DWORD bytesRead;
   ReadFile(hFile, buffer, size, &bytesRead, NULL);
   CloseHandle(hFile);

   // After (POSIX)
   int fd = open(path, O_RDONLY);
   ssize_t bytesRead = read(fd, buffer, size);
   close(fd);
   ```

2. **Introduce cross-platform abstractions** (Medium):
   - Create wrapper functions in `platform/` directory
   - Replace direct platform calls with wrapper calls
   - Example:
     ```cpp
     // platform/filesystem.h
     std::string get_home_directory();
     std::string get_temp_directory();
     bool file_exists(const std::string& path);
     ```

3. **Migrate libraries** (Medium):
   - Replace platform-specific libraries with cross-platform equivalents
   - Examples: CoreGraphics → Cairo, WinHTTP → libcurl, CFNetwork → libcurl
   - Adapt API usage to the new library's conventions

4. **Rewrite platform layers** (Hard):
   - For deeply platform-integrated code (e.g., Ghostty's AppKit renderer):
     1. **Map the call sites**: Find every place portable code calls into platform code
        ```bash
        # Find all calls from non-platform files into platform APIs
        grep -rn "NSApplication\|AppKit\|Metal\|CoreGraphics" --include="*.swift" --include="*.m" src/
        # Group by calling module to identify which components need what
        ```
     2. **Identify the interface boundary** — what the platform layer provides to the rest of the code
        - List every platform API function/method that portable code calls
        - Group related calls into interface categories (window, rendering, input, clipboard, etc.)
        - The interface is the MINIMAL set of operations the portable code needs
     3. **Define a clean abstraction interface** at that boundary
        - Keep the interface minimal — only what the application actually uses, not everything the platform provides
        - Use factory functions or dependency injection, not platform `#ifdef` in headers
        - The interface header must NEVER include platform-specific headers
     4. **Implement the interface for the target platform from scratch**
        - Do NOT try to translate platform code line-by-line
        - Instead, satisfy the interface contract using the target platform's native idioms
     5. **Wire the new implementation into the existing codebase**
        - The build system selects which implementation to compile based on target platform
   - For language migrations (Swift → C++/Rust):
     1. Identify Swift-specific idioms (optionals, protocols, extensions)
     2. Map to equivalent patterns in target language
     3. Rewrite module by module, maintaining the same public interface

5. **Handle filesystem differences** — see `guides/filesystem-migration.md` for case sensitivity, path separators, XDG conventions, and file permissions pitfalls

6. **Handle threading/IPC differences** — prefer `std::thread` for portability. See `guides/platform-api-mapping.md` for threading, IPC, and networking pitfalls

7. **Migrate resource handling**:
   - Icon formats: `.icns` (macOS) vs `.ico` (Windows) vs `.png` (Linux)
   - Font loading: CoreText vs DirectWrite vs Fontconfig
   - Asset bundling: `.app bundle` vs `.exe resources` vs `/usr/share/`

### Phase 5: Testing and Verification

**This phase requires actual execution.** You MUST compile the project and run the tests.
If a compiler or build tool is missing, install it before proceeding.

For **TDD track**: Tests were already written and run in Phase 4. This phase is final
verification — run ALL tests and confirm the pass rate. If tests fail, fix and re-run.

For **FI track**: Run all smoke tests written during Phase 4. Generate additional
integration tests for cross-feature interactions. Verify the full application launches
and performs basic operations.

Both tracks:
1. **Write platform-specific test configurations**:
   - CI configuration for the target platform
   - Test fixtures adapted for target platform paths and conventions

2. **Run all tests** — both existing portable tests and new migration tests
3. **Document any test failures** with root cause analysis

### Phase 6: Documentation

1. **Generate `MIGRATION-REPORT.md`** (see `templates/MIGRATION-REPORT.md.template`):
   - Source platform and target platform
   - Platform Dependency Inventory (complete PDI)
   - Complexity score and breakdown
   - Per-dependency migration decision with rationale
   - Files changed summary (count and list of modified files)
   - Build system changes
   - Known limitations and manual steps required
   - Testing results
   - Refinement todo list (items that need further work)

2. **Generate `PLATFORM-CHECKLIST.md`** (see `templates/PLATFORM-CHECKLIST.md.template`):
   - Build verification steps (compile, link, install)
   - Runtime verification steps (launch, basic operations, save/load)
   - Platform-specific checks (file paths, permissions, UI rendering)
   - Known issues and workarounds
   - Sign-off section

### Phase 7: Verification and Refinement

1. **Attempt to build** the migrated code on the target platform:
   - Resolve compilation errors
   - Resolve linker errors
   - Resolve runtime errors

2. **If the current OS is not the target platform**:
   - Use **Docker** for Linux testing: `docker run --rm -v $(pwd):/src -w /src ubuntu:22.04 bash -c "cmake -B build && cmake --build build"`
   - Use **GitHub Actions CI** for multi-platform testing (see `guides/testing-strategy.md` for CI config)
   - Use **cross-compilation** with CMake toolchain files for Windows targets
   - Mark all verification as "not tested on target" in MIGRATION-REPORT.md
   - Create a verification script the user can run on the actual target platform

3. **Run all tests**:
   - All existing portable tests should pass
   - All new migration tests should pass
   - Document any failures

4. **Create a refinement todo list**:
   - Items that need further work (couldn't be completed in this pass)
   - Items that need manual human intervention
   - Items that need target-platform testing (if cross-compiling)

5. **The refinement todo becomes the input** for `/migrate-anything:refine`

## Architecture Patterns & Pitfalls

### The Incremental Migration Pattern

The safest migration strategy is incremental: migrate one dependency at a time,
verify compilation after each, and commit frequently. This avoids the "big bang"
approach where everything breaks simultaneously and is impossible to debug.

```
Migration Order (by complexity):
─────────────────────────────────
Simple (1-3)     ████░░░░░░░░░░░░  ← Start here
Medium (4-7)     ████████████░░░░  ← Then these
Hard (8-10)      ████████████████  ← Last, and may need rewrite

After each dependency:
1. Compile → fix errors → commit
2. Run tests → fix failures → commit
3. Update MIGRATION-REPORT.md → commit
```

**Never batch changes across multiple dependencies without intermediate checks.**
A failed compilation with 5 simultaneous dependency changes is 10x harder to debug
than 5 sequential single-dependency changes.

### Hard Migration Phased Strategy

For Hard migrations (complexity 8+), the standard "migrate all at once, then refine"
approach doesn't work. These migrations require **phased rewrites** where each phase
delivers a testable increment:

**Phase A: Extract and Isolate** (do first)
1. Identify the portable core — code that doesn't depend on any platform API
2. Separate portable code from platform code into different directories/modules
3. Set up the cross-platform build system (CMake/Meson)
4. Create the platform abstraction interface (empty stubs for now)
5. Verify: portable code compiles on both source and target platforms

**Phase B: Infrastructure Layer** (do second)
1. Implement filesystem abstraction (path handling, config dirs)
2. Implement threading/process abstraction (if needed)
3. Implement logging and configuration
4. Verify: all infrastructure tests pass on target platform

**Phase C: First Major Subsystem** (pick the most impactful)
1. Choose one major platform dependency (UI OR graphics, not both)
2. Implement the target-platform version of that subsystem
3. Wire it into the abstraction interface
4. Verify: the subsystem works end-to-end on target platform

**Phase D: Remaining Subsystems** (one at a time)
1. Implement remaining platform dependencies one by one
2. After each: compile, test, commit
3. Verify: integration between subsystems works

**Phase E: Integration and Polish**
1. End-to-end testing of the full application
2. Performance testing on target platform
3. Fix remaining gaps
4. Package for target platform distribution

**Each phase should produce a commit-able, testable state.** If Phase C doesn't
compile, don't move to Phase D. The phases correspond roughly to:
- Phase A → `/migrate-anything` (initial pass, sets up structure)
- Phase B-E → `/migrate-anything:refine` (one phase per refine iteration)

### Preserve Existing Abstractions

If the project already has a platform abstraction layer, **extend it** rather than
replacing it. Rewriting an existing abstraction layer is wasted effort and introduces
regression risk.

### Prefer Cross-Platform Libraries

When replacing a platform-specific library, prefer established cross-platform
alternatives. See `guides/platform-api-mapping.md` for the full recommendation
table organized by category.

### The Rewrite Boundary

For Hard migrations (Ghostty-level), the key is identifying the **rewrite boundary** —
the interface between platform-specific code and portable code. Everything above the
boundary stays the same; everything below gets rewritten.

```
┌──────────────────────────┐
│   Application Logic      │ ← Unchanged
│   (portable code)        │
├──────────────────────────┤
│   Platform Abstraction   │ ← Extended or rewritten
│   (interface layer)      │
├──────────────────────────┤
│   Platform Implementation│ ← Completely rewritten
│   (AppKit/Win32/X11)     │
└──────────────────────────┘
```

### Language Migration Traps

When migrating between languages (e.g., Swift → C++), the agent already knows the idiom mappings (optionals, protocols, async/await, etc.). Focus on the **architectural mismatch**: source language idioms that don't have a clean target equivalent (e.g., Objective-C's dynamic dispatch model has no C++ analogue). Define clean interfaces at module boundaries and implement each side in its natural idiom.

### Dependency Migration Order

Not all dependencies are independent. Some depend on others. Migrate in this order:

1. **Build system** first — you need to compile to verify anything else
2. **Platform detection/preprocessor guards** — infrastructure for `#ifdef` blocks
3. **Filesystem and path handling** — foundational, used by everything else
4. **Threading and concurrency** — core runtime infrastructure
5. **System APIs** (process, IPC, networking) — mid-level services
6. **2D graphics and rendering** — if present, needed before UI
7. **UI framework** — highest-level, depends on graphics and system APIs
8. **Packaging and distribution** — last, after everything works

Migrating the UI before the graphics layer means you can't test rendering. Migrating
system APIs before the build system means you can't compile to verify.

### Header and Include Path Pitfalls

When migrating between platforms, include paths are a constant source of build failures:

- **Case sensitivity**: `#include "Renderer.h"` works on macOS (case-insensitive) but
  fails on Linux if the file is named `renderer.h`. Audit ALL includes.
- **System includes**: `<AppKit/AppKit.h>` doesn't exist on Linux. Use `#ifdef` guards:
  ```cpp
  #ifdef __APPLE__
  #include <AppKit/AppKit.h>
  #elif defined(__linux__)
  #include <gtk/gtk.h>
  #endif
  ```
- **Framework includes**: macOS uses `-framework AppKit` linking. On Linux, translate
  to `-lgtk-3` etc. The build system must handle this mapping.
- **Implicit includes**: Some headers are implicitly included on one platform but not
  another. Always include what you use explicitly.

### The Abstraction Layer Design Pattern

When a cross-platform abstraction layer is needed, use this pattern:

```cpp
// platform/platform.h — Pure interface, no platform includes
#pragma once
#include <string>
#include <memory>

namespace platform {

struct WindowConfig {
    std::string title;
    int width;
    int height;
};

class Window {
public:
    virtual ~Window() = default;
    virtual void show() = 0;
    virtual void hide() = 0;
    virtual void set_title(const std::string& title) = 0;
    // ... more methods
};

// Factory function — implemented per-platform
std::unique_ptr<Window> create_window(const WindowConfig& config);

} // namespace platform
```

```cpp
// platform/platform_linux.cpp — Linux/GTK implementation
#include "platform.h"
#include <gtk/gtk.h>

namespace platform {

class GtkWindow : public Window {
    GtkWindow* window_ = nullptr;
public:
    GtkWindow(const WindowConfig& config) {
        window_ = GTK_WINDOW(gtk_window_new(GTK_WINDOW_TOPLEVEL));
        gtk_window_set_title(window_, config.title.c_str());
        gtk_window_set_default_size(window_, config.width, config.height);
    }
    void show() override { gtk_widget_show(GTK_WIDGET(window_)); }
    void hide() override { gtk_widget_hide(GTK_WIDGET(window_)); }
    void set_title(const std::string& title) override {
        gtk_window_set_title(window_, title.c_str());
    }
};

std::unique_ptr<Window> create_window(const WindowConfig& config) {
    return std::make_unique<GtkWindow>(config);
}

} // namespace platform
```

**Key design rules for abstraction layers:**
1. The interface header must NEVER include platform-specific headers
2. Use factory functions (or dependency injection), not platform `#ifdef` in headers
3. Keep the interface minimal — expose only what the application actually uses
4. Each platform implementation is a separate `.cpp` file, conditionally compiled
5. The build system selects which `.cpp` to compile based on the target platform

### Don't Break the Build

After each dependency migration, verify the project still compiles. Do not batch
changes across multiple dependencies without intermediate compilation checks.

### Filesystem Case Sensitivity

Linux is case-sensitive; macOS and Windows are not. See `guides/filesystem-migration.md`
for detailed pitfalls and detection commands.

### Runtime vs Compile-Time Platform Differences

Some differences only manifest at runtime, not compile time:

| Difference | macOS | Linux | Trap |
|-----------|-------|-------|------|
| Stack size | 8MB (main thread) | 8MB (main), 2MB (pthread) | Thread stack overflow on Linux |
| `select()` FD limit | No hard limit | 1024 (FD_SETSIZE) | Network apps with many connections |
| `fork()` behavior | Copy-on-write | Copy-on-write | Same, but Linux has `CLONE_*` flags |
| `epoll` vs `kqueue` | `kqueue` | `epoll` | Completely different APIs |
| Shared library extension | `.dylib` | `.so` | Build system must handle |
| Default encoding | UTF-8 | Locale-dependent | String handling, file I/O |
| Signal handling | POSIX-like | POSIX | Linux has additional realtime signals |
| `/dev/null` vs `NUL` | `/dev/null` | `/dev/null` | Windows uses `NUL` |

## Debugging Failed Migrations

### Common Failure Modes

**1. Linker errors: undefined reference to platform functions**

Symptom: `undefined reference to 'NSApplicationMain'` or similar.

Cause: The code still calls source-platform APIs that don't exist on the target.

Fix: Search for all source-platform API calls. Use `grep -r "NSApplication\|AppKit\|Metal"` or
equivalent. Each unmigrated call must be wrapped in `#ifdef` or replaced.

**2. Header not found errors**

Symptom: `fatal error: 'AppKit/AppKit.h': No such file or directory`

Cause: Platform-specific includes without guards.

Fix: Add conditional compilation:
```cpp
#ifdef __APPLE__
#include <AppKit/AppKit.h>
#endif
```
Then ensure the code that uses these includes is similarly guarded.

**3. Implicit platform assumptions**

Symptom: Crashes or wrong behavior at runtime with no compile errors.

Common causes:
- `int` assumed to be 32-bit (it is on all mainstream platforms, but `long` varies)
- Pointer size assumptions (use `intptr_t` / `uintptr_t`)
- Byte order assumptions (use `ntohl()`/`htonl()` for network, `std::endian` for C++20)
- Struct packing differences between compilers
- `sizeof(wchar_t)` is 4 on Linux, 2 on Windows — use `char8_t` or `char32_t` explicitly

**4. Build system can't find libraries**

Symptom: `Could NOT find GTK3 (missing: GTK3_LIBRARY GTK3_INCLUDE_DIR)`

Cause: The library is named differently or installed in a non-standard location on the target.

Fix: Use `find_package()` with correct package names, add fallback paths:
```cmake
find_package(PkgConfig REQUIRED)
pkg_check_modules(GTK3 REQUIRED gtk+-3.0)
```

**5. Tests pass on source but fail on target**

Symptom: Tests that worked perfectly on macOS fail on Linux.

Common causes:
- Case-sensitive filesystem: test fixture filenames don't match references
- Thread timing: different scheduling behavior causes race conditions
- Locale differences: string sorting, number formatting
- Floating point: different FPU rounding on ARM vs x86

Fix: Make tests platform-aware. Use `#ifdef` for platform-specific expected values.
Never assume exact floating-point equality across platforms.

### Diagnostic Workflow

When a migration build fails, follow this diagnostic order:

1. **Read the FIRST error** — compiler errors cascade. Fix the first error, recompile.
2. **Check include paths** — 80% of first-build failures are missing includes.
3. **Check link libraries** — 15% are missing library links.
4. **Check for implicit dependencies** — the remaining 5% are platform assumptions.
5. **Isolate the failure** — if the full project doesn't compile, try compiling individual
   source files to narrow down which file has the issue.

## Principles & Rules

These are non-negotiable. Every migration MUST follow all of them.

**Execution:**
- **Do NOT spawn sub-agents or use agent teams** — perform all work directly in the main conversation. Sub-agents have poor quality for migration tasks. Only use sub-agents if the user explicitly requests it.
- **You MUST actually compile and run code, not just write it** — every phase that says "compile" or "test" means actually executing the build command and running the tests. Writing code without compiling is NOT acceptable.
- **If a compiler or build tool is missing, install it** — use `apt`, `brew`, `choco`, or download it. Do NOT skip compilation because a tool is missing. Ask the user for permission if unsure about the installation method.
- **Compilation failures must be fixed immediately** — do not proceed to the next phase if the code doesn't compile. Fix the error, recompile, and only then continue.

**Safety:**
- **Never modify the original source** — always work on a copy, branch, or fork.
- **Preserve git history** — commit after each successful phase for easy rollback.
- **Test incrementally** — verify compilation after each dependency migration.
- **Keep a rollback plan** — every phase should leave the project in a compilable state.

**Quality:**
- **Produce machine-readable migration reports** — JSON format available for programmatic use.
- **Document every migration decision** — the MIGRATION-REPORT.md must explain WHY each approach was chosen.
- **Rate complexity honestly** — do not underestimate the effort. A Hard migration should be flagged as such.
- **Use the target platform's conventions** — don't write "macOS-style" code on Linux. Follow the target platform's idioms.
- **Don't introduce unnecessary abstractions** — only create a platform abstraction layer when the code genuinely needs to run on multiple platforms.
- **Match source behavior, not source structure** — the migrated code should behave identically to the source, even if the internal structure is different.

**Testing:**
- **Existing portable tests MUST still pass** — migration should not break portable functionality.
- **Migration verification tests MUST be written** — prove the abstraction layer works.
- **Build verification is mandatory** — if it doesn't compile on the target, the migration is incomplete.
- **Test on the real target platform** — emulators and CI containers may not catch all issues.

**Documentation:**
- **Every migrated project MUST have a MIGRATION-REPORT.md** explaining what was done and why.
- **Every migrated project MUST have a PLATFORM-CHECKLIST.md** for manual verification.
- **Document known limitations explicitly** — not everything will work perfectly. Be honest about what's broken.

**Migration Scope:**
- **Start with a minimal viable migration** — get the project compiling and running basic functionality first, then expand.
- **Defer optimizations** — don't try to optimize for the target platform during initial migration. Get it working first.
- **Respect the source project's license** — check license compatibility before starting migration.

## Common Migration Scenarios

This table shows common migration patterns with their typical complexity and recommended approach.

| Source | Target | Typical Dependencies | Complexity | Recommended Approach |
|--------|--------|---------------------|------------|---------------------|
| macOS AppKit app | Linux | AppKit, CoreGraphics, Metal, Objective-C | Hard (8-10) | Rewrite UI with GTK/Qt, rewrite renderer with Vulkan/OpenGL |
| macOS AppKit app | Windows | AppKit, CoreGraphics, Metal | Hard (8-10) | Rewrite UI with WinUI/Qt, rewrite renderer with DirectX |
| Win32/C++ app | Linux | Win32 API, GDI, DirectX | Medium-Hard (6-8) | Replace Win32 with GTK/Qt, DirectX with Vulkan/OpenGL |
| Win32/C++ app | macOS | Win32 API, GDI | Medium-Hard (6-8) | Replace Win32 with AppKit or Qt |
| Qt app (Linux) | macOS/Windows | Qt, X11, D-Bus, udev | Medium (4-6) | Qt is cross-platform, replace X11/D-Bus integrations |
| GTK app (Linux) | macOS/Windows | GTK, X11, Cairo, D-Bus | Medium (5-7) | GTK works on macOS (Homebrew) and Windows (MSYS2), replace X11 |
| Swift/iOS app | Android | UIKit, CoreFoundation, Swift | Hard (9-10) | Complete rewrite with Kotlin/Flutter/React Native |
| C library (POSIX/Win32) | Cross-platform | POSIX or Win32 APIs | Simple (2-3) | Replace platform calls with equivalents |
| Portable runtime app | Any | Electron, JVM, Python, Go, Rust | Simple (1-2) | Already portable — check cgo/JNI/C extensions and platform paths |
| Game (custom engine) | Any | DirectX/Metal, custom platform layer | Hard (8-10) | Rewrite renderer and platform layer |

## Applying This to Specific Projects

The pattern is always the same: **detect platform code → map to equivalents → implement → test → document**. What varies is the scope of the rewrite.

### Example: Migrating a macOS Terminal Emulator to Linux

A project like Ghostty illustrates a Hard migration. See `guides/complexity-assessment.md`
for the detailed scoring breakdown. In summary: AppKit, Metal, Swift/ObjC, CoreText, and
the Xcode build system all need rewrites. Only the terminal emulation core (VT parsing,
PTY handling) is truly portable. This is why it's scored as Hard (10).

### Example: Migrating a Win32 Tool to Linux

A simpler case — a Windows CLI utility is typically Simple (2-3): mechanical API renames (`CreateFile` → `open()`, etc.), type changes (`DWORD` → `uint32_t`), and build system translation (VS → CMake). Most changes are direct replacements.

## Guides Reference

Detailed guides live in `guides/`. Use this table to decide which ones to read
based on the migration you're performing.

| Guide | Read when... | Phase |
|-------|-------------|-------|
| [`platform-api-mapping.md`](guides/platform-api-mapping.md) | Every migration — decision framework for HOW to migrate, pitfalls, and library recommendations | Phase 1, 2, 4 |
| [`complexity-assessment.md`](guides/complexity-assessment.md) | Scoping the migration effort | Phase 1 |
| [`build-system-migration.md`](guides/build-system-migration.md) | Changing the build system | Phase 3 |
| [`graphics-api-migration.md`](guides/graphics-api-migration.md) | Migrating Metal/DirectX/OpenGL/Vulkan | Phase 4 |
| [`ui-framework-migration.md`](guides/ui-framework-migration.md) | Migrating AppKit/Win32/GTK/Qt | Phase 4 |
| [`filesystem-migration.md`](guides/filesystem-migration.md) | Path handling, permissions, case sensitivity | Phase 4 |
| [`testing-strategy.md`](guides/testing-strategy.md) | Cross-platform test patterns, differential testing | Phase 3, 4, 5 |

## Directory Structure (Output)

When migrating a project, the output structure is:

```
<project-name>/
├── MIGRATION-REPORT.md          # What was done and why
├── MIGRATION-PLAN.md            # Planned approach (created in Phase 2)
├── PLATFORM-CHECKLIST.md        # Manual verification checklist
├── <original source tree>       # Modified source code
└── tests/
    ├── migration/               # Migration verification tests
    │   ├── test_platform_abstraction.py (or .cpp)
    │   ├── test_filesystem.py
    │   └── test_build.py
    └── <existing tests>         # Original tests (unmodified where possible)
```
