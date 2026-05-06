---
name: "migrate-anything"
description: "Migrate software source code between OS platforms — analyze platform dependencies, plan migration strategy, rewrite platform-specific code for Mac, Windows, or Linux."
---

# migrate-anything

Migrate software source code between operating system platforms. Analyzes platform-specific dependencies (AppKit, Win32, Metal, DirectX, etc.), maps them to target platform equivalents, and rewrites the code to work on the target OS.

## When to Use

- Porting platform-specific software to another OS (UI, graphics, system APIs, or all of the above)
- Migrating system-level code: file I/O, threading, IPC, networking between platforms
- Migrating GPU/graphics code: Metal ↔ Vulkan ↔ DirectX ↔ OpenGL
- Migrating UI frameworks: AppKit ↔ GTK/Qt ↔ Win32
- Making a platform-specific application cross-platform
- Migrating build systems: Xcode/VS → CMake
- Assessing migration feasibility before starting work

## Commands

| Command | Description |
|---------|-------------|
| `/migrate-anything <path> [target]` | Full migration pipeline (analyze → plan → implement → test → document) |
| `/migrate-anything:refine <path> [focus]` | Iteratively refine an existing migration |
| `/migrate-anything:analyze <path> [target]` | Read-only analysis — produces a report without changing code |
| `/migrate-anything:test <path>` | Run tests and update MIGRATION-REPORT.md with results |
| `/migrate-anything:validate <path>` | Validate a completed migration against standards |
| `/migrate-anything:map <from> <to>` | Show platform API equivalents and library recs |

## Complexity Levels

| Level | Score | Characteristics | Example |
|-------|-------|----------------|---------|
| **Simple** | 1-3 | Minor API replacements, mostly portable | C library with Win32 file I/O → POSIX |
| **Medium** | 4-7 | Library replacements, some architecture changes | Qt app with platform integrations |
| **Hard** | 8-10 | Full rewrite of platform layer, possible language change | GPU terminal (AppKit+Metal+Swift → GTK+Vulkan) |

## Examples

### Analyze Before Migrating
```bash
# See what a macOS → Linux migration would involve
/migrate-anything:analyze /home/user/ghostty linux
```

### Full Migration
```bash
# Migrate a macOS app to Linux
/migrate-anything /home/user/ghostty linux

# Migrate a Windows app to the current platform
/migrate-anything https://github.com/example/winapp
```

### Iterative Refinement
```bash
# After initial migration, refine specific areas
/migrate-anything:refine /home/user/ghostty-migrated "terminal renderer"
/migrate-anything:refine /home/user/ghostty-migrated "build system packaging"
```

### API Equivalents Reference
```bash
# See API equivalents between platforms
/migrate-anything:map macos linux
/migrate-anything:map windows linux
```

## Migration Methodology

The migration follows an 8-phase methodology defined in MIGRATE.md:

1. **Platform Detection** — Identify source and target platforms
2. **Dependency Analysis** — Build Platform Dependency Inventory (PDI)
3. **Feasibility Assessment** — Gate check for Hard migrations (language change, >60% platform code)
4. **Architecture Design** — Plan migration strategy per dependency
5. **Build System Migration** — Translate build system (Xcode/VS → CMake)
6. **Code Migration** — Implement replacements (Simple → Hard)
7. **Testing** — Verify compilation, write migration tests
8. **Documentation & Verification** — Generate reports, build, and create refinement todo

## For AI Agents

**Design philosophy**: Guides teach methodology and decision-making, NOT API lookups. The agent already knows API mappings. Guide content should focus on decision frameworks, non-obvious pitfalls, and architectural patterns.

When using this skill programmatically:

1. **Always run `/migrate-anything:analyze` first** — Understand scope before committing to migration
2. **Use `/migrate-anything:refine` repeatedly** — Complex migrations need multiple iterations
3. **Check MIGRATION-REPORT.md** after each run — Contains the refinement todo list
4. **Verify PLATFORM-CHECKLIST.md** — Manual verification may still be needed
5. **Respect complexity scores** — Hard migrations may need human guidance for architecture decisions

## Key Guides

| Guide | Content |
|-------|---------|
| `platform-api-mapping.md` | Decision framework for API migration + cross-platform library recs |
| `complexity-assessment.md` | Scoring matrix, decision tree, effort estimation |
| `build-system-migration.md` | Build migration pitfalls + minimal CMake pattern |
| `graphics-api-migration.md` | GPU migration decisions, coordinate system traps, rewrite pattern |
| `ui-framework-migration.md` | Paradigm shifts, event loop differences, rewrite boundary |
| `filesystem-migration.md` | Case sensitivity, XDG conventions, cross-platform solutions |
| `testing-strategy.md` | Migration-specific testing pitfalls and compile-test-fix loop |
