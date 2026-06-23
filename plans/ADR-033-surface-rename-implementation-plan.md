# ADR-033 Implementation Plan — Surface → (Gateway · JRP · Clients) Rename & Extraction

**Date:** 2026-06-23
**Status:** Draft plan — **gated on ADR-033 acceptance** (currently *Proposed*)
**Decision record:** [`decisions/ADR-033-surface-three-concerns.md`](../decisions/ADR-033-surface-three-concerns.md)

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
| `josyn-surface` | `5870271` *wip: sync 2026-06-22* | clean | **MVP-2b is committed** with old names (`JOSYN.Surface.Contracts`, `FakeSurfaceAgent`, `CompositeSurfaceAgent`, `SurfaceCommandHandler` references). |
| `josyn-backend` | `9217971` *wip: sync 2026-06-21* | clean | Surface-agent source **is committed and tracked** (commit `9217971`): `josyn-backend-surface-agent/` with `SurfaceCommandHandler.cs`, `ChangeJobArgumentCommand.cs`, `ISurfaceCommandHandler.cs`, `.csproj`, `.local-build/*`. (Note: `wip: sync` commit dates are unreliable — verified via path-scoped `git ls-files`, not by date.) |
| `josyn-toolbox` | `6e09a96` *wip: sync 2026-06-22* | 2 unrelated mods | Surface deploy tooling (`deploy-surface.ps1`, `launch-surface.cmd`, `surface.cmd`) appears committed. In the rename ripple zone. |
| `josyn-jrp` | — | — | **Does not exist.** Must be created. |
| `josyn-platform` | `2fbe420` | clean | ADR-033 + cross-refs committed (this repo). |

---

## 2. Preconditions (Phase 0 — do not skip)

- **P-0 — Acceptance.** ADR-033 is **Accepted**. Until then, this plan is read-only reference.
- **P-1 — Re-verify the working snapshot (§1) is still current.** The renames assume
  `josyn-backend-surface-agent/` is present and committed (verified 2026-06-23, commit `9217971`) and
  that `josyn-surface` MVP-2b is committed. Re-confirm with path-scoped `git ls-files` / `git status`
  in each repo before starting — not by commit date (the `wip: sync` timestamps are unreliable).
- **P-2 — Confirm `josyn-jrp` repo conventions.** Decide Pattern A vs B
  (`architecture/repo-structure-conventions.md`), `.local-build` layout
  (`architecture/local-build.md`: `clean.cmd` + `pack.cmd`), `nuget.config` local-feed path, target
  framework, and the **version string to copy from `josyn-jap`** (AGENTS.md §8 — never invent or bump).

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

### Phase 1 — Create `josyn-jrp` (contracts only, no EXE)
1. Scaffold the repo per P-2 (Pattern, `.local-build/clean.cmd` + `pack.cmd`, `nuget.config`,
   version copied from `josyn-jap`).
2. Create **`JOSYN.Jrp.Launch`**: the `start-session` request/response records (job name, base64
   args, allocated session GUID). Small, stable, transport-safe, identity-bearing (ADR-031 DS-2
   invariants: async-shaped use, wire-safe, actor/target/correlation fields, named error taxonomy).
3. Create **`JOSYN.Jrp.Surface`**: the read queries + control commands currently in
   `JOSYN.Surface.Contracts` (`ChangeJobArgument`, `ArgumentChangeOutcome` DTO, session/argument/
   schedule read records). **No DB shape may appear here** (ADR-031 DS-4).
4. `clean` → `pack` both. Verify `.nupkg`s land in the local feed.

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

- [ ] **P-1 resolved and recorded** (backend surface-agent source located/committed/reconstructed).
- [ ] `josyn-jrp` exists; `JOSYN.Jrp.Launch` + `JOSYN.Jrp.Surface` pack cleanly; version matches the
      `josyn-jap` convention (not bumped).
- [ ] No `JOSYN.Surface.Contracts` wire records remain; clients consume `JOSYN.Jrp.*`.
- [ ] `JOSYN.Backend.Gateway` builds; `SurfaceCommandHandler` lives inside it; **no backend type
      crosses `ISurfaceAgent`/JRP**; **no DB shape in `JOSYN.Jrp.*`**.
- [ ] Full chain builds in order `jrp → backend → surface → toolbox` from a **cleared** NuGet cache.
- [ ] All **six CLI verbs** pass from the deployed EXE; `bootstrap.ini` connection sneak still resolves.
- [ ] `FakeSurfaceAgent` / `CompositeSurfaceAgent` / DS-4 exception unchanged in behaviour.
- [ ] Platform docs (Phase 6) updated; ADR-033 flipped to **Accepted** (or its plan-reference noted).

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

This is a **mechanical rename + one repo extraction**, not a feature change. The backend
surface-agent source is confirmed present and committed (§1, commit `9217971`), so the "reconstruct"
risk does not apply — Phase 3 is a straight rename. The only genuinely open variable is **P-1's
re-verification** (snapshot may age between now and the follow-up session). Everything else is bounded
by the DoD checklist above. Nothing here begins before ADR-033 is Accepted.
