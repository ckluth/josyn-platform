# Deployment & Operations

Describes how JOSYN gets onto a machine and stays running. This chapter answers: how is
the system installed, how is it bootstrapped, how is it upgraded, and what must an operator
know to run it day-to-day.

## In Scope
- Installer — what it does, what it assumes
- Bootstrapping sequence
- Upgrade and rollback procedures
- Operator conventions and responsibilities
- Database backup and migration

## Out of Scope
- Physical folder structure and service topology → see [03-physical-architecture/](../03-physical-architecture/)
- Certificate and identity setup → see [04-identity-and-security/](../04-identity-and-security/)
- Compliance and audit logging → see [06-company-fit/](../06-company-fit/)

## Gray Areas & Dependencies
| Topic | Here | Elsewhere |
|-------|------|-----------|
| Certificate installation | Operational procedure: how certs are deployed | What certs are and why → [04-identity-and-security/](../04-identity-and-security/) |
| Database migration | Operational: when and how to run migrations | Schema and purpose → [03-physical-architecture/](../03-physical-architecture/) |

## Contents
- [job-deployment.md](job-deployment.md) — Job repository layout, self-contained vs framework-dependent, installer (TBD), bootstrapping (TBD)

## Related Decisions
*(none yet)*
