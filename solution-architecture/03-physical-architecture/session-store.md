# Session Store

## Overview

Every JOSYN job execution creates a **session record** in the session store.
The session store is a SQL Server database that serves three roles:
1. **Argument transfer** — the backend writes job arguments into the session row; `JAPServer` reads them
2. **Result collection** — `JAPServer` writes the job result (or error) back into the row
3. **Audit trail** — every execution is permanently recorded

## Technology

| Aspect | Choice |
|--------|--------|
| Database engine | SQL Server |
| ORM | EF Core (`Microsoft.EntityFrameworkCore.SqlServer`) |
| Schema namespace | `josyn` |
| Package | `JOSYN.Backend.SessionStore` |

## Session Lifecycle

```
SessionStarter  ──creates row──►  session store  (GUID + arguments)
JAPServer       ──reads args──►   session store  (GetRawArguments)
JAPServer       ──writes result─► session store  (PutRawResult / PutError)
Backend         ──polls result──► session store  (completion detection)
```

*(The current backend uses a 500 ms poll loop to detect completion.
A production implementation will use a push signal.)*

## Known Limitations

- No **session status column** — orphaned rows from failed process spawns are
  indistinguishable from running sessions. A `status` field (preparing / running /
  complete / failed) is needed for production.
- No separated **start / end timestamps**.
- No explicit **job identity** column linking the session to a registered job name.

These are known gaps, not final design choices.
