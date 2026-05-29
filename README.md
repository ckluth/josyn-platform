# JOSYN Platform

> **This repo is the authoritative architectural reference for the entire JOSYN platform.**
> Start here before working in any other repo.

---

## What is JOSYN?

JOSYN (Job System Next) is a platform for executing scheduled jobs as isolated executable processes. A central scheduler orchestrates job execution: it spawns job processes, hands them arguments via a named-pipe protocol, and receives results or error reports in return.

---

## The Five Repos

| Repo | Role | Namespace root |
|------|------|----------------|
| [`josyn-foundation`](../josyn-foundation) | Infrastructure primitives — Result pattern, serialization, IPC transport | `JOSYN.Foundation.*` |
| [`josyn-system`](../josyn-system) | Per-session JAP server — JAPServer EXE, shared contracts, logging | `JOSYN.System.*` |
| [`josyn-job-host`](../josyn-job-host) | Job execution runtime — library linked by each job executable | `JOSYN.System.JobHost` |
| [`josyn-backend`](../josyn-backend) | Scheduler and session-orchestration layer — triggers sessions, spawns processes | `JOSYN.Backend.*` |
| **`josyn-platform`** *(this repo)* | Architecture, decisions, and cross-cutting documentation | — |

### Why `josyn-job-host` has no dedicated namespace layer

Job executables are **decoupled consumers** of the JOSYN protocol, not internal components of the scheduling system. They link `josyn-job-host`, follow the protocol, and exit. This architectural separation is intentional and is reflected in the repo name (`josyn-job-host`, not `josyn-system-job-host`).

### Why `josyn-backend` is separate from `josyn-system`

`josyn-backend` owns the *when* and *what* of job execution (scheduling, session persistence, process spawning). `josyn-system` owns only the *how* of a single in-flight session (the JAP protocol server). They are coupled solely by the session GUID passed on the command line — there is no NuGet dependency between them. See [decisions/ADR-002-josyn-backend.md](decisions/ADR-002-josyn-backend.md).

---

## Vocabulary

| Word | Meaning in this codebase |
|------|--------------------------|
| **Platform** | The entire JOSYN ecosystem — all five repos together |
| **Backend** | The scheduler and session-orchestration layer (`josyn-backend`) |
| **System** | The per-session JAP protocol server and shared packages (`josyn-system`) |
| **Foundation** | Cross-cutting infrastructure primitives (`josyn-foundation`) |
| **Job Host** | The runtime embedded in each job executable (`josyn-job-host`) |

See [architecture/naming-conventions.md](architecture/naming-conventions.md) for the full naming guide.

---

## Architecture at a glance

```
┌──────────────────────────────────────────────────────────────────────────┐
│  JOSYN Platform                                                          │
│                                                                          │
│  ┌──────────────┐  spawns JAPServer.exe   ┌──────────────────────────┐  │
│  │  josyn-      │─────────────────────────►  JAPServer               │  │
│  │  backend     │  spawns job.exe          │  (josyn-system)          │  │
│  │              │──────────────────────┐  └──────────┬───────────────┘  │
│  └──────────────┘                      │             │ IPC (Named Pipes) │
│                                        │  ┌──────────▼───────────────┐  │
│                                        └──►  job.exe                 │  │
│                                           │  (josyn-job-host)        │  │
│                                           └──────────────────────────┘  │
│                                                                          │
│       josyn-system and josyn-job-host both depend on josyn-foundation   │
└──────────────────────────────────────────────────────────────────────────┘
```

Full architecture: [architecture/overview.md](architecture/overview.md)

---

## Key cross-cutting decisions

- All operations return `Result` / `Result<T>` — no exceptions propagate between layers
- All serialization is via `PropertyBag` — INI or JSON, culture `de-DE`
- IPC uses named pipes with GUID session isolation (`josyn-foundation-jip`)
- Error messages in **German**; documentation and session files in **English**
- All packages target `.NET 10.0`, `Nullable=enable`, `LangVersion=latest`

Coding standards: [architecture/coding-standards.md](architecture/coding-standards.md)

---

## Status

All repos at version `1.0.0-preview01` — active PoC phase. `josyn-backend` is a compilable stub.

See [decisions/](decisions/) for architectural decision records.
