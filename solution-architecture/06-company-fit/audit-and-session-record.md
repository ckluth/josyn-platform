# Audit and the Session Record

## The Session Store as Audit Trail

Every JOSYN job execution creates a persistent session record in the SQL Server session store.
This provides a built-in, structured audit trail:

- When did the job run?
- What arguments was it called with?
- What result did it produce?
- Did it succeed or fail? With what error?

For a company that must demonstrate audit-readiness, this is a significant improvement
over log-file-based approaches — the record is structured, queryable, and permanent.

## Current Audit Gaps

The current session store has known gaps that must be addressed before it qualifies
as production audit-ready:

| Gap | Impact | Required fix |
|-----|--------|--------------|
| No session status column | Orphaned rows (failed spawns) look like running sessions | Add `status` enum column |
| No separated timestamps | Cannot measure job duration | Add `StartedAt` and `CompletedAt` columns |
| No job identity column | Session row not linked to a registered job name | Add `JobName` or `JobId` foreign key |

These are known limitations, not permanent design choices.

## Compliance Considerations

*(TBD — what specific regulatory or company-policy obligations apply to job execution records?
E.g.: minimum retention period, access control on session data, personal data handling.)*
