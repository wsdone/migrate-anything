# Filesystem Migration Guide

Pitfalls and non-obvious issues when migrating filesystem code between platforms.

## Decision Framework

```
1. Is the code already using a cross-platform path library (std::filesystem, pathlib)?
   → YES: Check for case sensitivity issues and XDG conventions only
   → NO: Continue to 2

2. Is the filesystem usage limited to simple path joining and file I/O?
   → YES: Switch to std::filesystem (C++17) or pathlib (Python). Minimal effort.
   → NO: Continue to 3

3. Does the code rely on platform-specific file features (ACLs, extended attributes, file locking)?
   → YES: Create a platform/ abstraction layer for those features
   → NO: Direct replacement with cross-platform library
```

## The #1 Killer: Case Sensitivity

Linux is case-sensitive. macOS and Windows are not. Code that works on macOS/Windows **silently breaks** on Linux.

### What breaks

- `#include "utilities.h"` when the file is actually named `Utilities.h`
- `fopen("data/config.json")` when the file is `Data/Config.json`
- Build system references to source files with wrong case
- Asset/resource loading paths

### How to fix

After migrating to Linux, scan for case mismatches:

```bash
# Find #include directives that don't match actual files
find src -name "*.h" | while read f; do
    basename=$(basename "$f")
    grep -ri "#include.*\"$basename\"" src/ | grep -iv "$basename"
done
```

Or better: compile on Linux and let the compiler find every broken include.

## XDG Directory Conventions (Linux)

Linux has a standard for where apps store files. Follow it:

| Data Type | Path | Environment Variable Override |
|-----------|------|------------------------------|
| Config | `~/.config/appname/` | `$XDG_CONFIG_HOME` |
| Data | `~/.local/share/appname/` | `$XDG_DATA_HOME` |
| Cache | `~/.cache/appname/` | `$XDG_CACHE_HOME` |
| State/Logs | `~/.local/state/appname/` | `$XDG_STATE_HOME` |

Always check the environment variable first, fallback to the default path.

## Non-Obvious Differences

### Path separators
Windows uses `\`, everything else uses `/`. Use `std::filesystem::path` (C++17) or `pathlib.Path` (Python) for joining paths. Never hardcode separators.

### Max path length
- Windows: 260 chars (can be extended to 32K with registry setting or `\\?\` prefix)
- Linux: 4096 chars (PATH_MAX)
- macOS: 1024 chars (PATH_MAX)

Keep paths short. Use relative paths when possible.

### File locking
Windows locks files **mandatorily** — other processes can't open a locked file. POSIX locking is **advisory** — it only works if all processes cooperate. This difference can cause deadlocks when migrating Windows→Linux file locking patterns.

### Executable bit
Linux requires the execute permission bit to run binaries and scripts. Windows determines executability by extension (`.exe`, `.bat`). When deploying scripts to Linux, remember `chmod +x`.

### Hidden files
- macOS/Linux: Files starting with `.` are hidden
- Windows: `FILE_ATTRIBUTE_HIDDEN` flag hides files

### Symlinks
Windows requires Developer Mode or admin privileges to create symlinks. Consider:
- Junctions (for directories) — no special privileges needed
- Hard links (for files) — no special privileges needed
- Symlinks — require privilege

### Unicode in filenames
All modern OSes support Unicode filenames, but the normalization differs:
- macOS: Normalizes to NFD (decomposed)
- Linux: No normalization (preserves as-is)
- Windows: Preserves as-is

This means the same filename can be different byte sequences on macOS vs Linux. If your code compares filenames byte-by-byte, it will break.

## Cross-Platform Solutions

| Language | Library | Handles |
|----------|---------|---------|
| C++17 | `std::filesystem` | Paths, directory iteration, permissions |
| C++ | `Boost.Filesystem` | Pre-C++17, very portable |
| Python | `pathlib.Path` | Paths, glob, read/write |
| Rust | `std::fs` + `dirs` crate | Paths + XDG conventions |
| Go | `os` + `filepath` | Paths, directory operations |

For config file storage specifically:
| Language | Library |
|----------|---------|
| C++ | `XDG` macros + `std::filesystem` |
| Python | `platformdirs` (pip) |
| Rust | `dirs` crate |
| Go | `os.UserConfigDir()` |
