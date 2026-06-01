# Sanity Criteria — architecture

> Verify dependency chain integrity and structural rules.
> Read `architecture/overview.md` before evaluating — it contains
> the full component map, dependency graph, and IPC protocol details.

---

## Dependency Chain Rules

The following NuGet dependency edges are **permitted**. Any other cross-repo NuGet reference is a violation.

```
josyn-foundation   →  (nothing — zero outbound NuGet deps)
josyn-jap          →  josyn-foundation packages only
josyn-job-host     →  josyn-foundation packages + josyn-jap shared packages (Contract, Log)
josyn-backend      →  josyn-foundation packages + JOSYN.Jap.Shared.Contract + JOSYN.Jap.Shared.Log
                       (shared packages only — permitted because JOSYN.Jap.JAPServer lives in this repo)
                       JOSYN.Backend.SessionStarter itself: ResultPattern only
josyn-commons      →  nothing, OR josyn-foundation ResultPattern only
```

### Forbidden edges

| Forbidden reference | Reason |
|--------------------|--------|
| `josyn-foundation` → anything in platform | Foundation has zero outbound deps |
| `josyn-foundation` → `josyn-commons` | Commons is a utility satellite; Foundation never references it |
| `josyn-backend` → any `josyn-jap` package **other than** `Shared.Contract` / `Shared.Log` | Only the two shared packages are permitted; any other josyn-jap package reference is a violation |
| `josyn-backend` → `josyn-job-host` | No direct dependency |
| `JOSYN.Backend.SessionStarter` → any package other than `ResultPattern` | SessionStarter is isolated; JAPServer's deps are in JAPServer's project only |
| `josyn-jap` → `josyn-job-host` | Symmetric peers; they never reference each other |
| `josyn-job-host` → `josyn-jap` server packages | Symmetric peers |
| Any repo → `josyn-platform` | Platform is documentation only, not a code package |

---

## Checklist

### Per-repo dependency check

For each repo, inspect all `.csproj` `<PackageReference>` entries:

| Check | Verdict |
|-------|---------|
| Any forbidden edge present (see table above) | ❌ violation |
| josyn-foundation has any outbound NuGet dep | ❌ violation |
| josyn-backend references josyn-jap packages other than Contract/Log | ❌ violation |
| JOSYN.Backend.SessionStarter references anything other than ResultPattern | ❌ violation |
| josyn-commons references any platform package beyond ResultPattern | ❌ violation |
| All referenced packages are from josyn-foundation or josyn-jap shared, as permitted | ✅ pass |

### Structural checks

| Check | Verdict |
|-------|---------|
| josyn-jap contains NuGet packages only (Contract, Log) — no EXE project | ✅ expected |
| josyn-job-host consumes josyn-foundation + josyn-jap shared packages | ✅ expected |
| josyn-backend contains JOSYN.Jap.JAPServer (EXE) in its own solution | ✅ expected |
| josyn-backend SessionStarter spawns JAPServer.exe; JAPServer spawns job.exe | ✅ expected |
| Each solution in josyn-backend references only its own projects — no cross-solution project references | ✅ expected; any cross-solution project reference is ❌ violation |
| Named pipe names constructed dynamically from the session GUID | ✅ expected |
| Hardcoded pipe name string deviating from `JOSYN-REQ-<GUID>` / `JOSYN-RSP-<GUID>` pattern | ❌ violation |
| Any other deviation from the structural rules above | ❌ violation |

### Runtime coupling check

- `josyn-backend` spawns `JAPServer.exe` with `JOSYN-IPC <sessionGUID>` as the first CLI argument.
- `JOSYN.Jap.JAPServer` spawns `job.exe` with `JOSYN-IPC <sessionGUID>` as the first CLI argument.
- The session GUID is the only coupling between `JAPServer` and `job.exe` — verify no other shared state exists.

---

## Namespace / Assembly Integrity

Every assembly name must match its namespace root exactly. See `architecture/naming-conventions.md` for the full table. Flag any assembly whose name diverges from the expected pattern.

| Check | Verdict |
|-------|---------|
| Assembly name ≠ namespace root | ❌ violation |
| `JOSYN.JobHost` uses two-segment name (intentional exception, documented) | ✅ expected |
| All other platform-internal packages use `JOSYN.<Layer>.<Component>` | ✅ expected |
