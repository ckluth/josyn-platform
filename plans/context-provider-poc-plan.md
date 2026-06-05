# Context Provider PoC Plan

> Temporary. Goal: prove ADR-009 (Runtime Context Provider Pattern) with a minimal
> working implementation before committing to full production design.

---

## ⏭ Next session — start here

**Prompt to give the agent:**

> "Der prompt ist `C:\Users\chris\OneDrive\DevGit\josyn-roadmap\prompts\prompt-017.md`.
> Lies zuerst `josyn-platform/plans/context-provider-poc-plan.md` und
> `josyn-platform/decisions/ADR-009-runtime-context-provider-pattern.md`.
> Wir sind bei Step 3 (AdapterContracts-Package). Mach weiter."

**State to know:**
- Steps 1 + 2 are done and built. Pack-chain runs clean.
- The demo round-trip works end-to-end.
- `JOSYN.Backend.ConfigStore` is the adapter target — not `BootstrapConfig`.
- Step 3 needs a new minimal package `JOSYN.Backend.AdapterContracts` with `IConfigSource`.
- `ConfigStore` will then reference `AdapterContracts` and `SqlConfigSource` will implement `IConfigSource`.

---

## Where we are

- ADR-009 is **Accepted**. The pattern is decided.
- A working demo round-trip already exists (job spawning + JAP args flow).
- ✅ **Step 1 complete:** `BootstrapConfig` (INI file) is implemented and wired up.

---

## Architectural clarification (discovered during Step 1)

`BootstrapConfig` is **JOSYN-internal and never an adapter target.** It exists solely to
bootstrap the process — connection string, exe paths — before anything else can start.

The actual adapter target is the **4th Storage Realm member: Config.**

```
josyn.bootstrap.ini
  └── BootstrapConfig — JOSYN-internal, always present, not replaceable
        └── gets the process started (DB connection, paths)

JOSYN Config Storage  ← 4th Storage Realm member (not yet built)
  └── Runtime configuration in the DB
        → Built-in implementation: reads from josyn.config table
        → HAEVG adapter: replaces this with haevg-config-reader
        → THIS is the adapter target
```

---

## The ride in five steps

### Step 1 — Built-in file-based BootstrapConfig  ✅ DONE

---

### Step 2 — Config Storage Realm member

Design and implement Config as the 4th member of the storage realm.

- Define schema: `josyn.config` table — key/value pairs
- `IConfigRecord` convention (per storage.md)
- DDL script
- `JOSYN.Backend.ConfigStore` package — analogous to `SessionStore`
- Built-in reader: `SqlConfigSource` implements `IConfigSource`

**Done when:** Backend can read a config value by key from the DB.

---

### Step 3 — Adapter contracts package

The shared binary that both host and adapter will reference.

- New minimal package `JOSYN.Backend.AdapterContracts`
- Contains `IConfigSource` (and future adapter interfaces)
- Only dependency: `ResultPattern`
- `JOSYN.Backend.ConfigStore` references it for `IConfigSource`
- This package is the only shared dependency between JOSYN and any future adapter

**Done when:** Package builds; `SqlConfigSource` implements `IConfigSource` from it.

---

### Step 4 — ALC loader + rendezvous infrastructure

The machinery that makes late binding work.

- `AdapterLoadContext : AssemblyLoadContext` in josyn-backend
  (delegates `AdapterContracts` assembly to default context; isolates everything else)
- Bootstrap config: if `ConfigSourceType` key present → load adapter; if absent → use `SqlConfigSource`
- Fail fast with a clear message if the named type cannot be instantiated

**Done when:** Backend starts cleanly with no adapter (SQL path). Fails clearly with a bad type name.

---

### Step 5 — Fake HAEVG adapter + end-to-end demo

The proof.

- New project `HAEVG.Josyn.FakeAdapter` (separate assembly, not part of JOSYN)
- Implements `IConfigSource`, returns hardcoded fake config values
- Place DLL in `adapters/` folder, add type name to `josyn.bootstrap.ini`
- Backend picks it up at startup, uses fake values instead of DB

**Done when:** Two demo runs side by side:
  - Without adapter entry in bootstrap → `SqlConfigSource` (DB) used
  - With adapter entry in bootstrap → fake HAEVG values used

---

## What this PoC does NOT cover

- Access-token pre-loading (next adapter scenario after this)
- Per-job argument enrichment from adapter values (wiring into JAP args)
- Multiple adapters registered simultaneously
- Production-grade error messages and installer/deployment conventions
- Environment separation in Config Storage (deferred to ADR-010)
