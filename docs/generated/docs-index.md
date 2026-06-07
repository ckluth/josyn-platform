# Documentation Index

_Generated: 2026-06-07T12:49:05.5961829+02:00 — 119 documents across 10 repos._
_Semantic fields (type, perspective, state, summary) are filled by the AI enrichment pass._

## josyn-backend

| Name | Path | State | Type | Perspective | Summary |
|------|------|-------|------|-------------|---------|
| josyn-backend | README.md | thinking | overview | maintainer-architect, maintainer-developer, maintainer-operator | Will introduce the backend scheduler layer when full documentation exists. |
| JOSYN Error Report | db/error-report.md | settled | reference | maintainer-operator | Lists recent persisted backend error reports from the database. |
| JOSYN Session Report | db/session-report.md | settled | reference | maintainer-operator | Lists recent persisted job session records from the database. |
| JOSYN.Backend.GlobalConfig | josyn-backend-bootstrap-config/JOSYN.Backend.BootstrapConfig/README.md | settled | component-architecture | maintainer-developer | Explains the runtime configuration contract used by backend components. |
| JOSYN.Backend.ErrorHandler | josyn-backend-error-handler/JOSYN.Backend.ErrorHandler/README.md | settled | component-architecture | maintainer-developer, maintainer-operator | Explains the backend-wide error reporting endpoint and contract. |
| JOSYN.Jap.JAPServer | josyn-backend-jap-server/JOSYN.Jap.JAPServer/README.md | settled | component-architecture | maintainer-developer | Explains the backend server executable for JAP sessions. |
| JOSYN.Backend.JobRegistry | josyn-backend-job-registry/JOSYN.Backend.JobRegistry/README.md | settled | component-architecture | maintainer-developer, maintainer-operator | Explains the storage-backed registry of runnable jobs. |
| JOSYN.Backend.SessionStarter | josyn-backend-session-starter/JOSYN.Backend.SessionStarter/README.md | settled | component-architecture | maintainer-developer | Explains the component that allocates sessions and spawns JAPServer. |
| JOSYN.Backend.SessionStore | josyn-backend-session-store/JOSYN.Backend.SessionStore/README.md | settled | component-architecture | maintainer-developer, maintainer-operator | Explains the persistence component for JOSYN job sessions. |

## josyn-commons

| Name | Path | State | Type | Perspective | Summary |
|------|------|-------|------|-------------|---------|
| Advice for Maintainers of josyn-commons | MAINTAINERS.md | settled | reference | maintainer-architect, maintainer-developer, maintainer-operator | Gives repo-specific maintainer guidance for josyn-commons. |
| josyn-commons | README.md | settled | overview | maintainer-architect, maintainer-developer | Introduces josyn-commons as the domain-agnostic utility layer. |
| JOSYN.Commons.Log | josyn-commons-log/JOSYN.Commons.Log/README.md | settled | component-architecture | maintainer-developer | Explains the process-local file logging component in josyn-commons. |

## josyn-contoso

| Name | Path | State | Type | Perspective | Summary |
|------|------|-------|------|-------------|---------|
| josyn-contoso | README.md | settled | overview | maintainer-developer | Introduces the demo company adapter repo used for ADR-009. |

## josyn-foundation

| Name | Path | State | Type | Perspective | Summary |
|------|------|-------|------|-------------|---------|
| josyn-foundation | README.md | settled | overview | maintainer-architect, maintainer-developer | Introduces the foundation repo and its three core infrastructure packages. |
| josyn-foundation-jip | josyn-foundation-jip/README.md | settled | overview | maintainer-developer | Introduces the JIP transport sub-repo and links to package documentation. |
| Changelog | josyn-foundation-jip/JOSYN.Foundation.JIP/CHANGELOG.md | settled | reference | maintainer-developer | Records release history and notable changes for the JIP package. |
| JOSYN.Foundation.JIP | josyn-foundation-jip/JOSYN.Foundation.JIP/README.md | settled | component-architecture | maintainer-developer | Explains the named-pipe transport component used for JOSYN IPC. |
| JOSYN-IPC-Protocol | josyn-foundation-jip/JOSYN.Foundation.JIP/Docs/josyn-ipc-protocol-draft-01.md | thinking | architecture-detail | maintainer-architect, maintainer-developer | Drafts the layered IPC protocol used between JOSYN processes. |
| josyn-foundation-property-bag | josyn-foundation-property-bag/README.md | settled | overview | maintainer-developer | Introduces the PropertyBag sub-repo and links to package documentation. |
| Changelog | josyn-foundation-property-bag/JOSYN.Foundation.PropertyBag/CHANGELOG.md | settled | reference | maintainer-developer | Records release history and notable changes for the PropertyBag package. |
| JOSYN.Foundation.PropertyBag | josyn-foundation-property-bag/JOSYN.Foundation.PropertyBag/README.md | settled | component-architecture | maintainer-developer | Explains the flat record serialization component used in JOSYN IPC. |
| josyn-foundation-result-pattern | josyn-foundation-result-pattern/README.md | settled | overview | maintainer-developer | Introduces the ResultPattern sub-repo and links to package documentation. |
| Changelog | josyn-foundation-result-pattern/JOSYN.Foundation.ResultPattern/CHANGELOG.md | settled | reference | maintainer-developer | Records release history and notable changes for the ResultPattern package. |
| JOSYN.Foundation.ResultPattern | josyn-foundation-result-pattern/JOSYN.Foundation.ResultPattern/README.md | settled | component-architecture | maintainer-developer | Explains the result-based error handling component for JOSYN. |

## josyn-jap

| Name | Path | State | Type | Perspective | Summary |
|------|------|-------|------|-------------|---------|
| josyn-jap | README.md | settled | overview | maintainer-architect, maintainer-developer | Introduces josyn-jap as the shared protocol-contract layer. |
| JOSYN.Jap.Shared | josyn-jap-shared/README.md | settled | overview | maintainer-developer | Introduces the shared JAP libraries used by both session parties. |
| JOSYN.Jap.Shared.Contract | josyn-jap-shared/JOSYN.Jap.Shared.Contract/README.md | settled | component-architecture | maintainer-developer | Explains the shared application contract between JobHost and JAPServer. |

## josyn-job-host

| Name | Path | State | Type | Perspective | Summary |
|------|------|-------|------|-------------|---------|
| JOSYN.JobHost | JOSYN.JobHost/README.md | settled | component-architecture | job-developer, maintainer-developer | Explains the job runtime library used by JOSYN job executables. |

## josyn-platform

| Name | Path | State | Type | Perspective | Summary |
|------|------|-------|------|-------------|---------|
| AGENTS.md — JOSYN Platform | AGENTS.md | settled | agent-instruction | agent | Defines authoritative operating instructions for AI agents working in JOSYN repositories. |
| JOSYN Platform — Maintainer's Guide | MAINTAINERS.md | settled | reference | maintainer-architect, maintainer-developer, maintainer-operator | States the enduring architectural responsibilities and values for JOSYN maintainers. |
| JOSYN Platform | README.md | settled | overview | maintainer-architect, maintainer-developer, maintainer-operator, agent, stakeholder | Introduces JOSYN, its repositories, architecture, and this repo’s authority. |
| JOSYN Platform — Roadmap & Status | ROADMAP.md | in-progress | plan | maintainer-architect, stakeholder | Tracks platform status, milestones, completed work, and likely next steps. |
| Coding Standards — Functional-First C# | architecture/coding-standards.md | settled | reference | maintainer-architect, maintainer-developer, agent | Defines JOSYN’s functional-first C# principles and coding rules. |
| docs-index Vocabulary | architecture/docs-index-vocabulary.md | settled | reference | maintainer-architect, maintainer-developer, agent | Defines the closed vocabularies for docs-index semantic classification. |
| `.local-build` — Local Developer Tooling | architecture/local-build.md | settled | reference | maintainer-developer | Defines the purpose and conventions of the local build wrapper scripts. |
| Naming Conventions | architecture/naming-conventions.md | settled | reference | maintainer-architect, maintainer-developer, agent | Defines naming, namespace, and vocabulary rules across the platform. |
| Architecture Overview | architecture/overview.md | settled | architecture-overview | maintainer-architect, maintainer-developer, agent | Explains the platform structure, runtime flow, and dependency graph. |
| Repo Structure Conventions | architecture/repo-structure-conventions.md | settled | reference | maintainer-architect, maintainer-developer | Defines the allowed repository structure patterns used across JOSYN. |
| JOSYN Storage Realm | architecture/storage.md | settled | architecture-detail | maintainer-architect, maintainer-developer, maintainer-operator | Details the storage realm, schema conventions, and development setup. |
| ADR-001 — Platform Naming Vocabulary | decisions/ADR-001-platform-naming.md | settled | decision | maintainer-architect, maintainer-developer | Establishes the platform naming vocabulary and repo naming boundaries. |
| ADR-002 — Introducing `josyn-backend` as a Separate Layer | decisions/ADR-002-josyn-backend.md | settled | decision | maintainer-architect | Introduces josyn-backend as a separate platform layer. |
| ADR-003 — josyn-commons: A Generic Utility Layer | decisions/ADR-003-josyn-commons.md | settled | decision | maintainer-architect, maintainer-developer | Creates josyn-commons as the generic utility layer outside foundation. |
| ADR-004 — JAPServer Relocation into josyn-backend | decisions/ADR-004-japserver-relocation.md | settled | decision | maintainer-architect, maintainer-developer | Moves JAPServer implementation from josyn-jap into josyn-backend. |
| ADR-005 — Documentation Governance and Agentic Way of Work | decisions/ADR-005-documentation-governance.md | settled | decision | maintainer-architect, agent | Makes josyn-platform the authoritative home for documentation and agent guidance. |
| ADR-006 — Instance Types for Dependency-Holding Services | decisions/ADR-006-instance-types-for-dependency-holding-services.md | settled | decision | maintainer-architect, maintainer-developer | Defines instance-type rules for services that hold runtime dependencies. |
| ADR-007 — The JOSYN Storage Realm | decisions/ADR-007-storage-realm.md | settled | decision | maintainer-architect, maintainer-developer, maintainer-operator | Establishes the governed storage realm for durable platform state. |
| ADR-008 — LocalLog Relocation to josyn-commons | decisions/ADR-008-locallog-relocation.md | in-progress | decision | maintainer-architect, maintainer-developer | Proposes moving LocalLog into josyn-commons as shared logging infrastructure. |
| ADR-009 — Runtime Context Provider Pattern for External Integrations | decisions/ADR-009-runtime-context-provider-pattern.md | settled | decision | maintainer-architect, maintainer-developer | Defines runtime context providers for external integration and environment-specific data. |
| ADR-010 — Environment Separation and Installation Model | decisions/ADR-010-environment-separation.md | in-progress | decision | maintainer-architect, maintainer-operator | Explores the installation and separation model for DEV, INT, and PROD environments. |
| ADR-011 — JIP Wire-Type Extension: Enum Support in JipDispatcher | decisions/ADR-011-jip-enum-support.md | settled | decision | maintainer-architect, maintainer-developer | Adds enum wire-type support to JIP dispatcher registration. |
| ADR-012 — Maintainer Deployment (First Iteration) | decisions/ADR-012-maintainer-deployment.md | in-progress | decision | maintainer-architect, maintainer-operator | Outlines the first deployment model for maintainers during the PoC phase. |
| ADR-013 — Job Dev Mode: Running a Job Locally from the IDE | decisions/ADR-013-job-dev-mode.md | in-progress | decision | maintainer-architect, maintainer-developer, job-developer | Defines a local IDE workflow for running jobs during development. |
| ADR-014 — CLI `run-job` Command and Local Arguments Convention | decisions/ADR-014-cli-run-job-local-args.md | settled | decision | maintainer-developer, maintainer-operator | Defines the run-job CLI command and local argument conventions. |
| Enrich docs-index — AI enrichment prompt | docs/enrich-docs-index-prompt.md | settled | agent-instruction | agent | Provides the standing prompt for AI-based docs-index enrichment. |
| Context Provider PoC Plan | plans/context-provider-poc-plan.md | superseded | plan | maintainer-architect, maintainer-developer | Records the completed PoC plan for validating the context provider pattern. |
| SessionStore -> josyn-backend PoC Roadmap | plans/sessionstore-poc-roadmap.md | in-progress | plan | maintainer-architect, maintainer-developer | Plans the migration of the SessionStore prototype into josyn-backend. |
| PROP-001 — JAPServer Relocation into josyn-backend | proposals/PROP-001-japserver-backend-relocation.md | superseded | decision | maintainer-architect, maintainer-developer | Records the superseded proposal to move JAPServer into josyn-backend. |
| josyn-commons | repos/josyn-commons.md | settled | repo-summary | maintainer-architect, maintainer-developer | Summarizes the role, boundaries, packages, and current state of josyn-commons. |
| josyn-foundation | repos/josyn-foundation.md | settled | repo-summary | maintainer-architect, maintainer-developer | Summarizes the role, packages, and stability expectations of josyn-foundation. |
| josyn-jap | repos/josyn-jap.md | settled | repo-summary | maintainer-architect, maintainer-developer | Summarizes the protocol-contract role and package contents of josyn-jap. |
| josyn-job-host | repos/josyn-job-host.md | settled | repo-summary | maintainer-architect, maintainer-developer, job-developer | Summarizes the job runtime library, responsibilities, and package surface. |
| josyn-backend | repos/josyn-backend/overview.md | settled | repo-summary | maintainer-architect, maintainer-developer, maintainer-operator | Summarizes josyn-backend responsibilities, building blocks, and current architecture. |
| ADR-001 — Backend Building Block Model | repos/josyn-backend/decisions/ADR-001-backend-building-block-model.md | settled | decision | maintainer-architect, maintainer-developer | Defines the building block model used inside josyn-backend. |
| ADR-002 — SessionStore | repos/josyn-backend/decisions/ADR-002-session-store.md | settled | decision | maintainer-architect, maintainer-developer | Introduces SessionStore as a Category A backend component. |
| ADR-003 — GlobalConfig | repos/josyn-backend/decisions/ADR-003-global-config.md | settled | decision | maintainer-architect, maintainer-developer | Introduces GlobalConfig as a separate backend building block. |
| ADR-004 — Backend Restructuring: Trigger Executables, ErrorHandler, and SessionStarter Fire-and-Forget Correction | repos/josyn-backend/decisions/ADR-004-backend-restructuring.md | in-progress | decision | maintainer-architect, maintainer-developer | Proposes backend restructuring and corrected fire-and-forget session startup. |
| ADR-005 — JobRegistry | repos/josyn-backend/decisions/ADR-005-job-registry.md | settled | decision | maintainer-architect, maintainer-developer, maintainer-operator | Introduces JobRegistry as the authoritative record of runnable jobs. |
| ADR-006 — ErrorHandler | repos/josyn-backend/decisions/ADR-006-error-handler.md | in-progress | decision | maintainer-architect, maintainer-developer, maintainer-operator | Proposes a backend-wide error reporting component and contract. |
| Sanity Criteria — Decision-Maker Review | sanity/criteria-review.md | settled | manual | maintainer-architect | Guides periodic review of sanity criteria quality, coverage, and noise. |
| Sanity Check — Last Result (Retired) | sanity/last-result.md | superseded | reference | maintainer-developer, agent | Redirects retired sanity results to the new per-repo state files. |
| Sanity Check — Platform Overview | sanity/overview.md | settled | overview | maintainer-architect, maintainer-developer, agent | Shows the aggregated sanity status across platform repositories. |
| Sanity Check — Method and Manual | sanity/README.md | settled | manual | maintainer-developer, agent | Defines the sanity check method, profiles, workflow, and safety rules. |
| Sanity — Trigger Table | sanity/triggers.md | settled | reference | maintainer-developer, agent | Maps changed file paths to sanity categories for smart-mode runs. |
| Sanity Criteria — architecture | sanity/criteria/architecture.md | settled | reference | maintainer-architect, maintainer-developer, agent | Defines architecture criteria used during sanity checks. |
| Sanity Criteria — docs | sanity/criteria/docs.md | settled | reference | maintainer-developer, agent | Defines documentation completeness and quality criteria for sanity checks. |
| Sanity Criteria — principles | sanity/criteria/principles.md | settled | reference | maintainer-architect, maintainer-developer, agent | Defines principle-based coding criteria for sanity checks. |
| Sanity Criteria — standards | sanity/criteria/standards.md | settled | reference | maintainer-developer, agent | Defines naming and structural standards checked by sanity runs. |
| Sanity Criteria — tests | sanity/criteria/tests.md | settled | reference | maintainer-developer, agent | Defines test coverage, quality, and passing criteria for sanity checks. |
| Sanity State — josyn-backend | sanity/current-state/josyn-backend.md | thinking | reference | maintainer-developer, agent | Will report the current sanity status of josyn-backend once checked. |
| Sanity State — josyn-commons | sanity/current-state/josyn-commons.md | thinking | reference | maintainer-developer, agent | Will report the current sanity status of josyn-commons once checked. |
| Sanity State — josyn-foundation | sanity/current-state/josyn-foundation.md | settled | reference | maintainer-developer, agent | Records the latest sanity findings and fixes for josyn-foundation. |
| Sanity State — josyn-jap | sanity/current-state/josyn-jap.md | settled | reference | maintainer-developer, agent | Records the latest sanity findings and scope for josyn-jap. |
| Sanity State — josyn-job-host | sanity/current-state/josyn-job-host.md | thinking | reference | maintainer-developer, agent | Will report the current sanity status of josyn-job-host once checked. |
| Solution Architecture | solution-architecture/README.md | settled | overview | maintainer-architect, stakeholder | Introduces the solution architecture view of JOSYN in real deployment context. |
| JOSYN and its Predecessor | solution-architecture/01-context/josyn-and-jobsystem.md | settled | solution-architecture | maintainer-architect, stakeholder | Explains JOSYN’s purpose and its relationship to the legacy JobSystem. |
| Context | solution-architecture/01-context/README.md | settled | overview | maintainer-architect, stakeholder | Introduces the context chapter of the solution architecture. |
| Constraints & Settled Decisions | solution-architecture/02-constraints-and-settled-decisions/README.md | settled | overview | maintainer-architect, stakeholder | Introduces the chapter covering fixed constraints and settled decisions. |
| Settled Decisions | solution-architecture/02-constraints-and-settled-decisions/settled-decisions.md | settled | solution-architecture | maintainer-architect, stakeholder | Lists the non-negotiable baseline decisions of the JOSYN solution. |
| Folder Layout | solution-architecture/03-physical-architecture/folder-layout.md | settled | solution-architecture | maintainer-architect, maintainer-operator | Defines the fixed machine folder layout used by JOSYN. |
| Job Registry | solution-architecture/03-physical-architecture/job-registry.md | settled | solution-architecture | maintainer-architect, maintainer-operator | Explains how jobs are registered before JOSYN can execute them. |
| Machine Topology | solution-architecture/03-physical-architecture/machine-topology.md | settled | solution-architecture | maintainer-architect, maintainer-operator, stakeholder | Describes the single-machine deployment topology assumed by JOSYN. |
| Physical Architecture | solution-architecture/03-physical-architecture/README.md | settled | overview | maintainer-architect, maintainer-operator | Introduces the chapter describing JOSYN’s physical deployment structure. |
| Services and Executables | solution-architecture/03-physical-architecture/services-and-executables.md | settled | solution-architecture | maintainer-architect, maintainer-operator | Describes the executables and service roles in the backend runtime. |
| Session Store | solution-architecture/03-physical-architecture/session-store.md | settled | solution-architecture | maintainer-architect, maintainer-operator | Explains the session store’s runtime roles within the deployed solution. |
| Identity and Impersonation | solution-architecture/04-identity-and-security/identity-and-impersonation.md | settled | solution-architecture | maintainer-architect, maintainer-operator | Explains service identities, permissions, and job impersonation requirements. |
| Identity & Security | solution-architecture/04-identity-and-security/README.md | settled | overview | maintainer-architect, maintainer-operator | Introduces the chapter covering identity and security concerns. |
| Job Deployment | solution-architecture/05-deployment-and-operations/job-deployment.md | settled | solution-architecture | maintainer-operator, job-developer | Describes how a job is deployed into the JobRepository. |
| Deployment & Operations | solution-architecture/05-deployment-and-operations/README.md | settled | overview | maintainer-architect, maintainer-operator | Introduces the chapter covering deployment, bootstrapping, and operations. |
| Audit and the Session Record | solution-architecture/06-company-fit/audit-and-session-record.md | settled | solution-architecture | maintainer-architect, maintainer-operator, stakeholder | Explains how session records provide auditability for job executions. |
| Coexistence with JobSystem | solution-architecture/06-company-fit/coexistence-with-jobsystem.md | settled | solution-architecture | maintainer-architect, stakeholder | Explains the planned parallel coexistence of JOSYN and the legacy JobSystem. |
| Isolation vs. Integration | solution-architecture/06-company-fit/isolation-vs-integration.md | settled | solution-architecture | maintainer-architect, stakeholder | Explains JOSYN’s portability and its fit within company-specific infrastructure. |
| Maintainability and Knowledge Transfer | solution-architecture/06-company-fit/maintainability-and-knowledge-transfer.md | settled | solution-architecture | maintainer-architect, stakeholder | Explains how JOSYN is designed for maintainability and knowledge transfer. |
| Company Fit | solution-architecture/06-company-fit/README.md | settled | overview | maintainer-architect, stakeholder | Introduces the chapter about governance, compliance, and organisational fit. |
| Evolution Horizon | solution-architecture/07-evolution-and-lifecycle/evolution-horizon.md | settled | solution-architecture | maintainer-architect, stakeholder | Outlines realistic future evolution paths beyond the current Windows-only solution. |
| Evolution & Lifecycle | solution-architecture/07-evolution-and-lifecycle/README.md | settled | overview | maintainer-architect, stakeholder | Introduces the chapter covering future evolution and lifecycle boundaries. |
| YAGNI Decisions | solution-architecture/07-evolution-and-lifecycle/yagni-decisions.md | settled | solution-architecture | maintainer-architect, stakeholder | Records capabilities deliberately deferred under YAGNI discipline. |
| Documentation and Training — Overview | solution-architecture/08-documentation-and-training/overview.md | settled | solution-architecture | maintainer-architect, maintainer-developer, maintainer-operator, stakeholder | Summarizes current documentation and training coverage across the platform. |
| Documentation & Training | solution-architecture/08-documentation-and-training/README.md | settled | overview | maintainer-architect, maintainer-developer, maintainer-operator, stakeholder | Introduces the chapter covering documentation, onboarding, and training. |
| ArgGen — Inspection & Smoke-Test Guide | temp/arggen-inspection-guide.md | in-progress | manual | maintainer-developer | Provides a temporary inspection and smoke-test guide for ArgGen. |
| PoC Phases 2–5 — Inspection & Runtime Notes | temp/poc-phases-2-5-notes.md | in-progress | working-note | maintainer-developer | Captures temporary inspection and runtime notes from SessionStore PoC work. |
| Documentation Taxonomy — Working Note | thinking/documentation-taxonomy.md | thinking | working-note | maintainer-architect | Brainstorms a future taxonomy for classifying documentation across JOSYN repos. |

## josyn-playground

| Name | Path | State | Type | Perspective | Summary |
|------|------|-------|------|-------------|---------|
| AGENTS.md — JOSYN Playground | AGENTS.md | settled | agent-instruction | agent | Defines AI agent behavior for the consumer playground repository. |
| JOSYN Playground | README.md | settled | overview | maintainer-developer | Introduces the playground repo for demos, exploration, and experiments. |

## josyn-toolbox

| Name | Path | State | Type | Perspective | Summary |
|------|------|-------|------|-------------|---------|
| AGENTS.md — JOSYN Toolbox | AGENTS.md | settled | agent-instruction | agent | Defines AI agent behavior for the maintainer tooling repository. |
| JOSYN Toolbox | README.md | settled | overview | maintainer-developer, maintainer-operator | Introduces the toolbox repo for maintainer scripts and utilities. |
| deploy | deploy/README.md | settled | manual | maintainer-operator | Documents local deployment scripts used by maintainers during development. |
| doc-generator | doc-generator/README.md | settled | manual | maintainer-developer | Documents the HTML documentation generator and its intended usage. |
| docs-index-builder | docs-index-builder/README.md | settled | manual | maintainer-developer | Documents the tool that builds and updates docs-index.json. |
| JOSYN Toolbox | git-tools/README.md | settled | manual | maintainer-operator | Documents convenience scripts for machine bootstrap and Git remote synchronization. |

