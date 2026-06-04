# Evolution & Lifecycle

Describes where JOSYN might go and what it has deliberately deferred. This chapter answers:
what YAGNI decisions have been made and why, what the realistic evolution horizon looks like,
and what would need to change to cross it.

## In Scope
- YAGNI decisions — what was deferred, the reasoning, and the conditions under which it would be revisited
- Linux and Docker horizon — if, when, and at what cost
- Multi-runtime job support (e.g. Go jobs speaking the JAP protocol)
- Scalability — the price of a brighter future, and whether it is reasonable here
- Known eternal limitations

## Out of Scope
- Currently settled constraints → see [02-constraints-and-settled-decisions/](../02-constraints-and-settled-decisions/)
- Current physical state of the system → see [03-physical-architecture/](../03-physical-architecture/)

## Gray Areas & Dependencies
| Topic | Here | Elsewhere |
|-------|------|-----------|
| YAGNI boundary | What is *deferred* and why | What is *settled* → [02-constraints-and-settled-decisions/](../02-constraints-and-settled-decisions/) |

## Contents
- [yagni-decisions.md](yagni-decisions.md) — What was deliberately deferred: clustering, failover, management UI, multi-runtime jobs
- [evolution-horizon.md](evolution-horizon.md) — Linux/Docker horizon, multi-runtime job feasibility, scalability assessment

## Related Decisions
*(none yet)*
