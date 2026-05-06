# Cross-Platform Testing Strategy Guide

Decision framework and pitfalls for testing migrated software. The agent already knows how to write unit tests — this guide focuses on testing strategy specific to cross-platform migration.

## Decision Framework

```
1. Does the project have existing tests?
   → YES: Continue to 2
   → NO: Create a minimal test suite (compile test + abstraction tests). Continue to 4.

2. Are existing tests portable (no platform-specific dependencies)?
   → YES: Run them on target platform. Fix any failures. Done.
   → PARTIAL: Continue to 3
   → ALL platform-specific: Treat as "no tests" — create new suite

3. For platform-specific existing tests:
   → Can adapt with #ifdef or skip conditions: Adapt and run
   → Test a platform feature that no longer exists: Skip with documented reason
   → Test behavior that differs on target: Write separate target-platform version

4. Create migration verification tests:
   → Compilation test (MUST pass first)
   → Platform abstraction layer tests
   → Filesystem/path handling tests (case sensitivity)
   → Integration test (does the app launch?)
```

## Testing Layers (Migration Context)

| Layer | Purpose | Migration-Specific Value |
|-------|---------|------------------------|
| **Compilation Test** | Does the migrated code compile? | Catches missing includes, wrong link libraries, type mismatches |
| **Abstraction Tests** | Does the platform layer work? | Validates the rewrite boundary — the most critical test for migrations |
| **Ported Tests** | Do existing tests pass on target? | Catches behavioral differences in "equivalent" APIs |
| **Integration Tests** | Do components work together? | Catches runtime issues (stack size, FD limits, encoding) |
| **Regression Tests** | Did migration break anything? | Ensures portable code wasn't accidentally modified |

## Non-Obvious Testing Pitfalls

### Tests That Pass on Source But Fail on Target

This is the most common migration testing problem. Causes:

- **Case-sensitive filesystem**: Test fixtures reference `TestData/config.json` but the file is `testdata/config.json` on disk. Works on macOS, fails on Linux.
- **Thread timing differences**: Tests with implicit timing assumptions (sleep 0.1s, then assert). Different scheduling behavior causes race conditions on the target.
- **Locale and encoding**: String sorting, number formatting, date parsing differ. Tests comparing output strings fail.
- **Floating point**: ARM vs x86 FPU rounding. Never assert exact floating-point equality across platforms.
- **Default signal handlers**: Tests that rely on SIGPIPE behavior (macOS default differs from Linux).
- **`sizeof(wchar_t)`**: 2 bytes on Windows, 4 bytes on Linux/macOS. Any test serializing wchar_t data will fail cross-platform.

### The "It Compiled, Therefore It Works" Trap

Compilation passing means syntax and linking are correct. It does NOT mean:
- Runtime behavior is correct (different default stack sizes, FD limits)
- File operations work (permissions, case sensitivity)
- Threading works (different scheduling, race conditions)
- Graphics render correctly (driver differences, coordinate systems)

**Always run the integration test suite after compilation.**

### Testing Platform Abstraction Layers

The platform abstraction layer is the most critical component to test. If it's wrong, everything built on it is wrong. Test the interface, not the implementation:

```cpp
// Test the abstraction interface — run on ALL platforms
void test_filesystem_abstraction() {
    // These should work identically on all platforms
    auto home = platform::get_home_directory();
    assert(!home.empty());
    assert(platform::path_is_absolute(home));

    auto joined = platform::path_join({home, "documents", "file.txt"});
    assert(joined.find("file.txt") != string::npos);
    assert(!platform::file_exists(joined));  // Doesn't exist yet

    platform::write_file(joined, "test content");
    assert(platform::file_exists(joined));
    assert(platform::read_file(joined) == "test content");
    platform::delete_file(joined);
}
```

**Run abstraction tests on BOTH source and target platforms** if possible. The same test should pass on both — that's how you know the abstraction is correct.

### The Compile-Test-Fix Loop

During migration, follow this loop for each dependency:

```
1. Migrate dependency X
2. Compile → fix compilation errors → compile again
3. Run abstraction tests for X → fix failures
4. Run full test suite → fix any regressions
5. Commit (working state)
6. Move to next dependency
```

**Never skip step 3.** A broken abstraction layer makes all subsequent work unreliable.

## CI Configuration for Cross-Platform Testing

GitHub Actions multi-platform CI:

```yaml
name: Cross-Platform Tests
on: [push, pull_request]
jobs:
  test:
    strategy:
      matrix:
        os: [ubuntu-latest, macos-latest, windows-latest]
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@v4
      - name: Install dependencies (Linux)
        if: runner.os == 'Linux'
        run: sudo apt-get install -y libgtk-4-dev
      - name: Build
        run: cmake -B build && cmake --build build --parallel
      - name: Test
        run: cd build && ctest --output-on-failure
```

**Set up CI on day one of the migration.** Don't wait until the end. Cross-platform failures are much cheaper to fix when caught immediately.

## Handling Existing Test Suites

### When the source project has tests:

1. **Identify portable tests** — tests that don't call platform-specific APIs
2. **Mark platform-specific tests** with skip conditions:
   ```python
   @pytest.mark.skipif(platform.system() != "Darwin",
                       reason="Requires macOS AppKit")
   def test_appkit_window_creation():
       ...
   ```
3. **Create target-platform equivalents** for skipped tests
4. **Run all portable tests** to verify no regressions

### When the source project has NO tests:

Create a minimal migration verification suite:
1. Compilation test — `cmake --build build` succeeds
2. Launch test — application starts without crash
3. Abstraction tests — platform wrappers return valid values
4. File I/O test — read/write with target platform paths

This takes minimal effort and catches the most common migration failures.

## Testing Without Target Platform Access

If developing on a different platform than the target:

1. **Use CI** — GitHub Actions, GitLab CI for multi-platform testing
2. **Use containers** — Docker for Linux testing from any OS
3. **Generate a verification script** for manual testing on the target:
   ```bash
   #!/bin/bash
   set -e
   echo "Building..."
   cmake -B build && cmake --build build
   echo "Running tests..."
   cd build && ctest --output-on-failure
   echo "Migration verification PASSED"
   ```

## Functional Testing by Project Type

After compilation and abstraction tests pass, verify the migrated code actually **behaves correctly**. The testing approach depends on what kind of project was migrated:

### CLI Tools (very testable)

CLI tools are the easiest to functionally test — run the binary, check stdout/stderr and exit codes.

```bash
#!/bin/bash
# Functional test for migrated CLI tool
TOOL="./build/migrated-tool"

# Test: help flag works
$TOOL --help > /dev/null 2>&1
assert_exit_code 0 "--help should succeed"

# Test: version output
VERSION=$($TOOL --version)
assert_contains "$VERSION" "1.0" "version output"

# Test: input → output
$TOOL process input.txt -o output.txt
assert_exit_code 0 "process should succeed"
assert_file_exists output.txt
assert_file_contains output.txt "expected content"

# Test: error handling
$TOOL nonexistent-file 2>/dev/null
assert_exit_code_not 0 "missing file should fail"
```

Generate these tests by:
1. Reading the CLI's `--help` output
2. Identifying the main commands/flags
3. Writing test cases for each with sample inputs
4. Verifying outputs

### Libraries (testable)

Write a test program that links against the migrated library and calls its public API:

```c
// test_library.c — compile and run on target platform
#include <migrated_lib.h>
#include <assert.h>

int main() {
    // Init
    assert(lib_init() == 0);

    // Core functionality
    result_t* r = lib_process("test input");
    assert(r != NULL);
    assert(r->status == STATUS_OK);

    // Cleanup
    lib_free_result(r);
    lib_cleanup();
    return 0;
}
```

### Daemons/Services (moderately testable)

Test start/stop, port binding, and basic operations:

```bash
# Start the migrated service
./build/migrated-service --port 9090 &
PID=$!
sleep 2

# Test: responds to requests
curl -s http://localhost:9090/health | grep -q "ok"
assert_exit_code 0 "health endpoint"

# Cleanup
kill $PID
```

### GUI Applications (limited automated testing)

Automated GUI testing is generally not practical. Instead:
- Test the non-UI core logic (file handling, data processing, configuration)
- Test that the application binary launches without crashing
- If the UI framework supports headless mode (Qt: `-platform offscreen`, GTK: `GDK_BACKEND=virtual`), use it

## Differential Testing via SSH (Optional)

If the user provides SSH access to a machine running the **source platform**, you can compare the migrated software's behavior against the original:

```
User provides: --ssh user@host
```

### How it works

1. **Detect project type** and generate a shared set of test inputs
2. **Execute on source platform** via SSH: `ssh user@host "original-tool input.txt"`
3. **Execute locally** on target: `./migrated-tool input.txt`
4. **Compare results**: exit code, stdout, stderr, output files

### Output normalization

Before comparing, normalize outputs to remove platform-expected differences:

```
Filter out:
- Absolute paths (/usr vs /usr/local, C:\ vs /)
- Timestamps (normalize to epoch or strip)
- Memory addresses (0x7fff... vs 0x7ff0...)
- Temporary file paths (/tmp/xyz vs /var/folders/...)
- Platform-specific line counts in debug output
- Locale-dependent formatting (decimal separators)
```

### Example differential test

```bash
#!/bin/bash
SSH="user@source-host"
LOCAL="./build/migrated-tool"
REMOTE="original-tool"
INPUT="test_input.txt"

# Sync test input to source
scp "$INPUT" "$SSH:/tmp/migrate-test-input.txt"

# Run on both platforms
LOCAL_OUT=$(mktemp)
REMOTE_OUT=$(mktemp)

$LOCAL $INPUT > "$LOCAL_OUT" 2>&1
LOCAL_EXIT=$?

ssh "$SSH" "$REMOTE /tmp/migrate-test-input.txt" > "$REMOTE_OUT" 2>&1
REMOTE_EXIT=$?

# Normalize both outputs
normalize() {
    sed -e 's|/tmp/.\{1,20\}|[TEMP]|g' \
        -e 's|/var/folders/.\{1,30\}|[TEMP]|g' \
        -e 's|0x[0-9a-f]\{4,\}|[ADDR]|g' \
        -e 's|[0-9]\{4\}-[0-9]\{2\}-[0-9]\{2\}[T ][0-9:]\{8,\}|[TIMESTAMP]|g'
}

LOCAL_NORM=$(normalize < "$LOCAL_OUT")
REMOTE_NORM=$(normalize < "$REMOTE_OUT")

# Compare
if [ "$LOCAL_NORM" = "$REMOTE_NORM" ] && [ "$LOCAL_EXIT" = "$REMOTE_EXIT" ]; then
    echo "PASS: output matches source platform"
else
    echo "FAIL: output differs"
    diff <(echo "$REMOTE_NORM") <(echo "$LOCAL_NORM")
fi
```

### SSH requirements

- User must set up **passwordless SSH** (key-based auth) to the source machine
- Source machine must have the **original software installed and working**
- Test inputs must be available on both sides (agent syncs via `scp`)
- The agent should verify SSH connectivity before starting tests: `ssh -o ConnectTimeout=5 user@host "echo ok"`

## Key Testing Principles

1. **Test incrementally** — After each dependency migration, compile + test
2. **Never skip compilation** — If it doesn't compile, nothing else matters
3. **Test the abstraction, not the platform** — Focus on the interface layer
4. **Run on both platforms** — Same test, both source and target, same results
5. **Document known differences** — Some behavior legitimately differs; document it
6. **Fail fast** — Set up CI immediately, don't accumulate failures
