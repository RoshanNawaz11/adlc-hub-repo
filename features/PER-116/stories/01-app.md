# Story — PER-116

**Add `--threads=1` to lld on Linux CI to avoid image-link hangs**

---

## Story Description

LLD 23 occasionally hangs forever while linking images on Linux CI machines with an old system glibc, which deadlocks CI runs (JuliaLang/julia#64496). The issue attributes this to LLVM's parallel linking change (llvm/llvm-project#187989) combined with glibc bug 25847. The workaround is to pass `--threads=1` to lld when running on CI, detected by the `CI` environment variable equaling `"true"`.

All image links go through `Base.Linking.ld()` in `base/linking.jl`. Its final `else` branch sets the arguments for the `gnu` flavor on non-Windows, non-Apple systems. This story modifies that branch to include `--threads=1` when running on Linux with `CI=true`, and adds a regression test to `test/misc.jl`.

**Constraints:**

- `CI` must be read **at call time** inside `ld()`, not at build time via `const` or `@static`.
- Change only `base/linking.jl` and `test/misc.jl` — no new files, APIs, or environment variables.
- Out of scope: `src/`, `Compiler/`, `deps/`, LLD version, CI system libc changes.

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

## Technical Specification

### Files to Modify

1. **`base/linking.jl`** — the `ld()` function
2. **`test/misc.jl`** — regression test

### Implementation Details

#### 1. Modify `base/linking.jl` — `ld()` function

**Location:** Final `else` branch handling the `gnu` flavor on non-Windows, non-Apple systems (Linux).

**Change:** Add conditional logic to include `--threads=1` when:
- Platform is Linux (`Sys.islinux()` returns `true`)
- Environment variable `CI` equals `"true"` (checked at call time)

**Key constraint:** Read `CI` using `get(ENV, "CI", "")` or similar **inside the function body**, never in a `const` or `@static` block. The value must be determined at link time, not at system-image build time.

**Example pseudocode:**

```julia
function ld(...)
    # ... existing code ...
    else  # gnu flavor on Linux
        args = [...]
        
        # Add --threads=1 on Linux CI to avoid hang with old glibc
        if Sys.islinux() && get(ENV, "CI", "") == "true"
            push!(args, "--threads=1")
        end
        
        # ... rest of existing logic ...
    end
end
```

**Preserve:** All existing arguments and logic for both `strict_definitions=true` and `strict_definitions=false` cases.

---

#### 2. Add regression test to `test/misc.jl`

**Purpose:** Verify the `--threads=1` flag is correctly added on Linux CI and absent otherwise.

**Approach:** Use `withenv` to temporarily set `CI`, call `Base.Linking.ld()`, and inspect the `.exec` array.

**Test cases (only on Linux):**

1. **CI=true** → assert `"--threads=1"` is present
2. **CI unset** → assert `"--threads=1"` is absent
3. **CI=false** → assert `"--threads=1"` is absent

**Platform guard:** Wrap assertions in `if Sys.islinux()` to avoid failures on macOS/Windows.

**Example pseudocode:**

```julia
@testset "lld --threads=1 on Linux CI" begin
    if Sys.islinux()
        # Test CI=true case
        withenv("CI" => "true") do
            cmd = Base.Linking.ld(...)
            @test "--threads=1" in cmd.exec
        end
        
        # Test CI unset case
        withenv("CI" => nothing) do
            cmd = Base.Linking.ld(...)
            @test !("--threads=1" in cmd.exec)
        end
        
        # Test CI=false case
        withenv("CI" => "false") do
            cmd = Base.Linking.ld(...)
            @test !("--threads=1" in cmd.exec)
        end
    end
end
```

**Note:** The test must call `Base.Linking.ld()` with valid arguments to produce a command object. Consult existing tests in `test/misc.jl` or `base/linking.jl` for the required signature.

---

### Edge Cases and Clarifications

1. **Call-time vs build-time:** The `CI` check must happen **every time `ld()` is called**, not once when the system image is compiled. This ensures the flag can be toggled at runtime.

2. **Non-Linux platforms:** The conditional must short-circuit on `Sys.islinux()` before checking `CI`, so macOS and Windows are never affected.

3. **CI values other than "true":** Any value except the literal string `"true"` (case-sensitive) is treated as false. This includes `"1"`, `"TRUE"`, `"yes"`, etc.

4. **Existing arguments:** The order and presence of all other lld arguments must remain unchanged. The new flag can be appended or inserted at any position that does not break existing logic.

5. **Test environment:** The Dev worktree has no prebuilt Julia. Tests must run against a Julia built from the modified branch in that worktree.

---

## Verification

1. **Unit test:** Run `./julia --startup-file=no test/runtests.jl misc` on a Julia built from the branch. Verify the new test passes.

2. **Manual REPL check:** Start the built Julia and run:
   ```julia
   withenv("CI" => "true") do
       Base.Linking.ld(...).exec
   end
   ```
   Confirm `"--threads=1"` appears in the output on Linux, and does not appear with `CI` unset.

3. **Image linking:** Trigger an actual image link with `CI=true` and confirm the produced image is valid.

4. **Whitespace:** Run `make check-whitespace` and confirm it passes.

---

## Tests

| # | Scenario | Expected Outcome | Verification Method |
|---|----------|------------------|---------------------|
| 1 | Linux + CI=true | `"--threads=1"` in `Base.Linking.ld().exec` | Test in `test/misc.jl` |
| 2 | Linux + CI unset | `"--threads=1"` absent | Test in `test/misc.jl` |
| 3 | Linux + CI=false | `"--threads=1"` absent | Test in `test/misc.jl` |
| 4 | macOS/Windows + CI=true | `"--threads=1"` absent | Test in `test/misc.jl` (skipped on non-Linux) |
| 5 | All other arguments preserved with strict_definitions=true | Existing args unchanged | Test in `test/misc.jl` |
| 6 | All other arguments preserved with strict_definitions=false | Existing args unchanged | Test in `test/misc.jl` |
| 7 | Image linking works with CI=true | Valid image produced | Manual verification + existing test suite |
| 8 | Test suite passes | All tests pass | `./julia test/runtests.jl misc` |
| 9 | Whitespace check passes | No errors | `make check-whitespace` |

---

## Notes

- **Root cause:** LLVM parallel linking (llvm/llvm-project#187989) + glibc bug 25847 on old system libc.
- **Upstream references:** JuliaLang/julia#64496, glibc bug 25847.
- **LLD version:** 23.1.4 (unchanged by this story).
- **Out of scope:** Proving the hang is gone (intermittent, needs old-glibc CI machine), upgrading CI system libc, changing behavior for non-CI users.
