# ADR-017B-01 — Session-Starter Relocation into JAPServer

**Date:** 2026-06-12
**Status:** Superseded in part — JOSYN-IPC start mode removed 2026-06-14 (see closing note)

---

## Context

`JOSYN.Backend.SessionStarter` is currently a NuGet package consumed by orchestrators.
An orchestrator calls `ISessionStarter.StartSession(jobTypeName, arguments)`, which:

1. Validates the job is registered in `JobRegistry`.
2. Allocates a session GUID.
3. Persists a session record to `SessionStore`.
4. Spawns `JAPServer.exe JOSYN-IPC <guid>`.

Four orchestrator executables are planned: `listener`, `ticker`, `cli`, and `workflow-runner`.
Under the current model each of them would consume `SessionStarter`, making every orchestrator
responsible for session persistence, concurrency control, and process spawning — concerns
that have nothing to do with *when* or *why* a job starts.

Additionally:

- `JOSYN.Commons.Helpers.Turnstile` now exists and provides a cross-process, cross-user
  named mutex. Concurrent start requests for the same job (e.g. two tickers firing in overlap)
  must be serialised to avoid duplicate sessions. The right place to enforce this is at the
  session boundary itself, not in each orchestrator independently.
- `JAPServer` is already the runtime owner of a session: it holds the GUID, communicates
  with `job.exe`, reads arguments and writes results to `SessionStore`, and handles errors.
  Creating the session record is a natural extension of that ownership.

---

## Decision

### 1. Session-start logic moves into JAPServer

All session-start responsibilities — job registration check, GUID allocation, session
persistence, Turnstile concurrency control, and job.exe spawning with impersonation — move
from `JOSYN.Backend.SessionStarter` into `JOSYN.Jap.JAPServer`'s startup path.

Orchestrators become thin launchers. They describe *what* to run and supply caller context;
they do not manage the session lifecycle.

### 2. New JAPServer invocation mode: `JOSYN-START`

A second CLI mode is added alongside the existing `JOSYN-IPC` mode:

```
JOSYN.Jap.JAPServer.exe JOSYN-START <serialized-session-start-request>
```

| Mode | Caller provides | JAPServer does |
|------|-----------------|----------------|
| `JOSYN-IPC <guid>` | Pre-existing session GUID | Serves JAP protocol for the existing session |
| `JOSYN-START <request>` | Session-start request | Full session-start, then serves JAP protocol |

`JOSYN-IPC` is retained for migration compatibility and for scenarios where the session record
is created externally (e.g. legacy orchestrators during the transition period). It is not
deprecated by this ADR.

### 3. SessionStartRequest and the SessionLauncher packages

`SessionStartRequest` is a shared data contract consumed by both orchestrators and JAPServer.
Following the `JOSYN.Jap.Contract` pattern, it lives in a dedicated contracts package:

**`JOSYN.Backend.SessionLauncherContract`** — contracts only:
- `SessionStartRequest` record

```csharp
record SessionStartRequest
{
    string JobTypeName          { get; init; }
    string Arguments            { get; init; }   // job arguments file content, base64-encoded (see §3a)
    string TechnicalUserName    { get; init; }   // resolved from JobRegistry by SessionLauncher
    string CallerUser           { get; init; }
    string CallerDomain         { get; init; }
    string CallerApplication    { get; init; }
    string CallerMachine        { get; init; }
}
```

**`JOSYN.Backend.SessionLauncher`** — orchestrator-side launcher:
- `ISessionLauncher` / `SessionLauncher`

```csharp
public interface ISessionLauncher
{
    Result LaunchSession(SessionStartRequest request);
}
```

`SessionLauncher` owns all pre-launch preparation:
1. Validate the job is registered in `JobRegistry` — fail fast if not.
2. Resolve `TechnicalUserName` from `JobRegistry` and populate it in `SessionStartRequest`.
3. **Base64-encode** the raw arguments file content and store the encoded string in
   `SessionStartRequest.Arguments` (see §3a).
4. Serialize `SessionStartRequest` to INI (via `PropertyBag`), write to a temp file named
   `josyn-start-<throwaway-guid>.ini`, and spawn `JAPServer.exe JOSYN-START @<path>`.

JAPServer **trusts the request** — it never consults `JobRegistry`. The registry is the
launcher's concern.

**Dependencies of `JOSYN.Backend.SessionLauncherContract`:**
`JOSYN.Foundation.ResultPattern` only.

**Dependencies of `JOSYN.Backend.SessionLauncher`:**
`JOSYN.Backend.SessionLauncherContract`, `JOSYN.Foundation.PropertyBag`,
`JOSYN.Foundation.ResultPattern`, `JOSYN.Backend.BootstrapConfig`,
`JOSYN.Backend.JobRegistry`.

**CLI transport — temp file with `@`-prefix convention:**

```
JAPServer.exe JOSYN-START @<path>
```

The `@` prefix signals a file reference. `<path>` points to a temp file containing the
serialized `SessionStartRequest` (INI format, via `PropertyBag`). JAPServer reads and
**immediately deletes** the file before entering the Turnstile.

File naming: the orchestrator generates a throwaway `Guid.NewGuid()` solely for the
filename (e.g. `josyn-start-<spawn-guid>.ini`). This GUID has no further meaning and is
never stored. The session GUID remains allocated by JAPServer inside the Turnstile —
the invariant *"a session GUID in circulation always has a corresponding SessionStore record"*
is preserved.

Orphaned file cleanup: a file that survives longer than a fixed threshold (e.g. 5 minutes)
was never consumed and is safe to delete unconditionally. A trivial age-based wipe (startup
scan or maintenance job) is sufficient — no cross-reference with SessionStore is needed or
performed.

### 3a. Arguments encoding contract

`SessionStartRequest.Arguments` always carries the job arguments file content as a
**base64-encoded string** (`Convert.ToBase64String(File.ReadAllBytes(path))`).

**Why base64, not raw content:**
`PropertyBag` uses INI as its serialization format. INI has no quoting mechanism for
multiline values — embedding a multiline arguments file as a raw string in the temp file
would silently truncate it at the first newline. Base64 encoding collapses any content into
a single-line value that INI handles correctly.

`PropertyBag` is intentionally not extended to support multiline values. Keeping it flat
is a deliberate limitation. Base64 at the transport boundary is the explicit contract.

**Consistency across orchestrators:**
REST-based orchestrators (e.g. the planned `listener`) will naturally receive arguments as
a base64 string in their JSON request body — the REST transport convention. Having
`Arguments` always be base64 means every orchestrator encodes the same way, regardless of
how it received the arguments.

**Encoding boundary:**
- **Orchestrators encode** before populating `SessionStartRequest.Arguments`.
- **JAPServer decodes** (`Convert.FromBase64String`) after deserializing the request,
  before storing the raw INI content in `SessionStore`.
- **`GetRawArguments`** returns the raw decoded INI content to `job.exe`, unchanged.
  `job.exe` never sees base64.

The encoding/decoding is a transport-boundary concern only. It is invisible to `job.exe`
and to anything reading the `SessionStore` database directly.

### 4. JAPServer session-start sequence (JOSYN-START mode)

```
JAPServer.exe receives JOSYN-START @<path>
    │
    ├─ 1. Read and delete temp file; deserialise SessionStartRequest
    ├─ 2. Turnstile.Run(lockId: jobTypeName, worker: ...)   ◄─── lock acquired
    │       ├─ 3. Allocate Guid sessionGuid = Guid.NewGuid()
    │       ├─ 4. SessionStore.SaveNewSession(...)  ← persists record with ExecutionStatus = "preparing"
    │       ├─ 5. Spawn job.exe JOSYN-IPC <sessionGuid>
    │       │         under TechnicalUserName credentials via ICredentialProvider
    │       │         (TechnicalUserName is carried in SessionStartRequest — resolved by SessionLauncher)
    │       └─ 6. Accept/reject negotiation (pipes connected; first JAP exchange)
    │                 ├─ accepted → Turnstile releases; proceed to JAP serve loop
    │                 └─ rejected → mark session, Turnstile releases; exit
    │                                                        ◄─── lock released
    └─ 7. Enter JAP-protocol serve loop (accepted path only)
```

**Turnstile scope rationale:** the lock must be held until the accept/reject outcome is known.
Releasing at spawn time would be too early — a rejected job would have consumed the
concurrency slot without completing, and the next queued start for the same job would
proceed without knowing the prior attempt failed. Keeping the lock until negotiation
completes ensures the slot is only vacated when the session is definitively in-flight or
definitively failed.

The accept/reject negotiation protocol is a new platform feature and is not fully specified
in this ADR. Its design (message format, timeout, error semantics) is deferred to a
dedicated ADR. This ADR only establishes the boundary constraint: **negotiation must occur
inside the Turnstile-protected start phase**.

If any step within the Turnstile fails, the session record is left with
`ExecutionStatus = "preparing"` and no running job process — the same known limitation
documented in the `SessionStarter` sanity notes. Resolving orphaned sessions requires a
separate status-column ADR.

### 5. Turnstile lock granularity

The lock id is the `JobTypeName`. Only concurrent starts of the **same job** are serialised.
Starts of different jobs are independent and proceed in parallel.

### 6. Impersonation

`job.exe` is spawned under the Windows account stored as `TechnicalUserName` in `JobRegistry`.
The credentials required for impersonation (password, domain) are a **company concern** —
they depend on the company's identity infrastructure and secrets management policy.

The platform defines an extension point: **`ICredentialProvider`**. JAPServer receives it
via constructor injection and calls it to resolve credentials for a given `TechnicalUserName`
before spawning `job.exe`. The company supplies the implementation.

The design of `ICredentialProvider` — contract shape, package location, stub for development
— is deferred to a dedicated ADR (ADR-017B-03). Until that ADR is implemented, impersonation
is not active; `job.exe` is spawned under the JAPServer process identity.

### 7. New NuGet dependencies in JAPServer

`JOSYN.Commons.Helpers` is added (for `Turnstile`) and `JOSYN.Backend.SessionLauncherContract`
is added (for `SessionStartRequest` deserialization). Both are downward dependencies and do
not violate the platform layering rules. `JOSYN.Backend.SessionLauncher` itself is **not**
referenced by JAPServer — it is the orchestrator-side package.

### 8. Fate of JOSYN.Backend.SessionStarter

`JOSYN.Backend.SessionStarter` is **replaced** by `JOSYN.Backend.SessionLauncher`.
It will not be deleted immediately: legacy orchestrators may continue to use it during
the transition period. When all orchestrators have been migrated to `SessionLauncher`,
the old package is removed.

---

## Consequences

- Orchestrators have no session-lifecycle code. Each is reduced to: construct a
  `SessionStartRequest`, serialise it, and spawn `JAPServer.exe JOSYN-START <payload>`.
- All four planned orchestrators (`listener`, `ticker`, `cli`, `workflow-runner`) share
  an identical spawning contract — there is nothing orchestrator-specific in the
  session-start path.
- `JOSYN.Jap.JAPServer` acquires a new dependency (`JOSYN.Commons.Helpers`) and grows
  a startup branch. Its existing `JOSYN-IPC` path is unchanged.
- `JOSYN.Backend.SessionStarter` is replaced by `JOSYN.Backend.SessionLauncher`;
  the planned-components table in `overview.md` and `architecture/overview.md` will be
  updated when the implementation is complete.

---

## Closing note (2026-06-14)

The transition period is complete. `JOSYN.Backend.SessionStarter` had zero consumers outside
its own assembly — the `cli` orchestrator (the only live entry point) already used
`SessionLauncher`. The JOSYN-IPC **start mode** has therefore been removed from
`JOSYN.Jap.JAPServer` and the `josyn-backend-session-starter` sub-folder has been deleted.

`JAPServer` now accepts only `JOSYN-START @<path>`. The `JOSYN-IPC` string is retained as
`IPipesProtocol.MagicToken` — its role as the named-pipe protocol prefix passed to job
executables is unaffected.
