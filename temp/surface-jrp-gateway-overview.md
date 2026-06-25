# JOSYN Surface · JRP · Gateway — What We're Building (and Why)

A friendly orientation for curious colleagues.
Date: 2026-06-25 · Audience: anyone who wants to know what this corner of JOSYN is about.

---

## The one-paragraph version

JOSYN runs scheduled jobs as isolated processes, orchestrated by a headless backend. Headless is
great for machines but useless for humans — you can't *see* what happened. **Surface** is the
human window onto that headless platform. To make that window work across machines (your laptop
talking to a server's job data), we needed a clean network contract — that's **JRP**. And because
humans don't get to poke the database directly, every write goes through a guarded door on the
platform side — that's the **Gateway**. Three pieces, one goal: let people safely observe and nudge
a system that otherwise has no face.

---

## The three pieces

### 1. Surface — the human window (`josyn-surface`)

This is the part people actually touch. Today it's a command-line tool; later it'll grow a web UI.
You can ask it things like:

```
surface sessions --max 20        # what jobs ran recently, and how did they end?
surface error <guid>             # show me the full error for this failure
surface jobs                     # which jobs are registered?
surface arguments <job>          # what arguments is this job configured with?
surface schedule <job>           # when is this job supposed to run?
surface change-argument <job> <key> <value>   # change a configured argument
```

Five of those are **reads** (just looking). The sixth, `change-argument`, is a **write** (changing
something) — and that one is special, because writes are where you can break things. More on that
under Gateway.

The important design idea: Surface talks to a **seam** called `ISurfaceAgent`, not to a database or
a URL. Right now that seam is filled by throwaway scaffolding (`FakeAgent`) that reads the dev
database directly — a deliberate, dev-only shortcut so we could build and prove the human-facing
parts *before* the real networking exists. When the real network client (`HttpAgent`) arrives, it
slots into the exact same seam and **nothing above it changes**. That's the whole bet: build the
shape now, swap the plumbing later.

### 2. JRP — the contract between machines (`josyn-jrp`)

JRP = **JOSYN Remote Protocol**. It's the agreed-upon vocabulary two machines use to talk about
sessions, jobs, and errors over the network. Think of it as the *menu* both sides have memorised:
"if you ask me for a session summary, here are exactly the fields you'll get back."

JRP is **contracts only** — pure data shapes, no logic, no database, no executable. It ships as two
small packages:

- `JOSYN.Jrp.Launch` — the "please start a job" request/response. This is the *machine-facing* half
  of JRP (see "Not every caller is a human" below).
- `JOSYN.Jrp.Surface` — the read queries, the one write command, and all the data shapes the
  reporting side needs. This is the *human-facing* half.

**Why a separate repo for this?** Because a contract that two different worlds depend on shouldn't
live inside either world. If the wire vocabulary lived inside the backend, every UI tweak would drag
backend internals along with it. Keeping JRP standalone means the human side and the platform side
can evolve independently, as long as they both honour the menu.

> A small but important recent fix: one JRP data shape was secretly borrowing an *internal backend
> type* (the execution-status enum). That meant a backend implementation detail was leaking onto the
> public wire — exactly the kind of boundary erosion that's invisible until it bites. We gave JRP its
> own `SessionStatus` enum and translate the backend's status into it at the edge. Now the contract
> is genuinely self-contained: no backend type crosses the wire.

### 3. Gateway — the guarded door on the platform side (`JOSYN.Backend.Gateway`)

The Gateway is the **platform-resident** half of the conversation. It lives next to the real data,
and it's the *only* sanctioned way for the outside world to make changes. When Surface says "change
this argument," it doesn't touch the database — it asks the Gateway, and the Gateway decides whether
and how to carry that out.

Why bother with a doorman? Because "let the UI write directly to the store" is how every system
eventually grows a dozen inconsistent write paths and a security hole. One guarded door means one
place to validate, authorise, and audit every change.

> Honest status: today the Gateway is a **library** (a command handler other code calls in-process),
> not yet a standalone networked service. The full "hosted Gateway EXE listening over the network"
> is deliberately deferred until the network client lands. We've made sure the docs say *library
> today, hosted service later* rather than overpromising a daemon that doesn't exist yet.

---

## Not every caller is a human

Here's a wrinkle that's easy to miss if you only think of Surface as "a UI for people."

Sometimes the thing that wants to act on JOSYN **isn't a person** — it's another program or a
script, possibly on a different machine, that needs to **kick off a job-session remotely**. Think:
an external system finishes some work and wants to trigger a JOSYN job as the next step; or an ops
script that starts a session on a server from your laptop. No human is sitting there clicking. The
caller is code.

That "start a job-session from somewhere else" capability is a genuine **technical** need, separate
from the human "let me look and tweak" need. It's exactly why JRP has that second package,
`JOSYN.Jrp.Launch` — a small, stable contract for *"start this job, here are its arguments, give me
back the session id."* A thin client could bind **only** that package and do nothing else.

### The history: we used to have a separate Listener

Originally the plan had a **dedicated component for this** — a standalone `JOSYN.Backend.Listener`:
a small network service whose whole job was to sit on a machine, receive "start-job" requests over
the wire, and kick off the session. It was going to be its own EXE, deployed alongside the others.

Then we noticed something. The Listener's one and only capability — *receive a start request →
launch a session* — was the **same capability** the platform-resident agent (now the Gateway)
already needed anyway, for its own trigger / re-trigger verb. Both would be **resident on every
machine**. Both would be a network host. Keeping them separate meant **two doors doing the same
job** — two things to deploy, secure, and keep in sync, for no real gain.

So we made the call (ADR-032): **don't build the standalone Listener.** Fold `start-session` into
the single Gateway as just another verb. **One guarded door** — for both the human's "observe and
tweak" *and* the machine's "start a job remotely."

That consolidation is a big part of *why* the Gateway is framed the way it is: it isn't only the
write-door for humans editing arguments — it's also the **sole sanctioned network launch path** for
programs starting sessions. One component, two kinds of caller, one place to reason about safety.

> Status check: the launch contract (`JOSYN.Jrp.Launch`) exists today, but the Gateway doesn't yet
> *host* it over the network — that hosted service, plus a thin `SessionClient` for script/program
> callers, is part of the deferred work below. The point is the **shape** is already decided: one
> door, not two.

---

```
   You (human)
      │
      ▼
  Surface CLI            ← josyn-surface (the window)
      │
      │  speaks ISurfaceAgent (a seam, not a wire)
      ▼
  FakeAgent  ──reads──►  dev database directly   (throwaway, dev-only shortcut)
      │
      │  writes go a different way ↓
      ▼
  Gateway command handler  ←  JOSYN.Backend.Gateway (the guarded door)
      │
      ▼
  the real job/argument store

  ── and the shared vocabulary everyone speaks ──
  JRP contracts (josyn-jrp): session summaries, queries, the change command, SessionStatus…
```

Reads currently take the dev-only shortcut; the write verb already goes through the proper Gateway
door. When `HttpAgent` arrives, the shortcut disappears and *both* reads and writes travel over JRP
to the Gateway — without any change to the CLI or the data shapes.

---

## Where we are right now (MVP-2b)

- All **six CLI verbs** work end to end against the dev database.
- The wire contracts are cleanly extracted into their own `josyn-jrp` repo.
- JRP is genuinely boundary-clean (no backend types leak onto the wire).
- The build/packaging chain (`jrp → gateway → surface`) builds green from a cleared cache.
- Tests: 20/20 passing.

## What's deliberately *not* done yet

- **`HttpAgent`** — the real over-the-network client. This is the big next step; it retires the
  dev-only `FakeAgent` shortcut entirely.
- **A hosted Gateway service** — the standalone EXE that listens for remote requests.
- **`SessionClient` + `start-session`** — the hosted launch path: letting a program, script, or
  Surface itself actually *start* jobs remotely (not just observe and tweak them). This is where the
  retired Listener's capability finally ships — as a verb on the one Gateway, with a thin client for
  machine callers.
- **More write operations** and eventually a **web UI**.

---

## The mental model to take away

> **Surface** is the face. **JRP** is the shared language. **Gateway** is the guarded door.
> We built the *shapes* first — the seam, the contracts, the door — using dev-only scaffolding
> behind them, so that swapping in real cross-machine networking later is a plumbing change, not a
> redesign. If that bet holds, the day the network client lands, everything above it just works.

Questions welcome — poke any of us working on the surface concern.
