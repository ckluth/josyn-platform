# Evolution Horizon

## Linux and Docker

**Current state:** JOSYN is Windows-only. Backend services are Windows services.
Named pipes are used for IPC (supported on both Windows and Linux in .NET, but pipe naming
conventions and security descriptor handling differ). SQL Server is the session store.

**Horizon:** Linux and Docker are not planned, but the architecture does not structurally
exclude them. The technical work required would be:

| Blocker | Effort |
|---------|--------|
| Windows-service hosting | Replace with daemon / systemd equivalent |
| Named pipe access control | Minor differences on Linux, likely small |
| SQL Server | Runs on Linux; operationally different but not a fundamental blocker |
| Installer | Entirely different toolchain |

The named-pipe IPC choice (`JOSYN.Foundation.JIP`) is the most Windows-specific part
of the platform. It is a settled decision for now but would be the first thing to
reconsider if a Linux port became a real requirement.

## Multi-Runtime Jobs (Go, Python, etc.)

**Current state:** All first-party jobs are .NET executables.

**Horizon:** Any process that can:
1. Receive `JOSYN-IPC <guid>` as command-line arguments
2. Connect to the two GUID-named pipes (`JOSYN-REQ-<guid>`, `JOSYN-RSP-<guid>`)
3. Speak the JIP wire format: length-prefixed (int32 LE) UTF-8 JSON
4. Implement the three JAP operations: `GetRawArguments` / `PutRawResult` / `PutError`

...can be a JOSYN job — regardless of language. A Go job would need a small JAP client
library (~200 lines). The platform does not need to change.

This is noted as a **feasible future capability**, not a current commitment. The question
of whether to officially support non-.NET jobs (with documentation, client libraries, and
testing) is separate from the question of whether it is technically possible.

## Scalability

**Current state:** One backend per machine. No horizontal scaling.

**The question:** What would real scalability cost — and is it justified here?

Supporting multi-machine, coordinated, load-balanced JOSYN would require:
- Distributed session coordination (who owns which session?)
- A shared or replicated session store
- Failover protocols
- Changes to the `SessionStarter` and `TriggerAgent` design

For the current company scale, this is not justified. The right answer today is YAGNI.
If job volume grows to where a single machine cannot handle the load, the evolution path
would be re-evaluated from first principles at that point — not speculatively designed today.
