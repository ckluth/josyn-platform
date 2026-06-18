# ADR-025 Implementation Plan — SessionBroker Migration

**Status: COMPLETE** — 2026-06-18

**Goal:** Rename `JOSYN.Jap.JAPServer` to `JOSYN.SessionBroker`, extract it from
`josyn-backend` into a dedicated `josyn-session-broker` repo, and propagate all
naming and documentation changes across the platform.

**Decided in:** [`decisions/ADR-025-session-broker.md`](../decisions/ADR-025-session-broker.md)  
**Session origin:** 2026-06-18 — reasoning session, ADR written, Phase 1 completed.  
**Implementation session:** 2026-06-18 — Phases 2–8 + toolbox cleanup completed.

---

## Start here — three reads to get on track

Read these in order before doing anything else:

1. **`decisions/ADR-025-session-broker.md`** — the decision: full rationale, what changes, what does NOT change
2. **`architecture/overview.md`** — the updated platform picture: component map, dependency chain, spawn relationships
3. **`repos/josyn-backend/overview.md`** — what josyn-backend now is (library packages only, no EXE)

After those three reads, come back here for the phase plan.

---

## Key references to get back on track

| Document | Why read it |
|----------|-------------|
| `decisions/ADR-025-session-broker.md` | The decision record — full rationale, what changes, what does NOT change |
| `architecture/overview.md` | Updated component map and dependency chain — current state |
| `repos/josyn-backend/overview.md` | Updated backend repo summary — what's now in josyn-backend |
| `architecture/naming-conventions.md` | Updated namespace and vocabulary rules — `JOSYN.SessionBroker` two-segment rule |
| `decisions/ADR-004-japserver-relocation.md` | Historical context — why JAPServer landed in josyn-backend; ADR-025 supersedes its location decision |

---

## Phase log — all complete

### Phase 1 ✅ — josyn-platform documentation

All docs in this repo updated before any code was touched.

**Files changed:**
- `AGENTS.md` — `josyn-session-broker` added to repo table; `josyn-backend` description updated
- `architecture/naming-conventions.md` — vocabulary map, repo list, two-segment rule, namespace tree, assembly table
- `architecture/overview.md` — runtime flow, component map, dependency chain, spawn relationships
- `repos/josyn-backend/overview.md` — role, arch position, migration table (jap-server removed)
- `docs/docs-index.json` — ADR-025 entry added
- `decisions/ADR-025-session-broker.md` — the ADR itself (created)

Historical ADRs referencing `JAPServer` are intentionally left unchanged as historical records.

---

### Phase 2 ✅ — Create `josyn-session-broker` repo (`f0c8a37`, `483bf96`)

- Created `ckluth/josyn-session-broker` on GitHub (private)
- Cloned to `C:\DevGit\josyn-session-broker\`
- Copied 9 source files from `josyn-backend-jap-server\JOSYN.Jap.JAPServer\`
- Renamed `JAPServer.cs` → `SessionBroker.cs`; namespace `JOSYN.Jap.JAPServer` → `JOSYN.SessionBroker`; class `JAPServer` → `SessionBroker`
- Created `JOSYN.SessionBroker.csproj` (`AssemblyName=SessionBroker`, output: `SessionBroker.exe`)
- Created `JOSYN.SessionBroker.slnx`, `nuget.config` (→ `..\local-packages`), all `.local-build\` scripts, `README.md`, `.gitignore`
- Build: 0 warnings, 0 errors

**Key decisions made here:**
- `nuget.config` points to `..\local-packages` (one level up — Pattern A, not Pattern B)
- `AssemblyName=SessionBroker` (EXE output name), directory stays `JOSYN.SessionBroker\` — intentional mismatch per ADR-025
- `bootstrap.ini` copy removed from csproj; deployment handles INI placement

---

### Phase 3 ✅ — Update `josyn-backend` (`60d412a`)

- Deleted `josyn-backend\josyn-backend-jap-server\` entirely
- Removed JAPServer build step from `.local-build\build.cmd`
- Renamed `JapServerConstants.cs` → `SessionBrokerConstants.cs`; class renamed; `FolderName="SessionBroker"`, `ExeName="SessionBroker.exe"`
- Updated `SessionLauncher.cs` to use `SessionBrokerConstants`
- Updated `josyn-session-broker\Host.Entrypoint.cs` to use `SessionBrokerConstants.CliModeStart`
- Updated doc strings in `ISessionLauncher.cs`, `IBootstrapConfig.cs`, csproj descriptions, READMEs
- Repacked `JOSYN.Backend.Contracts` and `JOSYN.Backend.SessionLauncher`
- Rebuilt `josyn-session-broker` against updated packages: 0 warnings, 0 errors

**Note:** `JapServerConstants` → `SessionBrokerConstants` is a breaking public API change.
Only two consumers existed at the time: `SessionLauncher.cs` and `Host.Entrypoint.cs` — both updated.

---

### Phase 4 ✅ — Verify `repos/josyn-backend/overview.md` (`e46980f`)

Found and corrected significant pre-existing gaps (beyond ADR-025 scope):
- Wrong folder names (`SessionLauncherContract` → `JOSYN.Backend.Contracts`)
- Phantom entries removed (`josyn-backend-identity-adapter-contract`, `local-packages/`)
- `GlobalConfig` → `BootstrapConfig`
- Directory structure corrected throughout

---

### Phase 5 ✅ — Create `repos/josyn-session-broker.md` (`daa0849`)

Written from scratch: role, two-worlds architecture diagram, CLI contract, source structure
table, partial class breakdown, session lifecycle flow, full dependency list, build/package
notes, exit codes, sanity notes (including known `nuget.config` exception).

---

### Phase 6 ✅ — Update `docs/docs-index.json` (`798814b`)

- Removed 2 dead entries: `josyn-backend-jap-server/JOSYN.Jap.JAPServer/README.md` and
  `josyn-backend-session-starter/JOSYN.Backend.SessionStarter/README.md` (source files deleted)
- Updated `touched` on `repos/josyn-backend/overview.md`
- Added entry for `repos/josyn-session-broker.md` (type: `repo-summary`)
- Added entry for `josyn-session-broker/README.md` (type: `component-architecture`)

---

### Phase 7 ✅ — Check `josyn-contoso` (no changes needed)

Grepped for `JAPServer`/`JapServer` — no matches. Phase clean.

---

### Phase 8 ✅ — Sanity check on `josyn-session-broker` (`2b73e41`, `d44c0ab`, `5a0f24a`)

Full first-run check across all 5 categories. Findings and resolutions:

| Category | Result | Notes |
|---|---|---|
| architecture | ✅ | All dep edges legitimate per ADR-025 |
| standards | ✅ | `nuget.config` pattern exception documented |
| docs | ✅ | 1 violation fixed: stale "JAPServer session" in `AdapterManager` summary |
| tests | ⚠️ | Known gap — no test project; documented |
| principles | ✅ | 3 violations fixed: P1 (`Program` not static), P8 (`IAsyncDisposable`), P10 (null-forgiving comment); 1 minor P8 fixed (`HandleHandlerError` async) |

State file: `sanity/current-state/josyn-session-broker.md`  
Also: `josyn-session-broker` added to valid repo names in `sanity/README.md`.

---

### Additional ✅ — `josyn-toolbox` cleanup (`8d7519c`, `9bdd0a7`)

Two follow-up tasks not in the original plan:

**git-tools:** `josyn-session-broker` was missing from `cfg-repos-list.cmd` and
`josyn-clone-repos.cmd` — added after `josyn-backend` in both (deployment order).
`repo-push-all`, `repo-pull-all`, `repo-status`, and fresh-machine bootstrap now cover the repo.

**deploy:** `deploy-maintainer.ps1` still built from the deleted
`josyn-backend-jap-server\JOSYN.Jap.JAPServer.slnx` — `launch.cmd` would have failed on next
run. Fixed: `$JapServerRoot` → `$SessionBrokerRoot`, added `$SessionBrokerRepoRoot`, Step 4
now builds from `josyn-session-broker\JOSYN.SessionBroker.slnx`.

---

## Phase sequence summary

```
Phase 1  ✅  josyn-platform documentation
Phase 2  ✅  Create josyn-session-broker repo + migrate source
Phase 3  ✅  Remove JAPServer from josyn-backend; update spawn constants
Phase 4  ✅  Verify repos/josyn-backend/overview.md
Phase 5  ✅  Write repos/josyn-session-broker.md
Phase 6  ✅  Update docs-index.json
Phase 7  ✅  Check josyn-contoso — clean
Phase 8  ✅  Sanity check josyn-session-broker
Extra    ✅  josyn-toolbox: cfg-repos-list, josyn-clone-repos, deploy-maintainer
```

---

## Refinements for follow-up sessions

These items were identified during implementation but are out of scope for the migration itself.
Each is self-contained and safe to pick up independently.

### 1. Update `sanity/criteria/architecture.md`

The architecture criteria file still contains structural checks that reference
`JOSYN.Jap.JAPServer` hosted inside `josyn-backend`. These checks predate ADR-025 and are now
stale. A future session should update the criteria to reflect the new topology:
`josyn-session-broker` as a standalone repo, with `josyn-session-broker` in the expected
dependency chain checks.

**File:** `sanity/criteria/architecture.md`  
**Why deferred:** The criteria file is `josyn-platform` internal — not part of the
migration's safety contract. Updating it requires care to avoid invalidating any running sanity
check in progress.

### 2. Run sanity check on `josyn-backend`

The `josyn-backend` repo has never been sanity-checked (`Last checked: never` in overview).
Phase 3 made structural changes to it (deleted `josyn-backend-jap-server\`, renamed
`JapServerConstants` → `SessionBrokerConstants`). A sanity check is overdue.

**Command:** `run sanity-check --profile full --repo josyn-backend`

### 3. `convenience-scripts\nuget-total-rebuild.cmd`

This script (in `josyn-toolbox`) may reference `josyn-backend-jap-server` in its build order.
Not verified during this session — worth checking before running it on a fresh machine.

**File:** `C:\DevGit\josyn-toolbox\convenience-scripts\nuget-total-rebuild.cmd`
