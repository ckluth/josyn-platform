# ADR-017B-03 — ICredentialProvider (Impersonation Extension Point)

**Date:** 2026-06-12
**Status:** Placeholder — not yet specified

---

## Context

ADR-017B-01 establishes that `job.exe` is spawned under the Windows account stored as
`TechnicalUserName` in `JobRegistry`. Resolving the credentials needed for impersonation
(password, domain) is a **company concern** — it depends on the company's identity
infrastructure and secrets management policy (Credential Manager, encrypted config,
vault, etc.).

The platform defines an extension point (`ICredentialProvider`) and JAPServer depends on
it. The company supplies the implementation as an adapter (following the Contoso adapter
pattern). Until this ADR is implemented, `job.exe` is spawned under the JAPServer process
identity — impersonation is inactive.

---

## Open questions

- What is the exact contract of `ICredentialProvider`
  (input: `TechnicalUserName`; output: credentials record — what fields)?
- Which package does `ICredentialProvider` live in?
- Is a no-op development stub shipped by the platform, or left to the company?
- Does the company implementation live in `josyn-contoso` or a dedicated adapter repo?
- Which credential storage mechanism does the company adapter use
  (Windows Credential Manager, encrypted INI, secrets vault, …)?

---

## Decisions

*(to be filled in when this ADR is taken up)*

---

## Consequences

*(to be filled in when this ADR is taken up)*
