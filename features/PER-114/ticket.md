---
key: PER-114
type: story
status: todo
blocks: []
blocked_by: []
labels: []
repos: []
summary: Return `(missing, missing)` from `sincospi(missing)`
---

**Problem.** `sincospi(missing)` returns a single `missing`, so `s, c = sincospi(missing)` raises `MethodError: no method matching iterate(::Missing)`. `sincos(missing)` and `sincosd(missing)` already return `(missing, missing)`. In `base/math.jl`, `Missing` methods are generated from one shared list, and `:sincospi` is in that list (the loop that ends with `@eval $(f)(::Missing) = missing`, currently around line 1325). That method returns one `missing`. `sincos` and `sincosd` are not in the list, so they keep the two-value shape. The `sincospi` docstring says the function returns a `(sine, cosine)` tuple.

**Expected behavior.** `sincospi(::Missing)` returns `(missing, missing)`, and destructuring it succeeds. `sincos(missing)` and `sincosd(missing)` still return `(missing, missing)`. `sinpi(missing)`, `cospi(missing)`, and `tanpi(missing)` still return `missing`. Numeric `sincospi` results are unchanged.

**Scope.** Repo `Julia` only. Adjust the `Missing` method for `sincospi` in `base/math.jl` so it is not the single-value method from that shared list. Add a regression test in the existing `test/math.jl`. Do not add a new test file, and do not change the `Missing` result of any other math function. No changes under `src/`, `Compiler/`, or `deps/`.

**Acceptance criteria.**

* `sincospi(missing)` returns `(missing, missing)`, and `s, c = sincospi(missing)` succeeds with both values `missing`.
* `sincos(missing)` and `sincosd(missing)` still return `(missing, missing)`.
* `sinpi(missing)`, `cospi(missing)`, and `tanpi(missing)` still return `missing`.
* `julia --startup-file=no test/runtests.jl math` passes.
* `julia --startup-file=no contrib/check-whitespace.jl` passes.
