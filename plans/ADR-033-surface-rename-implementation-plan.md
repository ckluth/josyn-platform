# ADR-033 Implementation Plan — Surface → (Gateway · JRP · Clients) Rename & Extraction

**Date:** 2026-06-23 (Phase 1 executed 2026-06-24)
**Status:** **In progress** — ADR-033 **Accepted**; **Phase 1 complete**, Phases 2–6 pending
**Decision record:** [`decisions/ADR-033-surface-three-concerns.md`](../decisions/ADR-033-surface-three-concerns.md)

> **Progress (2026-06-24):** ADR-033 flipped to *Accepted* (`josyn-platform@9fe13b3`). `josyn-jrp`
> created (`josyn-jrp@3c443e9`, Pattern B) with **`JOSYN.Jrp.Launch`** and **`JOSYN.Jrp.Surface`**;
> both pack cleanly to `local-packages`. **Phase 1 (§5) is done** — see the per-phase ☑ markers and
> the Phase-1 outcome notes below. Next session starts at **Phase 2**.

> ADR-033 *decides*; this plan *sequences the work*. It exists because the rename spans **three
> git repos** plus the toolbox, threads a **NuGet pack/restore chain**, and turns on a **cross-repo
> precondition** the ADR could not see. Read the ADR first for the *why*; read this for the *order*.
> This plan does not start until ADR-033 is **Accepted** by the maintainer (AGENTS.md §5).

---

## 1. Ground-truth snapshot (verified 2026-06-23, `git` inspection)

The state the next session actually starts from — not the state the 2026-06-22 session summary
described. **Re-verify before acting; this snapshot may have aged.**

| Repo | HEAD (at snapshot) | Tree | Relevant fact |
|------|--------------------|------|---------------|
| `josyn-surface` | `5870271` *wip: sync 2026-06-22* | clean | **MVP-2b is committed** with old names (`JOSYN.Surface.Contracts`, `FakeSurfaceAgent`, `CompositeSurfaceAgent`, `SurfaceCommandHandler` references). **Phase 2 target — untouched as of 2026-06-24.** |
| `josyn-backend` | `9217971` *wip: sync 2026-06-21* | clean | Surface-agent source **is committed and tracked** (commit `9217971`): `josyn-backend-surface-agent/` with `SurfaceCommandHandler.cs`, `ChangeJobArgumentCommand.cs`, `ISurfaceCommandHandler.cs`, `.csproj`, `.local-build/*`. **Phase 3 target — untouched as of 2026-06-24.** (Note: `wip: sync` commit dates are unreliable — verified via path-scoped `git ls-files`, not by date.) |
| `josyn-toolbox` | `586b022` *wip: sync 2026-06-23* | clean | Surface deploy tooling (`deploy-surface.ps1`, `launch-surface.cmd`, `surface.cmd`) appears committed. In the rename ripple zone (Phase 5). |
| `josyn-jrp` | `3c443e9` *(created 2026-06-24)* | clean | **CREATED in Phase 1.** Pattern B; `josyn-jrp-launch/` (`JOSYN.Jrp.Launch`) + `josyn-jrp-surface/` (`JOSYN.Jrp.Surface`). Both pack cleanly to `local-packages`. |
| `josyn-platform` | `9fe13b3` | clean | ADR-033 **Accepted** (flipped from Proposed in `9fe13b3`); cross-refs committed (this repo). |

---

## 2. Preconditions (Phase 0 — do not skip)

- **P-0 — Acceptance. ☑ RESOLVED (2026-06-24).** ADR-033 is **Accepted** (`josyn-platform@9fe13b3`).
- **P-1 — Re-verify the working snapshot (§1) is still current.** The renames assume
  `josyn-backend-surface-agent/` is present and committed (verified 2026-06-23, commit `9217971`) and
  that `josyn-surface` MVP-2b is committed. Re-confirm with path-scoped `git ls-files` / `git status`
  in each repo before starting Phase 2/3 — not by commit date (the `wip: sync` timestamps are unreliable).
- **P-2 — `josyn-jrp` repo conventions. ☑ RESOLVED (2026-06-24).** Decided and applied:
  **Pattern B** (two sub-solutions `josyn-jrp-launch`, `josyn-jrp-surface`, mirroring `josyn-jap`);
  `.local-build` with batch-level orchestrator at root + single-target `clean.cmd`/`build.cmd`/`pack.cmd`
  per sub-solution; `nuget.config` local feed `..\..\local-packages`; TFM **`net10.0`**; **version
  `1.0.0-preview01`** copied from `josyn-jap` (not invented/bumped, AGENTS.md §8).

---

## 3. Decisions adopted from ADR-033's open questions

| ADR-033 Q | Plan adopts |
|-----------|-------------|
| **Q1** packaging granularity | **Two packages** — `JOSYN.Jrp.Launch` and `JOSYN.Jrp.Surface` — so a thin client (`SessionClient`) binds only `Launch`. Decided here, not deferred. |
| **Q2** client seam name | **Keep `ISurfaceAgent`** for now (client concern). Revisit at the `HttpAgent`/JRP-client phase. Out of scope for this rename. |
| **Q3** rename timing | **Moot** — MVP-2b is already committed (§1). This is a normal forward rename with its own commit(s). |
| **Q4** future non-surface JRP verbs | Out of scope. The two-package split already accommodates them. |

---

## 4. Cross-repo dependency & build order (the NuGet chain)

The rename inserts `josyn-jrp` as a **new upstream link**. Stale `.nupkg` was the root cause of the
2026-06-22 build break (AGENTS.md §8); extending the chain multiplies that risk. **Always
`clean.cmd` then `pack.cmd`, in dependency order, before any downstream restore.**

```
josyn-jrp  (NEW, contracts)            ── pack ──┐
                                                 ▼
josyn-backend  (Gateway + JobRegistry)  ── pack ──┐   (Gateway consumes JOSYN.Jrp.*)
                                                  ▼
josyn-surface  (CLI/clients)            ── restore/build   (clients consume JOSYN.Jrp.*)
                                                  ▼
josyn-toolbox  (deploy-surface)         ── re-deploy
```

**Mandatory ordering for every rebuild:** `josyn-jrp` → `josyn-backend` → `josyn-surface` →
`josyn-toolbox`. Clearing the **user NuGet cache** for each repacked package id is required
(AGENTS.md §8): `JOSYN.Jrp.Launch`, `JOSYN.Jrp.Surface`, and any backend package whose contract set
changed.

---

## 5. Work breakdown (phased, ordered)

### Phase 1 — Create `josyn-jrp` (contracts only, no EXE) — ☑ COMPLETE (2026-06-24)
1. ☑ Scaffold the repo per P-2 (Pattern B, `.local-build/clean.cmd` + `pack.cmd`, `nuget.config`,
   version `1.0.0-preview01` copied from `josyn-jap`).
2. ☑ Create **`JOSYN.Jrp.Launch`**: the `start-session` request/response records (job name, base64
   args, allocated session GUID). Small, stable, transport-safe, identity-bearing (ADR-031 DS-2
   invariants: async-shaped use, wire-safe, actor/target/correlation fields, named error taxonomy).
3. ☑ Create **`JOSYN.Jrp.Surface`**: the read queries + control commands currently in
   `JOSYN.Surface.Contracts` (`ChangeJobArgument`, `ArgumentChangeOutcome` DTO, session/argument/
   schedule read records). **No DB shape may appear here** (ADR-031 DS-4).
4. ☑ `clean` → `pack` both. Verified `.nupkg`s land in `local-packages`
   (`JOSYN.Jrp.Launch.1.0.0-preview01.nupkg`, `JOSYN.Jrp.Surface.1.0.0-preview01.nupkg`).

> **Phase-1 outcome notes (what was actually built — read before Phase 2):**
> - **`JOSYN.Jrp.Launch`** holds: `JrpTarget` (the renamed `SurfaceTarget`), `JrpErrorCategory`
>   (the renamed `SurfaceErrorCategory`), `StartSessionRequest`, `StartSessionResponse`.
>   `JrpErrorCategory` was placed **here, not in `.Surface`**, so both contract families share one
>   error vocabulary; a launch-only consumer still gets the taxonomy without the dashboard contract.
>   References `JOSYN.Jap.Contract` only.
> - **`JOSYN.Jrp.Surface`** holds: `ChangeJobArgument` (in `Commands/`), the 5 queries (in `Queries/`,
>   sub-namespace `JOSYN.Jrp.Surface.Queries`), the 7 DTOs (in `Dtos/`: `ArgumentChangeOutcome`,
>   `ArgumentSummary`, `ErrorDetail`, `JobArguments`, `JobSchedule`, `RegisteredJobSummary`,
>   `ScheduleEntrySummary`, `SessionSummary`), and `JrpError` (the renamed `SurfaceError` factory,
>   now producing `JrpErrorCategory`-prefixed messages). References `JOSYN.Jrp.Launch`,
>   `JOSYN.Foundation.ResultPattern`, `JOSYN.Backend.Contracts` (for `ExecutionStatus`).
> - **Name map for Phase 2/3 rewiring** — surface clients must repoint to these:
>   `SurfaceTarget`→`JrpTarget`, `SurfaceError`→`JrpError`, `SurfaceErrorCategory`→`JrpErrorCategory`,
>   namespace `JOSYN.Surface.Contracts`→`JOSYN.Jrp.Surface` (+ `JOSYN.Jrp.Surface.Queries` for queries,
>   `JOSYN.Jrp.Surface.Commands` for `ChangeJobArgument`) and `JOSYN.Jrp.Launch` for `JrpTarget`.
> - **No test project** was created (the contracts are pure records, mirroring `josyn-jap`'s
>   `JOSYN.Jap.Contract` which also ships test-less). Behavioural coverage stays in `josyn-surface`'s
>   existing test project, which Phase 2 repoints onto `JOSYN.Jrp.*`.
> - `ISurfaceAgent` was **not** touched — it stays a client seam in `josyn-surface` (Q2, §3, deferred).

### Phase 2 — Relocate wire contracts off `josyn-surface` / backend onto `josyn-jrp`
5. In `josyn-surface`: delete the relocated records from `JOSYN.Surface.Contracts`; add a package
   reference to `JOSYN.Jrp.*`; repoint `ISurfaceAgent`, `FakeSurfaceAgent`, `CompositeSurfaceAgent`,
   and the CLI to the `JOSYN.Jrp.*` types. `ISurfaceAgent` **stays** in `josyn-surface` (client seam).
6. Keep the explicit DB→DTO mapping inside `FakeSurfaceAgent` (throwaway DS-4) — it now maps to
   `JOSYN.Jrp.Surface` DTOs. No backend type crosses `ISurfaceAgent`.

> **Gate:** Re-confirm the §1 snapshot (P-1) is still current before starting Phase 3.

### Phase 3 — Rename the platform host: `SurfaceAgent` → `Gateway`
7. Rename `josyn-backend-surface-agent` → `josyn-backend-gateway`; assembly/namespace
   `JOSYN.Backend.SurfaceAgent` → **`JOSYN.Backend.Gateway`**.
8. Move `SurfaceCommandHandler` (the platform write path) into `JOSYN.Backend.Gateway`; repoint it to
   consume `JOSYN.Jrp.*` request/response types.
9. Keep `IJobArgumentWriter` / `ArgumentChangeOutcome` (backend-side) where they belong in
   `JOSYN.Backend.JobRegistry`; the Gateway maps backend types to `JOSYN.Jrp.Surface` DTOs explicitly
   (no backend type crosses JRP). `clean` → `pack` backend.

### Phase 4 — Rewire `josyn-surface` clients
10. Build `josyn-surface` against the repacked `josyn-jrp` (+ backend if the in-process
    `CompositeSurfaceAgent` references the Gateway write path directly). Confirm all six CLI verbs.

### Phase 5 — Toolbox / deploy ripple
11. Audit `deploy-surface.ps1`, `launch-surface.cmd`, `surface.cmd`, and the **`josyn.bootstrap.ini`
    connection sneak** for references to renamed assemblies/paths. Update, re-deploy
    `C:\ProgramData\JOSYN\Surface\`, re-run the six-verb smoke test from the deployed EXE.

### Phase 6 — Platform docs (this repo)
12. Update `repos/` summaries (add `josyn-jrp`; reflect Gateway in the backend summary),
    `architecture/overview.md`, and — if package-state tables list it — `architecture/storage.md`.
    (`naming-conventions.md` and `ROADMAP.md` already updated in `2fbe420`.)

---

## 6. Definition of Done

- [x] **P-1 resolved and recorded** — backend surface-agent source located & committed (`9217971`).
- [x] `josyn-jrp` exists; `JOSYN.Jrp.Launch` + `JOSYN.Jrp.Surface` pack cleanly; version matches the
      `josyn-jap` convention (`1.0.0-preview01`, not bumped).
- [ ] No `JOSYN.Surface.Contracts` wire records remain; clients consume `JOSYN.Jrp.*`. *(Phase 2)*
- [ ] `JOSYN.Backend.Gateway` builds; `SurfaceCommandHandler` lives inside it; **no backend type
      crosses `ISurfaceAgent`/JRP**; **no DB shape in `JOSYN.Jrp.*`**. *(Phase 3 — JRP side already clean)*
- [ ] Full chain builds in order `jrp → backend → surface → toolbox` from a **cleared** NuGet cache.
- [ ] All **six CLI verbs** pass from the deployed EXE; `bootstrap.ini` connection sneak still resolves.
- [ ] `FakeSurfaceAgent` / `CompositeSurfaceAgent` / DS-4 exception unchanged in behaviour.
- [ ] Platform docs (Phase 6) updated; ADR-033 flipped to **Accepted** ✅ *(done 2026-06-24)* — docs pending.

---

## 7. Traps carried forward (from the 2026-06-22 session)

- **Stale `.nupkg`** — the original break. `clean` **then** `pack`, in chain order; clear the user
  cache per package id (AGENTS.md §8). Never bump versions.
- **`ArgumentChangeOutcome` in two namespaces** — backend (`JOSYN.Backend.JobRegistry`) and the wire
  DTO (now `JOSYN.Jrp.Surface`). The Gateway maps explicitly; use a `using` alias in tests.
- **`change-argument` is change-only** — returns `[NotFound]` if job *or* argument record is absent
  (typo guard, ADR-030 D-9). It never creates.
- **Demo-data baseline** — `Contoso.DemoProduct.DemoJob / default`: smoke tests mutate then restore.
- **`surface.cmd`** needs `josyn-toolbox\convenience-scripts` on PATH and a prior `launch-surface.cmd`.

---

## 8. Deferred (not this plan)

- **Q2** — renaming `ISurfaceAgent` to a JRP-client seam: at the `HttpAgent` phase.
- **`HttpAgent`** — the real JRP transport (Windows Service/daemon hosting the Gateway over HTTPS);
  retires the `bootstrap.ini` sneak, `FakeSurfaceAgent`, and `CompositeSurfaceAgent` (ADR-031 DS-5).
- **`CreateJobArgument`**, **MVP-3 schedule writes**, **Blazor `JOSYN.Surface.Web`**,
  **`JOSYN.Surface.SessionClient`** (binds only `JOSYN.Jrp.Launch`).

---

## 9. Continuation note

This is a **mechanical rename + one repo extraction**, not a feature change. **Phase 1 is done**
(`josyn-jrp` created, both packages pack cleanly, ADR-033 Accepted). The next session **starts at
Phase 2**: gut the wire records from `JOSYN.Surface.Contracts` and repoint the surface clients +
tests onto `JOSYN.Jrp.*` using the **name map in the Phase-1 outcome notes** (§5). The backend
surface-agent source is confirmed present and committed (§1, commit `9217971`), so Phase 3 is a
straight rename. Re-verify **P-1** (snapshot may age) before touching `josyn-surface`/`josyn-backend`.
Everything else is bounded by the DoD checklist above.
