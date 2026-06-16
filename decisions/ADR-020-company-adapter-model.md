# ADR-020 — Company Adapter Model (Out-of-Process)

**Date:** 2026-06-16
**Status:** Accepted — supersedes ADR-009

---

## Context

ADR-009 established the company adapter pattern: adapter code is loaded into the
`josyn-backend` / `JAPServer` process via an isolated `AssemblyLoadContext` (ALC). The ALC
prevents version conflicts between the adapter's own dependencies and those already loaded
by the host. A shared contracts package preserves type identity for interface casts across
the load-context boundary.

This works — but it is architecturally analogous to the old `JobHost.exe` model that JOSYN
was built to escape. `JobHost` loaded job DLLs into its own process via reflection, required
careful assembly-resolve gymnastics, and could never fully prevent version conflicts. The ALC
is better discipline, but the structural risk is the same: **a third-party binary runs inside
the host's process space**. Any dependency brought in by the adapter is a potential conflict
source that neither the platform nor the adapter author fully controls.

JOSYN's own cure for JobHost was process-space separation via JIP. The same principle may
apply to adapters.

### Adapter candidates (examples)

- `IdentityAdapter` — resolves credentials / impersonation context
- `ConfigurationAdapter` — reads runtime config values from a company config service
- `LegacyConfigAdapter` — reads from a legacy config store
- `WorkflowAdapter` — notifies or queries a company workflow engine

All of these are **stateless lookup / persist tools**: they receive a request, return a
result, and hold no state worth preserving between calls.

---

## Options

### Option A — In-process ALC *(current, ADR-009)*

The adapter is a DLL deployed to an `Adapters/` subfolder. The host loads it via
`AssemblyLoadContext`, instantiates the implementation type by name, and calls it directly
as an in-process interface.

**Pros**
- Simple: no extra process, no IPC plumbing
- Low latency: direct in-process call
- Already implemented and demonstrated with `ContosoConfigSource`

**Cons**
- Shared process space: adapter dependencies live alongside the host's dependencies
- ALC isolation mitigates version conflicts but does not eliminate them structurally
- Adapter crash or unhandled exception can destabilise the host process
- Adapter must target a .NET version compatible with the host
- Architecturally contradicts the principle JOSYN itself was built on

---

### Option B — Out-of-process JIP executables *(preferred direction)*

Each adapter concern is a standalone EXE. It exposes a minimal JIP-based protocol — analogous
to how `job.exe` exposes the JAP protocol. The caller (josyn-backend or JAPServer) spawns
the adapter EXE, exchanges protocol calls over named pipes, and the adapter exits when done.

Each adapter type ships a minimal contracts package (analogous to `josyn-jap`) defining its
JIP protocol. The host references only the contracts; the adapter EXE implements them.

Two sub-models exist for the adapter process lifetime:

#### B1 — Session-scoped: spawn once, pipes stay open

The adapter EXE is spawned at JAPServer startup (or lazily on first use within a session).
The JIP pipes remain open for the duration of the session. Multiple protocol calls can be
made over the same connection. The adapter exits when JAPServer exits.

```
JAPServer starts
  │  spawn: ConfigurationAdapter.exe JOSYN-IPC <guid>   ← once per session
  │  JIP call: GetValue("ConnectionString")
  │  ... (session work) ...
  │  JIP call: GetValue("RetryPolicy")                  ← reuses open pipes
  │  JAPServer exits → adapter exits
```

**Additional pros:** Low per-call overhead after initial spawn; natural fit if multiple
calls to the same adapter are expected within a session.

**Additional cons:** Adapter process lifetime is coupled to JAPServer lifetime; a crash in
the adapter mid-session is a harder failure to recover from.

##### B1 lifecycle variants

Within B1, three loading strategies are available. The right choice may differ per adapter type.

**Eager (load at startup)**
The adapter EXE is spawned unconditionally when JAPServer starts, before any work begins.
The pipes are ready before the first call. Simple to reason about; the cost is paid upfront.
Suitable when the adapter is known to be needed for every session.

**Lazy (load on first use, unload when idle)**
The adapter EXE is spawned on the first call. If no further calls arrive within a defined
idle window, the process is torn down and the pipes are closed. A subsequent call respawns
it. Suitable when the adapter is needed only occasionally, or when startup cost should
not be paid for sessions that never use it.

**Permanent (keep loaded for the session lifetime)**
The adapter EXE is spawned on first use and kept alive until JAPServer exits — no idle
unload. Simpler than lazy: no idle timer, no re-spawn logic. Cost: a process that is
no longer needed continues to occupy a process slot until the session ends. Acceptable
when the adapter is lightweight and session duration is short.

#### B2 — Fire-and-forget: spawn a new EXE per request

A fresh adapter EXE is spawned for every individual request. The process performs exactly
one operation, returns the result via JIP, and exits immediately.

```
josyn-backend needs a config value:
  │  spawn: ConfigurationAdapter.exe JOSYN-IPC <guid>
  │  JIP call: GetValue("RuntimeEnvironment")
  │  adapter exits                                       ← clean, no residue
  └── spawns: JAPServer.exe ...
```

**Additional pros:** Maximally simple lifecycle — no open handles to manage, no crash
recovery needed; a failed call is just a failed spawn + error result. Each call is fully
isolated. Natural fit for infrequent, one-shot lookups.

**Additional cons:** Full spawn cost on every call; unsuitable if the same adapter is
called many times in rapid succession within a session.

---

**Pros common to both B sub-models**
- True process isolation: no shared assembly space, structurally guaranteed
- Adapter can target any .NET version independently
- Adapter crash has no effect on the calling process beyond the current call
- Consistent with JOSYN's own architectural cure (JIP + process separation)
- No `AssemblyLoadContext` complexity; no shared contracts package for type-identity

**Cons common to both B sub-models**
- Each adapter type requires a small JIP protocol contract package
- More deployment artefacts: adapter EXEs + their own dependency trees in `Adapters/`
- Caller must handle spawn failure as a first-class error path

---

### Option C — REST service

Each adapter concern is a standalone HTTP service. The caller makes HTTP requests to a
well-known local endpoint.

**Pros**
- Technology-agnostic: adapter can be written in any language / runtime
- Trivially callable from any component — josyn-backend, JAPServer, future components —
  without spawn mechanics
- Independently deployable and independently scalable
- Standard tooling (OpenAPI, logging, health checks)

**Cons**
- Requires a **running service**: adapter availability becomes an operational dependency
- josyn-backend cannot start if the adapter service is not running — a hard availability
  coupling that the other options do not have
- Network stack involved for what is essentially a local function call
- HTTP is heavy for stateless lookup tools; JSON serialization overhead per call
- Service lifecycle (start, stop, restart, version upgrades) must be managed separately
  from the JOSYN backend installation

---

## Comparison summary

| | Option A (ALC) | Option B (JIP EXE) | Option C (REST) |
|---|---|---|---|
| Process isolation | Partial (ALC) | Full | Full |
| .NET version freedom | No | Yes | Yes |
| Lifetime complexity | None | Session-scoped (B1) | High (daemon) |
| Operational dependency | None | None | Yes (service must be running) |
| Latency | Minimal | Spawn cost (tbd) | Network round-trip |
| Consistent with JOSYN model | No | Yes | No |
| Implementation complexity | Low | Medium | Medium–High |

---

## Decision

### 1. Adapter model: B1 — session-scoped out-of-process EXE

Each adapter concern is a standalone EXE. JAPServer spawns it once per session (or eagerly at
startup) and communicates over JIP named-pipe protocol calls. No `AssemblyLoadContext`, no
in-process loading, no reflection-based instantiation.

### 2. JAPServer is the exclusive adapter caller

JAPServer is the only process that spawns and communicates with adapter EXEs. Orchestrators
(`listener`, `ticker`, `cli`, `workflow-runner`) have no adapter knowledge. This is the
natural consequence of ADR-017B-01: session lifecycle — including all adapter calls required
to prepare a session — now lives inside JAPServer's startup path.

### 3. Deployment: `Adapters/` subfolder of JAPServer's binary directory

Adapter EXEs and their own dependency trees are deployed to an `Adapters/` folder relative
to `JAPServer.exe`. This folder is present in every deployment — standalone or company —
populated with whatever adapters that deployment requires.

### 4. Bootstrap config: per-concern key → EXE filename

Each adapter concern is registered via a dedicated key in `josyn.bootstrap.ini` under an
`[Adapters]` section. The value is the EXE filename within the `Adapters/` folder:

```ini
[Adapters]
ConfigurationAdapter = ConfigurationAdapter.exe
IdentityAdapter      = IdentityAdapter.exe
```

No assembly scanning. No discovery. Explicit and traceable — consistent with ADR-009's
explicit naming principle, adapted for EXE-based delivery.

### 5. Contract packages: one per adapter concern

Each adapter concern ships a dedicated JIP contract package defining its request/response
protocol. No shared adapter-protocol infrastructure package is introduced at this stage.
Adapter EXEs build directly on `JOSYN.Foundation.JIP`. If common scaffolding accumulates
across multiple adapter EXEs, a shared adapter-host library may be extracted at that point.

### 6. Adapters are mandatory — no graceful degradation

JAPServer's code has no conditional adapter logic. Every adapter call site unconditionally
requires a working adapter. A deployment without a configured adapter for a required concern
is a misconfigured deployment — JAPServer fails at startup.

Stub implementations (provided by `josyn-contoso` or equivalent) satisfy this requirement
in standalone and development deployments. Stubs implement the same JIP protocol contracts
as production adapters; they differ only in behavior (e.g. returning hardcoded or
filesystem-backed values rather than querying live company infrastructure).

---

## Consequences

- ADR-009's ALC-based adapter model is fully superseded and removed.
- `AdapterLoadContext` and `Host.ConfigSource.cs` are deleted from JAPServer.
- The `IConfigSource` / `SqlConfigSource` / `ConfigStore` naming is discarded. These
  components are replaced: each adapter concern gets a new JIP contract package; any
  SQL-backed config reading retained for internal use becomes JAPServer-internal logic
  under a name that reflects its actual role.
- `ContosoConfigSource` (`Contoso.Josyn.Adapter`) is replaced by a proper adapter EXE with
  a JIP protocol implementation and stub behavior.
- `josyn-backend-adapter-contracts/` (currently an empty placeholder) is the designated
  home for the first adapter contract package.
- Each adapter concern introduced in a future ADR follows this pattern: dedicated contract
  package + JAPServer call site + company adapter EXE + Contoso stub EXE.
- ADR-017B-03 (`ICredentialProvider`) is the first concrete adapter concern to be
  implemented under this model.

---

## Resolved questions

1. **Who is the caller?** ✅ **Resolved — JAPServer exclusively.**
   Orchestrators are thin launchers with no adapter knowledge. All adapter calls required
   to prepare a session belong to JAPServer's startup path (per ADR-017B-01).

2. **Deployment convention:** ✅ **Resolved — `Adapters/` subfolder of JAPServer's binary
   directory.** Carries over from ADR-009 but relocated to JAPServer's own directory,
   since JAPServer is now the caller.

3. **Bootstrap config format:** ✅ **Resolved — per-concern config key → EXE filename.**
   Each concern has its own key in `josyn.bootstrap.ini` under `[Adapters]`. The value
   names the EXE file within `Adapters/`. No discovery mechanism.

4. **Contract package model:** ✅ **Resolved — one contract package per adapter concern.**
   No shared adapter-protocol infrastructure package at this stage.

5. **No-op / stub adapters:** ✅ **Resolved — no platform-side fallbacks.**
   JAPServer fails at startup if a required adapter is absent or fails to start.
   Contoso (or equivalent) provides stub adapters for standalone and development deployments.

---

## Relation to other ADRs

- **ADR-009** (Runtime Context Provider Pattern): the ALC-based model described there is
  the starting point this ADR revises. ADR-009 is superseded by this ADR.
- **ADR-010** (Environment Separation): unaffected — the `josyn.bootstrap.ini`
  single-file-per-installation model applies regardless of which option is chosen.
- **ADR-017B-03** (ICredentialProvider): a concrete adapter use case; its design will
  be informed by whichever option this ADR accepts.
- **ADR-005B-01** (Backend Building Block Model): Option B adapter EXEs are Category B
  executables under that model — consistent, no change required.

---

## Opinion

**B1 (session-scoped EXE, pipes stay open) is the recommended model — applied uniformly
across all adapter types.**

**Against A:** The ALC model is acceptable only if you trust that adapter authors will
never bring in a dependency that conflicts with the host. Adapter authors are outside the
platform team — that trust cannot be enforced structurally. The risk is not hypothetical;
it is the same risk that made the old `JobHost` a maintenance wall. ALC is better
discipline, not a different category of solution.

**Against C:** A REST service introduces an availability dependency. The backend cannot
operate if the adapter service is not running. That coupling does not exist today and
buys nothing for stateless lookup tools. HTTP is also the wrong abstraction for a local
tool invocation — it is not a web API.

**B1 over B2 — one model, not two:** B1 is a strict superset of B2. An adapter that only
ever receives one call per session works under B1 without any structural difference at the
call site — the pipes simply close after one use. Once B1 infrastructure exists (which it
must, because at least one adapter needs session-scoped access), mandating B1 everywhere
is simpler than maintaining two models with rules about when to use which. Simplicity in
operations outweighs the theoretical overhead of keeping pipes open for single-call
adapters.
