---
description: Run tests for a migrated project and update MIGRATION-REPORT.md with results
subtask: true
---
# migrate-anything:test Command

Run all tests for a migrated project and update the migration documentation with results.

**Target source code**: $1
**SSH differential testing**: $2

## CRITICAL: Read MIGRATE.md First

**Before running tests, read `MIGRATE.md` (located in the plugin root, one directory up from this command).** It defines the testing standards, expected structure, and what constitutes a passing test suite.

## Arguments

- `$1` is the **source code path** (required). Local path to the migrated software source code.

- `$2` is an **SSH target for differential testing** (optional). Format: `user@host`. When provided, the agent runs the same test inputs on both the source platform (via SSH) and the target platform (locally), then compares outputs. Requires passwordless SSH key-based auth.

## What This Command Does

1. **Locate the project** — Verify the path exists and contains source code
2. **Detect project type** — CLI tool, library, daemon, or GUI application
3. **Attempt build** — Compile the migrated project on the current platform
4. **Run existing tests** — Execute any test suite found in the project
5. **Generate functional tests** — Based on project type (if applicable)
6. **Differential testing** (if `--ssh` provided) — Compare against source platform behavior
7. **Scan for remaining platform-specific code** — Grep for unmigrated API calls
8. **Update documentation** — Append test results to `MIGRATION-REPORT.md`

## Project Type Detection

Scan the project to determine its type:

| Indicator | Type | Functional Testing |
|-----------|------|-------------------|
| Has `main()` + parses arguments (argv, argparse, clap) | CLI tool | Generate subprocess tests |
| Exports symbols, has `.so`/`.dylib`/`.a` targets | Library | Generate API test program |
| Has `fork()`, signal handlers, socket bind | Daemon/service | Generate start/stop tests |
| Links GUI framework (GTK, Qt, AppKit) | GUI application | Limited: launch test only |
| Multiple of the above | Combined | Test each applicable layer |

## Functional Test Generation

After existing tests run, generate functional tests based on project type:

### For CLI tools:
1. Build the migrated binary
2. Run `./binary --help` to discover commands and flags
3. Generate test script that exercises each command:
   - `./binary --help` → exit code 0
   - `./binary --version` → output contains version string
   - `./binary process sample-input.txt` → output file exists and is valid
   - `./binary invalid-input` → non-zero exit code + error message
4. Execute the test script and record results

### For libraries:
1. Write a test program that calls the library's public API
2. Compile and link against the migrated library
3. Run the test program and verify assertions
4. Record results

### For daemons:
1. Start the service on a test port
2. Send requests and verify responses
3. Stop the service and verify clean shutdown

### For GUI apps:
1. Verify the binary launches without crash (use `--help` or headless mode)
2. If the framework supports headless (Qt: `-platform offscreen`, GTK: virtual backend), use it
3. Skip automated UI interaction testing

## SSH Differential Testing

When `$2` (SSH target) is provided:

1. **Verify connectivity**: `ssh -o ConnectTimeout=5 $2 "echo ok"`
2. **Detect source project type** on remote: identify the original binary/library
3. **Generate shared test inputs** that both platforms can process
4. **Sync inputs to source**: `scp test_inputs/* $2:/tmp/migrate-test/`
5. **Execute on both platforms** with the same inputs
6. **Normalize outputs** (strip paths, timestamps, memory addresses)
7. **Compare** normalized outputs and exit codes
8. **Report differences** as diffs with context

See `guides/testing-strategy.md` for normalization rules and example scripts.

## Test Discovery (existing tests)

The command also looks for existing tests:

1. **Migration verification tests** — `tests/migration/` directory
2. **Existing project tests** — Whatever test framework the project uses
3. **Compilation test** — If nothing else exists

### Test framework detection:
- **CMake/CTest**: `cmake --build build && cd build && ctest --output-on-failure`
- **pytest**: `python -m pytest -v --tb=short`
- **Make**: `make test` or `make check`
- **Cargo**: `cargo test`
- **Go**: `go test ./...`

## Output Format

Appended to MIGRATION-REPORT.md:

```markdown
## Test Results

Last run: 2026-05-07 14:30:00

### Build
Status: SUCCESS

### Project Type: CLI tool
Functional tests: 8 passed, 0 failed

### Existing Tests
12 passed, 0 failed

### Differential Testing (SSH: user@source-host)
Compared 5 test cases against source platform
Matches: 5/5 (100%)

### Remaining Platform-Specific Code
| Pattern | Count | Files |
...

**Summary**: 20 passed, 0 failed, coverage 90%
```

## Example

```bash
# Standard test run
/migrate-anything:test /home/user/ghostty-migrated

# With SSH differential testing against source platform
/migrate-anything:test /home/user/ghostty-migrated user@macos-source
```

## Success Criteria

- Build succeeds on the target platform
- All existing portable tests pass
- Functional tests pass (if project type supports them)
- Differential tests match (if SSH provided)
- MIGRATION-REPORT.md is updated with results

## Failure Handling

If tests fail:
1. Show which tests failed with error details
2. For differential failures: show the diff between source and target output
3. Suggest running `/migrate-anything:refine` to address the failures
4. Identify the likely cause (missing include, wrong library, behavioral difference)
