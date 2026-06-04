# Job Deployment

## What "deploying a job" means

Deploying a JOSYN job means placing the job executable and all its dependencies
into the correct subfolder of the `JobRepository`:

```
JobRepository/
└── MyCompany.MyProduct.MyJob/
    ├── MyCompany.MyProduct.MyJob.exe
    └── <dependencies — if not self-contained>
```

The subfolder name is the job's identity. The backend uses it to resolve the path to
`job.exe` when spawning a session. The naming convention for job folders should align
with the job executable name.

## Self-Contained vs Framework-Dependent

A JOSYN job executable can be published as:

| Mode | Behaviour | Trade-off |
|------|-----------|-----------|
| **Self-contained** | Ships with its own .NET runtime | Larger deployment artefact; no runtime pre-requisite on machine |
| **Framework-dependent** | Uses the .NET runtime installed on the machine | Smaller artefact; requires matching runtime version installed |

This is a per-job choice. JOSYN imposes no constraint — process isolation means each job
executable is free to make its own decision independently of other jobs or the backend.

## Installer

*(TBD — no installer exists yet. The backend is currently deployed manually during PoC.)*

## Bootstrapping

*(TBD — the bootstrapping sequence for a fresh machine installation is not yet defined.
Expected concerns: database connection configuration, service account setup, job repository
root path configuration, initial job registration.)*

## Upgrade

*(TBD — upgrade procedure for the backend and for individual job executables.)*
