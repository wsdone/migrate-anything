# Quick Start: migrate-anything

## 1. Analyze First

Before migrating, understand the scope:

```bash
/migrate-anything:analyze /path/to/your/app linux
```

This produces a Platform Dependency Inventory without modifying any code.

## 2. Run Full Migration

```bash
/migrate-anything /path/to/your/app linux
```

The agent will:
- Detect the source platform
- Analyze all platform-specific dependencies
- Plan and implement the migration
- Generate MIGRATION-REPORT.md and PLATFORM-CHECKLIST.md

## 3. Refine Iteratively

Complex migrations won't be complete in one pass. Run refine repeatedly:

```bash
/migrate-anything:refine /path/to/your/app
/migrate-anything:refine /path/to/your/app "UI framework"
/migrate-anything:refine /path/to/your/app "build system"
```

## 4. Validate

```bash
/migrate-anything:validate /path/to/your/app
```

## 5. Check Results

Read `MIGRATION-REPORT.md` for what was done and `PLATFORM-CHECKLIST.md` for manual verification steps.

## Tips

- Always analyze before migrating to understand scope
- Use refine with a focus area for complex migrations
- Check the refinement todo list after each run
- Validate after you think you're done
