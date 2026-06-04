# Machine Topology

## One Backend per Machine

A JOSYN installation lives on a single server machine — virtual or physical.
The backend services and the job repository reside on the same machine.
There is no cluster, no distributed setup, no cross-machine coordination.

Multiple machines exist in the company (e.g., one per environment: development, test,
production). Each operates as a completely independent JOSYN installation.

## No Clustering

JOSYN does not implement:
- Load balancing across machines
- Failover/fallback between machines
- Shared session stores across machines

This is a deliberate constraint inherited from `JobSystem` and settled for JOSYN.
See [settled-decisions.md](../02-constraints-and-settled-decisions/settled-decisions.md).
The future question of whether JOSYN could scale to clustering is deferred —
see [evolution-horizon.md](../07-evolution-and-lifecycle/evolution-horizon.md).

## Machine Requirements

*(To be defined — OS version, .NET runtime, SQL Server edition, disk layout,
service account requirements.)*
