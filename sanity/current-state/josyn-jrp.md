# Sanity State — josyn-jrp

**Last checked:** never
**Profile:** —

> `josyn-jrp` is the JRP wire-contract repo created under ADR-033 (Pattern B, mirroring `josyn-jap`).
> It holds two contracts-only packages — `JOSYN.Jrp.Launch` and `JOSYN.Jrp.Surface` — and no EXE.
> A formal sanity pass has not yet been run; this file is a placeholder so the repo is visible to the
> protocol. The notes below record the expected shape, to be confirmed on first check.

---

## Expected shape (to confirm on first sanity pass)

### Dependency chain

| Project | References | Expectation |
|---------|-----------|-------------|
| `JOSYN.Jrp.Launch` | `JOSYN.Jap.Contract`, `JOSYN.Foundation.ResultPattern` | contracts only; no backend type |
| `JOSYN.Jrp.Surface` | `JOSYN.Jrp.Launch`, `JOSYN.Foundation.ResultPattern` | contracts only; **no `JOSYN.Backend.*` reference** (severed under ADR-033 follow-up F-3 Option B) |

> **Layering invariant:** no backend type crosses JRP. `SessionSummary.ExecutionStatus` is typed as
> the JRP-owned `SessionStatus` wire enum (`Dtos/SessionStatus.cs`), not the backend `ExecutionStatus`.
> The backend→JRP status mapping lives at the read edge in `josyn-surface` (`FakeSurfaceAgent`).

### Namespace / Assembly integrity

| Assembly | Namespace root | Expectation |
|---------|---------------|-------------|
| `JOSYN.Jrp.Launch` | `JOSYN.Jrp.Launch` | match |
| `JOSYN.Jrp.Surface` | `JOSYN.Jrp.Surface` (+ `.Queries`, `.Commands`, `.Dtos`) | match |

### Structural / Runtime coupling

- Contracts only — no runtime spawn responsibilities, no DB shape, no hardcoded pipe/host names.

---

## Open items

- Run the first formal sanity pass (`architecture` + `standards` at minimum) and replace this
  placeholder with verified findings.
