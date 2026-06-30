# Current Plan — Surface / JRP / Gateway

> **Purpose.** A single public hand-off for the "surface → JRP → Gateway host" work stream.
> It names every existing reference, then states **what is done, what is left, and what is next**,
> so any future session can continue cold without reconstructing context.
>
> **Last updated:** 2026-06-26
> **Active work item:** Part C item #1 of the ADR-033 follow-up — the **JRP Gateway host EXE**.

---

## 1. Reference map (all documents on this topic)

### Decisions (`josyn-platform/decisions/`)

| ADR | Title | Status | Role in this stream |
|---|---|---|---|
| **ADR-030** | josyn-surface: The Human Window into a Headless Platform | Accepted (2026-06-21) | Origin: why a surface exists at all. |
| **ADR-031** | Surface Delivery Strategy (CQRS-lite, Agent Seam, MVP Phasing) | Accepted (2026-06-21) | `ISurfaceAgent` seam; DS-2 "design the seam for the boundary it will cross" (reason `JrpTarget` exists early). |
| **ADR-032** | No Standalone Listener: Session-Start Folds into the Surface Agent | Accepted | Killed the separate listener; session-start lives in the agent/host path. |
| **ADR-033** | josyn-surface is Three Concerns: the JRP Gateway, JRP Contracts, and Edge Clients | Accepted | **Parent.** Names **JRP**, the per-machine **Gateway** (T-1), the contracts-only `josyn-jrp` repo (T-2), the future `HttpAgent` (T-3). Transport = HTTPS/REST. |
| **ADR-034** | JRP HTTP Binding: Hosting Stack and Contract Authority | **Accepted (2026-06-26)** | **How** JRP is served: ASP.NET Core Minimal API + `IEndpoint`, OpenAPI projection + Scalar, API versioning; `josyn-jrp` is sole contract authority; Kiota rejected; explicit static endpoint registration (no reflection/DI). |
| **ADR-035** | JRP Addressing, Topology, and Bootstrap | **Accepted (2026-06-26)** | **Which server** a client talks to: peer model (no coordinator), env-scoped data verbs vs node-specific execution, one DB per env, env-local topology, DNS bootstrap seed, per-node addressing = hostname (D-7). |

### Plans (`josyn-platform/plans/`)

| Plan file | Role |
|---|---|
| `ADR-030-031-surface-mvp1-implementation.md` | Surface MVP1 build. |
| `ADR-030-031-surface-mvp2-plan.md` / `-mvp2b-session-summary.md` / `-mvp2-handoff.md` | Surface MVP2 build + hand-offs. |
| `ADR-033-surface-rename-implementation-plan.md` | The josyn-surface → JRP/Gateway/edge rename execution. |
| `ADR-033-followup-plan.md` | **The immediate predecessor.** Parts A+B executed; **Part C item #1 (Gateway host) is the work tracked here.** |
| **`current-plan-surface-jrp-gateway.md`** | **This file** — the live continuation plan. |

### Code (the host under construction)

- Repo: `josyn-backend`, sub-folder **`josyn-backend-gateway-host`** (Pattern B EXE, not packed).
- Project: `JOSYN.Backend.Gateway.Host` (`Microsoft.NET.Sdk.Web`, net10.0-windows).
- Contract repo consumed: **`josyn-jrp`** (`JOSYN.Jrp.Launch`, `JOSYN.Jrp.Surface`) — unchanged, no version bump.

---

## 2. The arc in one paragraph

`josyn-surface` was re-conceived (ADR-033) as three concerns: the **JRP contracts** (`josyn-jrp`,
records-as-contract), the **Gateway** (a platform-resident, per-machine EXE that serves the JRP API
over HTTPS/REST, owns `start-session`, and reads from the real backend stores), and **edge clients**.
ADR-034 then fixed *how* JRP is served (hosting stack + contract authority); ADR-035 fixed *which
server* a client addresses (peer model, env-scoped reads vs node-specific launch, DNS bootstrap). The
current build stands up that Gateway host EXE.

---

## 3. What is DONE

**Decisions (ratified 2026-06-26)**
- ADR-034 and ADR-035 authored, rubber-duck-reviewed, four design decisions (A/B/C/D) folded in, and
  **flipped to Accepted**. Cross-references wired between ADR-033 ↔ 034 ↔ 035.
- The four confirmed decisions now baked into the ADRs:
  - **A** — `start-session` rejects a launch whose `Target.Machine ≠ this host` (ADR-035 D-2).
  - **B** — `Environment` validated on **every** verb; responses carry the real serving env (ADR-035 D-2).
  - **C** — per-node addressing: `Machine` *is* the DNS hostname, `https://<host>:<well-known port>`
    by convention; topology stores identity only (ADR-035 **D-7**).
  - **D** — endpoints registered by an **explicit static list**, no reflection/DI (ADR-034 D-2).
- `docs/docs-index.json` deprecated platform-wide (AGENTS.md §6 note).
- Carried-over fix from prior session: `StartSessionResponse` emptied (GUID allocated in
  SessionBroker, not pre-allocated by Gateway); `JOSYN.Jrp.Launch` repacked.

**Backend store extensions (G-1)** — packages extended + repacked (no version bump):
- `SessionStore`: `ISessionStore.GetRecentSessions(int)` + impl.
- `ErrorHandler`: read seam `IErrorReadStore.GetByUid(Guid)` + impl (⚠ softens ADR-011B-01 write-only
  stance — flag for a doc note / mini-ADR at review).

**Gateway host scaffold + handlers (G-2 / G-3 / G-4)** — builds green:
- Project, `.slnx`, `nuget.config`, Pattern B `.local-build` scripts.
- `Program.cs`, `HostFactory.cs`, `IEndpoint.cs`.
- 5 read handlers: `GetRegisteredJobsHandler`, `GetJobArgumentsHandler`, `GetJobScheduleHandler`,
  `GetRecentSessionsHandler`, `GetErrorDetailHandler`.
- `StartSessionHandler` (fire-and-forget; `SessionLauncher.LaunchSession` static; empty response).

**`EndpointRegistry.cs` rewrite + G-5 Endpoints + G-6 Config** — builds green, all 7 verbs live:
- `EndpointRegistry.All` replaced by `MapEndpoints(GatewayStartup)` — explicit instantiation, no reflection.
- `GatewayStartup.cs` record: connection string, backend root, env, listen URL.
- `JrpHttpResult.cs`: `Result<T>` → `IResult`; `JrpErrorCategory` → HTTP status (D-7).
- `TargetValidation.cs`: `ParseAndValidate` (query params) + `ValidateEnvironment` + `ValidateMachine`.
- 7 endpoint files (one per JRP verb): `GetRegisteredJobsEndpoint`, `GetRecentSessionsEndpoint`,
  `GetJobArgumentsEndpoint`, `GetJobScheduleEndpoint`, `GetErrorDetailEndpoint`,
  `ChangeJobArgumentEndpoint`, `StartSessionEndpoint`.
- `Program.cs` updated: loads `FileBootstrapConfig`, validates `RuntimeEnvironment`, falls back to
  `https://localhost:5001` for missing `GatewayListenUrl`.
- `FileBootstrapConfig` + `IBootstrapConfig` extended with optional `RuntimeEnvironment?` and
  `GatewayListenUrl?`; repacked at same version. INI example updated.
- `HostFactory.cs` updated: accepts `GatewayStartup`, sets listen URL via `UseUrls`.

**G-7 Verify** — all 7 verbs smoke-tested against DEV DB:
- `GET /v1/jobs` → 200 (live job data) ✅
- `GET /v1/sessions?maxCount=5` → 200 (3 sessions) ✅
- `GET /v1/jobs/{job}/arguments` → 200 (argument record) ✅
- `GET /v1/jobs/{job}/schedule` → 200 (schedule + entries) ✅
- `GET /v1/errors/{uid}` → 404 `[NotFound]` ✅
- `PUT /v1/jobs/{job}/arguments/{arg}` → 200 before/after ✅
- `POST /v1/sessions` wrong machine → 400 machine mismatch ✅
- `POST /v1/sessions` correct machine → validation passed; 500 no EXE in dev layout (expected) ✅
- OpenAPI `/openapi/v1.json` served ✅
- Environment mismatch → 400 ✅

**G-8 Docs** — all four targets updated:
- `architecture/overview.md`: Gateway Host added to Component Map + dependency chain.
- `architecture/naming-conventions.md`: Gateway vocabulary, namespace tree, assembly table updated.
- `repos/josyn-backend/overview.md`: directory tree, planned components (✅), build notes, sanity notes.
- `ROADMAP.md`: M6 status, What's done, What's in progress, What's next all current.

---

## 4. In progress / PAUSED

*(nothing — all items through G-8 are complete)*

---

## 5. What is LEFT / NEXT (gated — AGENTS.md §5 confirmation required)

- **G-9  Commit (gated)** — dependency order: `josyn-backend` (BootstrapConfig repack + host) →
  `josyn-platform` (docs + plan). §5 go-ahead per repo.
- **MVP-3  HttpAgent** — retire `FakeAgent` / `CompositeSurfaceAgent` / `bootstrap.ini` sneak
  (ADR-031 DS-5). Implement `ISurfaceAgent` as a thin hand-written `HttpClient` over `JOSYN.Jrp.*`
  packages (ADR-034 D-6). Add `JOSYN.Surface.SessionClient` (binds `JOSYN.Jrp.Launch` only).
  Add `CreateJobArgument` verb. MVP-3 schedule writes. Requires the Gateway host to be deployed
  and reachable.

---

## 6. Blockers & open dependencies

- **Job-registry placement gap (hard blocker for `start-session` candidate resolution).** The registry
  records *that* a job is registered, not *on which servers it is installed*. ADR-035 D-4/D-5 depend on
  a per-job **install set**. Tracked as separate later work — **name it, do not build it here.**
- **Pre-existing drift (not in this stream's scope):** ADR-033's header says `Accepted` but its
  Continuation Brief still reads "*State of play. Proposed.*" — flag for the maintainer.

---

## 7. Key technical facts for a cold start

- **Addressing model (ADR-035):** peer model, NO master/router (a "directory" answering *who are the
  peers* is allowed; a *router* that forwards is forbidden). ONE DB per environment → data verbs are
  env-scoped (any Gateway answers identically); `start-session` is node-specific. Two-phase launch:
  env-scoped candidate read → client picks one by its own policy → node-specific execute. Bootstrap =
  naming convention + DNS (`josyn-<env>` → ≥1 live Gateway); env DB owns full membership. Per-node
  endpoint = `https://<Machine>:<well-known JRP port>`, `Machine` = DNS hostname.
- **Contract authority (ADR-034):** `josyn-jrp` records are the single source of truth; OpenAPI is a
  *bounded* (no-type-drift) projection, not a definer; host-phase acceptance check = every verb has one
  endpoint referencing its `JOSYN.Jrp.*` types + `JrpError`→status metadata reviewed. Kiota rejected;
  `HttpAgent` (deferred, ADR-033 T-3) is a thin hand-written client over the shared packages.
- **Read mapping reference:** `josyn-surface/JOSYN.Surface.FakeAgent/FakeSurfaceAgent.cs` (DS-4
  throwaway) — reuse its 5 read mappings + the `ExecutionStatus → SessionStatus` switch; replicate at
  the host edge so no backend enum crosses JRP.
- **Build/code quirks:** `Microsoft.NET.Sdk.Web` SDK; `AddApiVersioning()` returns
  `IApiVersioningBuilder` (call separately, cannot fluently chain `IServiceCollection` ext);
  `SessionLauncher.LaunchSession` is **static**, returns plain `Result` (fire-and-forget, no GUID);
  class/namespace collisions need `using X = Namespace.Class;` aliases (`SessionLauncher`,
  `SessionStore`, `ISessionStore`); `.local-build` `.cmd` scripts reject unknown args — call with none;
  AGENTS.md §8: never bump `<Version>`, always `clean.cmd` before `pack.cmd`.

---

## 8. Governance reminders

- Every write + git op is subject to the **AGENTS.md §5 confirmation gate**. ADRs A/B/C/D were
  confirmed and are applied; status flips were the maintainer's explicit act.
- All artefacts in English (AGENTS.md §4).
- Excluded repos (`josyn-docs`, `josyn-guide`) must never be touched.
