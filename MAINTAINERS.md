# JOSYN Platform — Maintainer's Guide

## *A Letter to Whoever Maintains This Next*

This file is the constitutional reference for everyone who works in the JOSYN platform —
today, and in the years to come.

Read it once. Return to it when decisions get hard.

---

## What we are building

JOSYN is a platform for executing scheduled jobs as isolated, autonomous processes.
It is not a framework. It is not a library. It is an architecture — a set of repos,
contracts, and conventions that work together because they were designed to.

The platform is small by intention. Every layer exists because it earned its place.
Every boundary exists because it protects something worth protecting.

---

## The dependency graph is the architecture

In JOSYN, the structure of who depends on whom is not an implementation detail.
It is the architecture itself.

The layers are:

```
josyn-foundation   — bedrock; stable forever; referenced by all, references nothing
josyn-commons      — utility satellite; open for growth; referenced by all except foundation
josyn-jap          — per-session protocol server; the man-in-the-middle; the game-changer!
josyn-job-host     — job execution runtime: the job-developer's world...
josyn-backend      — scheduler, session orchestrator, runtime owner; tip of an iceberg...
josyn-platform     — this repo; documentation, decisions, architecture
```

These relationships are not suggestions. They are contracts.

A dependency that runs in the wrong direction does not announce itself.
It compiles. It ships. The tests pass. Life goes on — for a while.

But the graph is now wrong. Future code grows around that mistake without knowing
it is there. Decisions are made against an architecture that has already drifted from
its intended shape. The drift compounds quietly, year after year, until the cost of
correction becomes the cost of a rewrite.

Think of it as cancer. The violation may never surface. You may ship it, maintain it,
and retire the codebase — all without ever feeling the pain. Some systems survive their
architectural cancers peacefully until their natural end. You might be lucky.
Or the decline sets in slowly, then accelerates — increasing friction, failing abstractions,
a rewrite that could have been avoided. All from one small, unreviewed dependency edge.

**Protect the graph. Every edge in the wrong direction is a slow-burning fuse.**

---

## Foundation is forever

`josyn-foundation` is the one layer that must never change shape.
Its contracts are load-bearing. Every other repo in the platform stands on them.

Adding to foundation is a major decision. Changing it is an event.
Breaking it — even in a backward-compatible way — requires deliberate coordination
across all consuming repos.

When in doubt: foundation does not grow. The consuming layer adapts.

---

## Commons is not foundation

`josyn-commons` exists precisely because not everything can be foundation.
It is a growing toolbox — useful, domain-agnostic, always backward-compatible.

The critical boundary: **foundation never references commons.**
Commons may grow; foundation may not follow.

When a helper in commons proves foundational, it does not move automatically.
That is an architectural decision, documented in an ADR, made deliberately.

---

## Backend is its own world

`josyn-backend` is listed in one line above. Do not let that mislead you.

Its public role — scheduler, session orchestrator, runtime supervisor — is the narrow contract it exposes to the rest of the platform. Everything beneath that contract is backend territory: scheduling strategies, retry logic, concurrency models, lifecycle state machines, session store, persistence contracts, observability hooks, execution infrastructure...

The backend is designed to evolve independently. Internals may deepen. Subsystems may emerge. That growth is acceptable — and expected — as long as it stays inside the boundary and does not widen the contract.

Do not promote backend internals into shared abstractions without an explicit architectural decision. What lives in the backend, stays in the backend, until the boundary itself is deliberately changed.

Purely internal backend decisions belong in backend-local documentation. Only decisions that affect the contract, the dependency graph, or other repos belong in a platform ADR.

---

## Errors are values. Always.

JOSYN has one mechanism for expressing failure: `Result` / `Result<T>`.

There are no exceptions propagating between layers.
There are no silent swallows.
There are no `try/catch` blocks above the bottom-of-call-graph boundary.

Every failure is a value. Every failure can be inspected, propagated, and logged.
This is not a preference. It is a platform contract.

New code that throws across a layer boundary is a defect — regardless of how
convenient it felt to write.

---

## The vocabulary is precise by design

Every word in the JOSYN vocabulary carries a specific meaning.
"Backend" is not "JAP". "Foundation" is not "Commons". "Platform" is the whole.

Using words loosely in documentation, commit messages, and code comments
degrades the shared mental model that makes cross-repo work possible.

When you introduce a new concept, give it a name, and document it.
When a name no longer fits, change it — with an ADR.

See [architecture/naming-conventions.md](architecture/naming-conventions.md).

---

## Decisions are documented

Every significant architectural choice in JOSYN lives in a decision record (ADR).

An ADR is not bureaucracy. It is institutional memory. It answers the question
"why is it this way?" — a question that will be asked, guaranteed, by the person
who comes after you.

When you make a decision that future maintainers will need to understand:
write the ADR. Keep it short. State the context, the decision, and the consequences.

See [decisions/](decisions/).

---

## What it means to maintain this platform

Maintaining JOSYN means more than writing code that compiles.

It means keeping the architecture honest — catching the dependency that points
the wrong way, the abstraction that leaks domain knowledge, the convenience
that quietly couples two things that should not know about each other.

It means reading the ADRs before making decisions that touch existing contracts.
It means writing the ADR when you make a decision worth remembering.
It means asking: *"Does this addition earn its place?"*

The platform you work in today was built carefully. The person who works in it
after you deserves to inherit that care — not debt.

---

## A final word

Good architecture is not a destination. It is a discipline.

Every commit is either an investment or a withdrawal. Most are small. The balance
compounds — in both directions.

Keep the graph clean. Keep the vocabulary precise. Keep the contracts honest.

That is the work.
