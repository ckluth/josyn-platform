# MVP-2 Handoff — josyn-surface (session close, 2026-06-21)

**Purpose:** Cold-start brief for the next session. The MVP-2b implementation is **complete and
verified but NOT yet committed** in either repo. This document records exactly what was built,
what state the working trees are in, and what the next session should do first.

**Companion documents:**
- `ADR-030-031-surface-mvp2-plan.md` — the governing design/plan (weighing, decisions Q-A/Q-B, build order).
- `ADR-030-031-surface-mvp1-implementation.md` — MVP-1 record.
- `decisions/ADR-030-josyn-surface.md`, `decisions/ADR-031-surface-delivery-strategy.md` — the ADRs.

---

## ⟢ Status at session close

| Item | State |
|------|-------|
| **MVP-2a** (read verbs: `jobs`, `arguments`, `schedule`) | ✅ SHIPPED — `josyn-surface` commit `eff997e`, pushed |
| **MVP-2b** (`change-argument` — first platform write) | ✅ IMPLEMENTED + VERIFIED, ⏳ **UNCOMMITTED** in both repos |
| Build | ✅ 0 warnings, 0 errors |
| Tests | ✅ 20/20 pass |
| Manual smoke-test | ✅ Before/After correct; `[NotFound]` on unknown job & unknown arg; demo data restored |
| **MVP-3** (schedule writes) | ⛔ DEFERRED |

> ⚠️ Per AGENTS.md §5 the confirmation gate applies: the next session must get an explicit
> go-ahead from the maintainer **before** running any `git commit` / `push`.

---

## ⟢ What was built (MVP-2b)

The platform's **first honest write**: change the content of an existing job-argument record,
end-to-end from CLI to SQL, through a real platform-resident command handler (DS-5 — *not* via
FakeAgent's direct-DB access, which stays read-only by construction, DS-4).

### Architecture — the DS-4/DS-5 seam

```
CLI: change-argument <job> <arg> <content|@file>
  └─> ISurfaceAgent.ChangeJobArgument(...)            (JOSYN.Surface.Contracts)
        └─> CompositeSurfaceAgent                     (JOSYN.Surface.FakeAgent)
              ├─ reads  ──> FakeSurfaceAgent          (DS-4 throwaway, read-only helper)
              └─ writes ──> ISurfaceCommandHandler    (DS-5, platform-resident)
                              └─> SurfaceCommandHandler   (JOSYN.Backend.SurfaceAgent)
                                    └─> IJobArgumentWriter.ChangeArgument(...)
                                          └─> SqlJobRegistry  (JOSYN.Backend.JobRegistry)
                                                └─> SaveChanges()
```

Key discipline: `FakeSurfaceAgent` is now a **read-only helper** (no longer implements
`ISurfaceAgent`). `CompositeSurfaceAgent` is the single full `ISurfaceAgent` — reads delegate to the
fake, writes delegate to the platform handler. The fake-read exception is never stretched to cover
mutations.

### `josyn-backend` — uncommitted changes

**`josyn-backend-job-registry/` (modified):**
- `JOSYN.Backend.JobRegistry/Contracts/ArgumentChangeOutcome.cs` *(new)* — record `{ JobName, ArgumentName, Before, After }`.
- `JOSYN.Backend.JobRegistry/Contracts/IJobArgumentWriter.cs` *(new)* — write-side interface, **separate** from read-focused `IJobRegistry`; `Result<ArgumentChangeOutcome> ChangeArgument(jobName, argumentName, content)`.
- `JOSYN.Backend.JobRegistry/SqlJobRegistry.cs` *(modified)* — now implements `IJobArgumentWriter` too. `ChangeArgument`: guard job exists → guard arg exists (change-only, **never creates**) → capture Before → update Content → `SaveChanges()` → return outcome.

**`josyn-backend-surface-agent/` (new Pattern-B library project):**
- `JOSYN.Backend.SurfaceAgent.slnx`, `nuget.config` (→ `..\..\local-packages`)
- `JOSYN.Backend.SurfaceAgent/JOSYN.Backend.SurfaceAgent.csproj` — version `1.0.0-preview01`; refs `JOSYN.Foundation.ResultPattern` + `JOSYN.Backend.JobRegistry`.
- `ChangeJobArgumentCommand.cs` — full envelope `{ Actor, Environment, Machine, CorrelationId, JobName, ArgumentName, Content }` (fields present from day 1; enforcement deferred).
- `ISurfaceCommandHandler.cs` + `SurfaceCommandHandler.cs` — takes a connection string, builds `SqlJobRegistry` as `IJobArgumentWriter`, delegates.
- `.local-build/{build,clean,pack}.cmd` — mirror job-registry; `clean.cmd` purges `josyn.backend.surfaceagent` from the NuGet cache.

### `josyn-surface` — uncommitted changes

**`JOSYN.Surface.Contracts/`:**
- `Dtos/ArgumentChangeOutcome.cs` *(new)* — surface-native DTO; same shape by value, **no backend type crosses the seam**.
- `Commands/ChangeJobArgument.cs` *(new)* — surface command `{ Target, Actor, CorrelationId, JobName, ArgumentName, Content }`.
- `ISurfaceAgent.cs` *(modified)* — added `Task<Result<ArgumentChangeOutcome>> ChangeJobArgument(ChangeJobArgument, CancellationToken)` (now 5 reads + 1 write; all async + CT per DS-2).

**`JOSYN.Surface.FakeAgent/`:**
- `CompositeSurfaceAgent.cs` *(new)* — the single full `ISurfaceAgent`; maps backend → surface outcome explicitly.
- `FakeSurfaceAgent.cs` *(modified)* — dropped `: ISurfaceAgent`; now a read-only helper.
- `JOSYN.Surface.FakeAgent.csproj` *(modified)* — added `JOSYN.Backend.JobRegistry` + `JOSYN.Backend.SurfaceAgent` package refs.

**`JOSYN.Surface.Cli/SurfaceCli.cs`** *(modified)* — constructs `CompositeSurfaceAgent(FakeSurfaceAgent, SurfaceCommandHandler)`; added `change-argument` verb with `@file` content support; added `RenderArgumentChange`; updated usage text (6 verbs).

**`JOSYN.Surface.Test/ChangeJobArgumentTests.cs`** *(new)* — 5 tests: envelope shape, DTO transport-safety, success mapping, NotFound propagation, full-envelope pass-through (via `CaptureCommandHandler` stub). Uses `using SurfaceOutcome = JOSYN.Surface.Contracts.ArgumentChangeOutcome;` to disambiguate.

### Local NuGet feed (`C:\DevGit\local-packages\`)
- `JOSYN.Backend.JobRegistry.1.0.0-preview01.nupkg` — repacked (new interface + outcome).
- `JOSYN.Backend.SurfaceAgent.1.0.0-preview01.nupkg` — new package.

---

## ⟢ Continuation point — do this next

### 1. Commit the uncommitted work (after maintainer go-ahead)

Two repos, both with staged-and-untracked MVP-2b changes. Suggested order: **backend first**
(surface depends on its packages), then surface.

**`josyn-backend`** — currently on `wip: sync` commits (`5a0bdd0`). Files to add:
```
 M josyn-backend-job-registry/JOSYN.Backend.JobRegistry/SqlJobRegistry.cs
 ?? josyn-backend-job-registry/JOSYN.Backend.JobRegistry/Contracts/ArgumentChangeOutcome.cs
 ?? josyn-backend-job-registry/JOSYN.Backend.JobRegistry/Contracts/IJobArgumentWriter.cs
 ?? josyn-backend-surface-agent/
```
Suggested message: `Add IJobArgumentWriter + JOSYN.Backend.SurfaceAgent (surface write path)`

**`josyn-surface`** — on `eff997e`. Files to add:
```
 M JOSYN.Surface.Cli/SurfaceCli.cs
 M JOSYN.Surface.Contracts/ISurfaceAgent.cs
 M JOSYN.Surface.FakeAgent/FakeSurfaceAgent.cs
 M JOSYN.Surface.FakeAgent/JOSYN.Surface.FakeAgent.csproj
 ?? JOSYN.Surface.Contracts/Commands/
 ?? JOSYN.Surface.Contracts/Dtos/ArgumentChangeOutcome.cs
 ?? JOSYN.Surface.FakeAgent/CompositeSurfaceAgent.cs
 ?? JOSYN.Surface.Test/ChangeJobArgumentTests.cs
```
Suggested message: `Add MVP-2b: change-argument (first platform write via CompositeSurfaceAgent)`

> Both messages should carry the standard `Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>` trailer.

### 2. Documentation follow-ups (platform repo)
- **`ROADMAP.md`** — register MVP-2 (2a + 2b) as a complete milestone.
- **`architecture/storage.md`** — stale: JobRegistry is **Existing** (not "Planned"); add a `JobScheduleStore` row.
- **`ADR-030-031-surface-mvp2-plan.md`** — flip MVP-2b status line from "DESIGNED & CONFIRMED" to "SHIPPED" once committed.

### 3. Future increments (not started)
- **`CreateJobArgument`** — the distinct *create* command (the other half of the Q-A decision; MVP-2b deliberately did change-only).
- **MVP-3 — schedule writes** (`AddScheduleEntry`, `SuspendSchedule`, `ResumeSchedule`): harder semantics (child collections, cron validation). Deferred.
- **HttpAgent phase** — stand up the EXE host/transport (Windows Service/daemon + HTTPS REST) wrapping the *same* `SurfaceCommandHandler`; surface swaps `CompositeSurfaceAgent` for an `HttpAgent`. Nothing above `ISurfaceAgent` changes.

---

## ⟢ Traps & invariants for the next agent

- **NuGet (AGENTS.md §8):** version is **never** bumped — `1.0.0-preview01` throughout. When repacking: run `clean.cmd NOPAUSE` (purges `%USERPROFILE%\.nuget\packages\…`) **before** `pack.cmd NOPAUSE`. The surface must restore **after** the backend packs, or it binds a stale cached `.nupkg` missing the new types.
- **`ArgumentChangeOutcome` lives in two namespaces** — `JOSYN.Backend.JobRegistry` (backend return) and `JOSYN.Surface.Contracts` (durable surface DTO). `CompositeSurfaceAgent` maps backend → surface explicitly; no backend type may cross `ISurfaceAgent`. In tests, alias one to disambiguate.
- **Result API (`JOSYN.Foundation.ResultPattern`):** `Result<T>.Success(value)`, `Result<T>.Fail(message)`, implicit `Error`/`Exception` → `Result<T>`, `.ToResult<TOther>()` to re-type a failure. There is **no** `.Failure()` method.
- **`change-argument` is change-only** — returns `[NotFound]` if the job OR the argument is absent; it never silently creates a record (guards against typos).
- **`josyn-backend-surface-agent/nuget.config`** points two levels up (`..\..\local-packages`) — it sits one folder deeper than the component root.
- **Demo data state:** the smoke-test mutated then **restored** `Contoso.DemoProduct.DemoJob / default` to `Message=Hello contoso-demo-job`, `RepeatCount=0`, `ScheduledFor=19.06.2026`, `IsHighPriority=False`, `Budget=0,00`. Dev DB is clean.
