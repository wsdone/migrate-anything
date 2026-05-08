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
   - Use the agent's built-in API knowledge for specific source → target mappings
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


4. **Differential testing (optional)**: If SSH access to the source platform is available, compare migrated behavior against the original.

#### Shared Implementation Guidance (both tracks)

1. **Apply direct replacements** (Simple): API renames, path separators, type changes, `#ifdef` guards
2. **Introduce cross-platform abstractions** (Medium): wrapper functions in `platform/` directory
3. **Migrate libraries** (Medium): replace platform libraries with cross-platform equivalents
4. **Rewrite platform layers** (Hard): map call sites, identify interface boundary, define abstraction, implement for target, wire in
5. **Handle filesystem differences** (case sensitivity, path separators, XDG conventions)
6. **Handle threading/IPC** (prefer `std::thread` for portability)
7. **Migrate resources**: icon formats, font loading, asset bundling

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
   - Use **GitHub Actions CI** for multi-platform testing
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

## Guides Reference

| Guide | Read when... | Phase |
|-------|-------------|-------|
| [`complexity-assessment.md`](guides/complexity-assessment.md) | Scoping the migration effort, scoring complexity | Phase 1 |
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
