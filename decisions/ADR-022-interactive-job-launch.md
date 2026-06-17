# ADR-022 — Headless / Interactive Distinction for job.exe Launch

**Date:** 2026-06-17
**Status:** Accepted — amends ADR-021 § 2 (`ImpersonatedProcess`)

---

## Context

ADR-021 established `ImpersonatedProcess.Start` with `CreateNoWindow = true` as a fixed
property. That is correct for production: JAPServer runs as a Windows service with no
interactive desktop, and the job process must not attempt to allocate one.

The CLI (`JOSYN.Backend.CLI`) launches sessions for a qualitatively different purpose:
developer test runs, ad-hoc maintenance, and integration verification. In these cases the
caller is a human at a terminal who expects to see what the job is doing. With
`CreateNoWindow = true` the job's console output was silently swallowed.

The question is how to model the distinction without fracturing the launch path.

---

## Decision

### 1. A single `Interactive` flag threads the call chain

A `bool Interactive { get; init; } = false` property is added to both
`SessionLaunchRequest` (produced by the caller) and `SessionStartSpec` (resolved and
forwarded by `SessionLauncher`). The field is optional everywhere — callers that omit it
get the existing headless behaviour at no cost.

The property is **not** named `Headless`. The dominant state (production, service context)
is the default. Callers that want non-default behaviour set the flag explicitly. No silent
inversion of semantics is possible.

```csharp
// SessionLaunchRequest / SessionStartSpec — shared new property
public bool Interactive { get; init; } = false;
```

### 2. `ImpersonatedProcess.Start` gains a `headless` parameter

```csharp
public static Result<int> Start(
    string            exePath,
    string            arguments,
    string            password,
    WindowsCredential credential,
    bool              headless = true)   // ← new; default preserves prior behaviour
```

`headless` is mapped directly to `CreateNoWindow`:

```csharp
CreateNoWindow = headless,   // false = attach to caller's console (interactive/dev mode)
```

The default is `true` — a callee with no opinion gets `CreateNoWindow = true`, which is
the correct service-context default. The parameter name is `headless` rather than
`interactive` to keep `ImpersonatedProcess` domain-agnostic (it does not know why the
process is being launched).

### 3. `LaunchJobAndStorePid` forwards the flag

`LaunchJobAndStorePid` in `Host.Prepare.cs` receives `bool interactive = false` and passes
`headless: !interactive` to `ImpersonatedProcess.Start`. The negation is local to the
call site and is named at the call site via a named argument — no silent inversion.

### 4. CLI always sets `Interactive = true`

```csharp
new SessionLaunchRequest
{
    ...
    Interactive = true   // CLI-launched sessions are dev/debug/maintenance
}
```

No CLI flag or switch is needed. Any session the CLI starts is by definition interactive.
The character of the caller is sufficient to determine the mode.

---

## Observable behaviour

When the CLI starts a job:

- JAPServer is launched by `SessionLauncher` with `UseShellExecute = false` and no
  `CreateNoWindow` override — it inherits the CLI process's console.
- `job.exe` is launched with `CreateNoWindow = false` — Windows attaches it to the same
  console (via `CreateProcessWithLogonW`, which uses console attachment rather than handle
  inheritance, so this works across account boundaries).
- Console output from `job.exe` appears in the operator's terminal window.

The CLI remains **fire-and-forget** (ADR-014): it exits immediately after spawning JAPServer.
JAPServer and job.exe outlive the CLI process, but the console itself (owned by the shell
that launched the CLI) remains open. Output arriving after the CLI exits still appears in
the terminal, interleaved with the shell prompt. This is acceptable for dev/debug use; it
is not suitable for scripted output capture.

When a production scheduler (Ticker, Listener) starts a job:

- `Interactive` is not set → defaults to `false` → `headless = true` → `CreateNoWindow = true`.
- Behaviour is identical to pre-ADR-022.

---

## Consequences

- `JOSYN.Backend.Contracts` (`SessionLaunchRequest`, `SessionStartSpec`) — new optional
  field. All existing callers are source-compatible; the field defaults to `false`.
- `JOSYN.Backend.SessionLauncher` — copies `Interactive` from request to spec. No
  behavioural change for callers that omit it.
- `JOSYN.Commons.Helpers` / `ImpersonatedProcess` — new optional `headless` parameter.
  All existing call sites are source-compatible; the default is `true`.
- `JOSYN.Jap.JAPServer` / `Host.Prepare.cs` — forwards the flag. No behavioural change
  for non-interactive sessions.
- `JOSYN.Backend.CLI` — sets `Interactive = true`. Console output from job.exe is now
  visible to the operator.
- No new Windows privilege requirements: `CreateNoWindow = false` with
  `CreateProcessWithLogonW` requires no additional ACLs beyond those already established
  in ADR-021.

---

## Relation to Other ADRs

- **ADR-021** (Impersonated Process Launch): amended. The `CreateNoWindow` property is no
  longer unconditionally `true`; it is now controlled by the `headless` parameter.
- **ADR-014** (CLI `run-job`): the "fire-and-forget" characterisation remains accurate.
  This ADR adds the caveat that job console output is now visible in the operator's
  terminal (see § Observable behaviour above).
