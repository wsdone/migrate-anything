# migrate-anything

AI-powered cross-platform software migration agent. Migrate software source code between macOS, Windows, and Linux.

**[中文文档](README_CN.md)**

> Inspired by [cli-anything](https://github.com/HKUDS/CLI-Anything) — an AI agent that builds CLI interfaces for GUI applications. migrate-anything applies the same agent-driven methodology to the problem of cross-platform code migration.

## What It Does

migrate-anything analyzes software source code, identifies platform-specific dependencies, and rewrites the code to work on a target operating system. It handles:

- **System-level code**: File I/O, threading, processes, IPC, networking
- **Graphics APIs**: Metal, DirectX, OpenGL, Vulkan
- **UI frameworks**: AppKit, Win32, GTK, Qt
- **Build systems**: Xcode, Visual Studio → CMake/Meson
- **Architecture**: Platform abstraction layers, rewrite boundaries
- **Language migration**: Swift/ObjC → C++/Rust (when needed)

## Installation

**One-command install** (recommended):
```bash
/plugin marketplace add wsdone/migrate-anything
/plugin install migrate-anything@migrate-anything
```

**Manual install** — add to `~/.claude/settings.json`:
```json
{
  "enabledPlugins": {
    "migrate-anything@migrate-anything": true
  },
  "extraKnownMarketplaces": {
    "migrate-anything": {
      "source": {
        "source": "github",
        "repo": "wsdone/migrate-anything"
      }
    }
  }
}
```

## Commands

| Command | Description | Modifies Code? |
|---------|-------------|---------------|
| `/migrate-anything` | Full migration pipeline (Phases 0-7) | Yes |
| `/migrate-anything:analyze` | Read-only analysis of platform dependencies | No |
| `/migrate-anything:test` | Run tests and update reports | No |
| `/migrate-anything:validate` | Validate against MIGRATE.md standards | No |
| `/migrate-anything:refine` | Iterative improvement of migration coverage | Yes |
| `/migrate-anything:map` | Display API equivalents for a platform pair | No |

## Typical Workflow

```
Step 1: Analyze (optional, read-only)
  /migrate-anything:analyze /path/to/source

Step 2: Migrate
  /migrate-anything:migrate-anything /path/to/source

Step 3: Refine loop (repeat until coverage is acceptable)
  /migrate-anything:refine /path/to/source
  → Refine automatically compiles, tests, and commits after each fix

Step 4: Validate
  /migrate-anything:validate /path/to/source
```

### Automated Refinement with Ralph

Use [ralph-wiggum](https://github.com/anthropics/claude-code/tree/main/plugins/ralph-wiggum) to run refine in an automated loop:

```bash
# Auto-refine until coverage is acceptable
/ralph-wiggum:ralph-loop /migrate-anything:refine /path/to/source
```

### Refine Focus

The refine command accepts a natural language description as the second parameter:

```bash
# Focus on a specific area
/migrate-anything:refine /path/to/source "graphics rendering pipeline"

# Set a coverage goal
/migrate-anything:refine /path/to/source "feature coverage below 90%, keep going"

# Report a UI bug for the agent to fix
/migrate-anything:refine /path/to/source "white border on right side, cannot resize window"
```

**Language migration** — add `--lang <language>` to force a language rewrite (e.g., Swift → Rust). Only use this when the source language is unavailable on the target platform. Most migrations do NOT need this.

```bash
/migrate-anything:migrate-anything /path/to/source --lang rust
```

## Example Scenarios

**UI migration:**
| What | Source | Target | Complexity |
|------|--------|--------|------------|
| macOS native app | macOS (AppKit + CoreGraphics) | Linux (GTK 4) | Hard |
| Windows desktop app | Windows (WPF/WinForms) | Linux (Qt) | Hard |
| Qt app with native integrations | Linux (Qt + D-Bus + udev) | macOS (Qt + native APIs) | Medium |

**Graphics migration:**
| What | Source | Target | Complexity |
|------|--------|--------|------------|
| GPU-accelerated terminal | macOS (Metal + Swift) | Linux (Vulkan) | Hard |
| Game engine | Windows (DirectX + custom platform) | Linux (Vulkan) | Hard |

**System / architecture migration:**
| What | Source | Target | Complexity |
|------|--------|--------|------------|
| System utility | Windows (Win32 API) | Linux (POSIX) | Simple |
| Kernel extension | macOS (kext) | Linux (eBPF) | Hard |
| C library with platform I/O | Windows (Win32 file API) | Linux (POSIX) | Simple |
| Portable runtime app | Any (Electron/JVM/Python/Go) | Any | Simple |

## License

MIT
