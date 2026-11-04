# Requirements — PER-116

**Story:** Add `--threads=1` to lld on Linux CI to avoid image-link hangs

---

## Background

LLD 23 occasionally hangs forever while linking images on Linux CI machines with an old system glibc, which deadlocks CI runs (JuliaLang/julia#64496). The issue attributes this to LLVM's parallel linking change (llvm/llvm-project#187989) combined with glibc bug 25847. The workaround is to pass `--threads=1` to lld on CI, detected by the `CI` environment variable.

All image links go through `Base.Linking.ld()` in `base/linking.jl`. Its final `else` branch sets the arguments for the `gnu` flavor on non-Windows, non-Apple systems. This story changes that branch to include `--threads=1` when running on Linux with `CI=true`, and adds a regression test to `test/misc.jl`.

---

## Acceptance Criteria

### AC-1: Single-threaded lld on Linux CI

**Given** the system is Linux  
**And** the environment variable `CI` equals `"true"`  
**When** `Base.Linking.ld()` is called  
**Then** the returned command's `.exec` array includes `"--threads=1"`

---

### AC-2: Multi-threaded lld when CI is unset or false

**Given** the system is Linux  
**And** the environment variable `CI` is unset or equals any value other than `"true"`  
**When** `Base.Linking.ld()` is called  
**Then** the returned command's `.exec` array does not include `"--threads=1"`

---

### AC-3: Other arguments unchanged

**Given** the system is Linux  
**And** `Base.Linking.ld()` is called with `strict_definitions` set to `true`  
**When** the command is built  
**Then** all other arguments (e.g., `--no-undefined`) are present and unchanged  

**And Given** `strict_definitions` is `false`  
**Then** the arguments for that mode are also unchanged

---

### AC-4: Non-Linux systems unaffected

**Given** the system is macOS, Windows, or any non-Linux platform  
**And** the environment variable `CI` equals `"true"`  
**When** `Base.Linking.ld()` is called  
**Then** the returned command does not include `"--threads=1"`

---

### AC-5: CI read at call time

**Given** the implementation of the conditional logic  
**When** the code is reviewed  
**Then** the `CI` environment variable is read inside the `ld()` function body, not in a `const` or `@static` block  
**And** the value is not fixed at system-image build time

---

### AC-6: Regression test in `test/misc.jl`

**Given** a test case in `test/misc.jl`  
**When** the test is run with `withenv("CI" => "true")` on Linux  
**Then** the test asserts that `Base.Linking.ld().exec` includes `"--threads=1"`  

**And When** the test is run with `withenv("CI" => nothing)` on Linux  
**Then** the test asserts that `Base.Linking.ld().exec` does not include `"--threads=1"`  

**And When** the test is run with `withenv("CI" => "false")` on Linux  
**Then** the test asserts that `Base.Linking.ld().exec` does not include `"--threads=1"`  

**And** the test only asserts the flag presence on Linux (no assertions on macOS or Windows)

---

### AC-7: Image linking still works

**Given** a Julia build with this change  
**And** the environment variable `CI` equals `"true"`  
**When** `Base.Linking.link_image` is called  
**Then** the image is produced successfully and is valid

---

### AC-8: Test suite passes

**Given** a Julia built from the modified branch in the Dev worktree  
**When** `./julia --startup-file=no test/runtests.jl misc` is run  
**Then** all tests pass

---

### AC-9: Whitespace check passes

**Given** the modified source files  
**When** `make check-whitespace` is run  
**Then** the check passes with no errors

---

## Technical Notes

- **File to modify:** `base/linking.jl` — the `ld()` function's `else` branch (the `gnu` flavor on Linux).
- **Test file:** `test/misc.jl` — add one new test case using `withenv` to cover AC-6.
- **Read `CI` at call time:** Use `get(ENV, "CI", "")` or similar inside the function, not a top-level constant.
- **Platform detection:** Use `Sys.islinux()` to guard the conditional.
- **No new files, APIs, or environment variables:** Only modify existing code and tests.
- **Out of scope:** Changes to `src/`, `Compiler/`, `deps/`, LLD version, or the CI system libc.

---

## Verification Approach

1. Run the test suite with `CI=true` and `CI` unset on a Dev worktree Julia built from the branch.
2. Inspect `Base.Linking.ld().exec` in a REPL with `withenv` to confirm the flag presence.
3. Trigger an actual image link with `CI=true` and confirm the image is valid.
4. Run `make check-whitespace` on the modified files.
