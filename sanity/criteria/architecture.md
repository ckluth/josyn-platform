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
josyn-backend      →  josyn-foundation packages only  (NO dep on josyn-jap)
josyn-commons      →  nothing, OR josyn-foundation ResultPattern only
```

### Forbidden edges

| Forbidden reference | Reason |
|--------------------|--------|
| `josyn-foundation` → anything in platform | Foundation has zero outbound deps |
| `josyn-foundation` → `josyn-commons` | Commons is a utility satellite; Foundation never references it |
| `josyn-backend` → `josyn-jap` NuGet packages | Backend couples via session GUID only, not NuGet |
| `josyn-backend` → `josyn-job-host` | No direct dependency |
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
| josyn-backend references josyn-jap packages | ❌ violation |
| josyn-commons references any platform package beyond ResultPattern | ❌ violation |
| All referenced packages are from josyn-foundation or josyn-jap shared, as permitted | ✅ pass |

### Structural checks

| Check | Verdict |
|-------|---------|
| josyn-jap and josyn-job-host both consume the same foundation packages | ✅ expected |
| josyn-backend spawns JAPServer.exe — no NuGet dep, coupling is session GUID only | ✅ expected |
| Named pipe names follow `JOSYN-REQ-<sessionGUID>` / `JOSYN-RSP-<sessionGUID>` pattern | ✅ expected |
| Any deviation from the above | ❌ violation |

### Runtime coupling check

- `josyn-backend` spawns `JAPServer.exe` and `job.exe` with `JOSYN-IPC <sessionGUID>` as the first CLI argument.
- The session GUID is the only coupling between backend and JAP/job — verify no other shared state exists.

---

## Namespace / Assembly Integrity

Every assembly name must match its namespace root exactly. See `architecture/naming-conventions.md` for the full table. Flag any assembly whose name diverges from the expected pattern.

| Check | Verdict |
|-------|---------|
| Assembly name ≠ namespace root | ❌ violation |
| `JOSYN.JobHost` uses two-segment name (intentional exception, documented) | ✅ expected |
| All other platform-internal packages use `JOSYN.<Layer>.<Component>` | ✅ expected |
