---
description: Validate a completed migration against MIGRATE.md standards
subtask: true
---
# migrate-anything:validate Command

Validate a completed migration against MIGRATE.md standards and best practices.

**Target source code**: $1

## CRITICAL: Read MIGRATE.md First

**Before validating, read `MIGRATE.md` (located in the plugin root).** Validation checks compliance with the methodology, architecture patterns, and quality requirements defined there. You need to know the standards to check against them.

## Arguments

- `$1` is the **source code path** (required). Local path to the migrated software source code.

## What This Command Validates

### 1. Required Documentation
- `MIGRATION-REPORT.md` exists and contains:
  - Source/target platform summary
  - Complete Platform Dependency Inventory
  - Per-dependency migration decision with rationale
  - Files changed summary
  - Known limitations
  - Testing results
- `MIGRATION-PLAN.md` exists and documents the planned approach
- `PLATFORM-CHECKLIST.md` exists with verification steps

### 2. Build Verification
- Project compiles on the target platform
- No unresolved platform-specific errors
- Build system correctly handles platform conditionals
- All dependencies are available on target platform

### 3. Platform-Specific Code Coverage
- Scan for remaining source-platform API calls
- Verify all platform dependencies have been addressed
- Check for remaining `#ifdef` blocks that need attention
- Verify platform abstraction layer is complete

### 4. Test Verification
- All existing portable tests pass
- Migration verification tests exist and pass
- No test regressions introduced

### 5. Code Quality
- No hardcoded source-platform paths
- No case-sensitivity issues in includes
- Cross-platform path handling used consistently
- Platform abstraction follows MIGRATE.md patterns

## Validation Report

```
Migration Validation Report
===========================
Source: macOS → Target: Linux
Project: ghostty

Documentation (4/4 checks passed)
Build Verification (3/5 checks passed)
  ✗ Unresolved Metal imports in src/renderer/metal.m
  ✗ Missing Vulkan dependency in CMakeLists.txt
Platform Coverage (15/20 dependencies migrated)
  ✗ NSCursor → not yet mapped
  ✗ NSAppearance → not yet mapped
  ...
Tests (12/15 passed)
  ✗ test_renderer_basic: FAIL (segfault)
  ✗ test_window_create: FAIL (missing GTK init)
  ✗ test_file_dialog: SKIP (not yet implemented)
Code Quality (8/10 checks passed)
  ✗ Hardcoded path: "/Applications/Ghostty.app/..."
  ✗ Case mismatch: #include "renderer.h" vs "Renderer.h"

Overall: NEEDS WORK (42/49 checks)
Refinement needed: 7 items
```

## Success Criteria

- All documentation checks pass
- Project compiles on target platform
- All tests pass
- No remaining source-platform API calls (or explicitly documented)
- Code quality checks pass

## Example

```bash
/migrate-anything:validate /home/user/ghostty-migrated
```
