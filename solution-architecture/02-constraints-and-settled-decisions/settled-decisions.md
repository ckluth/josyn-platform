# Settled Decisions

These things are **not up for discussion**. They are the fixed baseline of the JOSYN platform.
Opening any of these requires an explicit decision record and consensus — not just a PR.

## Technology: .NET and C#

**.NET and C# are settled.** All backend components, all infrastructure packages, and all
first-party job executables use .NET (currently `net10.0`) and C#. This is a company-wide
constraint, not just a JOSYN one.

> Note: The IPC protocol could in principle support a job written in another language (e.g. Go)
> if it speaks the JAP protocol via named pipes. This is a future possibility, not a current
> commitment. See [07-evolution-and-lifecycle](../07-evolution-and-lifecycle/).

## Execution Model: Process Isolation

**Each job runs as an independent OS process.** This is the architectural cure that JOSYN
provides over the old `JobHost` model. There is no shared-process loading, no assembly-resolve
gymnastics. Each `job.exe` is a standalone executable that targets its own .NET version freely.

This is not a preference — it is the structural foundation of the platform.

## IPC: Named Pipes (JIP)

**JIP (JOSYN Inter-Process) over named pipes is the IPC mechanism.** Two unidirectional pipes
per session: one for requests (`JOSYN-REQ-<guid>`), one for responses (`JOSYN-RSP-<guid>`).
Wire format: length-prefixed binary (int32 LE + UTF-8). Session isolation via GUID-named pipes.

There is no plan to replace this with HTTP, gRPC, or any other network transport. Note: `System.IO.Pipes` abstracts over platform primitives — on Linux it uses Unix domain sockets under the hood — so the JIP transport layer is portable in principle without any protocol change. A Linux port is not planned but is not structurally blocked.

## Session Store: SQL Server

**SQL Server is the session store.** Every job session is persisted. EF Core is the ORM.
The schema namespace is `josyn`. No alternative database engine is planned.

## Topology: Single Machine per Installation

**There is no clustering, no load-balancing, no fallback machine.** Each JOSYN installation
is a single backend on a single server. Multiple machines exist in the company, but they
operate independently — they do not form a cluster. This matches the `JobSystem` model and
is accepted as a deliberate constraint.

## Error Handling: Result Pattern, No Thrown Exceptions

**`JOSYN.Foundation.ResultPattern` is the sole mechanism for propagating failures across
component boundaries.** No thrown exceptions escape process or IPC boundaries. `Result` and
`Result<T>` are returned; callers check `Succeeded` before proceeding. Exceptions are caught
and converted to error values at entry points.

## Serialization Culture: de-DE

**`de-DE` (German) is the serialization culture** for all `PropertyBag` round-trips. Decimal
separators, date formats, and number formatting all follow German conventions. This is a
company convention tied to the business data the jobs process.

## Foundation: Zero Outbound Dependencies

**`josyn-foundation` has zero outbound NuGet dependencies.** It is the infrastructure bedrock.
Any `<PackageReference>` in any `josyn-foundation` project is a violation, with no exceptions.
Internal dependency chain within foundation is fixed:
`ResultPattern` (no deps) ← `PropertyBag` ← `JIP`.

## Jobs Are Executables, Not Class Libraries

**JOSYN jobs are standalone executables (`job.exe`), not class-library assemblies (`.dll`).**
This is the key structural improvement over JobSystem. The old `JobHost` model required loading
job DLLs via reflection into a shared process. JOSYN eliminates this entirely. Job executables
are first-class OS processes.
