# Folder Layout

JOSYN has well-known, fixed locations on a machine. The convention of predictable paths
is intentional — it makes deployment, troubleshooting, and automation reliable.

## Backend Installation Folder

The backend executables and services reside in a well-defined location on the machine.

*(Exact path TBD — to be aligned with the installer design.)*

## Job Repository

```
JobRepository/
└── <JobName>/
    ├── <JobName>.exe
    └── <dependencies — if not self-contained>
```

Jobs physically reside on the same machine as the backend that orchestrates their execution.
Each job has its own subfolder within the `JobRepository` root. The folder name identifies
the job. A job consists of one or more binaries in its subfolder.

This is a direct carry-over from `JobSystem`, where the equivalent was `JobSystem.JobRepository`.
The key improvement: jobs are now executables (`job.exe`) rather than class-library assemblies
(`.dll`). Assembly-resolve pain is gone. Shared process-space is gone.

## Log Folder

Each process writes its logs to a well-known log directory.
The `LocalLog` component (`JOSYN.Jap.Shared.Log`) writes per-date files:

```
<LogDirectory>/
├── <causer>/
│   └── yyyy-MM-dd.log     ← causer-specific (error routing by source process)
└── yyyy-MM-dd.log         ← process-level log
```

*(Log directory paths TBD — to be aligned with installer/deployment conventions.)*
