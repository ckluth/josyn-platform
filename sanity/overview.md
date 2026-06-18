# Sanity Check — Platform Overview

**Generated:** 2026-06-18T19:27 UTC

| Repo | docs | tests | principles | architecture | standards | Last checked |
|------|------|-------|-----------|-------------|-----------|-------------|
| josyn-foundation | ✅ | ✅ | ✅ | ✅ | ✅ | 2026-05-31T12:10 UTC |
| josyn-jap | — | — | — | ✅ | ✅ | 2026-05-31T12:49 UTC |
| josyn-job-host | — | — | — | — | — | never |
| josyn-backend | — | — | — | — | — | never |
| josyn-commons | — | — | — | — | — | never |
| josyn-session-broker | ❌ | ⚠️ | ❌ | ✅ | ✅ | 2026-06-18T19:27 UTC |

---

## josyn-foundation — All violations resolved (2026-05-31)

## josyn-jap — All violations resolved (2026-05-31)

## josyn-session-broker — First check (2026-06-18) — violations open

- **docs**: `AdapterManager.cs` class `<summary>` still says "JAPServer session"
- **principles**: 3 violations — `Program.cs` (P1), `AdapterProcess.cs` (P8), `Host.Entrypoint.cs` (P10) — plus 1 minor (P8 in `Host.Server.cs`)
