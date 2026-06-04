# YAGNI Decisions

These are things JOSYN **deliberately does not do yet**, with the reasoning recorded here.
Each entry names the deferred capability, explains why it was deferred, and states the
condition under which it would be revisited. YAGNI is a discipline — not an excuse
for leaving things vague.

## No Clustering / No Load Balancing

**Deferred.** The `JobSystem` predecessor runs on single machines with no clustering.
JOSYN inherits this topology. The cost of adding clustering — distributed session
coordination, failover logic, shared state, consensus protocols — is significant.
The benefit is unclear until the company runs jobs at a volume where a single machine
is a genuine bottleneck or reliability risk.

**Revisit condition:** If the company runs jobs at a scale where single-machine reliability
or throughput is insufficient, or if machines need to coordinate execution.

## No Fallback / Failover

**Deferred.** Follows from no clustering. If the backend machine goes down, scheduled
jobs do not run during the outage. This is accepted operational behaviour, consistent
with the current `JobSystem`.

**Revisit condition:** Same as clustering.

## No Management UI

**Deferred.** Job registration and configuration lives in the company config-manager
(or will migrate to a JOSYN-owned database table — see
[job-registry.md](../03-physical-architecture/job-registry.md)).
A dedicated JOSYN management UI for registering jobs, viewing session history,
and triggering manual runs is not planned.

**Revisit condition:** When the company config-manager is no longer a suitable host for
job registry data, or when operator demand makes a purpose-built UI clearly justified.

## No Multi-Runtime Job Support (Yet)

**Deferred.** All first-party jobs are .NET executables. The JAP protocol is technically
transport-agnostic — a Go or Python job could speak it — but no tooling, client libraries,
or testing for non-.NET jobs exists.

**Revisit condition:** When a concrete requirement for a non-.NET job arises.
See [evolution-horizon.md](evolution-horizon.md) for what this would take.
