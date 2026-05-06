---
description: Refine an existing migration by addressing remaining platform-specific issues
subtask: true
---
# migrate-anything:refine Command

Refine an existing cross-platform migration to address remaining issues and expand coverage.

**Target source code**: $1
**Focus area**: $2

## CRITICAL: Read MIGRATE.md First

**Before refining, read `MIGRATE.md` (located in the plugin root, one directory up from this command).** All new code must follow the same standards as the original migration. MIGRATE.md is the single source of truth for architecture, patterns, and quality requirements.

## Arguments

- `$1` is the **source code path** (required). Local path to the migrated software source code. Must be the same tree that was migrated with `/migrate-anything`.

- `$2` is the **focus area** (optional). A natural-language description of the area to focus on. When provided, skip broad gap analysis and target the specified area.

  Examples:
  - `"graphics rendering pipeline"`
  - `"file dialog and save/load integration"`
  - `"window management and event handling"`
  - `"build system and packaging"`

## What This Command Does

This command is used **after** an initial migration has been performed with `/migrate-anything`. It analyzes remaining platform-specific code, identifies gaps, and iteratively improves the migration.

### Step 1: Inventory Current Coverage
- **Read `MIGRATION-PLAN.md`** for the sub-project list and acceptance criteria
- If sub-projects exist, present the sub-project kanban board:
  ```
  | Sub-project | Priority | Status | Acceptance |
  |-------------|----------|--------|------------|
  | 1. Terminal core | must-have | complete | 5/5 ✓ |
  | 2. Config system | must-have | partial | 2/4 ✓ |
  | 3. Rendering engine | must-have | pending | 0/5 ✓ |
  | 4. Window management | must-have | pending | 0/4 ✓ |
  | 5. Input handling | nice-to-have | pending | 0/3 ✓ |
  ```
- **Pick the next sub-project**: highest priority (must-have first), then prefer `partial` over `pending`
  (a partially done sub-project should be finished before starting a new one)
- If `$2` (focus area) is provided, override the pick and focus on the specified area
- Scan the migrated source code for remaining platform-specific code in the chosen sub-project's scope
- Use targeted grep patterns to find source-platform API calls
- **If no sub-projects are defined** (simple migration), fall back to full codebase scan
- Calculate coverage metrics for the chosen sub-project scope

**Checkpoint**: Present the sub-project board and confirm which sub-project to work on this iteration.

### Step 2: Source Cross-Reference (before implementation)
- Read the **original source module** for the chosen sub-project
- Enumerate every public function, class, method, and feature in the original
- For each one, check if a corresponding implementation exists in the migrated code
- List what's missing vs what's already done
- This establishes the ground truth — the original source code is the specification

### Step 3: Implement
- Address gaps identified in Step 2, in priority order (build blockers first)
- Follow the same patterns established in the original migration
- Follow MIGRATE.md standards for all new code
- **After each fix**: compile → test → commit
- For TDD track: show test is RED before implementing, GREEN after

**Checkpoint**: Compile and test after each individual fix. Do not batch fixes.

### Step 4: Verify Against Acceptance Criteria
- Go through the sub-project's acceptance criteria checklist
- **Actually run the tests** and show the output (not a summary — real test output)
- **Actually grep** for source-platform API calls in scope and show the count
- **Source cross-reference**: re-read the original source module and verify every feature
  has a migrated equivalent. List any that are still missing.
- If ALL criteria pass → update sub-project status to `complete`
- If ANY criterion fails → status stays `partial`, document what's still needed

### Step 5: Update Documentation
- Update migration verification tests to cover new code paths
- Document any tests that still fail with root cause

### Step 5: Update Documentation
- Update `MIGRATION-REPORT.md` with new changes (dependencies migrated, files changed, test results)
- Update `PLATFORM-CHECKLIST.md` with new verification steps
- Update the refinement todo list — remove completed items, add newly discovered items
- Update the coverage map with new status
- **Recalculate migration coverage %** and compare with previous run — show progress
- Generate a **Refine Summary** at the end:
  ```
  ## Refine Summary
  - Sub-project worked on: {name} ({status before} → {status after})
  - Acceptance criteria: {met}/{total} passed
  - Source cross-reference: {matched}/{total} original features have migrated equivalent
  - Source-platform API calls remaining in scope: N
  - Tests: A passed, B failed, C new
  - Unmet criteria (if any): [list what's still needed]
  - Next sub-project to work on: {name} ({priority}, {status})
  ```

## Example

```bash
# Broad refinement — agent finds all remaining gaps
/migrate-anything:refine /home/user/ghostty-migrated

# Focused refinement — target a specific area
/migrate-anything:refine /home/user/ghostty-migrated "terminal renderer and Metal to Vulkan"
/migrate-anything:refine /home/user/winapp-migrated "dialog boxes and file picker integration"
/migrate-anything:refine /home/user/myapp-migrated "build system CMake packaging"
```

## Success Criteria

- All existing tests still pass (no regressions)
- New code follows MIGRATE.md standards
- Coverage meaningfully improved (more dependencies migrated)
- Build blockers resolved (code compiles, or remaining errors documented)
- Documentation updated to reflect changes
- Refinement todo list updated

## Notes

- Refine is incremental — run it multiple times to steadily improve coverage
- Each run should focus on a coherent set of related issues
- The agent should present the gap analysis before implementing, so the user can steer priorities
- Refine never removes existing code — it only adds or fixes
