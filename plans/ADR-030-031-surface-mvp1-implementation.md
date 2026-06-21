# MVP-1 Implementation Plan — josyn-surface (Reporting precursor)

**Derived from:** ADR-030 (Accepted 2026-06-21) + ADR-031 (Accepted 2026-06-21).
**Scope:** MVP-1 only — read-only, local CLI, DEV DB via FakeAgent. No REST, no agent EXE,
no aggregator, no Blazor, no commands. (Those are MVP-2+ and out of scope here.)

---

## ⟢ Continuation Brief (read first) — status as of 2026-06-21

**MVP-1 is COMPLETE, verified, committed, and pushed.** A follow-up session can start fresh
from the points under "Naturally next" below.

### Repo facts
- **Location:** `C:\DevGit\josyn-surface` (sibling of `josyn-platform`).
- **GitHub:** https://github.com/ckluth/josyn-surface — **public**, default branch `main`,
  tracking `origin/main`. Matches all sibling repos.
- **Initial commit:** `00cf470` "Scaffold josyn-surface MVP-1 (read-only CLI via FakeAgent)"
  (with Co-authored-by Copilot trailer).
- **Layout:** Pattern A — `JOSYN.Surface.slnx` + `nuget.config` (local feed `..\local-packages`)
  + `.local-build/` at root; four sibling projects.
- **Build/test:** clean Release build (0 warnings, 0 errors); `dotnet test` → 7/7 pass.
- **Manual run verified against `josyn-db-local`:**
  `sessions [--max N]` returns live rows; `error <guid>` returns the `[NotFound]` taxonomy path;
  no-args prints usage.

### What we DID (this session)
1. Authored, rubber-duck-reviewed, revised **ADR-031** (delivery strategy, DS-1..DS-5).
2. Promoted **ADR-030** and **ADR-031** to **Accepted** (2026-06-21).
3. Updated **ROADMAP.md** (M6 milestone + in-progress/next). `docs/docs-index.json` left ON HOLD.
4. Scaffolded the **`josyn-surface`** repo (4 projects, see skeleton below).
5. Built clean, wrote/passed 7 tests, verified CLI against the dev DB.
6. `git init`, initial commit, created the public GitHub repo, pushed `main`.

### What is LEFT in MVP-1
- **Nothing required.** MVP-1 scope is fully delivered.
- Optional polish (defer unless wanted): richer CLI formatting/paging; a FakeAgent integration
  test that reads a *seeded* `josyn-db-local` (current tests are pure mapping/taxonomy, no DB);
  a short repo `repos/josyn-surface.md` summary in `josyn-platform`.

### What is NATURALLY next (MVP-2 — new session)
Governed by ADR-031 **DS-5**. Do NOT start without re-reading ADR-031.
1. **First write command** (e.g. `RetriggerSession`) — but per **DS-5 gating decision** the
   surface repo must NEVER link `SessionLauncher`/launch code. The mutation runs in a **real,
   platform-resident agent EXE in `josyn-backend`**; the surface calls it across the seam.
   ⇒ MVP-2 is blocked on that agent existing. Likely the true first step is *standing up that
   agent EXE in josyn-backend*, not surface code.
2. **HttpAgent** — second `ISurfaceAgent` implementation (REST). Validates the seam (DS-2).
   The named error taxonomy (`SurfaceErrorCategory`, recoverable via `SurfaceError.CategoryOf`)
   was designed to map onto HTTP status codes here.
3. **Command envelope** — introduce the write-side shape defined in ADR-031 DS-5 when (1) lands.
4. **Retire the DS-4 exception** — when a real agent serves reads, delete `FakeAgent`'s direct
   DB access (it is throwaway by design; the durable `Contracts` survive untouched).

### Watch-outs / open items
- **DSQ-1 (seam shape)** still open: MVP-1 uses two explicit async query methods — fine for now;
  generalise to a single `Send(query)` only if it earns its keep.
- **DSQ-3 (cross-machine naming)** still open — irrelevant until multi-machine reach (MVP-3+).
- **Hardcoded DEV connection string** (`tu.josyn`/`josyn` @ `localhost\SQLEXPRESS01`) lives in
  `JOSYN.Surface.Cli\SurfaceCli.cs`, overridable via env `JOSYN_SURFACE_DEV_CONNECTION`. It is now
  in a PUBLIC repo — same disposable local credential already public in `josyn-backend`, so the
  posture is consistent. Move it out of source when the HttpAgent phase arrives.
- **Hard seam rule (DS-4):** no DB row/table shape may ever cross `ISurfaceAgent`. FakeAgent maps
  rows → DTOs internally. Preserve this in any change.

---

## Grounding facts (verified against the codebase, 2026-06-21)

- **Storage:** SQL Server, EF Core, schema `josyn`, dev DB `josyn-db-local`, login `tu.josyn`/`josyn`.
  Bootstrap: `josyn-backend/db/bootstrap-local-dev.sql`.
- **Tables:** `josyn.SessionStore` (existing), `josyn.ErrorStore` (existing).
- **Record contracts (read shape reference, NOT to be reused verbatim across the seam):**
  - `JOSYN.Backend.Contracts.IJobSessionRecord` — UID, JobTypeName, Arguments, Result, JobVersion,
    UserName/UserDomain, ClientApplication/ClientMachine, TecUser?, Started, ExecutionStatus,
    Progress?, Finished?, JapServerProcessId, JobHostProcessId, LastWriteTime?, Host?.
  - `JOSYN.Backend.ErrorHandler.IErrorRecord` — UID, OccurredAt, Causer, Message, CallStack?,
    ExceptionDetails?, JobName?, SessionGuid?.
- **Critical finding:** existing store APIs are insufficient for reporting.
  `ISessionStore` exposes only `GetSession(id)` + write/concurrency methods — **no list/recent/filter**.
  `IErrorHandler` is write-only. ⇒ FakeAgent MUST read the DB directly (DS-4 is necessary, not just
  convenient), and the surface DTOs are designed on API terms, not inherited from a store contract.

---

## Open decisions to lock before/at first code

| # | Decision | Options | Recommendation |
|---|----------|---------|----------------|
| P-1 | Repo structure pattern for `josyn-surface` | A (one solution, multiple projects) vs B (multi-solution) | **DECIDED: Pattern A** (2026-06-21). MVP-1 is one cohesive deliverable; migrate to B when the Blazor shell / aggregator become separate solutions. |
| P-2 | Seam shape (DSQ-1) | single `ISurfaceAgent.Send(query)` vs explicit query methods | For a read-only MVP, **`ISurfaceAgent` with two async query methods** is simplest and still swap-safe. Generalise to `Send` only if it earns its keep. |
| P-3 | Read mechanism in FakeAgent | direct EF Core `internal sealed` DbContext vs raw ADO/SQL | **EF Core, `internal sealed`** (matches storage.md convention; DbContext never public). Confined to FakeAgent. |

---

## Target skeleton (Pattern A; subject to P-1)

```
josyn-surface/                         ← new repo, sibling of josyn-platform
├── JOSYN.Surface.slnx
├── nuget.config                       ← local feed (consumes backend Contracts package for ExecutionStatus etc. if needed)
├── README.md
├── .local-build/                      ← clean.cmd / build.cmd per local-build.md
├── JOSYN.Surface.Contracts/           ← DURABLE: query records + response DTOs + ISurfaceAgent + wire-safe Result usage
├── JOSYN.Surface.FakeAgent/           ← THROWAWAY: EF Core read of josyn-db-local, DB→DTO mapping
├── JOSYN.Surface.Cli/                 ← MOSTLY DURABLE: CLI shell, renders DTOs
└── JOSYN.Surface.Test/                ← tests for contracts + handlers (+ FakeAgent against a known dev DB)
```

---

## Durable contracts (designed on API terms — DS-2 invariants)

**Response DTOs (transport-safe, identity-bearing, trimmed to reporting need):**

- `SessionSummary` — Uid, JobTypeName, ExecutionStatus, Started, Finished?, UserName, ClientMachine,
  Environment, Machine. *(No process IDs / Host — not reporting concerns.)*
- `ErrorDetail` — Uid, OccurredAt, Causer, Message, CallStack?, ExceptionDetails?, JobName?,
  SessionGuid?, Environment, Machine.

**Query records (immutable):**

- `GetRecentSessions(Environment, Machine, RangeBound)` → `Result<IReadOnlyList<SessionSummary>>`
- `GetErrorDetail(Environment, Machine, Guid ErrorUid)` → `Result<ErrorDetail>`

**Seam (P-2):**

```csharp
public interface ISurfaceAgent
{
    Task<Result<IReadOnlyList<SessionSummary>>> GetRecentSessions(GetRecentSessions query, CancellationToken ct);
    Task<Result<ErrorDetail>>                   GetErrorDetail(GetErrorDetail query, CancellationToken ct);
}
```

- Async + CancellationToken from day 1 (DS-2 invariant), even though FakeAgent answers synchronously.
- Every query carries Environment + Machine identity (DS-2).
- `Result` failure cases use the named taxonomy: NotFound, Unavailable, Unauthorized, Timeout, (Partial later).
- DTOs and Result crossing the seam are wire-safe (no exceptions/stacks; those stay in logs).

---

## Throwaway FakeAgent (DS-4 — DEV-only, read-only)

- `internal sealed` EF Core DbContext over `josyn-db-local`, single configured DEV connection
  (no runtime connection-string switching — ADR-010).
- Reads `josyn.SessionStore` / `josyn.ErrorStore`, maps rows → durable DTOs **inside** FakeAgent.
- **Hard rule:** no DB/table shape crosses `ISurfaceAgent`. Mapping is throwaway.
- Source-level marker noting the temporary D-8/D-17 exception and its removal trigger (real agent).

---

## Build order (step-by-step)

1. **Repo + skeleton** (P-1) — create `josyn-surface`, slnx, nuget.config, README, `.local-build/`.
2. **Contracts project** — DTOs, query records, `ISurfaceAgent`, decide Result error taxonomy surface.
3. **FakeAgent project** — EF Core DbContext, two reads, DB→DTO mapping, DEV-only config.
4. **CLI shell** — two verbs (`sessions`, `error <guid>`), renders DTOs; idiomatic above the seam.
5. **Tests** — contract/mapping tests; FakeAgent read test against a seeded `josyn-db-local`. ✅ done — 7 tests pass.
6. **Manual verification** — run CLI against dev DB; confirm output matches the legacy
   `get-session-report` / `get-error-report` capability (not code — D-11). ✅ done — `sessions`
   returns live rows from `josyn-db-local`; `error <guid>` returns the `[NotFound]` taxonomy path.
7. **Docs** — register `josyn-surface` + MVP-1 in `ROADMAP.md` ✅ done (2026-06-21).
   `docs/docs-index.json` **ON HOLD** (maintainer decision 2026-06-21: derived human-facing
   artifact, not maintained for now).

---

## Explicitly NOT in MVP-1

REST/HttpAgent, the platform-resident agent EXE, any command/mutation (incl. RetriggerSession),
aggregator, machine registry, Blazor, auth/RBAC enforcement, cross-machine reach.
