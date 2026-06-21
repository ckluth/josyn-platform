# MVP-2 Plan — josyn-surface (Job arguments & schedule reach; first write)

**Derived from:** ADR-030 (Accepted 2026-06-21) + ADR-031 (Accepted 2026-06-21), MVP-1 plan
(`ADR-030-031-surface-mvp1-implementation.md`).
**Status:** MVP-2a SHIPPED (commit `eff997e`, pushed). MVP-2b DESIGNED & CONFIRMED — ready to
implement (awaiting go-ahead). MVP-3 deferred.
**Scope:** Job-argument and job-schedule **reach** for `josyn-surface`. Read-only first (MVP-2a),
then the platform's **first write** (MVP-2b) as the smallest honest mutation. Schedule writes are
explicitly deferred to MVP-3.

This plan is a planning artefact. Per AGENTS.md §5 the confirmation gate still applies to every
write/git operation when implementation begins — this document does not pre-authorise code.

---

## ⟢ Continuation Brief (read first)

A follow-up session can cold-start from here. MVP-1 is COMPLETE, committed, pushed
(`josyn-surface` @ `00cf470`). This plan covers what is **naturally next** after MVP-1, refining
ADR-031 **DS-5** for two concrete use-cases the maintainer wants now: **read/change job arguments**
(FR-9) and **read/add/change schedule entries** (FR-10).

**The decision that shapes everything:** both use-cases split cleanly along the read/write line the
ADRs already draw. Reads are deliverable now inside the read-only DS-4 exception. Writes are the
first platform mutation and are gated by DS-5 on a real platform-resident agent EXE — they may
**not** ride FakeAgent's direct-DB access (DS-4 is read-only by construction).

**Chosen shape (confirmed):**
- **MVP-2a** — read verbs for *both* arguments and schedules. Useful immediately, zero debt.
- **MVP-2b** — **change an argument record** as the platform's first write; vehicle to stand up the
  minimal platform-resident agent EXE + command envelope.
- **MVP-3** — schedule add/edit/suspend/resume writes, once the write seam is proven.

---

## Grounding facts (verified against `josyn-backend`, 2026-06-21)

Both backing stores already exist and are **read-only by contract** today.

**Job registry & arguments** — `JOSYN.Backend.JobRegistry`:
```csharp
public interface IJobRegistry
{
    Result<IJobRegistrationRecord>                GetByName(string name);
    Result<IReadOnlyList<IJobRegistrationRecord>> GetAll();
    Result<IArgumentRecord>                       GetArgument(string jobName, string argumentName);
}
public interface IJobRegistrationRecord { string Name; string TechnicalUserName; IReadOnlyList<IArgumentRecord> ArgumentRecords; }
public interface IArgumentRecord        { string JobName; string Name; string Content; }
```

**Job schedules & entries** — `JOSYN.Backend.JobScheduleStore`:
```csharp
public interface IJobScheduleStore { Result<IReadOnlyList<IJobScheduleRecord>> GetAll(); }
public interface IJobScheduleRecord      { string JobName; bool Suspended; DateOnly? SuspendedUntil; IReadOnlyList<IJobScheduleEntryRecord> Entries; }
public interface IJobScheduleEntryRecord { string ArgumentRecordName; string ScheduleDefinition; int? ToleranceMinutes; }
```

**Critical findings:**
1. **Reads are richer than MVP-1's.** Unlike `ISessionStore` (no list method — which forced the
   DS-4 direct-DB read), both stores already expose `GetAll`. The read path is well-supported.
2. **No write API exists on either store.** Mutating an argument or a schedule entry requires a
   *new* write method on the platform store contract — a `josyn-backend` change, not a surface one.
3. **`architecture/storage.md` is stale.** It lists `JobRegistry` as "Planned" and omits
   `JobScheduleStore` entirely. Both exist. (Doc fix — tracked below, separate from MVP-2.)
4. **Referential shape:** `IArgumentRecord.JobName` → registry; `IJobScheduleEntryRecord.`
   `ArgumentRecordName` → an argument record. So registry+arguments is the natural foundation the
   schedule view builds on.

---

## Why this split (the weighing)

| Half | Use-case | ADR posture | Cost | Verdict |
|------|----------|-------------|------|---------|
| **Read** | Show arguments / schedule entries for a job | Inside DS-4 (read-only); reuses existing `GetAll`/`GetByName`/`GetArgument` | Low — new CLI verbs + DTOs + mapping | **Deliver now (MVP-2a)** |
| **Write** | Change an argument record | DS-5 first command: needs write API on the store **and** a platform-resident agent EXE; DS-4 does **not** cover writes | Medium — backend write method + minimal agent EXE + command envelope | **First write (MVP-2b)** |
| **Write** | Add/edit schedule entry, suspend/resume | Same gate as above, **plus** child-collection semantics + `ScheduleDefinition` (cron-like) validation | High — several distinct mutations | **Defer (MVP-3)** |

**Arguments win as the first write** because an argument upsert is a single flat record
(`{JobName, Name, Content}`) — no child collections, no schedule-definition validation, no process
spawn. It is the smallest mutation that still exercises the full command path (envelope → seam →
platform-resident handler → store write), which is exactly what a first write should prove.

**Hard rule preserved:** the write does **not** extend DS-4. DS-4 is read-only by construction; a
write through FakeAgent's direct DB access would break ADR-031 and must be flagged, not done. The
write lands platform-side from day one (DS-5, D-17/D-19).

---

## MVP-2a — Read reach (arguments + schedules), read-only

Extends MVP-1's read surface. Stays entirely inside DS-4. No platform-resident agent needed.

### Durable contracts (designed on API terms — DS-2 invariants)

Response DTOs (transport-safe, identity-bearing, trimmed to reporting need):

- `JobArguments` — `Environment, Machine, JobName, TechnicalUserName, IReadOnlyList<ArgumentSummary>`
  where `ArgumentSummary = { Name, Content }`.
- `JobSchedule` — `Environment, Machine, JobName, Suspended, SuspendedUntil?,`
  `IReadOnlyList<ScheduleEntrySummary>` where
  `ScheduleEntrySummary = { ArgumentRecordName, ScheduleDefinition, ToleranceMinutes? }`.

Query records (immutable, identity-bearing per DS-2):

- `GetJobArguments(Environment, Machine, string JobName)` → `Result<JobArguments>`
- `GetJobSchedule(Environment, Machine, string JobName)`   → `Result<JobSchedule>`
- (optional) `GetRegisteredJobs(Environment, Machine)`     → `Result<IReadOnlyList<...>>` — a
  discovery verb so the user can list job names before drilling in. Decide when writing the verb.

Seam — extend `ISurfaceAgent` with the new async query methods (same shape as MVP-1; resolves
nothing new in DSQ-1 — keep the explicit-methods style unless it stops paying for itself).

`Result` failure cases reuse the named taxonomy (`NotFound`, `Unavailable`, `Unauthorized`,
`Timeout`). No DB/table shape crosses the seam (DS-4 containment rule).

### FakeAgent reads (DS-4 — DEV-only, read-only)

Two options for the read source — **decide at implementation, flag the choice**:
- **(a)** Continue direct EF Core read of `josyn-db-local` (consistent with MVP-1's FakeAgent), map
  rows → DTOs internally. Simplest; same throwaway posture.
- **(b)** Consume the existing `IJobRegistry` / `IJobScheduleStore` store *packages* in-process and
  map their `IxxxRecord` contracts → surface DTOs. Cleaner (no second DB-shape duplication) but
  introduces a surface→store package coupling that must be justified vs D-8/D-17 and removed when
  the real agent lands.

Recommendation: **(a)** for strict consistency with MVP-1 and the throwaway contract; revisit at
the HttpAgent phase. Either way, the **mapping is throwaway**; the DTOs above are durable.

### CLI verbs

- `arguments <jobName> [--env E] [--machine M]` → renders `JobArguments`.
- `schedule <jobName> [--env E] [--machine M]`  → renders `JobSchedule`.
- (optional) `jobs` → lists registered job names.

### Tests

- Contract/mapping tests for the two new DTOs (pure, no DB).
- FakeAgent read test against a seeded `josyn-db-local` (mirrors MVP-1's manual-verify step).
- Taxonomy test: unknown `jobName` → `[NotFound]`.

### Out of scope for MVP-2a
Any mutation; the platform-resident agent; REST/HttpAgent; schedule semantics beyond display.

---

## MVP-2b — First write: `ChangeJobArgument` (DS-5 command)

The platform's first mutation. Refined 2026-06-21 after a rubber-duck pass and two maintainer
decisions. **The surface never writes the DB directly and never links launch code.** Write logic is
placed platform-side in `josyn-backend` from day one; only the *transport* is deferred (in-process
invocation, explicitly sanctioned by DS-5).

**Scope locked:** ship **`ChangeJobArgument` only** (change an *existing* argument). `CreateJobArgument`
is a distinct, later command (maintainer decision A, 2026-06-21) — never an upsert. The two are kept
separate so a typo can never silently create a record.

### The agent vision (context — see also ADR-030 D-5/D-16/D-17)

The "platform-resident agent" ultimately is **one long-lived process per JOSYN machine** (Windows
Service / systemd daemon) exposing a clean, distinct **HTTPS/REST listener** — the only thing allowed
to touch that machine's stores and to spawn sessions there (the C-2 execution channel). It has two
separable parts:
1. **Command logic** — "validate this command, then write the store." Cheap, load-bearing, platform code.
2. **Host** — the service shell + Kestrel + REST + auth that exposes (1) over the network. Heavy plumbing.

**MVP-2b builds (1) as a library and defers (2) entirely** (maintainer decision B, 2026-06-21:
"library now, no empty EXE host yet"). The surface process loads the backend assembly and calls the
command handler **in-process** — same code the future service will host, minus the socket. When the
HttpAgent phase arrives, the EXE host wraps the unchanged handler and the surface swaps its in-process
write-adapter for an `HttpAgent`; nothing above `ISurfaceAgent` or inside the handler changes (the DS-2
swap guarantee, applied to the write side).

```
CLI ──> ISurfaceAgent.ChangeJobArgument(cmd)        (surface process, DEV box)
             │
        [write adapter] ──direct method call──> SurfaceCommandHandler   (josyn-backend library)
             │                                   .HandleChangeJobArgument(cmd)
        (reads still go via                            │
         FakeAgent direct-DB, DS-4)                     ▼
                                              IJobArgumentWriter / SqlJobRegistry
                                                        │
                                                        ▼
                                              josyn.ArgumentRecords  (UPDATE existing)
```

### Platform-side work (`josyn-backend`)

1. **New write-side interface — `IJobArgumentWriter`** in `JOSYN.Backend.JobRegistry`. The write does
   **not** go on the read-focused `IJobRegistry` (rubber-duck #4 — avoid broadening a query contract
   for all readers):
   ```csharp
   public interface IJobArgumentWriter
   {
       // Changes an EXISTING argument's content. NotFound if the (job, argument) pair does not exist.
       Result<ArgumentChangeOutcome> ChangeArgument(string jobName, string argumentName, string content);
   }
   ```
   - Returns `Result<ArgumentChangeOutcome>`, not bare `Result` (rubber-duck #3/#7): the outcome carries
     `Before` (previous content) and `After` (new content) read **atomically** inside the write, so the
     CLI shows a truthful before/after without a racy read-modify-read.
   - `ChangeArgument` is *change-only*: `[NotFound]` if the job or the argument record is absent — it
     never creates (maintainer decision A).
   - `SqlJobRegistry` (existing class) implements it alongside `IJobRegistry`. No `<Version>` bump
     (NuGet rule §8); `clean.cmd` then `pack.cmd` to republish.

2. **New backend library project — `JOSYN.Backend.SurfaceAgent`** (Pattern-B: own solution + `.local-build`).
   Holds the platform-resident command logic. One public class:
   ```csharp
   public sealed class SurfaceCommandHandler(string connectionString)
   {
       // Takes the FULL command envelope — actor/target/correlation are NOT dropped (rubber-duck #1).
       public Result<ArgumentChangeOutcome> HandleChangeJobArgument(ChangeJobArgumentCommand command);
   }
   ```
   - Internally constructs `new SqlJobRegistry(connectionString)` (as `IJobArgumentWriter`) and calls
     `ChangeArgument`. It is the long-lived home of all future writes; it is **not** the surface's
     FakeAgent and does not reuse it.
   - `ChangeJobArgumentCommand` is a backend-side command type carrying `{ Actor, Environment, Machine,
     CorrelationId, JobName, ArgumentName, Content }`. Enforcement of actor/audit is deferred, but the
     fields exist from day 1 (DS-5).
   - **No EXE host, no REST, no Kestrel, no auth** in this increment (maintainer decision B).
   - Packs as a NuGet library to the local feed, consumed by the surface.

### Surface-side work (`josyn-surface`)

3. **Command record — `ChangeJobArgument`** in Contracts (durable, wire-safe, envelope-bearing):
   `{ SurfaceTarget Target, string Actor, Guid CorrelationId, string JobName, string ArgumentName, string Content }`
   → returns `Result<ArgumentChangeOutcome>` (a surface-Contracts DTO with `Before`/`After`, **not** a
   backend/EF shape — DS-2 "no DB shape crosses the seam", rubber-duck #10).

4. **Seam shape (DSQ-1 — decided).** Keep the single `ISurfaceAgent` and add one async command method
   `ChangeJobArgument(ChangeJobArgument command, CancellationToken ct = default)`. The read/write
   asymmetry is already visible from the method names; a `Query`/`Command` split earns nothing here and
   is reversible later. Async by shape (seam invariant) even though the handler runs synchronously
   underneath (rubber-duck #8 — surface stays async, backend may stay sync).

5. **Write adapter, not a fake mutation (rubber-duck #2).** `FakeSurfaceAgent` keeps **only** its
   read methods (DS-4 stays read-only by construction). The command path is served by a **separate**
   `ISurfaceAgent` composition that delegates `ChangeJobArgument` to an in-process
   `SurfaceCommandHandler` while continuing to serve reads from the existing FakeAgent. Conceptually:
   `CompositeSurfaceAgent(reads: FakeSurfaceAgent, writes: PlatformCommandAdapter)`. The read-only DS-4
   exception is thereby **closed, not extended**, for the argument domain.

6. **CLI verb — `change-argument <jobName> <argName> <content-or-@file>`** (`@file`/stdin for
   non-trivial JSON payloads). Deliberate by design (DS-5 / NFR-2). Renders the `Before`/`After` the
   handler returned — no separate read.

### NuGet cross-repo sequencing (rubber-duck #9 — make the cache trap explicit)

The surface today does **not** reference the backend JobRegistry package; MVP-2b adds references to
both the updated `JOSYN.Backend.JobRegistry` and the new `JOSYN.Backend.SurfaceAgent`. Because versions
are **not** bumped (§8), a stale cached `.nupkg` would hide the new `IJobArgumentWriter` /
`SurfaceCommandHandler`. Mandatory order:
1. Backend: implement write interface + handler.
2. Backend: `clean.cmd` then `pack.cmd` for **both** changed/new packages.
3. **Clear the user NuGet cache** for those package IDs (§8) before the surface restores.
4. Surface: restore + build against the fresh packages.

### Tests
- `SqlJobRegistry.ChangeArgument`: existing arg → `Updated` with correct `Before`/`After`; unknown
  argument → `[NotFound]`; unknown job → `[NotFound]`. (Mapping/guard logic; DB-integration optional.)
- `SurfaceCommandHandler.HandleChangeJobArgument`: passes the full envelope through; surfaces the
  handler's `Result` faithfully.
- Surface contract: `ChangeJobArgument` round-trip; `ArgumentChangeOutcome` is a Contracts DTO (no EF
  types leak).
- End-to-end (in-process): CLI `change-argument` → composite agent → handler → store; a subsequent
  `arguments` read reflects the change; `Before`/`After` match.

### Out of scope for MVP-2b
`CreateJobArgument` (distinct later command); REST/HttpAgent and any EXE host (deferred — decision B);
schedule writes (MVP-3); confirmation-gate/audit *enforcement*; any session spawn (this is a pure store
write — it does **not** cross C-2 and must not pull in `SessionLauncher`).

---

## MVP-3 (deferred) — Schedule writes

Once the write seam is proven by MVP-2b: add/edit a schedule entry, suspend/resume a schedule. These
carry the harder semantics deliberately left out of the first write:
- Child-collection mutation (entries under a schedule).
- `ScheduleDefinition` validation (cron-like grammar) — needs its own decision.
- Suspend/resume as distinct commands with `SuspendedUntil` handling.
All ride the same command envelope + platform-resident agent established in MVP-2b.

---

## Build order (step-by-step)

**MVP-2a — ✅ DONE (commit `eff997e`, pushed):**
1. ✅ Contracts — `JobArguments`, `JobSchedule`, `RegisteredJobSummary` DTOs + 3 query records; extended `ISurfaceAgent`.
2. ✅ FakeAgent reads — direct EF read + DB→DTO mapping (option (a)); split into partials.
3. ✅ CLI — `jobs`, `arguments`, `schedule` verbs.
4. ✅ Tests (8 new; 15/15 pass) + manual verify against `josyn-db-local`.

**MVP-2b — confirmed, ready to implement:**
5. **Backend write API** — `IJobArgumentWriter.ChangeArgument` → `Result<ArgumentChangeOutcome>`
   (Before/After, change-only, `[NotFound]` if absent), implemented on `SqlJobRegistry`. Doc it.
6. **Backend `JOSYN.Backend.SurfaceAgent` library** — `SurfaceCommandHandler.HandleChangeJobArgument`
   taking the full `ChangeJobArgumentCommand` envelope; delegates to the writer. No EXE/REST.
7. **Backend pack + cache-clear** — `clean.cmd`/`pack.cmd` both packages; clear NuGet cache (§9 above).
8. **Surface contracts** — `ChangeJobArgument` command + `ArgumentChangeOutcome` DTO; add async
   `ISurfaceAgent.ChangeJobArgument`.
9. **Surface write adapter** — `CompositeSurfaceAgent` (reads → FakeAgent, writes → in-process
   `SurfaceCommandHandler`); FakeAgent stays read-only.
10. **Surface CLI** — `change-argument <job> <arg> <content-or-@file>`; render Before/After.
11. **Tests** — writer guard logic, handler envelope pass-through, contract round-trip, in-process E2E.
12. **Docs** — register MVP-2 in `ROADMAP.md`; correct stale `architecture/storage.md`
    (JobRegistry = Existing; add JobScheduleStore row). `docs/docs-index.json` remains ON HOLD per
    the MVP-1 maintainer decision.

---

## Open decisions — resolved

| # | Decision | Resolution (2026-06-21) |
|---|----------|-------------------------|
| Q-1 | MVP-2a read source in FakeAgent | **(a) direct EF read** — shipped. |
| Q-2 | Seam shape (DSQ-1) at first command | **Single `ISurfaceAgent` + one async command method.** No Query/Command split. |
| Q-3 | Write API name/shape | **`IJobArgumentWriter.ChangeArgument` → `Result<ArgumentChangeOutcome>`** (separate write interface; not on `IJobRegistry`; not an upsert). |
| Q-4 | Argument `Content` input in CLI | **inline arg or `@file`/stdin** for non-trivial payloads. |
| Q-5 | Discovery verb `jobs` | **Included** in MVP-2a — shipped. |
| Q-A | Create vs change as first write | **`ChangeJobArgument` only**; `CreateJobArgument` is a distinct later command. |
| Q-B | Platform agent: library vs EXE now | **Library now** (`JOSYN.Backend.SurfaceAgent`); **no EXE/REST host yet.** |

---

## Watch-outs

- **DS-4 is read-only — do not extend it to writes.** The argument write lands platform-side
  (MVP-2b), never through FakeAgent's DB access. Flag any temptation to "just write the DB from the
  surface for DEV."
- **No DB shape crosses `ISurfaceAgent`** (DS-4 containment) — for both reads and the write.
- **No version bumps; clear NuGet cache before re-pack** (AGENTS.md §8) when the backend
  JobRegistry package gains the write method.
- **This command does not cross C-2.** It is a store write, not a session spawn — keep
  `SessionLauncher` out of it. (The spawn-based `RetriggerSession`, ADR-031 DS-5's original MVP-2
  example, remains a separate future increment.)
- **`storage.md` staleness** is real and should be corrected, but as a doc fix tracked in step 9 —
  not folded into the code work.
- **Confirmation gate (AGENTS.md §5)** applies to every file/git operation when implementation
  begins; this plan authorises nothing on its own.

---

## Explicitly NOT in MVP-2

REST/HttpAgent; the cross-machine layer (DSQ-3 name); schedule writes (MVP-3); session spawn /
`RetriggerSession` / `SessionLauncher`; confirmation-gate & audit *enforcement*; aggregator; machine
registry; Blazor; auth/RBAC enforcement.
