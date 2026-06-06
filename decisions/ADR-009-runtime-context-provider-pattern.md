# ADR-009 — Runtime Context Provider Pattern for External Integrations

**Date:** 2026-06-05
**Status:** Accepted

---

## Context

JOSYN will be deployed in two distinct environments:

- **Standalone (MIT):** No company-specific dependencies. Config comes from a local file.
- **HAEVG:** The company provides services — configuration, identity management, auth tokens,
  and potentially others — via NuGet packages (e.g., `haevg-config-reader`) that must not
  become compile-time dependencies of JOSYN.

As JOSYN matures, multiple HAEVG-specific integration contracts of varying complexity are
expected. A general-purpose pattern is needed — one that works for a simple key-value config
reader today and for a richer multi-method identity provider tomorrow.

### Constraints established during design

- A HAEVG-specific host EXE (different binary per environment) is explicitly rejected.
  JOSYN ships one binary; deployment context is a configuration concern.
- One-to-many IPC (many concurrent JAPServer processes reaching one shared adapter service)
  is rejected: high implementation complexity and at odds with JOSYN's session-isolation model
  (see ADR-004, Challenge 3 rebuttal).
- No HTTP. No DI containers.

---

## Decision

### 1. Adapter pattern: three parties, no compile-time coupling

```
JOSYN                        Adapter assembly              External NuGet
─────────────────────        ─────────────────────         ─────────────────
IContextProvider       ◄──── HaevgContextProvider ──────►  haevg-config-reader
(ships with JOSYN)           (HAEVG-owned, separate)       (untouched)
```

JOSYN defines haevg-agnostic interfaces in `Backend.GlobalConfig`. Adapter assemblies live
outside JOSYN, are not distributed with the standalone build, and are the only layer that
references HAEVG NuGet packages.

### 2. Adapters are invoked exclusively by josyn-backend — at session start, before spawning

This is the structural core of the decision.

`josyn-backend` is the single orchestrating process. It resolves all external context
(configuration, tokens, identity, credentials) **before** spawning JAPServer. The resolved
values are passed as structured arguments. JAPServer passes relevant values through to job.exe
via the JAP arguments mechanism.

```
josyn-backend
  │  1. IContextProvider.GetValue(...)   ← only adapter call
  │  2. IContextProvider.AcquireToken(...)
  │  3. Build full argument set
  └── spawns: JAPServer.exe (receives everything it needs)
                └── spawns: job.exe (receives everything it needs as arguments)
```

**JAPServer and job.exe never call the adapter.** They are pure receivers of pre-resolved data.

### 3. Jobs are pure functions

A job receives arguments and returns a result. Everything it needs for execution — tokens,
config values, identity context — arrives as arguments. Jobs do not make ad-hoc calls to
external infrastructure at runtime. This is consistent with JOSYN's functional-first principle.

If a job implements a REST client, the required access token is pre-loaded by the backend and
delivered as a job argument.

### 4. One-to-one JIP is sufficient

Because the adapter is called only from `josyn-backend` (a single process), the many-to-one
topology never arises. The existing JIP infrastructure and session model require no changes.

### 5. Rendezvous: convention folder + config-driven type name

The adapter assembly is not referenced at compile time. The bootstrap config file names the
adapter type:

```json
{
  "ConnectionString": "...",
  "ContextProviderType": "HAEVG.Josyn.Adapter.HaevgContextProvider, HAEVG.Josyn.Adapter"
}
```

JOSYN loads the assembly from a well-known `Adapters/` subfolder of the backend installation
and instantiates the type via `Type.GetType()` + `Activator.CreateInstance()`. No assembly
scanning. No MEF. This is a deliberate, bounded use of reflection — the second in the platform
after `[JobEntryPoint]` dispatch.

When `ContextProviderType` is absent, `josyn-backend` uses its built-in file-based provider.
This is the standalone path.

---

## Consequences

- `josyn-backend` gains a `Backend.GlobalConfig` abstraction with an `IContextProvider`
  interface (or equivalent). The built-in implementation reads from the local config file.
- The `Adapters/` folder becomes a deployment-level concern: present and populated in HAEVG
  installations, empty or absent in standalone.
- Jobs cannot depend on runtime adapter availability. Any information a job might need must be
  anticipatable at session-start time and expressible as a job argument.
- The pattern generalises: future HAEVG integrations (identity, notifications, workflow)
  follow the same structure — new methods on `IContextProvider`, or additional interfaces
  resolved the same way.

---

## Open questions (to resolve before accepting)

1. **Single interface vs. multiple interfaces:** ✅ **Resolved — separate interfaces per concern.**
   Each integration (config, identity, tokens, notifications, …) gets its own interface and
   its own registration entry in the bootstrap config. No catch-all adapter. This keeps each
   contract minimal, independently replaceable, and independently versioned.

2. **Adapter assembly isolation:** ✅ **Resolved — isolated `AssemblyLoadContext`, analogous
   to `JobAssemblyLoadContext` in the legacy system.**
   The adapter is loaded in its own `AssemblyLoadContext` so its dependency versions cannot
   conflict with those already loaded by `josyn-backend`. The ALC delegates loading of the
   shared adapter-contract assemblies back to the default context, preserving type identity
   for interface casts. This requires the adapter-facing interfaces to live in a dedicated,
   minimal contracts package — separate from `Backend.GlobalConfig` — that both the host and
   the adapter assembly reference. Analogous to `josyn-jap` as a contracts-only repo.

3. **Error handling at load time:** ✅ **Resolved — fail fast, no fallback.**
   If a configured adapter type cannot be found or instantiated, `josyn-backend` must not
   start. A fallback to the built-in provider would silently run the backend with missing
   or wrong context — there is no second-best strategy when elementary functionality is
   absent at startup. The error is surfaced as a hard startup failure with a clear message.

---

## Decision challenge — objections and rebuttals

This decision was stress-tested by deliberately arguing against it. Six challenges were
raised; each is preserved here with its rebuttal to document why the decision holds.

---

### Challenge 1 — "YAGNI: only config reading is needed today"

*The only concrete requirement today is reading a connection string. The full adapter pattern —
isolated `AssemblyLoadContext`, dedicated contracts package, convention folder, per-concern
interfaces — is significant engineering for a problem that may never grow beyond one
`GetValue(string key)` call.*

**Rebuttal:** The constraint that motivates the entire pattern — no compile-time dependency
on `haevg-config-reader` — exists today, not hypothetically. The pattern is the minimum viable
response to a constraint that is already in force. The ALC complexity is bounded and paid once;
the marginal cost of each subsequent adapter integration is near zero. Starting with a direct
reference and refactoring later, when real integrations arrive under time pressure, costs more
and carries migration risk. YAGNI applies to features; it does not apply to architectural
boundaries that are cheaper to establish now than to retrofit later.

---

### Challenge 2 — "YAGNI: HAEVG integration may never materialise"

*JOSYN is MIT-licensed and may be used purely standalone. The adapter machinery — contracts
package, ALC loader, convention folder — is dead weight in standalone deployments.*

**Rebuttal:** The standalone path bears zero cost from this decision. When `ContextProviderType`
is absent from the bootstrap config, no assembly is loaded, no ALC is created, and the built-in
file-based provider is used. The adapter machinery is invisible to standalone deployments.
The cost of the pattern is borne exclusively by the deployment that needs it.

---

### Challenge 3 — "Scalability: pre-loading breaks for data-driven lookups"

*If a job processes hundreds of records, each associated with a different responsible user,
those users cannot be known at session start. The pre-load-everything principle breaks down
for data-driven identity resolution.*

**Rebuttal:** This challenge conflates two distinct concerns. The adapter pattern addresses
**infrastructure context**: who is authorised to run this job, which token should the job use
to call downstream services. It does not address **business-logic data**: which users are
associated with which records — that is the job's own domain problem. A job that needs to
resolve user identity for business records should call its downstream service directly, using
the pre-loaded token as a credential. No adapter call from JAPServer or job.exe is ever needed.
The adapter and the business logic operate at different layers and must not be conflated.

---

### Challenge 4 — "Scalability: multiple adapters at startup add latency"

*As the number of adapter interfaces grows — config, identity, tokens, notifications — startup
time grows with each external service call. A slow identity service delays every session start.*

**Rebuttal:** Adapters are called once per session start, not per request. The number of
distinct adapter interfaces is expected to remain small. Where latency becomes a real concern,
the correct tool is caching within the adapter — the adapter implementation may cache results
for a configurable TTL. This is the adapter's own implementation concern; JOSYN's startup
sequence invokes the adapter and takes the result, without knowledge of how it was obtained.
No architectural change to the pattern is required.

---

### Challenge 5 — "The config-driven type name is a security risk"

*The bootstrap config names an arbitrary assembly and type. If the file is writable by an
attacker, arbitrary code executes inside the `josyn-backend` process.*

**Rebuttal:** The bootstrap config file already contains the database connection string. If it
is writable by an attacker, the connection string is compromised and the system is fully owned
regardless of adapter loading. The adapter type name does not introduce a new attack surface
that is not already present. Securing the bootstrap config file is an operational responsibility
that is independent of this decision.

---

### Challenge 6 — "A separate contracts package is premature overhead"

*Requiring a dedicated contracts NuGet package — with its own version, publish pipeline, and
release cadence — adds friction for what is currently a single interface. This is engineering
overhead without immediate payoff.*

**Rebuttal:** The contracts package is not an organisational preference — it is a technical
necessity imposed by the `AssemblyLoadContext` type-identity constraint. Without a shared
binary that both the host (default context) and the adapter (isolated context) load as the
same identity, interface casts fail at runtime. There is no lighter alternative once the
isolation decision is accepted. The package will be small and stable — analogous to `josyn-jap`,
which is similarly minimal and similarly load-bearing. Small and stable is a feature.
