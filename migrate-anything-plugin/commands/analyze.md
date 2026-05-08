---
description: Analyze source code for platform-specific dependencies without making changes
subtask: true
---
# migrate-anything:analyze Command

Analyze source code for platform-specific dependencies and produce a detailed report. This is a **read-only** operation — no code is modified.

**Target source code**: $1
**Target language**: --lang

## CRITICAL: Read MIGRATE.md First

**Before analyzing, read `MIGRATE.md` (located in the plugin root, one directory up from this command).** Phases 0 and 1 define the platform detection and dependency analysis methodology. Follow those standards precisely.

## Arguments

- `$1` is the **source code path or GitHub repo URL** (required).
- `--lang <language>` is the **target programming language** (optional). Forces a language migration assessment (e.g., `--lang rust`, `--lang cpp`). **Only use this when the source language is unavailable or unsuitable on the target platform.** Most migrations do NOT need this flag.

## What This Command Does

This command performs Phase 0 and Phase 1 of the migration methodology, producing a comprehensive analysis without modifying any code.

### Step 1: Source Acquisition

- If `$1` is a GitHub URL, clone it to a local working directory
- Verify the local path exists and contains source code
- Derive the project name from the directory name

### Step 2: Platform Detection

Identify the source platform by scanning for code indicators:

| Platform | Indicators to Scan For |
|----------|----------------------|
| macOS | `#import <AppKit/>`, `NSApplication`, `Metal`, `CoreGraphics`, `.xcodeproj/`, `Package.swift` with `.macOS`, Swift `import Cocoa` |
| Windows | `#include <windows.h>`, `WinMain`, `HWND`, `HRESULT`, `CreateFileW`, `.sln`, `.vcxproj`, `DirectX`, `WPF` |
| Linux | `#include <X11/`, `wayland-client.h`, `dbus/`, `udev`, `/proc/`, `/sys/`, `gtk/gtk.h` |
| Cross-platform | `#ifdef _WIN32`, `#ifdef __APPLE__`, `#ifdef __linux__`, existing platform abstraction layers |

**Use grep/find patterns** to scan all source files. Report confidence level (high/medium/low).

**Checkpoint**: Report detected platform with evidence.

### Step 3: Platform Dependency Inventory (PDI)

Scan all source files and build the PDI. For each dependency found:

- **Classify** into category: UI Framework / Graphics API / System API / Build System / Language Runtime / Hardware Access / Service Integration
- **Count** files affected and approximate lines of code
- **Rate complexity** using `guides/complexity-assessment.md` scoring matrix
- **Note** the specific API calls found and their usage patterns

**Checkpoint**: Present the complete PDI table.

### Step 4: Complexity Assessment

- Calculate the overall complexity score using `guides/complexity-assessment.md`
- If `--lang` is specified, add language migration complexity to the score
- Apply the scoring matrix and decision tree
- Apply adjustment modifiers (no tests, unfamiliar codebase, etc.)
- Report the final score and complexity tier (Simple / Medium / Hard)

### Step 5: Strategy Recommendation

For each dependency in the PDI, recommend a migration strategy:

| Strategy | When to Use |
|----------|------------|
| Direct Replacement | 1:1 API mapping exists |
| Cross-Platform Abstraction | Must support multiple platforms |
| Library Migration | Replace entire library with cross-platform equivalent |
| Rewrite | No equivalent exists |

Use the agent's built-in API knowledge to find target platform equivalents.

### Step 6: Effort Estimation

Estimate effort in developer-weeks based on:
- Project size (lines of code)
- Complexity tier
- Number of dependencies
- Effort multipliers from `guides/complexity-assessment.md`

## Output Format

The command produces an analysis report containing:

```
Platform Migration Analysis
===========================
Project: ghostty
Source: macOS (detected — high confidence)
Target: Linux (specified)

Platform Dependency Inventory:
| Dependency | Category | Files | LoC | Complexity |
|------------|----------|-------|-----|------------|
| AppKit     | UI       | 45    | 3200| Hard (9)   |
| Metal      | Graphics | 12    | 1800| Hard (8)   |
| CoreGraphics| 2D     | 8     | 600 | Medium (5) |
| std::filesystem| System| 3   | 150 | Simple (2) |

Overall Complexity: Hard (8.2/10)
Platform-specific code: ~62% of codebase

Recommended Strategy:
- UI: AppKit → GTK 4 (Rewrite)
- Graphics: Metal → Vulkan (Rewrite)
- 2D: CoreGraphics → Cairo (Library Migration)
- Filesystem: Already portable (No change)

Estimated Effort: Large project, Hard complexity → 8-16 developer-weeks
Risk Level: HIGH — multiple hard dependencies, language migration needed
```

## When to Use

- Before starting a migration, to understand the scope
- To compare migration difficulty between projects
- To decide whether migration is feasible
- To plan the migration approach
- As a first step before `/migrate-anything` or `/migrate-anything:refine`

## Example

```bash
# Analyze Ghostty for Linux migration (current OS)
/migrate-anything:analyze /home/user/ghostty

# Analyze a Windows app for current platform
/migrate-anything:analyze https://github.com/example/winapp

# Analyze with language migration in scope (Swift → Rust)
/migrate-anything:analyze /projects/myapp --lang rust
```
