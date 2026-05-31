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
josyn-jap          — the protocol backbone; grows deliberately; existing members never break
josyn-job-host     — open for evolution; the job developer's only door into the platform
josyn-backend      — scheduler, session orchestrator, runtime owner; tip of an iceberg...
josyn-platform     — this repo; documentation, decisions, architecture
```

One repository stands outside this graph entirely:

```
josyn-sandbox      — consumer; references the platform freely; the platform does not know it exists
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

## JAP is the protocol backbone

`josyn-jap` is the application protocol layer every job execution flows through.
It sits on the JIP transport — which lives in foundation and is truly bedrock —
and defines the high-level conversation between the scheduler world and the job world.

That conversation is not frozen. Every new capability that job-host gains must have
a voice in the protocol — JAP grows upward from the needs of the job world.

That growth is healthy and necessary. But it must be deliberate:

- New members on `IJosynApplicationProtocol` require coordination with `josyn-job-host`,
  which implements the client side of every member.
- Existing members must never change semantics — they are active contracts in every
  running session.
- Changes to JAP's shared packages (`Jap.Shared.*`) ripple to both `JAPServer` and
  `JobHost` simultaneously. Review both sides before committing.

The transport below all of this — the named pipes, the wire format, the session GUID
convention — is JIP's territory, defined in foundation. That does not move.
Everything JAP adds above it can grow.

---

## Job Host is the job developer's gateway

`josyn-job-host` is the platform's handshake with the world outside.
Job developers link it, follow the protocol, and write business logic.
That is all they should need to do.

Job Host is designed to evolve — more features, more relief for the job developer's daily work, more support
to get things done, and more room for the person in front of the keyboard — to focus on the problem they actually came to solve.

This evolution is healthy and expected. But every addition is a commitment to every job developer
who links it.

Two rules govern that evolution:

1. **Stay lean.** Job developers should encounter only what they need.
   No JAP concepts should leak into the consumer-facing surface.
   No abstractions that serve the platform's convenience at the expense of the consumer's clarity.

2. **No YAGNI.** Add only for a concrete reason. Remove only with deliberate migration
   and deprecation — removals are expensive for a library that job authors bind to.

The `[JobEntryPoint]` attribute and `Core.Run(args)` are the entire surface most job authors
will ever see. Protect their simplicity.

---

## Backend is its own world

`josyn-backend` is listed in one line above. Do not let that mislead you.

Its public role — scheduler, session orchestrator, runtime supervisor — is the narrow contract it exposes to the rest of the platform. Everything beneath that contract is backend territory: scheduling strategies, retry logic, concurrency models, lifecycle state machines, session store, persistence contracts, observability hooks, execution infrastructure...

The backend is designed to evolve independently. Internals may deepen. Subsystems may emerge. That growth is acceptable — and expected — as long as it stays inside the boundary and does not widen the contract.

Do not promote backend internals into shared abstractions without an explicit architectural decision. What lives in the backend, stays in the backend, until the boundary itself is deliberately changed.

Purely internal backend decisions belong in backend-local documentation. Only decisions that affect the contract, the dependency graph, or other repos belong in a platform ADR.

---

## Sandbox is not the platform

`josyn-sandbox` is a consumer repository. It may reference any platform repo it needs.
The platform never references it back — not in NuGet, not via project references, not through any convention or discovery mechanism.

This is the one rule that defines its place: **the platform does not know josyn-sandbox exists.**

Because of this, sandbox carries none of the platform's obligations. It is not maintained to platform standards. Its code may be rough, experimental, or incomplete. That is not a deficiency — it is the point. It is the maintainer's space to demonstrate the living system, run exploratory integration scenarios, and develop new concepts before they earn a place in the real architecture.

When something in sandbox matures into a real feature, it moves — rewritten and reviewed — into the appropriate platform repo. The sandbox is not a permanent home.

---

## Two stability philosophies

Foundation stands apart — it has no philosophy, only permanence.
Everything below it falls into one of two modes.

The **grows deliberately** layers — `jap` and `job-host` — are expected to expand,
but only additively and only with coordination. JAP's application protocol acquires
new members as the platform matures; Job Host's consumer API grows as job developer
needs become clear. In both cases: additions are decisions, existing members are
commitments, and every change ripples to at least one other repo.

The **open-for-evolution** layers — `backend` and `commons` — are expected to grow and
deepen more freely. Commons accumulates domain-agnostic helpers; backend's internals
may expand significantly behind its narrow contract. Growth is healthy — as long as it
earns its place and stays inside the boundary.

Knowing which mode a layer operates in tells you how much care a change deserves.

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

- **Keep the graph clean.** 
- **Keep the vocabulary precise.**
- **Keep the contracts honest.**

That is the work.
