# Maintainability and Knowledge Transfer

## The Reality: Maintainers Come and Go

The people who build JOSYN today will not maintain it forever. New developers will join.
Original authors will move on. This is not a risk to mitigate — it is a certainty to
design for.

A platform that can only be maintained by those who built it is a liability, not an asset.

## What This Rules Out

**No esoteric technology choices** without clear justification and documentation.
If a technology requires deep specialist knowledge that cannot be acquired in a reasonable
time, the cost of that choice must be weighed honestly. Fashionable or clever is not
a justification.

**No tribal knowledge.** If understanding a design decision requires a conversation with
the original author, the design is incomplete. The reasoning must live in the code,
the architecture docs, or a decision record — not in someone's head.

**No hidden complexity.** Complexity that lives in undocumented conventions, implicit
assumptions, or "everyone knows that" shortcuts is the hardest kind to transfer.
Prefer explicit over implicit. Prefer boring over clever.

## What This Requires

**Decision records for non-obvious choices.** When a design decision was made,
the reasoning must be recorded. Not just *what* was decided — *why*, and *what
was rejected and why*. This is the purpose of the `decisions/` folder.

**Architecture documentation that a newcomer can follow.** The `josyn-platform` repo
is the answer to "where do I start if I know nothing?" It must be maintained as the
entry point for new maintainers, not just as a historical record.

**Code that explains itself.** The JOSYN coding standards (static-first, Result pattern,
no DI containers, no OOP-by-default) are not arbitrary preferences. They reduce the
cognitive surface area that a maintainer must master. A static method is easier to
follow than a chain of injected abstractions.

**Dependency discipline.** Minimising external dependencies reduces the surface of
"things a new maintainer must learn before touching the code". Every additional
NuGet dependency is a knowledge transfer burden.

## The Platform's Own Documentation as a Maintainability Tool

`josyn-platform` exists partly to serve this goal. If a new developer can clone it,
read the architecture overview, the coding standards, and the decision records, and
then productively work on any of the platform repos — the documentation is doing its job.

That bar should be measured against, not assumed.
