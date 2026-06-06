# Company Fit

> "Good governance is essential — but it comes at a price! Let's always keep the balance: not too strict, not too loose. No overlapping or conflicting concerns between maintenance shortcuts and vital change-audit needs. No implementation or maintenance burden from pure bureaucratic noise. The answer lies in a clean and transparent solution-architecture that always weighs these conflicting interests wisely."

Describes how JOSYN fits into the company's world — technically, organisationally, and
over time. This chapter asks three questions: How does JOSYN protect its platform identity
while adapting to a specific company? How does the company migrate from JobSystem without
disruption? And how does the system remain maintainable as the people who built it move on?

## In Scope
- JOSYN's agnostic identity vs. company-specific tailoring (adapters, extension points)
- The MIT-licence angle — what platform agnosticity means in practice
- Integration with the company's development and release process
- Governance — who owns what decisions
- Compliance and regulatory obligations
- Audit-readiness — what must be logged and traceable
- Migration from JobSystem — parallel existence, what to keep, what to overcome
- Maintainability and knowledge transfer — no expert dependencies, no tribal knowledge
- Organisational identity management (Active Directory, policies)

## Out of Scope
- Technical security implementation → see [04-identity-and-security/](../04-identity-and-security/)
- Deployment procedures → see [05-deployment-and-operations/](../05-deployment-and-operations/)

## Gray Areas & Dependencies
| Topic | Here | Elsewhere |
|-------|------|-----------|
| Identity management | Organisational: AD governance, access policies | Technical: impersonation, service accounts → [04-identity-and-security/](../04-identity-and-security/) |
| Job registry governance | Who is allowed to register jobs, approval process | Physical structure of the registry → [03-physical-architecture/](../03-physical-architecture/) |
| Extension points | What the contracts are and who owns what | Physical implementation (e.g. GlobalConfig) → [03-physical-architecture/](../03-physical-architecture/) |

## Contents
- [isolation-vs-integration.md](isolation-vs-integration.md) — Platform agnosticity, extension points, MIT-licence intent
- [coexistence-with-jobsystem.md](coexistence-with-jobsystem.md) — Migration model, parallel existence, what to keep vs overcome
- [audit-and-session-record.md](audit-and-session-record.md) — Session store as audit trail, current gaps, compliance considerations (TBD)
- [maintainability-and-knowledge-transfer.md](maintainability-and-knowledge-transfer.md) — No expert dependencies, knowledge transfer as a design constraint

## Related Decisions
*(none yet)*
