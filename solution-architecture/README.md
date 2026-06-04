# Solution Architecture

This section describes JOSYN as a **concrete, deployed system in a company's reality**.
It is the Solution Architecture view — distinct from the platform and code architecture
documented elsewhere in this repo. Where the platform view asks *"how is JOSYN designed?"*,
this view asks *"how does JOSYN actually exist — on machines, in organisations, over time?"*

## Chapters

| # | Chapter | What it covers |
|---|---------|----------------|
| 1 | [Context](01-context/) | What JOSYN is, who uses it, what it replaces, what it connects to |
| 2 | [Constraints & Settled Decisions](02-constraints-and-settled-decisions/) | The eternals — things that are fixed and not up for discussion |
| 3 | [Physical Architecture](03-physical-architecture/) | Machines, folder layout, services, backend structure, database, job registry |
| 4 | [Identity & Security](04-identity-and-security/) | Certificates, impersonation, domain users, authentication |
| 5 | [Deployment & Operations](05-deployment-and-operations/) | Installer, bootstrapping, conventions, maintenance |
| 6 | [Company Fit](06-company-fit/) | Governance, compliance, audit-readiness, politics, development process |
| 7 | [Evolution & Lifecycle](07-evolution-and-lifecycle/) | Docker/Linux horizon, YAGNI boundaries, multi-runtime jobs |
| 8 | [Documentation & Training](08-documentation-and-training/) | Public documentation, training materials, onboarding |

## How to Navigate

Start with a chapter that matches your question. Each chapter README describes its scope,
what it explicitly excludes, and where overlapping topics are handled instead. Content lives
in dedicated files within each chapter folder — the README is the map, not the territory.

Architectural decisions are stored canonically in [`decisions/`](../decisions/) at the repo
root. You will never need to browse that folder directly — every decision relevant to a topic
is linked from the content that depends on it.
