# Job Registry

## What is the Job Registry?

Before a job can be executed by JOSYN, it must be **registered**. Registration associates
a job (its name, its executable path in the job repository) with metadata that cannot
live inside the job executable itself:

- Which domain user is used for impersonation when running this job
- Schedule configuration (when to run)
- Any other per-job configuration

Simply having the executable in the `JobRepository` folder is not sufficient — the registry
provides the decoupled metadata store.

## Current State (JobSystem)

In the existing `JobSystem`, job registration lives in the **company's proprietary
config-manager database**. This is an external dependency outside JOSYN's control.

## Open Question

> **Should JOSYN have its own job registry, independent of the company config-manager?**

**Arguments for an own registry** (a table in the JOSYN database):
- JOSYN becomes more self-contained and portable across environments
- The session store database is already there — a `jobs` table is a natural addition
- Removes the external dependency on the company config-manager
- Enables JOSYN to own its own configuration lifecycle

**Arguments for staying with the config-manager:**
- Established operational process — operators already know it
- Single source of truth for all company system configuration
- Avoids building a job registration UI in JOSYN itself

This is an **open decision**. A decision record will be created when this is resolved.
See also [06-company-fit](../06-company-fit/) for the governance angle on this question.
