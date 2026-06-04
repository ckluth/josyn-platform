# Constraints & Settled Decisions

Documents what is not negotiable. This chapter saves time by preventing the re-opening of
closed debates. It distinguishes between things that are *settled* (decided and stable) and
things that are *deliberately deferred* (YAGNI — the latter belong in chapter 7).

## In Scope
- Technology choices (.NET, C#)
- Topology constraints (no clustering, no load-balancing, no fallback)
- Language and runtime constraints
- Other fixed baseline assumptions

## Out of Scope
- Open architectural decisions → see [decisions/](../../decisions/)
- Future evolution and deferred options → see [07-evolution-and-lifecycle/](../07-evolution-and-lifecycle/)

## Gray Areas & Dependencies
| Topic | Here | Elsewhere |
|-------|------|-----------|
| YAGNI boundaries | What is *settled* and why | What is *deferred* and the horizon for revisiting → [07-evolution-and-lifecycle/](../07-evolution-and-lifecycle/) |

## Contents
- [settled-decisions.md](settled-decisions.md) — All settled constraints: .NET/C#, process isolation, named pipes, SQL Server, single-machine topology, Result pattern, de-DE culture, zero foundation dependencies

## Related Decisions
*(none yet)*
