# Guide — Session Launch

**Relates to:** ADR-007 (Session-Starter Relocation)
**State:** Settled

---

## What happens when you start a job?

This guide explains the session-launch flow for a reader unfamiliar with the
`SessionLauncher` / `JAPServer` split. For the formal decisions see ADR-007.

---

## The core idea

Orchestrators do not manage sessions. They describe *what* to run and hand it off.
Everything else — GUID allocation, persistence, concurrency control, process spawning —
is `JAPServer`'s concern.

```mermaid
flowchart TD
    subgraph Orchestrators["Orchestrators (thin launchers)"]
        CLI["CLI\nrun-job DemoJob args.ini"]
        REST["REST listener\n(future)"]
        TimeSched["TimeScheduler\n(future)"]
        WfRunner["WorkflowRunner\n(future)"]
    end

    subgraph Launcher["JOSYN.Backend.SessionLauncher (library)"]
        L1["1. Check job is registered"]
        L2["2. Resolve TechnicalUserName"]
        L3["3. Base64-encode arguments"]
        L4["4. Write temp file\njosyn-start-&lt;guid&gt;.ini"]
        L5["5. Spawn JAPServer.exe\nJOSYN-START @&lt;path&gt;"]
    end

    subgraph JAPServer["JOSYN.Jap.JAPServer (JOSYN-START mode)"]
        J1["Read + delete temp file"]
        J2["Decode base64 arguments"]
        J3["Turnstile.Run — per job name"]
        J4["Allocate session GUID"]
        J5["SaveNewSession → SessionStore"]
        J6["Spawn job.exe\nJOSYN-IPC &lt;guid&gt;"]
        J7["JAP protocol loop\nGetEnvironment · GetRawArguments · PutResult/Error"]
        J8["Send shutdown token → exit cleanly"]
    end

    CLI -->|SessionStartRequest| L1
    REST -->|SessionStartRequest| L1
    TimeSched -->|SessionStartRequest| L1
    WfRunner -->|SessionStartRequest| L1
    L1 --> L2 --> L3 --> L4 --> L5

    L5 -->|temp INI file| J1
    J1 --> J2 --> J3 --> J4 --> J5 --> J6 --> J7 --> J8
```

---

## Why this split?

Previously every orchestrator carried its own session-start logic. Four orchestrators →
four copies of the same GUID allocation, persistence, and concurrency code.

Now:

- **Orchestrators** construct a `SessionStartRequest` and call `SessionLauncher.LaunchSession()`.
  One line. No session lifecycle knowledge required.
- **`SessionLauncher`** (a NuGet library) handles pre-launch checks and spawns JAPServer.
- **JAPServer** owns the session from start to finish — GUID, `SessionStore`, Turnstile
  lock, job process, and protocol loop.

---

## The temp file

JAPServer is a separate process — arguments can't be passed on the command line (size
limit, no audit trail). The orchestrator writes a short-lived INI file, passes its path
prefixed with `@`, and JAPServer reads and **immediately deletes** it.

The filename contains a throwaway `Guid.NewGuid()` solely to avoid collisions — it is
never stored. The session GUID is allocated later, inside JAPServer, inside the Turnstile.

---

## The base64 detail

Job arguments are an INI file. That content has to travel inside `SessionStartRequest`,
which is itself serialised as an INI file (the temp file). INI has no quoting mechanism
for multiline values — embedding raw INI-in-INI would silently truncate at the first
newline.

The rule: `SessionStartRequest.Arguments` is **always base64-encoded**.

| Step | Who | What |
|------|-----|------|
| Populate `SessionStartRequest.Arguments` | Orchestrator | `Convert.ToBase64String(File.ReadAllBytes(path))` |
| Deserialize request from temp file | JAPServer | plain string — still base64 |
| Decode before `SessionStore.SaveNewSession` | JAPServer | `Convert.FromBase64String(...)` |
| `GetRawArguments` response to `job.exe` | JAPServer | plain INI — base64 never visible to job |

This is a transport-boundary concern only. It is invisible to `job.exe` and to anything
reading the `SessionStore` database directly.

The same rule applies to future orchestrators: a REST listener will receive arguments as
a base64 string in its JSON body — it populates `Arguments` directly, without re-encoding.
The contract is uniform across all orchestrators regardless of how they received the
arguments.
