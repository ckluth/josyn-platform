# ADR-011 — JIP Wire-Type Extension: Enum Support in JipDispatcher

**Date:** 2026-06-05
**Status:** Accepted

---

## Context

`JipDispatcher.RegisterAll` auto-discovers handler methods on a protocol interface (e.g.
`IJosynApplicationProtocol`) via reflection and registers them as JIP request handlers.

At the time of writing, it enforces exactly two supported method signatures:

| Signature | Semantics |
|-----------|-----------|
| `Task<Result<string>> Method()` | No input; returns a string payload |
| `Task<Result> Method(string)` | Takes a string payload; returns nothing |

These shapes match JIP's wire format: a request carries an optional `string` data field,
and a response carries an optional `string` data field. JIP is — and remains — a
**string-only transport**.

The problem surfaced when `IJosynApplicationProtocol.GetEnvironment()` was added with the
return type `Task<Result<RuntimeEnvironment>>`. `RuntimeEnvironment` is an enum, not a
`string`. `RegisterAll` hit the `else` branch and failed at startup with a runtime error.

Two repair strategies were considered.

---

## Options considered

### Option A — Keep JIP string-only; handle serialization at JAP endpoints

`IJosynApplicationProtocol.GetEnvironment()` is changed to return `Task<Result<string>>`.
Each endpoint (server returns `env.ToString()`, client parses with `Enum.Parse<T>`) handles
the conversion manually.

**Rejected because:** it leaks JIP's transport constraint into the JAP contract and into
every future implementation. The interface no longer expresses intent — it expresses wire
mechanics. Every consumer must know and replicate the `ToString`/`Parse` contract manually,
with no compiler enforcement and no single point of failure.

### Option B — Extend JipDispatcher to support enums transparently

`RegisterAll` gains a third supported shape:

| Signature | Semantics |
|-----------|-----------|
| `Task<Result<TEnum>> Method()` where `typeof(TEnum).IsEnum` | No input; returns an enum value serialized as its name string |

The dispatcher serializes outbound enum values via `Enum.ToString()` and deserializes
inbound responses via `Enum.Parse<TEnum>()`. The wire is still a string — this is purely
a convention layer concern. The JAP interface retains its typed signature.

**Chosen.**

---

## Decision

`JipDispatcher.RegisterAll` is extended with a third supported method shape:
`Task<Result<TEnum>> Method()` where `TEnum` is an `enum` type.

The supported method shapes for `RegisterAll` are now:

| Signature | Wire encoding |
|-----------|---------------|
| `Task<Result<string>> Method()` | String payload as-is |
| `Task<Result<TEnum>> Method()` where `typeof(TEnum).IsEnum` | `Enum.ToString()` / `Enum.Parse<TEnum>()` |
| `Task<Result> Method(string)` | String payload as-is |

**The rule is explicit and closed:** JIP supports exactly `string` and `enum` at the
convention layer. Records, primitives, value types, and complex objects are not supported
and will continue to fail with a clear error at startup.

---

## Rationale

Enums are a safe, bounded extension:

1. **Lossless wire representation.** `Enum.ToString()` and `Enum.Parse()` are deterministic
   and reversible. No serializer dependency; no ambiguity.
2. **Foundation learns nothing about domain types.** The extension uses only
   `typeof(T).IsEnum` — pure reflection. `JipDispatcher` remains domain-agnostic.
3. **Eliminates a class of boilerplate.** Without this, every typed enum endpoint on every
   protocol implementation must manually implement `ToString`/`Parse` with no enforcement.
4. **Closed set.** Unlike opening for `int` or `record`, the enum case is bounded: the
   serialization contract is universal across all .NET enums, requires no configuration,
   and cannot silently diverge between endpoints.

**The slippery slope is closed by the explicit rule.** Enums are permitted because their
wire representation is a platform primitive. Any request to extend `RegisterAll` further
requires a new ADR.

---

## Consequences

- `IJosynApplicationProtocol.GetEnvironment()` returns `Task<Result<RuntimeEnvironment>>` —
  the interface stays type-safe and expressive.
- `JAPServer.GetEnvironment()` returns `RuntimeEnvironment.DEV` (or the configured value);
  `RegisterAll` serializes it transparently.
- `JAPClient.GetEnvironment()` receives the string from the wire; `RegisterAll`'s reflection
  wrapper deserializes it to `RuntimeEnvironment` before the return value reaches the caller.
- `architecture/overview.md` is updated to document all three supported `RegisterAll` shapes.
- The startup error message in `RegisterAll`'s `else` branch is updated to list all three
  supported signatures.

---

## Relation to other ADRs

- No existing ADR is superseded.
- This ADR is purely additive to the JIP convention layer described in
  `architecture/overview.md` under *IPC Protocol*.
