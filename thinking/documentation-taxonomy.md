# Documentation Taxonomy — Working Note

**Date:** 2026-06-06
**Status:** Thinking — not settled, not an ADR

---

## What this is

A working note capturing a brainstorming session on how to bring structure and order
to the growing pile of documentation across all JOSYN repos. Nothing here is decided.
This is a place to hold the ideas until they are ready to be acted on.

---

## The core idea

**Tags decouple semantic meaning from file location.**

Instead of forcing every document into a folder hierarchy that can only express one
organisational axis at a time, documents stay where they live and carry tags that
express their meaning along multiple axes simultaneously.

This makes it possible to navigate the documentation in different ways — by type,
by audience, by maturity — without reorganising the file tree every time the
perspective changes.

---

## The three axes

### 1. Type
*What kind of document is this?*

Drives discovery and routing. A reader looking for "all decisions" or "all references"
should be able to filter on this.

Vocabulary: to be defined. Candidates from the brainstorm:
technical architecture (high-level), technical architecture (in-depth),
component/sub-architecture, solution architecture, reference/standards,
manual/operational, decision/ADR, working note/thinking, overview/entry-point, repo summary,
**work-overview** (milestones, what's done, what's left — a progress/status type that does not
exist yet but was explicitly identified as missing).

### 2. Audience / Perspective
*Who is this for, in what role?*

The key insight here: a person is not just a role — they wear different hats at
different times. A maintainer acting as technical architect needs different documents
than the same person acting as operator or developer. "Perspective" may be richer
than a flat audience list.

Candidates: maintainer-as-architect, maintainer-as-developer, maintainer-as-operator,
job-developer, agent, manager/stakeholder, external/curious.

### 3. State / Maturity
*How much should the reader trust this?*

Not a navigation axis — a **trust signal**. A reader landing on a document needs to
know whether they are reading settled architecture or someone's half-finished thinking.
Acting on a draft as if it were a decision is a real risk.

Candidates: thinking (raw), in-progress, settled, superseded.

---

## A dropped axis — Importance

The original brainstorm included **importance** as a fourth axis (`fundamental`, `medium`, `temporary`).
It was not elevated to load-bearing status during discussion. The reason: importance is largely
derivable from type and state — a settled, platform-wide architecture document is implicitly
fundamental. Making it an explicit tag adds maintenance burden without adding navigational value.
Kept here so the decision not to include it is visible.

---

## The human-interface idea

There is already a tool that compiles Markdown documentation from a repo into a browsable HTML site.
Currently scoped to one branch ("solution-architecture"). The long-term vision: all repos compile
together into a single site with an audience-aware entry point — a "cool entry-guard" that routes
the reader based on who they are and what they want.

This is not to be evolved further now. But it is the reason the tag model matters: without
tag-aware tooling consuming the tags, the taxonomy has no visible payoff. The HTML compiler
becoming tag-aware is the bridge between the taxonomy model and the human-interface vision.

---

## Two hard guardrails

These were identified during the discussion and must not be forgotten:

1. **`josyn-platform` owns the tag schema.** A closed vocabulary per axis, defined
   here, governs all repos. Free-form tags collapse fast.

2. **Value is locked behind tooling.** The tag model only pays off when the HTML
   documentation compiler becomes tag-aware — filtered views, audience routing,
   the "entry-guard" concept. Without that, tags are invisible decoration.

---

## What happens next (when ready)

- Define closed vocabularies for each axis (small, ≤6 values per axis).
- Add a `documentation-taxonomy` entry to the Knowledge Map in `AGENTS.md`.
- Make the HTML compiler tag-aware.
- Only then: tag existing documents incrementally.

---

## What stays out of scope (for now)

- The mechanism for carrying tags (frontmatter, sidecar file, registry) — implementation detail.
- The `thinking/` folder itself as a concept — its name signals maturity by location.
  Documents here are raw. No further state signal needed for this folder.
