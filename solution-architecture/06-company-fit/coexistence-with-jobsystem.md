# Coexistence with JobSystem

## The Migration Challenge

JOSYN does not replace JobSystem on a single cutover date. The two systems run in parallel
during the migration period. Jobs are migrated incrementally — some run on JOSYN, others
still run on JobSystem, on the same machine.

## How Coexistence Works

The clean seam is `SessionStarter`. Each system has its own implementation:

| System | SessionStarter behaviour |
|--------|--------------------------|
| **JobSystem** | Spawns `JobHost.exe <sessionUID>` |
| **JOSYN** | Spawns `JAPServer.exe JOSYN-IPC <guid>` |

Both can run in parallel without interfering. The `TriggerAgent` (or workflow adapter) routes
each job to the correct starter based on whether it has been migrated.

## What to Keep, What to Overcome

Not everything in `JobSystem` is a problem. Some things are settled and sensible.
The migration is not a rewrite-everything exercise — it is a targeted cure for real pain.

| Aspect | Keep | Overcome |
|--------|------|----------|
| Single machine per installation | ✅ Settled constraint | — |
| SQL Server session store | ✅ Carry forward | Extend schema: status column, timestamps, job identity |
| Job repository as a folder tree | ✅ Carry forward | Jobs become executables, not class-library DLLs |
| Per-session GUID | ✅ Carry forward | Now also names the IPC pipes |
| `TriggerAgent` / scheduling concept | ✅ Carry forward | Rewrite in JOSYN, clean of legacy coupling |
| `WorkflowAdapter` concept | ✅ Carry forward | Rewrite as a proper extension point |
| `JobHost.exe` shared process model | — | ❌ Eliminated — replaced by process-isolated `job.exe` |
| WCF server | — | ❌ Eliminated — replaced by named-pipe IPC |
| Assembly-reflection-based job loading | — | ❌ Eliminated — jobs are independent executables |
| Config-manager as job registry | ❓ Open question | May stay or migrate to JOSYN-owned table — see [job-registry.md](../03-physical-architecture/job-registry.md) |



A single job is the migration unit. Migrating a job means:
1. Rewriting it as a `net10.0` Console EXE using `JOSYN.JobHost`
2. Deploying it to the JOSYN `JobRepository`
3. Switching its `SessionStarter` routing from old to new

The old job assembly remains deployed until the switch is complete and verified.

## Political Context

JOSYN is a ground-up rewrite of a settled, production-critical system. This has
organisational implications:

- Stakeholders who depend on JobSystem need confidence in the migration approach
- The incremental coexistence model is partly a technical choice, partly a political one —
  it allows risk to be managed one job at a time
- Governance around when a job is "ready to migrate" needs to be defined with the teams
  who own those jobs

*(Further governance detail: see [governance.md](governance.md) when created.)*
