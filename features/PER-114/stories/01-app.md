# Story — Return `(missing, missing)` from `sincospi(missing)`

**ID:** PER-114  
**Type:** Bug Fix  
**Track:** Quick  
**Epic:** (standalone backlog item)  
**Covers:** N/A  
**Status:** Draft  
**Assignee:** (unassigned)  

---

## Problem Statement

`sincospi(missing)` currently returns a single `missing` value, which causes a `MethodError` when destructured (`s, c = sincospi(missing)` fails with "no method matching iterate(::Missing)"). This behavior is inconsistent with the function's documented contract (returns a `(sine, cosine)` tuple) and with the analogous functions `sincos(missing)` and `sincosd(missing)`, both of which correctly return `(missing, missing)`.

The root cause is that `sincospi` is in the shared list of functions in `base/math.jl` (around line 1325) that generates `Missing` methods as `@eval $(f)(::Missing) = missing`, producing a single-value return. `sincos` and `sincosd` are not in that list, so they preserve the two-element tuple shape for `Missing` inputs.

---

## Expected Behavior

- `sincospi(missing)` returns `(missing, missing)`, and destructuring it succeeds with both values `missing`.
- `sincos(missing)` and `sincosd(missing)` continue to return `(missing, missing)` (no change).
- Single-value `Missing` methods (`sinpi`, `cospi`, `tanpi`) continue to return `missing` (no change).
- Numeric `sincospi` results for non-`Missing` inputs remain unchanged.

---

## Scope

**Repository:** `Julia` only.

**In scope:**
- Adjust the `Missing` method for `sincospi` in `base/math.jl` (around line 1325) so it returns `(missing, missing)` instead of `missing`.
- Add a regression test in `test/math.jl` covering the destructuring case.

**Out of scope:**
- Changes to `sincos(missing)` or `sincosd(missing)` (already correct).
- Changes to `sinpi`, `cospi`, `tanpi`, or any other math function's `Missing` method.
- New test files.
- Changes under `src/`, `Compiler/`, or `deps/`.

---

## Given/When/Then Acceptance Criteria

### AC-1: `sincospi(missing)` returns a two-element tuple

**Given** the Julia REPL is started with `julia --startup-file=no`  
**When** I evaluate `sincospi(missing)`  
**Then** the result is `(missing, missing)`

**And when** I evaluate `s, c = sincospi(missing)`  
**Then** the destructuring succeeds  
**And** `s` is `missing`  
**And** `c` is `missing`

---

### AC-2: `sincos(missing)` and `sincosd(missing)` remain unchanged

**Given** the Julia REPL is started with `julia --startup-file=no`  
**When** I evaluate `sincos(missing)`  
**Then** the result is `(missing, missing)`

**And when** I evaluate `sincosd(missing)`  
**Then** the result is `(missing, missing)`

---

### AC-3: Single-value `Missing` methods remain unchanged

**Given** the Julia REPL is started with `julia --startup-file=no`  
**When** I evaluate `sinpi(missing)`  
**Then** the result is `missing` (not a tuple)

**And when** I evaluate `cospi(missing)`  
**Then** the result is `missing` (not a tuple)

**And when** I evaluate `tanpi(missing)`  
**Then** the result is `missing` (not a tuple)

---

### AC-4: Math test suite passes

**Given** the repository is in a clean state  
**When** I run `julia --startup-file=no test/runtests.jl math`  
**Then** all tests pass with exit code 0

---

### AC-5: Whitespace check passes

**Given** the repository is in a clean state  
**When** I run `julia --startup-file=no contrib/check-whitespace.jl`  
**Then** the check passes with exit code 0

---

## Technical Specification

### File Locations

- **Implementation:** `base/math.jl` (around line 1325)
  - The `sincospi(::Missing)` method is currently generated from a shared list that produces single-value `missing` returns.
  - The loop ends with `@eval $(f)(::Missing) = missing`.
  - `:sincospi` is in that list; `:sincos` and `:sincosd` are not.
  - **Change:** Remove `:sincospi` from the shared list and add an explicit method definition: `sincospi(::Missing) = (missing, missing)`.

- **Test:** `test/math.jl` (existing file)
  - Add a regression test covering `sincospi(missing)` returning `(missing, missing)` and successful destructuring.

### Implementation Approach

1. Locate the shared `Missing` method generation loop in `base/math.jl` (around line 1325).
2. Remove `:sincospi` from the list of functions that generate `@eval $(f)(::Missing) = missing`.
3. Add an explicit method definition immediately after the loop or in the appropriate location:
   ```julia
   sincospi(::Missing) = (missing, missing)
   ```
4. In `test/math.jl`, add a test case:
   ```julia
   @test sincospi(missing) === (missing, missing)
   @test (s, c = sincospi(missing); s === missing && c === missing)
   ```
5. Run `julia --startup-file=no test/runtests.jl math` to verify the fix.
6. Run `julia --startup-file=no contrib/check-whitespace.jl` to verify code style.

### Verification Approach

- **Manual REPL check:** Evaluate `sincospi(missing)` and `s, c = sincospi(missing)` to confirm the result is `(missing, missing)` and destructuring succeeds.
- **Regression test:** The new test case in `test/math.jl` will catch future regressions.
- **Existing behavior preservation:** Verify `sincos(missing)`, `sincosd(missing)`, `sinpi(missing)`, `cospi(missing)`, and `tanpi(missing)` still return their current values by running the full math test suite.

---

## Open Questions

None.

---

## Notes

- The `sincospi` docstring states the function returns a `(sine, cosine)` tuple. The current `Missing` method violates this contract.
- This is a single-method fix with no impact on other `Missing` methods or numeric results.
- The fix aligns `sincospi(missing)` with the behavior of `sincos(missing)` and `sincosd(missing)`.
