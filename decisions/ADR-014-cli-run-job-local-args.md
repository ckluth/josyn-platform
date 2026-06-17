# ADR-014 — CLI `run-job` Command and Local Arguments Convention

**Date:** 2026-06-06  
**Status:** Accepted

---

## Context

`JOSYN.Backend.CLI` is the operator-facing executable for manually triggering job sessions.
Its current implementation (`Program.cs`) has hardcoded demo values and a single static
function `HardCodedDemoSessionStart()`. This is a PoC placeholder — the comment in the file
reads: *"replace with real inputs once CLI arg parsing is added"*.

The first production use case for the CLI is:

> An operator or maintainer wants to start a specific job with specific arguments — for
> integration testing, ad-hoc maintenance, or operator-triggered execution outside the normal
> scheduler flow — without modifying or recompiling the binary.

This requires two things:

1. A command-line argument processor in the CLI.
2. A convention for storing reusable, named argument files alongside deployed job executables,
   with optional convenience launchers.

---

## Decision

### The `run-job` Command

The CLI's first production command (exact syntax is an implementation detail; the contract is):

```
JOSYN.Backend.CLI.exe <run-job> <jobname> [<argument-file>]
```

| Token | Required | Description |
|-------|----------|-------------|
| `<run-job>` | yes | Sub-command selector; exact form decided at implementation |
| `<jobname>` | yes | Registered job type name, as stored in `josyn.JobRegistrations` (e.g. `Contoso.DemoProduct.DemoJob`) |
| `<argument-file>` | no | Path to a `.ini` file in `PropertyBag` INI format; omitting it is valid only for zero-parameter jobs |

**Dispatch:** the CLI reads `<argument-file>` into a raw string and calls
`ISessionStarter.StartSession(jobName, rawArguments)`. The job registry lookup (`IJobRegistry`)
and process spawning remain unchanged.

**Registry-only resolution:** `<jobname>` is always resolved via `josyn.JobRegistrations`.
There is no direct-path bypass. If the job is not registered, the CLI fails with a clear error
before a session is created. This is consistent with how the scheduler triggers jobs at runtime.

**Fire-and-forget:** `StartSession` spawns `JAPServer` and returns a GUID immediately. The CLI
does not wait for the job to complete. Errors that occur during job execution (including
argument deserialisation failures) surface in the `ErrorStore` and `LocalLog`, not on the
terminal. This is the CLI's intended character — it is a session trigger, not a job monitor.

> **Console output (ADR-022):** Although the CLI exits immediately, `job.exe` is launched with
> `CreateNoWindow = false` when the session originates from the CLI (`Interactive = true`).
> JAPServer inherits the CLI's console; `job.exe` is attached to the same console via
> `CreateProcessWithLogonW`. Any output the job writes to stdout/stderr appears in the
> operator's terminal window, interleaved with the shell prompt, while the job runs.
> This is intentional — CLI sessions are dev/debug/maintenance runs where output visibility
> matters. It is not a guarantee of ordered or capturable output.

**Exit codes** (inherit from the current pattern):

| Code | Meaning |
|------|---------|
| `0` | Session started successfully |
| non-zero | Session could not be started; error written to console |

---

### Resolution Rule

When `<argument-file>` is omitted:

- The CLI passes an empty string to `StartSession`.
- This is valid for zero-parameter jobs. `JobInvoker` short-circuits before calling
  `GetRawArguments()` when `parameters.Length == 0` — the empty string is never read.
- For jobs that require arguments, `JobInvoker` must detect the empty string early and return
  a clear error (see *Required Code Fix* below).

The CLI does **not** auto-discover `local-arguments\arguments-default.ini`. Its behaviour is
fully determined by what was typed. The `.cmd` launcher is the right place to encode
"use this specific file for this context".

---

### Required Code Fix — `JobInvoker` Guard

Traced during design: `PropertyBag.Deserialize("")` fails with `"No sections found."` — a
serialiser-level message that gives the operator no hint of the actual mistake.

`JobInvoker.CreateInvocationArguments` must add a guard before attempting deserialisation:

```csharp
private static async Task<Result<object[]?>> CreateInvocationArguments(
    MethodInfo func, IJosynApplicationProtocol japClient)
{
    var parameters = func.GetParameters();
    if (parameters.Length == 0) return Result<object[]?>.Success(null);

    var rawArguments = await japClient.GetRawArguments();
    if (!rawArguments.Succeeded)
        return Result<object[]?>.Propagate(rawArguments.ToResult<object[]?>());

    if (string.IsNullOrWhiteSpace(rawArguments.Value))
        return Result<object[]?>.Fail(
            "Job erwartet Argumente, aber es wurden keine übergeben.");

    // ... existing deserialisation path
}
```

This fix belongs in `josyn-job-host`. It produces a readable entry in the `ErrorStore` when
an operator starts a parameterised job without an argument file.

---

### The `local-arguments` Convention

> **Scope:** `local-arguments\` is a CLI and maintainer convention. It is **not** a production
> execution vehicle. The `ticker-service` must not read from this folder without a dedicated
> architectural decision. How scheduled job arguments reach the ticker at runtime is an
> explicitly deferred question — see *Open Questions* below.

Each job folder in the deployed job repository may optionally contain a `local-arguments\`
subfolder:

```
$JobRepositoryRoot\
  Contoso.DemoProduct.DemoJob\
    Contoso.DemoProduct.DemoJob.exe
    local-arguments\                          <- entirely optional; generated at deploy time
      _launcher.cmd                           <- shared launcher; CLI path lives here once
      arguments-default.ini
      arguments-default.cmd                   <- one-liner; calls _launcher.cmd
      arguments-integration-test.ini
      arguments-integration-test.cmd          <- one-liner; calls _launcher.cmd
```

**Rules:**

- The subfolder name is fixed: `local-arguments`.
- The entire folder is **optional**. Jobs with no `[JobEntryPoint]` parameters need not have
  it at all — `ArgGen` will not generate one.
- `local-arguments\` is a **deploy-time artefact**, not a source-controlled artefact. It is
  generated by `ArgGen` during deployment and must not be committed to the job repo.
- `.ini` files may exist without a corresponding `.cmd`. The `.cmd` is a convenience — not
  a required pairing.
- The naming pattern `arguments-<context>.ini` / `arguments-<context>.cmd` is a convention,
  not enforced by the platform.
- `ArgGen` generates the `default` context only. Additional named contexts (e.g.
  `arguments-integration-test`) are hand-authored post-deploy if needed.

---

### The `.ini` File

Standard `PropertyBag` INI format — the same format used for job arguments at runtime:

```ini
Message=Hallo JOSYN
RepeatCount=3
ScheduledFor=06.06.2025
IsHighPriority=False
Budget=1499,99
```

Culture is `de-DE` throughout (decimal separators, date formats), consistent with the
platform-wide serialization convention.

---

### The `.cmd` Launchers

The `local-arguments\` folder uses a two-level structure to avoid repeating the CLI path
in every per-context file.

**`_launcher.cmd` — the shared launcher:**

Contains the CLI path, the echo header, and the `CHOICE` confirmation gate. The CLI path
is injected by `ArgGen` at deploy time.

```batch
@echo off
CHCP 1252 >nul

set JOSYN_CLI=C:\ProgramData\JOSYN\CLI\JOSYN.Backend.CLI.exe
set JOB_NAME=Contoso.DemoProduct.DemoJob

echo.
echo  Job:   %JOB_NAME%
echo  Args:  %~1
echo.

set /p CONFIRM="ENTER zum Starten, CTRL-C zum Abbrechen... "

echo.
"%JOSYN_CLI%" run-job "%JOB_NAME%" "%~1"
exit /b %ERRORLEVEL%
```

**Per-context `.cmd` — a one-liner:**

Each argument set gets a thin wrapper that passes its `.ini` path to `_launcher.cmd`:

```batch
@call "%~dp0_launcher.cmd" "%~dp0arguments-default.ini"
```

Notes:
- `%~dp0` expands to the directory of the calling `.cmd` file — the `.ini` path is always
  self-relative, requiring no hardcoded absolute path for the argument file itself.
- The CLI path in `_launcher.cmd` **is** hardcoded to the installation. This is consistent
  with the `launch.*.cmd` convention (see `architecture/local-build.md`): maintainer scripts
  may contain machine-specific paths. If the path changes, one file is edited.
- The `CHOICE` gate is intentional. There is no scripted bypass — scripts invoke the CLI
  directly and do not use the `.cmd` launchers.
- The `.cmd` files are implicitly environment-specific (the CLI path encodes the installation).
  On INT, a separate set of `.cmd` files points to the INT installation.

---

### `ArgGen` — Deploy-Time Scaffold Generator

`ArgGen` is a standalone console tool (`josyn-toolbox\arg-gen\`) that generates the
full `local-arguments\` scaffold for a job after it has been published.

**Call signature:**

```
ArgGen.exe <job-exe-path> --cli-path <cli-exe-path> [--output-dir <dir>]
```

| Argument | Required | Description |
|----------|----------|-------------|
| `<job-exe-path>` | yes | Path to the published job executable |
| `--cli-path` | yes | Path to `JOSYN.Backend.CLI.exe` on the target machine |
| `--output-dir` | no | Output directory; defaults to `local-arguments\` next to the job exe |

**What it generates (full scaffold):**

- `arguments-default.ini` — property names with type-appropriate placeholder values
- `_launcher.cmd` — shared launcher with CLI path, echo header, and `CHOICE` confirmation gate
- `arguments-default.cmd` — one-liner wrapper calling `_launcher.cmd` with the default INI

**How it works internally:**

1. Loads the published job assembly in an isolated `AssemblyLoadContext` (resolves
   dependencies from the job folder — no runtime execution occurs).
2. Finds the `[JobEntryPoint]` method by attribute name.
3. Detects single-record vs. flat-parameter signature (via `<Clone>$` — same heuristic as
   `JobInvoker.RetrieveInvocationArguments`).
4. Derives property names and emits type-appropriate placeholder values (de-DE culture,
   consistent with the platform-wide serialization convention).
5. Includes a `; TypeName` comment line in the generated INI for documentation.
6. Job type name is derived from the exe file name — no extra input required.

**Placeholder values by type:**

| Type | Placeholder |
|------|-------------|
| `string` | *(empty)* |
| integer types | `0` |
| `decimal` / `float` / `double` | `0,00` |
| `bool` | `False` |
| `DateOnly` | today in `dd.MM.yyyy` |
| `DateTime` / `DateTimeOffset` | now |
| `Guid` | `00000000-0000-0000-0000-000000000000` |
| `TimeSpan` / `TimeOnly` | `00:00:00` |
| `enum` | first declared value name |
| `T?` (nullable) | *(empty)* |

**Exit behaviour:**

| Condition | Exit code | Output |
|-----------|-----------|--------|
| Scaffold written | `0` | Files in `--output-dir` |
| Job has no parameters | `0` | Nothing written; message to stderr |
| Error (not found, no entry point, etc.) | non-zero | Error message to stderr |

**Integration in `deploy-maintainer.ps1`:**

`ArgGen` is called after each job publish step, passing `$CliRoot` as `--cli-path` and
the published job folder as `--output-dir`. This replaces any hand-authored
`local-arguments\` content in deployed job folders.

> **Note:** The `deploy-maintainer.ps1` script split (backend A vs. job B) proposed earlier
> in this document remains an open question and is not required for ArgGen integration.
> ArgGen is added as an additional step in the existing monolithic script.

---

## Open Questions

> **⚠ DEFERRED — Production argument sourcing for the `ticker-service`**
>
> How scheduled job arguments reach the `ticker-service` at runtime is an open architectural
> decision that must be resolved before the scheduling subsystem is designed.
>
> Candidate options:
> - **A — Database:** arguments stored inline in the scheduling descriptor or a dedicated
>   `josyn.JobArgumentSets` table. Change path: SQL migration → source control → pipeline.
> - **B — Source-controlled files:** `.ini` files version-controlled in the job repo,
>   deployed by pipeline. `local-arguments\` on production becomes a pipeline output.
> - **C — Management surface:** arguments managed via an admin API/UI, stored in DB,
>   with built-in audit and versioning.
>
> **Governance constraint (non-negotiable):** the chosen mechanism must support pipeline-based
> change management with full audit trail. RDP-based file editing is not an acceptable
> change path for production argument sets.
>
> `local-arguments\` must **not** be assumed as the answer for the ticker-service without
> an explicit decision captured in a follow-up ADR.

1. **`local-arguments` source of truth:** ~~Where do the `.ini` and `.cmd` files live before
   deployment?~~ **Resolved — Option B.** `local-arguments\` is a deploy-time artefact
   generated by `ArgGen`. Files are never checked into the job repo.

2. **`_launcher.cmd` generation:** ~~If `ArgGen` writes `_launcher.cmd`, it must know the
   installation path at deploy time.~~ **Resolved.** `ArgGen` receives the CLI path via
   `--cli-path`. The value is provided by `deploy-maintainer.ps1` as `$CliRoot`.

3. **`ArgGen` — record type name as INI comment:** **Resolved.** The generated `.ini`
   includes a `; TypeName` comment line for documentation. No platform impact.

4. **`deploy-maintainer.ps1` split — manifest vs. literal list:** Still open. The split into
   backend (A) and job (B) functions is desirable but not required for `ArgGen` integration.
   Deferred until the script grows unwieldy or multiple job repos are in play.

---

## Consequences

- The CLI gains a real command surface; `HardCodedDemoSessionStart()` can be removed.
- Operators can trigger any registered job with any named argument set without IDE or source
  access.
- The `local-arguments\` convention is self-documenting: the presence and naming of `.ini`
  files communicates which execution contexts the maintainer has prepared.
- The `CHOICE` gate in `_launcher.cmd` prevents accidental execution from the file explorer.
- A single `_launcher.cmd` edit updates the CLI path for all argument sets in a job folder.
- `ArgGen` generates the full `local-arguments\` scaffold at deploy time — hand-authored
  argument files in job repos are eliminated. The `contoso-demo-job\local-arguments\` hack
  can be removed once `ArgGen` is integrated into `deploy-maintainer.ps1`.

---

## Relation to Other ADRs

- **ADR-012** (Maintainer Deployment): `local-arguments\` lives inside the job folder
  deployed by ADR-012. `_launcher.cmd` references the CLI path established by ADR-012. The
  proposed script restructuring extends ADR-012's scope.
- **ADR-013** (Job Dev Mode): `dev-args.ini` uses the same `PropertyBag` INI format as
  `local-arguments`. The two conventions address different lifecycle stages: IDE debugging
  (ADR-013) vs. operator execution against a deployed installation (this ADR).
- **ADR-005** (JobRegistry): the `run-job` command resolves `<jobname>` exclusively via
  `josyn.JobRegistrations` — the same registry used at runtime by `SessionStarter`.
- **ADR-010** (Environment Separation): `_launcher.cmd` hardcodes an installation path and is
  therefore implicitly environment-specific. This is acceptable for maintainer tooling but
  should be noted when INT deployment is formalised.



