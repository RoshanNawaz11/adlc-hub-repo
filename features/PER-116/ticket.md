---
key: PER-116
type: story
status: todo
blocks: []
blocked_by: []
labels:
- ci
repos: []
summary: Run lld single-threaded on Linux CI to avoid image-link hangs
---

**Problem.** LLD 23 occasionally hangs forever while linking images on Linux CI machines with an old system glibc, which deadlocks CI runs (JuliaLang/julia#64496). The issue attributes this to LLVM's parallel linking change (llvm/llvm-project#187989) combined with glibc bug 25847. It is more likely on aarch64 but can also happen on x86_64. The issue suggests `--threads=1` on CI or a newer system libc. The problem is in the system libc, not the glibc Julia links against. This repo uses LLD 23.1.4.

**Cause location.** All image links go through `Base.Linking.ld()` in `base/linking.jl` (`link_image_cmd` calls `ld`). Its final `else` branch sets the arguments for the `gnu` flavor on non-Windows, non-Apple systems. Nothing there limits lld's threads today.

**Expected behavior.**

* On Linux, when the environment variable `CI` equals `"true"`, `ld()` includes `--threads=1` in the lld command.
* When `CI` is unset or has any other value, the lld command is unchanged.
* macOS, Windows and other non-Linux systems are unchanged, even when `CI=true`.

**Scope.** Repo `Julia` only. Change `ld()` in `base/linking.jl` and add a regression test to the existing `test/misc.jl`. Do not add a new test file, a new environment variable, or a new public API. Do not change `src/`, `Compiler/`, `deps/`, `.buildkite-external-version` or the LLD version.

**Implementation constraint.** `CI` must be read when `ld()` is called. Do not read it in a `const` or inside `@static`, because the value would be fixed when the system image is built, not when the link runs.

**Acceptance criteria.**

* On Linux with `CI=true`, `"--threads=1"` is in `Base.Linking.ld().exec`.
* On Linux with `CI` unset, and with `CI=false`, `"--threads=1"` is not in `Base.Linking.ld().exec`.
* All other arguments from `ld()` are unchanged, with `strict_definitions` both true and false.
* The new test covers the cases above using `withenv`, and only asserts the flag on Linux.
* `Base.Linking.link_image` still produces a working image with `CI=true`.
* `./julia --startup-file=no test/runtests.jl misc` passes on a Julia built from this branch.
* `make check-whitespace` passes.

**Test environment.** The Dev worktree has no built Julia, and an installed Julia does not include changes to `base/`. Do not report the acceptance tests as passing from an installed Julia.

**Out of scope.** Proving the hang is gone (it is intermittent and needs an old-glibc CI machine), upgrading the CI system libc, and changing behavior for non-CI users on old glibc.
