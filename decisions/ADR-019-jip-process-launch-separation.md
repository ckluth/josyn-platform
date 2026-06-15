# ADR-019 — JIP: Separate Process Launch from Pipe Server

**Date:** 2026-06-15
**Status:** Accepted

---

## Context

`PipesServer.RunAsync` previously accepted a `ClientExePath` in `IServerStartArguments`
and launched the client executable internally via a private `StartClientExe` method.
An `OnClientProcessStarted` callback was provided to hand the resulting process ID back
to the caller.

This design had a structural flaw: the PID callback fired asynchronously *inside*
`RunAsync`, after the pipe server was already running. Any failure in the callback
(e.g., failing to persist the PID to the session store) could not stop the preparation
flow — it could only fire an error notification. The caller had no synchronous,
checkable `Result` at the point where it mattered.

---

## Decision

Process launch is removed from `PipesServer` entirely.

1. `ClientExePath` and `OnClientProcessStarted` are removed from `IServerStartArguments`
   and `ServerStartArguments`.

2. A new public method is added to `PipesServer`:
   ```csharp
   public static Result<int> StartClientProcess(string exePath, Guid sessionKey)
   ```
   This launches the executable with the JIP session key as a CLI argument and returns
   the process ID. It encapsulates the `PipesProtocol` argument construction, which
   belongs in the JIP layer.

3. Callers are responsible for calling `StartClientProcess` before `RunAsync`,
   handling the `Result<int>`, and persisting the PID as explicit, sequential steps.
   Only after all pre-flight steps succeed may `RunAsync` be called.

4. If a post-launch step fails before `RunAsync` is called, the caller is responsible
   for killing the process (best-effort, by PID) before routing to the error path.

---

## Rationale

1. **Explicit sequencing over hidden callbacks.** A `Result<int>` returned synchronously
   before `RunAsync` is started gives the caller a clear, checkable failure point.
   A void callback fired inside an already-running async task does not.

2. **Single responsibility.** `PipesServer` owns pipe communication. Process lifecycle
   is the caller's concern. Mixing the two produced an invisible coupling between the
   transport layer and session store persistence logic.

3. **No behaviour change for reconnect.** The previous guard
   `if (ClientExePath != null && reConnect) return Fail(...)` existed only because
   auto-launch and reconnect were incompatible. With launch separated out, `RunAsync`
   no longer needs this guard.

---

## Consequences

- `IServerStartArguments` and `ServerStartArguments` no longer carry `ClientExePath`
  or `OnClientProcessStarted`.
- `PipesServer.RunAsync` is simpler: it only manages the pipe lifecycle.
- `Host.Starter` in `josyn-backend-jap-server` now explicitly launches the job executable,
  persists the PID, and checks the result before calling `RunAsync`.
- Any future caller of `PipesServer` that needs to start a client process must use
  `PipesServer.StartClientProcess` and handle the result before calling `RunAsync`.
