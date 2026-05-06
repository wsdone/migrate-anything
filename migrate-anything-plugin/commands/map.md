---
description: Display platform API equivalents for a source-target platform pair
subtask: true
---
# migrate-anything:map Command

Display platform API equivalents and cross-platform library recommendations for migrating between two operating systems.

**Source platform**: $1
**Target platform**: $2

## Arguments

- `$1` is the **source platform** (required). Values: `macos`, `windows`, `linux`
- `$2` is the **target platform** (required). Values: `macos`, `windows`, `linux`

## What This Command Does

Uses the agent's built-in API knowledge and the decision framework in `guides/platform-api-mapping.md` to display relevant API equivalents for the specified source → target platform pair. Also shows cross-platform library recommendations from the guide that can replace platform-specific code entirely.

### Step 1: Validate Arguments

Confirm both `$1` and `$2` are valid platform names. Report an error if either is missing or invalid.

### Step 2: Generate API Equivalents

Read `guides/platform-api-mapping.md` for the decision framework and cross-platform library recommendations. Use your built-in knowledge to produce the source → target API equivalents for each category.

### Step 3: Display Output

For each API category, display a formatted table showing the source API → target API mapping.

### Output Sections

The command displays API equivalents for:

1. **File System Operations** — Path handling, directories, permissions
2. **Process Management** — Create/terminate processes, environment
3. **Threading** — Threads, mutexes, condition variables
4. **IPC** — Pipes, shared memory, message queues
5. **Networking** — Sockets, DNS, HTTP
6. **Graphics/GPU** — 3D APIs, 2D drawing, windowing
7. **UI Frameworks** — Window management, widgets, dialogs
8. **Audio** — Playback, recording
9. **System Services** — Logging, scheduling, power management
10. **Security** — Encryption, key storage, TLS
11. **Hardware Access** — Bluetooth, USB, serial, camera
12. **Cross-Platform Alternatives** — Recommended libraries for each category

### Cross-Platform Library Recommendations

For each category, the command also lists recommended cross-platform libraries that can replace platform-specific code entirely. **Prefer these over direct API mappings** when they exist — they handle platform differences internally.

### Example Output (macOS → Linux)

```
File System — Path Operations
=============================
| Operation | macOS API | Linux API |
|-----------|-----------|-----------|
| Home dir  | NSHomeDirectory() | $HOME / getenv("HOME") |
| Temp dir  | NSTemporaryDirectory() | /tmp / getenv("TMPDIR") |
| Config    | ~/Library/Application Support/ | ~/.config/ (XDG) |
...

Cross-Platform Alternative: std::filesystem (C++17), pathlib (Python)

Threading
=========
| Operation | macOS (POSIX) | Linux (POSIX) |
|-----------|---------------|---------------|
| Create    | pthread_create() | pthread_create() |
| Mutex     | pthread_mutex_t | pthread_mutex_t |
...

Cross-Platform Alternative: std::thread, std::mutex (C++11)

Graphics/GPU
============
| Category | macOS | Linux |
|----------|-------|-------|
| 3D API   | Metal | Vulkan / OpenGL |
| 2D       | CoreGraphics | Cairo |
...

Cross-Platform Alternative: wgpu, bgfx, SDL, Cairo
```

## Example

```bash
# Show macOS → Linux API mappings
/migrate-anything:map macos linux

# Show Windows → Linux API mappings
/migrate-anything:map windows linux

# Show macOS → Windows API mappings
/migrate-anything:map macos windows

# Show Linux → macOS (reverse direction)
/migrate-anything:map linux macos
```

## Notes

- This is a read-only reference command — no code is modified
- The guide `guides/platform-api-mapping.md` provides the decision framework and library recommendations; the agent supplies specific API equivalents from its built-in knowledge
- When a direct API mapping doesn't exist, the table shows a recommended cross-platform alternative
- For same-platform mappings (e.g., `linux linux`), display cross-platform library recommendations only
- For detailed filesystem migration pitfalls, see `guides/filesystem-migration.md`
- For detailed graphics migration patterns, see `guides/graphics-api-migration.md`
- For detailed UI migration patterns, see `guides/ui-framework-migration.md`
