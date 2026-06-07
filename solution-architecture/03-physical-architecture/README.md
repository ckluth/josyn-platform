# Physical Architecture

Describes how JOSYN physically exists on a machine. This chapter answers: what runs where,
what lives in which folder, how the parts relate to each other at the infrastructure level.
It is the concrete, spatial picture of the system.

## In Scope
- Machine topology (one backend per machine, no clustering)
- Well-known folder locations (backend, job repository)
- Windows services and executables
- Database — purpose, location, scope
- Job repository layout
- Job registry — structure and ownership

## Out of Scope
- How to install or upgrade the system → see [05-deployment-and-operations/](../05-deployment-and-operations/)
- Identity, certificates, service accounts → see [04-identity-and-security/](../04-identity-and-security/)
- Security policies and compliance → see [06-company-fit/](../06-company-fit/)

## Gray Areas & Dependencies
| Topic | Here | Elsewhere |
|-------|------|-----------|
| Database | Schema, purpose, location | Backup and migration procedures → [05-deployment-and-operations/](../05-deployment-and-operations/) |
| Job registry | Physical structure and data | Governance of who registers jobs → [06-company-fit/](../06-company-fit/) |

## Contents
- [machine-topology.md](machine-topology.md) — One backend per machine, no clustering, machine requirements
- [folder-layout.md](folder-layout.md) — Backend folder, job repository structure, log folder layout
- [services-and-executables.md](services-and-executables.md) — Current and planned services, per-session JAPServer, CLI contract
- [session-store.md](session-store.md) — SQL Server session store, session lifecycle, known gaps
- [job-registry.md](job-registry.md) — What the registry is, current state, open decision on ownership

## Related Decisions
*(none yet)*
