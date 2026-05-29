# josyn-job-host

**Role:** Job execution runtime — a NuGet library linked by every JOSYN job executable.
When a job process starts, it calls `Core.Run(args)` and the library handles all
protocol communication, argument deserialization, reflection dispatch, and result routing.

**Location:** `C:\Users\chris\OneDrive\DevGit\josyn-job-host`
**Namespace:** `JOSYN.Jap.JobHost`
**Version:** `1.0.0-preview01`

---

## Architectural position

`josyn-job-host` is a **decoupled consumer** of the JOSYN protocol.
It speaks `IJosynApplicationProtocol` but is not part of `josyn-jap`.
This is an intentional architectural decision — job executables are first-class
external participants, not internal scheduler components.

The repo name (`josyn-job-host`, no "system") reflects this.

---

## Structure

```
josyn-job-host/
├── JOSYN.Jap.JobHost/
│   ├── Core.cs                              ← ICore — main entry point
│   ├── JobInvoker.cs                        ← reflection-based dispatch (internal)
│   ├── JAPClient.cs                         ← IJosynApplicationProtocol over Named Pipes
│   ├── ArgumentsComparer.cs                 ← delegate (placeholder, unused)
│   ├── Contracts/
│   │   └── ICore.cs                         ← static abstract contract for Core
│   └── Attributes/
│       ├── JobEntryPointAttribute.cs        ← marks the single job method
│       ├── BeforeJobEntryPointAttribute.cs  ← pre-job hook (placeholder)
│       ├── JobArgumentsAttribute.cs         ← marks argument type (placeholder)
│       ├── JobResultAttribute.cs            ← marks result type (placeholder)
│       └── ParallelExecutionAllowedAttribute.cs  ← parallel flag (placeholder)
├── JOSYN.Jap.JobHost.Test/
│   ├── JobInvokerTests.cs                   ← 7 tests
│   └── JobInvokerTestSupport.cs             ← fakes, stubs, stub records
└── JOSYN.MyDemoJob/                         ← reference job executable
    ├── Mandatory/MyFirstJob.cs              ← [JobEntryPoint] example
    └── Optional/
        ├── MyArguments.cs                   ← argument record (diverse types)
        └── MyResult.cs                      ← result record
```

---

## Core.Run — entry point

Every job executable calls this from `Main`:

```csharp
// Program.cs in any job executable
private static async Task<int> Main(string[] args) => await Core.Run(args);
```

```csharp
public sealed class Core : ICore
{
    public static async Task<int> Run(string[] args)
    {
        // 1. Set UTF-8 console encoding
        // 2. Connect to JAPServer via JAPClient (parses session GUID from args)
        // 3. JobInvoker.InvokeJob(japClient)
        // 4. On success: exit 0
        // 5. On failure: LocalLog.Error + PutError(ErrorReport) + exit -2
        // 6. On IPC failure: LocalLog.Error + exit -1
    }
}
```

Exit codes:
- `0` — job completed successfully
- `-1` — IPC connection failed
- `-2` — job invocation failed

---

## JobInvoker — reflection dispatch

```csharp
// Public-facing (used in tests)
internal static Task<Result> InvokeJob(IJosynApplicationProtocol japClient,
    IEnumerable<Type> types);

// Flow:
// 1. FindJobFunction(types)           ← finds exactly one [JobEntryPoint] method
// 2. CreateInvocationArguments(...)   ← GetRawArguments() + PropertyBag.Deserialize
// 3. method.Invoke(null, args)        ← static call
// 4. ProcessJobResult(...)            ← PropertyBag.Serialize + PutRawResult()
```

### Argument deserialization

- Zero parameters: invoked with `null` (no server round-trip)
- Single record parameter: `PropertyBag.Deserialize(raw, paramType)` — detected via `<Clone>$` method presence
- Multiple parameters: `PropertyBag.Deserialize(raw, parameterInfos[])`

### Result serialization

- `void` return or `null` with nullable annotation → `Result.Success`, no `PutRawResult` call
- `null` without nullable annotation → `Result.Error("...NULL...")`
- Any value → `PropertyBag.Serialize(value, IniDictionarySerializer.Serialize)` + `PutRawResult`

---

## JAPClient

```csharp
internal sealed class JAPClient : IJosynApplicationProtocol
{
    // Static factory — parses session GUID from args, connects pipes
    internal static Task<Result<JAPClient>> CreateConnectedClient(string[] args);

    public Task<Result<string>> GetRawArguments();
    public Task<Result> PutRawResult(string result);
    public Task<Result> PutError(string serializedError);
}
```

`PutError` serializes an `ErrorReport` via `PropertyBag` before sending.

---

## Attributes

| Attribute | Target | Status | Purpose |
|-----------|--------|--------|---------|
| `[JobEntryPoint]` | Method | ✅ Enforced | Marks the single entry method; exactly one must exist |
| `[BeforeJobEntryPoint]` | Method | ⏳ Placeholder | Pre-job initialization hook; not yet invoked |
| `[JobArguments]` | Class | ⏳ Placeholder | Documents argument type; not validated at runtime |
| `[JobResult]` | Class | ⏳ Placeholder | Documents result type; not validated at runtime |
| `[ParallelExecutionAllowed(bool)]` | Method | ⏳ Placeholder | Scheduler hint; not read by job host (scheduler's concern) |

All placeholder attributes carry `<remarks>Not yet implemented</remarks>` in their XML docs.

---

## How to author a job executable

```csharp
// 1. Create a net10.0 Console App
// 2. Reference JOSYN.Jap.JobHost NuGet package
// 3. Program.cs:
private static async Task<int> Main(string[] args) => await Core.Run(args);

// 4. Job class:
public static class MyJob
{
    [JobEntryPoint]
    public static MyResult Execute(MyArguments args)
    {
        // business logic
        return new MyResult { ... };
    }
}

// 5. Argument and result types must be C# records
public sealed record MyArguments
{
    public required string Name { get; init; }
    public int Count { get; init; }
}

public sealed record MyResult
{
    public required string Message { get; init; }
    public bool Succeeded { get; init; }
}
```

The scheduler spawns the executable with `JOSYN-IPC <sessionGUID>` as args.

---

## Dependencies

```xml
<PackageReference Include="JOSYN.Foundation.JIP" Version="1.0.0-preview01"/>
<PackageReference Include="JOSYN.Foundation.PropertyBag" Version="1.0.0-preview01"/>
<PackageReference Include="JOSYN.Foundation.ResultPattern" Version="1.0.0-preview01"/>
<PackageReference Include="JOSYN.Jap.Shared.Contract" Version="1.0.0-preview01"/>
<PackageReference Include="JOSYN.Jap.Shared.Log" Version="1.0.0-preview01"/>
```

---

## Test coverage

7 NUnit tests in `JobInvokerTests`:

| Test | Scenario |
|------|----------|
| `InvokeJob_NoEntryPoint_Fails` | Zero `[JobEntryPoint]` methods → error with attribute name |
| `InvokeJob_MultipleEntryPoints_Fails` | Two entry points → German "Mehrere" error |
| `InvokeJob_VoidEntryPoint_Succeeds` | Void method, no args → success |
| `InvokeJob_RecordArgument_DeserializesAndExecutes` | Record arg deserialized from INI → success |
| `InvokeJob_RecordReturn_PutsSerializedResult` | Record result serialized → `LastPutResult` set |
| `InvokeJob_GetArgumentsFails_PropagatesError` | `GetRawArguments` fails → error propagated |
| `InvokeJob_JobThrowsUnhandledException_ReturnsGermanError` | Unhandled throw → German error message |

---

## Build & Package

```
.local-build/pack.cmd    ← packs JOSYN.Jap.JobHost to ../../local-packages/
.local-build/test.cmd    ← dotnet test
```

License: MIT | Company: HAEVG AG | Target: net10.0
