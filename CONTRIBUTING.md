# Contributing to migrate-anything

## Project Structure

```
migrate-anything/
├── .claude-plugin/
│   └── marketplace.json                  # Marketplace registration
├── migrate-anything-plugin/              # Installed plugin
│   ├── .claude-plugin/
│   │   └── plugin.json                   # Plugin metadata
│   ├── commands/                         # Slash commands
│   ├── guides/                           # Methodology and decision frameworks
│   ├── templates/                        # Report templates
│   └── MIGRATE.md                        # Core methodology (source of truth)
├── skills/
│   └── migrate-anything/
│       └── SKILL.md                      # Skill definition
└── .claude/
    └── commands/
        └── migrate-dev.md                # Development meta-command
```

## Adding a New Guide

1. Create a new `.md` file in `migrate-anything-plugin/guides/`
2. Follow the existing guide format: title, overview, tables, code examples
3. Reference the guide from `MIGRATE.md` in the "Guides Reference" table
4. Include the guide in the relevant command phase descriptions

## Adding a New Command

1. Create a new `.md` file in `migrate-anything-plugin/commands/`
2. Include YAML frontmatter with `description` and `subtask: true`
3. Document arguments, what the command does, success criteria, and examples
4. Reference `MIGRATE.md` as the methodology source
5. Update `SKILL.md` to list the new command

## Updating MIGRATE.md

MIGRATE.md is the single source of truth. When updating it:

- All commands reference it — changes affect the entire plugin
- Add concrete code examples for new concepts
- Keep the phase structure (Phase 0-7) intact
- Update the "Guides Reference" table when adding/removing guides

## Style Guidelines

- **Guides teach methodology, NOT API lookups.** The agent already knows API mappings. Focus on decision frameworks, non-obvious pitfalls, and architectural patterns.
- Use markdown tables for decision criteria, paradigm comparisons, and library recommendations — not for exhaustive API mapping
- Include code examples that illustrate PATTERNS (abstraction layer, rewrite boundary), not API translations
- Include code examples in C/C++ (the most common migration target language)
- Add Python examples for scripting/testing patterns
- Keep guides focused — Claude supplements with its own knowledge
- Use consistent section headers across guides

## Testing Changes

1. Restart Claude Code to reload the plugin
2. Test with `/migrate-anything:analyze <path>` on a real project
3. Verify `/migrate-anything:map <from> <to>` shows correct mappings
4. Check that all command files load without errors

## Development Loop

Use `/migrate-dev` to iteratively develop the plugin. It reads the plan, checks progress, and implements the next batch of files.
