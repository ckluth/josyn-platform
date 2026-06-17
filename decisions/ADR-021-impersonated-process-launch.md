# ADR-021 — Impersonated Process Launch for job.exe

**Date:** 2026-06-17
**Status:** Accepted — implements ADR-017B-03 § credential usage

---

## Context

ADR-017B-03 established the `IdentityAdapter` contract and wired the `GetPassword` call into
`Host.Prepare.cs`. The returned password was plumbed through to `LaunchJobAndStorePid` with a
TODO marker:

```csharp
string technicalUserPassword   // TODO (ADR-017B-03): use for impersonated Process.Start
```

ADR-019 moved process launch out of `PipesServer` and into the caller as an explicit
`Result<int>`-returning step. The current `PipesServer.StartClientProcess` performs a plain
`Process.Start` with no credentials — the JAP server process identity is inherited by `job.exe`.

Two constraints shape the implementation:

1. **No user profile.** The technical user account has no local profile on the execution
   machine. Windows creates a temporary profile on first logon if `LoadUserProfile = true`;
   that must not happen here. `job.exe` must therefore never depend on environment variables
   backed by a user profile (`%APPDATA%`, `%USERPROFILE%`, etc.).

2. **Non-interactive context.** The JAP server is started by a Windows service. There is no
   interactive desktop. The child process must not attempt to create a visible window.

A third consideration — **Linux readiness** — is addressed as a position decision (§ 5 below).

---

## Decisions

### 1. Impersonation lives in `Host.Prepare.cs`, not in Foundation

`PipesServer` is a transport primitive (ADR-019). It must not acquire knowledge of identity
or credentials. `Host.Prepare.cs` already owns the prepare phase; it is the correct call site
to decide *how* the process is launched.

`LaunchJobAndStorePid` stops delegating to `PipesServer.StartClientProcess` for the
impersonated path and builds the `ProcessStartInfo` directly, using:

```csharp
PipesProtocol.CreateClientStartCLIArguments(sessionGuid.ToString())
```

`PipesServer.StartClientProcess` is retained unchanged for tests, demos, and any future
non-impersonated callers.

### 2. Two new utilities in `josyn-commons-helpers`

The building blocks are reusable (UPN parsing, impersonated process launch) and carry no
JOSYN-domain knowledge — they satisfy the commons guard rule (ADR-003). They live in the
existing `JOSYN.Commons.Helpers` package alongside `Turnstile`.

Both types carry `[SupportedOSPlatform("windows")]`.

#### `WindowsCredential`

A small validated value type that wraps a Windows UPN string.

```csharp
[SupportedOSPlatform("windows")]
public readonly record struct WindowsCredential
{
    public string Upn { get; }   // "username@domain"

    public static Result<WindowsCredential> Parse(string upn);
}
```

`Parse` rejects strings that do not contain exactly one `@` character and are not otherwise
empty. Bare usernames (no `@`) are **not accepted** — technical accounts are always
domain-qualified in JOSYN deployments (see § 4).

#### `ImpersonatedProcess`

A static launcher that builds and starts a `ProcessStartInfo` with Windows credentials.

```csharp
[SupportedOSPlatform("windows")]
public static class ImpersonatedProcess
{
    public static Result<int> Start(
        string           exePath,
        string           arguments,
        string           password,
        WindowsCredential credential);
}
```

`ProcessStartInfo` is built internally with:

| Property | Value | Rationale |
|---|---|---|
| `UserName` | `credential.Username` | Account name — part before `@` |
| `Domain` | `credential.Domain` | Domain or machine name — part after `@`; works for both AD and local accounts |
| `PasswordInClearText` | `password` | Windows-only string variant; `SecureString` is deprecated in .NET 6+ |
| `UseShellExecute` | `false` | Required for credential-based launch |
| `CreateNoWindow` | `true` | Service context; no interactive desktop |
| `LoadUserProfile` | `false` | Technical user has no local profile |

Returns the process ID on success.

### 3. `TechnicalUserName` format: UPN (`username@domain`)

`SessionStartSpec.TechnicalUserName` is defined as a UPN string: `username@domain`.

This works for both Active Directory domain accounts (`svc_job@corp.local`) and local machine
accounts (`svc_job@MACHINENAME`). `ImpersonatedProcess` splits the UPN into `UserName` and
`Domain` parts for `ProcessStartInfo` internally — this is an implementation detail.
Callers always deal with the whole UPN string.

The `IIdentityAdapter.GetPassword` contract receives the full UPN string unchanged.

### 4. Bare usernames are not supported

Local accounts (no domain) are rejected at the `WindowsCredential.Parse` call site with a
descriptive error. The rationale:

- Technical users in JOSYN are domain accounts by design; local accounts offer no meaningful
  isolation boundary between the scheduler and the job.
- Allowing bare usernames would require a separate code path (no UPN, no clear `Domain`
  value) that buys nothing architecturally.

If a future deployment genuinely requires local account impersonation, this decision can be
revisited as a follow-up ADR.

### 5. Linux readiness: explicit gate, no abstraction

Impersonation on Linux uses `sudo -u <user>` rather than credential injection via
`CreateProcessWithLogonW`. The password produced by `IIdentityAdapter.GetPassword` is
meaningless in that context — `sudo` with `NOPASSWD` is the expected Linux pattern. This
makes the Linux path a different adapter contract shape, not merely a different
`ProcessStartInfo` construction.

**Decision:** No cross-platform abstraction is introduced now. The call site in
`LaunchJobAndStorePid` carries an explicit OS guard:

```csharp
if (!OperatingSystem.IsWindows())
    return Result.Fail("Impersonated process launch is not supported on non-Windows platforms.");
```

`[SupportedOSPlatform("windows")]` on both commons types makes this a compile-time signal
as well.

When Linux support is needed it requires:
- A new `ImpersonatedProcess.StartOnLinux` (or equivalent) using `sudo -u <user>`.
- A revised adapter contract or a separate `ILinuxIdentityAdapter` — the password model
  does not translate.
- `sudoers` configuration on the target machine: the service account must be granted
  `NOPASSWD` rights to run `job.exe` as the technical user.

The code seam is clean; the operational and contract cost is visible.

---

## `LoadUserProfile = false` — job.exe contract constraint

`job.exe` must not depend on environment variables or file-system paths derived from the
user profile: `%APPDATA%`, `%LOCALAPPDATA%`, `%USERPROFILE%`, `%TEMP%` (user-scoped).

This constraint is documented here and must be carried in `architecture/coding-standards.md`
under the job implementation rules.

System-scoped paths (`%ProgramData%`, `%TEMP%` machine-level, `%SystemRoot%`) are available
and safe to use.

---

## Consequences

- `JOSYN.Commons.Helpers` gains two new types: `WindowsCredential` and `ImpersonatedProcess`.
  The package version is bumped; all consumers must update their reference.
- `Host.Prepare.cs` / `LaunchJobAndStorePid` calls `ImpersonatedProcess.Start` instead of
  `PipesServer.StartClientProcess`. `PipesServer.StartClientProcess` is untouched.
- `SessionStartSpec.TechnicalUserName` documentation is updated to specify UPN format.
- The `TODO (ADR-017B-03)` marker in `Host.Prepare.cs` is resolved.
- Job developers are bound by the `LoadUserProfile = false` constraint — documented in
  coding-standards.

---

## Resolved questions

- **Where does the impersonation logic live?** ✅ `Host.Prepare.cs` via `ImpersonatedProcess` in commons
- **`TechnicalUserName` format?** ✅ UPN (`username@domain`)
- **`Domain` field on `ProcessStartInfo`?** ✅ Explicit domain/machinename split from UPN — hidden inside `ImpersonatedProcess`, not visible to callers
- **`SecureString` vs `PasswordInClearText`?** ✅ `PasswordInClearText` — `SecureString` is deprecated in .NET 6+
- **`LoadUserProfile`?** ✅ `false` — no local profile exists; job.exe must not rely on one
- **Bare local usernames?** ✅ Not supported — domain accounts only
- **Linux readiness?** ✅ Explicit OS gate; no abstraction; cost documented; deferred
- **Window station / desktop ACL grants (service context)?** ✅ Not needed — `CreateNoWindow = true` + headless job.exe
