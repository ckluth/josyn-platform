# Identity and Impersonation

## Service Accounts

The JOSYN backend services (future `TriggerAgent`, `Service`, etc.) run under Windows service
accounts. These accounts need sufficient privileges to:
- Read the job repository folder
- Connect to the SQL Server session store
- Spawn child processes (`JAPServer.exe`, `job.exe`)

*(Specific account requirements, naming conventions, and privilege scopes: TBD)*

## Job Impersonation

Each registered job is associated with a **domain user account** for impersonation.
When a job executes, it runs under that domain identity — not the backend service account.

This is a carry-over from `JobSystem` and serves important purposes:
- Jobs that access company resources (file shares, databases, APIs) do so under the
  appropriate business identity
- Audit logs in downstream systems reflect the correct user, not a generic service account
- Privilege separation: the backend service account is not over-privileged

The impersonation user is configured in the job registry — see
[job-registry.md](../03-physical-architecture/job-registry.md).

## Named Pipe Access Control

Named pipes used by JIP (`JOSYN-REQ-<guid>`, `JOSYN-RSP-<guid>`) are per-session and
GUID-named. Because both `JAPServer.exe` and `job.exe` are spawned by the same backend
(or by `JAPServer` itself), pipe access control must permit both processes to connect.

*(Access control configuration and security descriptor requirements: TBD)*

## Certificates

*(TBD — no information available yet on what certificates JOSYN requires, if any.
Likely relevant if the backend exposes HTTP endpoints or uses certificate-based
authentication for inter-service communication.)*
