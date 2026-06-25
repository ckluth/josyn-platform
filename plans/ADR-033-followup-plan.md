# ADR-033 Follow-up Plan — Remediation, Commits, and Deferred Work

> **Status:** proposed (awaiting maintainer go-ahead per AGENTS.md §5).
> **Predecessor:** [`ADR-033-surface-rename-implementation-plan.md`](ADR-033-surface-rename-implementation-plan.md)
> — its six phases (rename + repo extraction) are **executed and verified**
> (josyn-surface builds 0/0, 20/20 tests, all six CLI verbs smoke-tested live; `JOSYN.Jrp.*`
> and `JOSYN.Backend.Gateway` packed to `local-packages`). That plan's own ☑ markers are stale
> — see Phase F-4 below.
>
> This document captures the gaps a rubber-duck review surfaced **after** execution, plus the
> commit sequencing and the work the original plan deliberately deferred (§8 of the predecessor).

---

## 0. Why this exists

The original plan was a clean mechanical rename. Execution went green, but a post-hoc review
found seven gaps the plan either created or never accounted for, none of which break the current
build but all of which will bite the next maintainer or a fresh clone:

1. A **contract-layering contradiction** baked into the predecessor plan itself.
2. **Backend repo-root build scripts** that silently skip the new Gateway package.
3. **Commit sequencing** that can produce a broken fresh clone if done out of order.
4. A **stale retired `.nupkg`** left in the shared feed — the exact §7 trap.
5. **Canonical docs** (ADR plan markers, architecture overview, ROADMAP, surface README) that
   now disagree with the executed reality.
6. **Gateway documented as an EXE/host** but implemented as a library only.
7. **`josyn-jrp` absent from the sanity protocol** despite being a source-of-truth contracts repo.

Plus the genuinely-deferred forward work (HttpAgent, SessionClient, write-path expansion).

---

## 1. Ground truth (re-confirm before acting)

Working trees are **uncommitted** in four repos. Re-run before touching anything:

```
git -C C:\DevGit\josyn-jrp      status --short   # expected: clean (Phase 1, already committed)
git -C C:\DevGit\josyn-surface  status --short   # Phase 2+3 edits (modified/deleted/untracked)
git -C C:\DevGit\josyn-backend  status --short   # surface-agent deleted, gateway untracked (rename)
git -C C:\DevGit\josyn-toolbox  status --short   # deploy-surface.ps1 comment
git -C C:\DevGit\josyn-platform status --short   # docs-index, backend overview, josyn-jrp.md, this file
```

`local-packages` currently holds **both** `JOSYN.Backend.Gateway.1.0.0-preview01.nupkg` and the
orphaned `JOSYN.Backend.SurfaceAgent.1.0.0-preview01.nupkg`.

> **Out of scope (maintainer instruction):** `josyn-docs` and `josyn-guide`.
> **NuGet discipline (AGENTS.md §8):** never bump versions; `clean.cmd` (clears user cache) then
> `pack.cmd`, in dependency order.

---

## Part A — Remediation (gaps 1–7)

### Phase F-1 — Backend repo-root build/clean/pack include Gateway  *(gap #2 — Critical)*

The root `C:\DevGit\josyn-backend\.local-build\{build,pack,clean}.cmd` enumerate sub-solutions by
explicit name. None list `josyn-backend-surface-agent` (old) **or** `josyn-backend-gateway` (new),
so a maintainer running the **repo-root** pack silently does not rebuild Gateway, and may publish a
stale downstream. `repos/josyn-backend/overview.md` currently *claims* the root pack includes
Gateway — that claim is false until this phase lands.

Steps:
1. Add `josyn-backend-gateway` to root `build.cmd` (after `JOSYN.Backend.JobRegistry`, since Gateway
   depends on JobRegistry + `JOSYN.Jrp.*`).
2. Add `josyn-backend-gateway` to root `pack.cmd` in dependency order (after `job-registry`).
3. Add the Gateway user-cache clear to root `clean.cmd`; while there, audit the existing root-clean
   entries for any folder that no longer exists and fix/remove stale ones.
4. Verify: from a cleared cache, root `clean.cmd NOPAUSE` → `pack.cmd NOPAUSE` produces
   `JOSYN.Backend.Gateway.1.0.0-preview01.nupkg`.

**DoD:** root pack rebuilds Gateway; `overview.md`'s claim is now true.

---

### Phase F-2 — Remove the stale retired package  *(gap #4 — Should-fix)*

1. Delete `C:\DevGit\local-packages\JOSYN.Backend.SurfaceAgent.1.0.0-preview01.nupkg`.
2. Clear `%USERPROFILE%\.nuget\packages\josyn.backend.surfaceagent` (extracted cache dir).
3. Grep all in-scope repos one more time for `JOSYN.Backend.SurfaceAgent` to confirm nothing would
   re-resolve it (review already found none in active code — this is the belt-and-braces check).

**DoD:** the retired package id is gone from the shared feed and the user cache; no consumer references it.

---

### Phase F-3 — JRP↔backend contract-layering decision  *(gap #1 — Critical; maintainer decision required)*

`JOSYN.Jrp.Surface` references `JOSYN.Backend.Contracts` so `SessionSummary.ExecutionStatus` can
reuse the backend `ExecutionStatus` enum (predecessor plan L108). This **contradicts** the same
plan's L134 ("no backend type crosses JRP"): `ExecutionStatus` is a backend type, and it crosses
JRP through a public DTO. This shipped in Phase 1 — it predates the Phase 2–6 work — and it does not
break the build (`backend-contracts` is a stable upstream package already in the feed). But it makes
the documented `jrp → backend → surface` chain inaccurate; the real chain is
`backend-contracts → jrp → backend-gateway → surface`.

**This is a design call, not a mechanical fix — do not change unilaterally.** Two options to put to
the maintainer:

- **Option A — Keep the dependency.** Accept that JRP's read DTOs reuse the backend enum. Then:
  correct the predecessor plan's L134 wording and the dependency-chain note, and document the true
  build order (`backend-contracts` precedes `jrp`).
- **Option B — Sever it.** Introduce a JRP-owned wire enum (e.g. `JOSYN.Jrp.Surface.SessionStatus`),
  map backend `ExecutionStatus` → JRP enum at the read edge (inside `FakeSurfaceAgent` / future
  `HttpAgent`), and drop the `JOSYN.Backend.Contracts` package reference from `JOSYN.Jrp.Surface`.
  This restores a contracts-only JRP and the `jrp`-first build order.

**DoD:** maintainer picks A or B; the chosen path is implemented and the predecessor plan's
contradiction is resolved (not left dangling).

---

### Phase F-4 — Reconcile canonical docs with executed reality  *(gap #5 — Should-fix)*

The single-source-of-truth repo and the active repo README disagree with what was built:

1. **Predecessor plan** (`ADR-033-surface-rename-implementation-plan.md`): mark Phases 2–6 and the
   DoD §6 as done (or stamp the doc "executed — see follow-up plan"). Today only Phase 1 is checked.
2. **`architecture/overview.md`**: contains **no** JRP/Gateway references at all — still describes the
   pre-rename backend/JAP flow. Add the JRP contract layer and the Gateway write path.
3. **`ROADMAP.md`**: names are updated but milestone status/next-steps still say the extraction is
   upcoming. Correct status to reflect the completed rename.
4. **`josyn-surface/README.md`**: still says `JOSYN.Surface.Contracts` owns query records, DTOs, and
   the error taxonomy, and describes MVP-1/read-only. Rewrite around JRP contracts + the Gateway
   write path; `JOSYN.Surface.Contracts` now owns only the `ISurfaceAgent` client seam.

**DoD:** the four documents describe the post-rename world; no stale "Contracts owns DTOs" or
"extraction is next" language remains.

---

### Phase F-5 — Gateway doc/artifact alignment  *(gap #6 — Should-fix)*

`architecture/naming-conventions.md` defines Gateway as a "per-machine EXE that hosts JRP", but the
implemented `JOSYN.Backend.Gateway` is a **library** (`OutputType` default), and
`GatewayCommandHandler` remarks defer service/daemon hosting. There is no JRP host EXE and no
`start-session` implementation yet.

Resolve the overstatement (no code beyond docs unless the maintainer wants the host now):
- Clarify naming-conventions / overview to "Gateway **command library** today; hosted Gateway **EXE**
  later (SessionClient / start-session phase)", **or**
- If a host was actually intended in this phase, scope the EXE — but that is forward work and belongs
  in Part B, not here.

**DoD:** docs no longer claim a Gateway EXE/host exists; the library-vs-host boundary is explicit.

---

### Phase F-6 — Bring `josyn-jrp` into the sanity protocol  *(gap #7 — Should-fix)*

The new source-of-truth contracts repo is invisible to the platform sanity machinery:
1. Add `sanity/current-state/josyn-jrp.md` (stub following the existing per-repo format).
2. Add `josyn-jrp` to `sanity/overview.md` and the supported-repo list in `sanity/README.md`.
3. Add the `sanity/current-state/josyn-jrp.md` entry to `docs/docs-index.json` (a `repos/josyn-jrp.md`
   entry already exists; the sanity entry does not).

**DoD:** `josyn-jrp` appears everywhere the other repos do in the sanity protocol and docs index.

---

## Part B — Commit sequencing  *(gap #3 — Critical)*

`josyn-surface` depends on `JOSYN.Jrp.*` **and** `JOSYN.Backend.Gateway`; Gateway depends on
`JOSYN.Jrp.*`. Committing surface before its upstreams are committed and packable would break a
fresh clone. After Part A is complete and confirmed, commit in dependency order, each with the
AGENTS.md Co-authored-by trailer:

```
Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>
```

Order:
1. **`josyn-jrp`** — (already committed in Phase 1; include only if Phase F-3 Option B touched it).
2. **`josyn-backend`** — Gateway rename (Phase 3) **+** root build/pack/clean scripts (Phase F-1).
   Stage so git records the surface-agent→gateway move as a rename, not delete+add.
3. **`josyn-surface`** — wire-contract relocation + Gateway rewire (Phase 2+3).
4. **`josyn-toolbox`** — deploy comment fix (Phase 5).
5. **`josyn-platform`** — docs (Phase 6 + F-4/F-5/F-6 + this plan).

> **Confirmation gate (AGENTS.md §5):** each repo's commit requires explicit maintainer go-ahead.
> Nothing in Part B runs without it.

**DoD:** all five repos committed in order; a fresh clone + `clean`/`pack` in the same order
(`backend-contracts → jrp → backend-gateway → surface → toolbox`) builds green from source + feed.

---

## Part C — Deferred forward work (predecessor §8)  *(future phases — not this session)*

Captured so it is not lost; each is its own future effort, gated separately:

- **`ISurfaceAgent` rename (Q2).** Deliberately kept as the client seam. Revisit when the real
  `HttpAgent` lands and the seam's final name/shape is known.
- **`HttpAgent`.** The over-the-wire `ISurfaceAgent` implementation that talks JRP to a hosted
  Gateway, replacing the in-process `CompositeSurfaceAgent` for cross-machine use.
- **`JOSYN.Surface.SessionClient`** and the **Gateway EXE/host** that consumes the currently
  unconsumed `JOSYN.Jrp.Launch.StartSessionRequest/Response` (gap #8 — future-facing API today).
- **Write-path expansion:** `CreateJobArgument`, MVP-3 schedule writes — the Gateway already
  receives `Target`/`Actor`/`CorrelationId` on `ChangeJobArgument` (gap #9, preserved but unused);
  wire them into validation/routing/audit when the host exists.
- **Blazor `JOSYN.Surface.Web`** front end over `ISurfaceAgent`.

---

## Execution order summary

```
Part A (remediation, no commits):  F-1 → F-2 → F-3(decision) → F-4 → F-5 → F-6
Part B (commits, gated):           jrp → backend → surface → toolbox → platform
Part C:                            future sessions, separately planned
```

Every write and every git operation in this plan is subject to the AGENTS.md §5 confirmation gate.
