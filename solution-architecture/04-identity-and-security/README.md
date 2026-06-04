# Identity & Security

Describes how JOSYN handles identity and secures itself and the jobs it runs. This chapter
answers: who is allowed to do what, how are domain identities used, and how is trust
established between components.

## In Scope
- Windows identity and service accounts
- Job impersonation — domain users connected to jobs
- Certificates — purpose, location, lifecycle
- Authentication between components

## Out of Scope
- Network and firewall rules (infrastructure team concern, outside JOSYN)
- Application-level authorisation logic → see [architecture/](../../architecture/)
- Compliance and audit requirements around identity → see [06-company-fit/](../06-company-fit/)

## Gray Areas & Dependencies
| Topic | Here | Elsewhere |
|-------|------|-----------|
| Certificates | Technical: what certs, where, how used | Deployment: how certs are provisioned → [05-deployment-and-operations/](../05-deployment-and-operations/) |
| Identity management | Technical configuration and impersonation | Organisational: AD governance, policies → [06-company-fit/](../06-company-fit/) |

## Contents
- [identity-and-impersonation.md](identity-and-impersonation.md) — Service accounts, per-job domain user impersonation, named pipe access control, certificates (TBD)

## Related Decisions
*(none yet)*
