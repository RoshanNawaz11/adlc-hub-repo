# Requirements — Return `(missing, missing)` from `sincospi(missing)`

## Given/When/Then Acceptance Criteria

### AC-1: `sincospi(missing)` returns a two-element tuple

**Given** the Julia REPL is started with no startup file  
**When** I evaluate `sincospi(missing)`  
**Then** the result is `(missing, missing)`

**And when** I evaluate `s, c = sincospi(missing)`  
**Then** the destructuring succeeds  
**And** `s` is `missing`  
**And** `c` is `missing`

---

### AC-2: `sincos(missing)` and `sincosd(missing)` remain unchanged

**Given** the Julia REPL is started with no startup file  
**When** I evaluate `sincos(missing)`  
**Then** the result is `(missing, missing)`

**And when** I evaluate `sincosd(missing)`  
**Then** the result is `(missing, missing)`

---

### AC-3: Single-value `Missing` methods remain unchanged

**Given** the Julia REPL is started with no startup file  
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

## Technical Notes

### File Locations

- **Implementation**: `base/math.jl` around line 1325
  - The `sincospi(::Missing)` method is currently generated from a shared list that produces single-value `missing` returns
  - The loop ends with `@eval $(f)(::Missing) = missing`
  - `:sincospi` is in that list; `:sincos` and `:sincosd` are not
- **Test**: `test/math.jl` (existing file; add regression test here)

### Scope Boundaries

**In scope:**
- Adjust the `Missing` method for `sincospi` in `base/math.jl` so it returns `(missing, missing)` instead of `missing`
- Add a regression test in `test/math.jl` covering the destructuring case

**Out of scope:**
- Changes to `sincos(missing)` or `sincosd(missing)` — already correct
- Changes to `sinpi(missing)`, `cospi(missing)`, `tanpi(missing)`, or any other math function's `Missing` method
- New test files
- Changes under `src/`, `Compiler/`, or `deps/`
- Numeric `sincospi` results for non-`Missing` inputs

### Implementation Constraint

The `sincospi` docstring states the function returns a `(sine, cosine)` tuple. The current `Missing` method violates this contract by returning a single `missing` value instead of a two-element tuple.
