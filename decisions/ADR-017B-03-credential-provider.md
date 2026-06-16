# ADR-017B-03 — IdentityAdapter (Impersonation Extension Point)

**Date:** 2026-06-16
**Status:** Accepted

---

## Context

ADR-017B-01 establishes that `job.exe` is spawned under the Windows account stored as
`TechnicalUserName` in `JobRegistry`. Resolving the credentials needed for impersonation
is a **company concern** — it depends on the company's identity infrastructure and secrets
management policy.

ADR-020 establishes the adapter model: each company concern is an out-of-process EXE
communicating with JAPServer over JIP. This ADR applies that model to the impersonation
extension point and names it **IdentityAdapter**.

Until this ADR is implemented, `job.exe` is spawned under the JAPServer process identity —
impersonation is inactive.

---

## Decisions

### 1. Naming: IdentityAdapter (not ICredentialProvider)

The extension point is named `IdentityAdapter`, consistent with ADR-020's adapter naming
convention. The JIP protocol interface is `IIdentityAdapter`.

### 2. Contract: single method — `GetPassword`

```
GetPassword(string username) → Result<string>
```

Maps directly to the JipDispatcher's `Task<Result<string>> Method(string)` signature.
Input: the `TechnicalUserName` value from `JobRegistry`.
Output: the plain-text password for that account.

Domain resolution is deferred. JAPServer uses an empty domain string for the initial
implementation (sufficient for local accounts and many domain configurations).

### 3. Contract package: `JOSYN.Backend.IdentityAdapter.Contract`

Lives in `josyn-backend` / `josyn-backend-adapter-contracts/`. This is the empty placeholder
created in anticipation of ADR-020. The package contains only `IIdentityAdapter`.
Dependencies: `JOSYN.Foundation.ResultPattern` only.

### 4. No platform stub

The platform does not ship a stub implementation. `josyn-contoso` owns the stub for
standalone and development deployments.

### 5. Stub implementation in `josyn-contoso`

`josyn-contoso` provides `Contoso.IdentityAdapter.exe`. It implements the `IIdentityAdapter`
JIP protocol and returns passwords read from a local INI file. This makes dev credentials
explicit and configurable without hardcoding them in source.

### 6. HAEVG implementation: deferred

The real HAEVG adapter (Credential Manager, vault, or encrypted config) and its repo
placement are deferred. The contract and stub are sufficient to activate impersonation
in development and to unblock JAPServer implementation.

---

## Resolved questions

- **Contract shape:** ✅ `GetPassword(string username) → Result<string>`
- **Package:** ✅ `JOSYN.Backend.IdentityAdapter.Contract` in `josyn-backend-adapter-contracts/`
- **Platform stub:** ✅ None — Contoso owns the stub
- **Stub repo:** ✅ `josyn-contoso`
- **Credential storage (stub):** ✅ Local INI file
- **Credential storage (HAEVG):** ⬜ Deferred
- **Domain:** ⬜ Deferred — empty string used initially

---

## Consequences

- `JOSYN.Backend.IdentityAdapter.Contract` is the first adapter contract package produced
  under ADR-020. It establishes the pattern for all subsequent adapter contracts.
- JAPServer gains a `GetPassword` call site before `LaunchJobAndStorePid` — impersonation
  becomes active once the adapter is configured.
- `josyn-contoso` gains a new `contoso-identity-adapter/` subfolder and EXE.
- The bootstrap config for Contoso/dev deployments must include:
  ```ini
  [Adapters]
  IdentityAdapter = Contoso.IdentityAdapter.exe
  ```
