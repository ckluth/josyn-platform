# Documentation and Training — Overview

## Current State

No formal end-user or operator documentation exists yet.
The `josyn-platform` repository serves as the primary knowledge base for developers
working on the platform itself.

## Planned Documentation

| Document | Audience | Status |
|----------|----------|--------|
| Operator Guide | System operators | Not started |
| Job Developer Guide | Job authors | Not started |
| Migration Guide | Teams migrating jobs from JobSystem | Not started |
| API Reference | Job developers | Seed exists: XML docs in `JOSYN.JobHost` and architecture docs in `josyn-platform` |

## Training Needs

**Job developers:**
- How to author a JOSYN job — the `[JobEntryPoint]` pattern
- Argument and result records — `PropertyBag` serialization, supported types
- Error handling — the `Result` pattern, what to return on failure
- How to test a job executable

**Operators:**
- Installation and bootstrapping of the backend
- Job deployment to the `JobRepository`
- Monitoring and log file locations
- Troubleshooting common failures (IPC errors, failed sessions)

**Migration teams:**
- What changes when moving a job from JobSystem to JOSYN
- The coexistence model — how old and new jobs run in parallel
- Migration checklist per job

## Notes

The `josyn-job-host` package is the primary developer-facing API. Its XML documentation
and the `josyn-platform` architecture docs form the natural seed of the Job Developer Guide.
Writing that guide is the highest-leverage documentation investment for JOSYN right now.
