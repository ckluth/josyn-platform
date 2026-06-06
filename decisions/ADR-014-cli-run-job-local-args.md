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

## Decision (Draft)

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
    local-arguments\                          <- entirely optional
      _launcher.cmd                           <- shared launcher; CLI path lives here once
      arguments-default.ini
      arguments-default.cmd                   <- one-liner; calls _launcher.cmd
      arguments-integration-test.ini
      arguments-integration-test.cmd          <- one-liner; calls _launcher.cmd
```

**Rules:**

- The subfolder name is fixed: `local-arguments`.
- The entire folder is **optional**. Jobs with no operator-facing argument sets need not have
  it at all.
- `.ini` files may exist without a corresponding `.cmd`. The `.cmd` is a convenience — not
  a required pairing.
- The naming pattern `arguments-<context>.ini` / `arguments-<context>.cmd` is a convention,
  not enforced by the platform.

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
is injected here once — either by the operator manually or by a future auto-generation tool.

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

## Discussion: `deploy-maintainer.ps1` Restructuring

*This is an idea to explore, not a confirmed decision. Captured here because it extends the
local-arguments concept into the deployment pipeline. The deploy topic is deliberately out of
scope for the current design iteration.*

### Proposed Split

The current `deploy-maintainer.ps1` is a monolithic script. The proposal is to split it into
two logical units:

**A — Backend deployment** (executed once per environment, infrequently):
- Publish `JOSYN.Jap.JAPServer`, `JOSYN.Backend.CLI`, adapters, `josyn.bootstrap.ini`

**B — Job deployment** (a reusable function, called once per job):
- Publish `<Job.slnx>` to `$JobRepositoryRoot\<JobTypeName>\`
- Optionally generate `local-arguments\arguments-default.ini` and `_launcher.cmd`

### Auto-Generation of `arguments-default.ini`

The deploy script (part B) could auto-generate an INI template by:

1. Loading the published job assembly via reflection.
2. Finding the `[JobEntryPoint]` method (`JobInvoker.FindJobFunction` logic).
3. Inspecting the parameter signature (single record or parameter list, as
   `JobInvoker.RetrieveInvocationArguments` already distinguishes).
4. Emitting property names with type-appropriate placeholder values.

**Candidate tool — `ArgGen`:** a lightweight console tool
(`ArgGen.exe <job-exe-path> [--output <file>]`) that the deploy script calls after publish.
Natural home: `josyn-sandbox`, consistent with `deploy-maintainer.ps1` (ADR-012 rationale).

If the job has no `[JobEntryPoint]` parameters, no file is generated and `local-arguments\`
may be omitted.

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

1. **`local-arguments` source of truth:** Where do the `.ini` and `.cmd` files live before
   deployment?
   - Option A: version-controlled in the job repo, published as `Content / Copy if newer`.
   - Option B: generated entirely at deploy time by `ArgGen`; never checked in.
   - Option C (hybrid): `.ini` is generated at deploy time; `.cmd` is hand-authored and
     version-controlled.
   The right answer depends on whether `ArgGen` is built and whether job authors want
   argument files under source control.

2. **`_launcher.cmd` generation:** If `ArgGen` writes `_launcher.cmd`, it must know the
   installation path at deploy time. This is available in `deploy-maintainer.ps1` as
   `$CliRoot`. No open question for the script case — but if the `.cmd` is version-controlled
   (Option A / C above), the path must be a known convention or a placeholder.

3. **`ArgGen` — record type name as INI comment:** Should the generated `.ini` include a
   comment line with the source record type name (e.g. `; DemoArguments`) for documentation?
   No platform impact — a cosmetic decision for `ArgGen`.

4. **deploy-maintainer.ps1 split — manifest vs. literal list:** Should the set of jobs be a
   literal array in the script (simple, PoC-appropriate) or driven by a manifest file
   (flexible, adds an artefact)? Literal array is sufficient for the current scale.

---

## Consequences

- The CLI gains a real command surface; `HardCodedDemoSessionStart()` can be removed.
- Operators can trigger any registered job with any named argument set without IDE or source
  access.
- The `local-arguments\` convention is self-documenting: the presence and naming of `.ini`
  files communicates which execution contexts the maintainer has prepared.
- The `CHOICE` gate in `_launcher.cmd` prevents accidental execution from the file explorer.
- A single `_launcher.cmd` edit updates the CLI path for all argument sets in a job folder.

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



