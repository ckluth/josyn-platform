# `.local-build` — Local Developer Tooling

`.local-build` is a set of thin `.cmd` wrapper scripts that give a **maintainer or platform
developer** fast, opinionated entry points into common build tasks — directly from a terminal,
without IDE involvement.

---

## Purpose and Audience

These scripts exist because during active platform development the same sequence of commands
is typed repeatedly: build in the right configuration, clear the cached NuGet package, pack,
run tests. `.local-build` codifies those sequences once per solution (and once per repo for
multi-solution repos) so they can be invoked with a single call.

**Primary audience:** maintainers and platform developers working locally.

`.local-build` scripts are **not for automated pipelines**. CI/CD workflows define their own
build steps and must never delegate to these scripts. The assumptions baked in here — local
NuGet cache paths, output directories under `C:\Temp`, hardcoded pipe arguments in launch
scripts — are developer-machine assumptions that have no place in a pipeline.

---

## Non-Scope

| What `.local-build` is NOT | What owns it instead |
|---|---|
| A CI/CD build stage | The pipeline definition in the CI system |
| An SDK or toolchain installer | Machine setup / provisioning scripts |
| A NuGet feed publisher | The CI release workflow |
| An environment configuration tool | Outside scope of this platform |
| A deployment or launch orchestrator | Production runtime tooling |

---

## Two Characters

`.local-build` appears at two structurally distinct levels and the two have different
characters. Understanding which character a script set has tells you immediately what it can
and cannot do.

### Single-Target

Operates on **exactly one solution** (`.slnx`). The scripts locate their solution one level
up from `%~dp0` and act on it exclusively. They know nothing about other solutions in the repo.

This character is used:
- in **Pattern A repos** — the repo-root `.local-build/` is the only one; it is single-target
- in **sub-solution folders inside Pattern B repos** — each sub-solution has its own

**Typical script surface:**

| Script | Action |
|--------|--------|
| `build.cmd [Release\|Debug]` | Builds the solution; defaults to Release |
| `build.release.cmd` | Shorthand — calls `build.cmd Release` |
| `build.debug.cmd` | Shorthand — calls `build.cmd Debug` |
| `test.cmd` | Runs all tests in the solution |
| `pack.cmd` | Packs the NuGet package to the local feed |
| `clean.cmd` | Removes this package's entry from the local NuGet cache |
| `all.cmd` | Chains: clean → build → test → pack |
| `launch.release.cmd` | Launches the executable with release output (executables only) |
| `launch.debug.cmd` | Launches the executable with debug output (executables only) |

Not every script is present in every `.local-build/`. Include only what applies to that
solution: a library has no `launch.*.cmd`; a solution without tests omits `test.cmd`.

**`clean.cmd` note:** at the single-target level, clean does *not* run `dotnet clean` on
`bin/obj`. It removes the package from the local NuGet cache (`%USERPROFILE%\.nuget\packages\`).
This is the developer-specific operation: it forces downstream projects to resolve a fresh
copy of the package on the next build, which is the common need when iterating on a library
that is consumed locally.

**`launch.*.cmd` note:** launch scripts may contain hardcoded local paths (output directories,
pipe names). This is acceptable — they are intentionally per-developer-machine scripts. Do not
attempt to generalise them.

---

### Batch / Orchestrator

Operates on **all sub-solutions in a Pattern B repo**, in dependency order. It contains no
build logic of its own — it delegates entirely to each sub-solution's single-target
`.local-build/`. The dependency order is encoded directly in the script.

This character is used **exclusively at the repo root of Pattern B repos**.

**Typical script surface:**

| Script | Action |
|--------|--------|
| `build.cmd [Release\|Debug]` | Calls `build.cmd` in each sub-solution in dependency order |
| `test.cmd` | Calls `test.cmd` in each sub-solution in dependency order |
| `pack.cmd` | Calls `pack.cmd` in each sub-solution |
| `clean.cmd` | Calls `clean.cmd` in each sub-solution |
| `all.cmd` | Chains: clean → build → test → pack across all sub-solutions |

`all.cmd` is the top-level **"build the whole repo"** entry point. It only exists at the
batch level.

---

## Assignment by Pattern

| Repo pattern | `.local-build` level | Character |
|---|---|---|
| **Pattern A** | Repo root | Single-target |
| **Pattern B** | Repo root | Batch / orchestrator |
| **Pattern B** | Sub-solution folder | Single-target |

A Pattern B sub-solution may omit its own `.local-build/` only if it has no scripts at all
(e.g., a sub-folder that is documentation only). For any sub-solution that is built, tested,
or packed, the single-target `.local-build/` is required — the batch-level scripts delegate
to it and will fail without it.

---

## Script Rules

- All scripts start with `@echo off` and `CHCP 1252`.
- `build.cmd` accepts an optional `[Release|Debug]` argument and defaults to `Release`.
- Errors propagate via `exit /b %ERRORLEVEL%`. No silent failures.
- Scripts do not `pause` in normal operation. Pauses are either removed or commented out.
- All paths are derived from `%~dp0`. No hardcoded absolute paths except in `launch.*.cmd`
  (see above).
- The `all.cmd` script at either level does **not** accept a configuration argument. It
  always builds Release. Debug builds are invoked explicitly via `build.debug.cmd`.
