---
description: Migrate software source code from one OS platform to another (full pipeline)
subtask: true
---
# migrate-anything Command

Migrate software source code between operating system platforms.

**Target source code**: $1
**Target language**: --lang

## CRITICAL: Read MIGRATE.md First

**Before doing anything else, you MUST read `MIGRATE.md` (located in the plugin root, one directory up from this command).** It defines the complete methodology, architecture standards, and implementation patterns. Every phase below follows MIGRATE.md. Do not improvise — follow the migration specification.

## Arguments

- `$1` is the **source code path or GitHub repo URL** (required). Either:
  - A **local path** to the software source code (e.g., `/home/user/ghostty`, `./myapp`)
  - A **GitHub repository URL** (e.g., `https://github.com/ghostty-org/ghostty`)

  If a GitHub URL is provided, clone the repo locally first, then work on the local copy.

- `--lang <language>` is the **target programming language** (optional). Forces a language migration in addition to platform migration (e.g., `--lang rust`, `--lang cpp`, `--lang go`). **Only use this when the source language is unavailable or unsuitable on the target platform** (e.g., Swift → Linux requires a language change). Most platform migrations do NOT need this flag — only use it when you explicitly want to rewrite in a different language.

## What This Command Does

This command implements the complete migrate-anything methodology to transform platform-specific source code into code that runs on the target platform. **All phases follow the standards defined in MIGRATE.md.**

### Phase 0: Platform Detection and Source Acquisition
- If `$1` is a GitHub URL, clone it to a local working directory
- Verify the local path exists and contains source code
- **Detect source platform** by scanning for indicators:
  - macOS: AppKit, Foundation, Metal, `.xcodeproj`, `Package.swift` with `platform: .macOS`
  - Windows: `<windows.h>`, Win32, DirectX, `.sln`, `.vcxproj`, WPF, WinUI
  - Linux: X11, Wayland, D-Bus, udev, `/proc/`, `/sys/`
- **Determine target platform** — always the current running OS (migration must be testable locally)
- Derive the project name from the directory name

**Checkpoint**: Present platform detection results to user. Confirm source → target pair.

### Phase 1: Platform Dependency Analysis
- **If `MIGRATION-PLAN.md` already exists** (from a previous `/migrate-anything:analyze` run), read it and skip to Phase 2
- Otherwise, scan all source files and build a **Platform Dependency Inventory (PDI)**
- Classify each dependency: UI Framework / Graphics API / System API / Build System / Language Runtime / Hardware Access / Service Integration
- Rate complexity per dependency: Simple (1-3) / Medium (4-7) / Hard (8-10)
- Calculate overall **Complexity Score** using `guides/complexity-assessment.md` scoring matrix
- Estimate total lines of platform-specific code and percentage of codebase
- If `--lang` is specified, add language migration to the PDI with its own complexity score

**Checkpoint**: User reviews PDI. Proceed only after confirmation. If complexity is Hard (8+), warn the user that this may require significant rewrites and multiple refine iterations. If `--lang` is specified, warn that language migration adds significant scope.
- If overall complexity is Hard (8+), run the feasibility decision tree from MIGRATE.md Phase 1.5
- Assess whether migration is the right approach or if a full rewrite/alternative is recommended
- If the gate says **STOP** (language change for entire codebase): recommend full rewrite instead
- If the gate says **WARN** (>60% platform-specific): present honest assessment and alternatives
- If no tests and sparse docs: add risk note

**Checkpoint**: Present feasibility assessment to user. If STOP or WARN, get explicit confirmation to proceed before Phase 2.

### Phase 2: Migration Architecture Design
- Map each dependency to target platform equivalent using the agent's built-in API knowledge
- Decide migration strategy per dependency:
  - **Direct Replacement** for 1:1 API mapping (e.g., `CreateFileW` → `open()`)
  - **Cross-Platform Abstraction** for multi-platform support (e.g., `platform/` directory)
  - **Library Migration** for whole-library replacement (e.g., CoreGraphics → Cairo)
  - **Rewrite** for components with no equivalent (e.g., AppKit → GTK)
- Design the cross-platform abstraction layer if needed (see MIGRATE.md "Abstraction Layer Design Pattern")
- Determine migration order following MIGRATE.md "Dependency Migration Order"
- Document in `MIGRATION-PLAN.md` with: source/target, per-dep mapping, strategy, estimated effort, migration order
- **Present the migration plan to the user** for approval

**Checkpoint**: User reviews and approves the migration plan before any code changes.

### Phase 3: Build System + Test Framework Setup
- Analyze existing build system (Xcode, VS, Makefile, etc.)
- Translate to cross-platform build system (prefer CMake)
- Handle compiler flags, link libraries, platform conditionals
- Set up test framework on target platform
- **Verify the build system compiles** before proceeding to Phase 4
- Translate build system to cross-platform (prefer CMake)

**Checkpoint**: Build system produces a valid (possibly incomplete) project structure.

### Phase 4: Code Migration (TDD or FI-driven)

Follow the track determined in Phase 1. See MIGRATE.md Phase 4 for full details.

**Track A (TDD — good test coverage):**
- Migrate tests to target platform (all should be RED)
- Build skeleton with stubs
- Implement features one by one until tests turn GREEN
- **Green tests / total tests = coverage progress**

**Track B (FI-driven — few/no tests):**
- Build skeleton with stubs (get compiling first)
- Implement features from the Feature Inventory, must-have first
- Write a smoke test for each implemented feature
- **FI coverage = migrated features / total features**

**Both tracks:** After each feature implementation, compile → test → commit.
- Rewrite platform layers as needed (Hard)
- Handle filesystem, threading, and IPC differences
- Scan for case sensitivity issues in includes and file references
- See guides for specific patterns:
  - Handle graphics, UI, and filesystem differences

**Checkpoint**: Commit after each dependency. If build breaks, fix before moving to next dependency.

### Phase 5: Testing and Verification
- Run ALL tests and confirm pass rate
- TDD track: confirm final GREEN rate, fix any remaining RED tests
- FI track: run all smoke tests, verify feature coverage
- Generate integration tests for cross-feature interactions
- Verify the full application launches and performs basic operations
- See `guides/testing-strategy.md` for methodology

**Checkpoint**: All tests must pass (or failures must be documented with root cause).

### Phase 6: Documentation
- Generate `MIGRATION-REPORT.md` using `templates/MIGRATION-REPORT.md.template`:
  - Source/target platform summary
  - Complete PDI with complexity scores
  - Per-dependency migration decision with rationale
  - Files changed summary (count and list)
  - Build system changes
  - Known limitations and manual steps required
  - Testing results (pass/fail counts)
  - Refinement todo list
- Generate `PLATFORM-CHECKLIST.md` using `templates/PLATFORM-CHECKLIST.md.template`:
  - Build verification steps (compile, link, install)
  - Runtime verification steps (launch, basic operations)
  - Platform-specific checks
  - Known issues and workarounds
  - Sign-off section

### Phase 7: Verification
- Attempt to build on target platform
- Run all tests
- Document failures with root cause analysis
- Create prioritized refinement todo list for `/migrate-anything:refine`
- If the project compiles and tests pass, mark migration as "complete"
- If not, mark as "needs refinement" and list specific gaps

## Output Structure

```
<project-name>/
├── MIGRATION-REPORT.md          # What was done and why
├── MIGRATION-PLAN.md            # Planned approach
├── PLATFORM-CHECKLIST.md        # Manual verification
├── <original source tree>/      # Modified source code
└── tests/migration/             # Migration verification tests
```

## Success Criteria

The command succeeds when:
1. Platform Dependency Inventory is complete and accurate
2. MIGRATION-PLAN.md documents all decisions with rationale
3. Code compiles on the target platform (or build errors are documented)
4. All existing portable tests still pass
5. Migration verification tests pass
6. MIGRATION-REPORT.md is generated with full details
7. PLATFORM-CHECKLIST.md is generated for manual verification
8. Refinement todo list identifies remaining work

## Example

```bash
# Migrate Ghostty (macOS) to Linux (current OS)
/migrate-anything /home/user/ghostty

# Migrate a Windows app to current platform
/migrate-anything https://github.com/example/winapp

# Migrate a Mac app to Linux, rewriting Swift to Rust
/migrate-anything /projects/myapp --lang rust
```
