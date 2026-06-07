# Isolation vs. Integration

## JOSYN's Identity as a Platform

JOSYN started as — and could remain as — a standalone, MIT-licensed platform.
It has no hard dependency on any specific company's infrastructure, tooling, or conventions.
This is deliberate. It is what makes JOSYN reusable and portable.

This agnosticity is an asset worth protecting. It must not be quietly eroded by convenience
shortcuts that bake in company-specific assumptions at the platform level.

## The Tension

Every real deployment of JOSYN lives inside a specific company's world:
- A specific Active Directory domain
- A specific SQL Server installation
- A specific config-manager or job registry
- A specific deployment process and governance structure
- A specific set of business conventions

JOSYN must fit that world smoothly — but without *becoming* it.

## The Solution: Explicit Extension Points

The answer to this tension is **explicit, named extension points** — adapters, plugins,
and configuration interfaces that separate the stable platform core from the company-specific
layer.

| Concept | Platform responsibility | Company responsibility |
|---------|------------------------|------------------------|
| Session store | `ISessionStore` contract + EF Core implementation | Connection string, schema location, retention policy |
| Global config | `IGlobalConfig` contract | `CompanyConfig` implementation with real values |
| Job registry | Registry contract (TBD) | Where registry data lives (config-manager, own table) |
| Scheduling | Trigger and ticker contracts | Actual schedule definitions per job |
| Identity / impersonation | Impersonation hook (TBD) | Domain user mapping per job |

The platform owns the contracts. The company owns the implementations.
This keeps JOSYN honest about its boundaries and gives companies a clear, stable surface to build on.

## What This Means in Practice

- **No company-specific code in platform packages.** `JOSYN.Foundation.*`, `JOSYN.Jap.*`,
  `JOSYN.JobHost` must never contain any reference to HAEVG AG, the company config-manager,
  or any other company-specific concept.
- **`HardcodedGlobalConfig` is a placeholder**, not a model. The real `GlobalConfig`
  implementation belongs to the company adapter layer — outside the platform proper.
- **Extension points should be documented as such.** A company should be able to read the
  platform docs and understand exactly where to plug in — without reverse-engineering internals.

## The MIT Licence Angle

The MIT licence is not just a legal choice — it signals intent. JOSYN is meant to be
reusable beyond HAEVG AG. Keeping the platform agnostic is what makes that real.
If company-specific logic bleeds into the platform core, the licence becomes fiction.
