# ADR-023 — Introduce `josyn-adapter-contracts` as the Dedicated Adapter Contract Repo

**Date:** 2026-06-17
**Status:** Accepted

---

## Context

ADR-020 established the out-of-process adapter model: each company concern is a standalone
EXE communicating with JAPServer over JIP. Each adapter concern ships a dedicated contract
package defining its JIP protocol. ADR-017B-03 and the ConfigurationAdapter work produced
the first two such packages:

- `JOSYN.Backend.ConfigurationAdapter.Contract`
- `JOSYN.Backend.IdentityAdapter.Contract`

Both currently live inside `josyn-backend` — because that was the most convenient home at
the time they were created. The `JOSYN.Backend.*` prefix they carry reflects where they sat,
not what they are.

These packages are **not** backend internals. They are the rendezvous layer between the
JOSYN platform and the company-world adapter EXEs. JAPServer references them as a caller;
company adapter EXEs (e.g. `Contoso.IdentityAdapter.exe`) reference them as implementers.
Neither side owns them — they belong to the boundary itself.

Keeping them in `josyn-backend` obscures this. The `JOSYN.Backend.*` prefix signals
"backend-internal" to any reader. A consumer of `JOSYN.Backend.IdentityAdapter.Contract`
from `josyn-contoso` sees a dependency that looks like a backend implementation detail —
which is precisely the wrong message.

As the adapter surface grows (WorkflowAdapter, LegacyConfigAdapter, and others are
anticipated in ADR-020), the misplacement compounds: `josyn-backend` accumulates packages
whose true audience is the company-world boundary, not the backend layer.

---

## Decision

### 1. New repo: `josyn-adapter-contracts`

Introduce a dedicated thin repository named **`josyn-adapter-contracts`**. Its sole purpose
is to hold JIP protocol contract packages for adapter concerns. It contains no executable
code, no implementation logic, and no framework dependencies beyond
`JOSYN.Foundation.ResultPattern`.

The repo follows **Pattern B** (multi-solution) — each adapter concern is an independently
releasable package with its own solution, `nuget.config`, and `.local-build/` scripts.

### 2. Rename the packages — `JOSYN.Backend.*` → `JOSYN.Adapter.*`

Moving the packages without renaming them would preserve the misleading prefix. The correct
namespace segment is `Adapter`, reflecting the new vocabulary layer this repo introduces:

| Current name | New name |
|---|---|
| `JOSYN.Backend.ConfigurationAdapter.Contract` | `JOSYN.Adapter.ConfigurationAdapter.Contract` |
| `JOSYN.Backend.IdentityAdapter.Contract` | `JOSYN.Adapter.IdentityAdapter.Contract` |

All future adapter contract packages follow the same pattern:
`JOSYN.Adapter.<ConcernName>.Contract`.

### 3. `josyn-backend` loses the two sub-folders and their pack entries

The solution sub-folders `josyn-backend-configuration-adapter-contract` and
`josyn-backend-identity-adapter-contract` are deleted from `josyn-backend`. The corresponding
entries in `josyn-backend\.local-build\pack.cmd` are removed.

### 4. Deploy script gains a new pack step

`deploy-maintainer.ps1` gains an `Invoke-Pack` call for `josyn-adapter-contracts` inserted
in dependency order — after `josyn-jap` (which provides `JOSYN.Foundation.JIP`) and before
`josyn-backend` (which consumes the adapter contracts via JAPServer).

### 5. Naming conventions document is updated

`architecture/naming-conventions.md` gains `Adapter` as a recognised namespace segment and
`josyn-adapter-contracts` as a registered repo.

---

## Rationale

**The contracts define a boundary, not a layer.**
`JOSYN.Backend.*` names a layer — the scheduler and session-orchestration layer. Adapter
contracts are not part of that layer; they sit at the seam between the platform and the
company world. A dedicated repo makes that seam explicit and navigable as a standalone
artefact.

**The analogy with `josyn-jap` is exact.**
`josyn-jap` exists to hold the JAP protocol contracts (`JOSYN.Jap.Contract`) — the boundary
between the platform and job executables. `josyn-adapter-contracts` is the same structure
one layer outward: the boundary between the platform and company adapter EXEs.
One boundary, one repo.

**The rename is mandatory.**
A package named `JOSYN.Backend.*` that lives in a repo named `josyn-adapter-contracts` and
is consumed by `josyn-contoso` is incoherent. The prefix must match the boundary, not the
prior accident of placement.

**Two packages today, more tomorrow.**
ADR-020 anticipates WorkflowAdapter, LegacyConfigAdapter, and others. Each will produce its
own contract package. The new repo absorbs them naturally; `josyn-backend` would not.

---

## Consequences

- `josyn-backend` becomes a purer backend repo: all packages it ships are genuinely
  backend-internal or backend-executable artefacts.
- `josyn-contoso` updates its `PackageReference` entries for both renamed packages.
- JAPServer updates its `PackageReference` entries for both renamed packages.
- The platform repo table in `AGENTS.md` gains the new repo.
- Every future adapter concern introduced under ADR-020 produces its contract package in
  `josyn-adapter-contracts`, not in `josyn-backend`.

---

## Relation to other ADRs

- **ADR-020** (Company Adapter Model): this ADR sharpens the structural consequence of
  ADR-020's §5 ("one contract package per adapter concern") by giving those packages a
  dedicated home.
- **ADR-017B-03** (IdentityAdapter): the first concrete artefact affected. Its contract
  package is renamed and relocated.
- **ADR-017** (Backend Contracts Package): unaffected — `JOSYN.Backend.Contracts` remains
  in `josyn-backend` and is correct there. Adapter contracts are a separate concern.
- **ADR-004** (JAPServer Relocation): precedent for relocating an artefact to the repo whose
  identity it belongs to, even when moving it requires updating consumers.
