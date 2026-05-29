# josyn-foundation

**Role:** Infrastructure primitives — no business logic, no topology awareness.
Consumed by all other repos as NuGet packages.

**Location:** `C:\Users\chris\OneDrive\DevGit\josyn-foundation`
**Version:** `1.0.0-preview01`

---

## Packages

```
josyn-foundation-result-pattern/   →  JOSYN.Foundation.ResultPattern
josyn-foundation-property-bag/     →  JOSYN.Foundation.PropertyBag
josyn-foundation-jip/              →  JOSYN.Foundation.JIP
```

### Dependency chain within foundation

```
JOSYN.Foundation.ResultPattern   (no deps)
         ▲                 ▲
         │                 │
 PropertyBag              JIP
```

---

## JOSYN.Foundation.ResultPattern

**Purpose:** Errors-as-values. The single mechanism for propagating failures across the entire platform.

### Key types

```csharp
// Void result
sealed record Result : IResult<Result>
{
    bool Succeeded { get; }
    string? ErrorMessage { get; }
    Exception? Exception { get; }
    IReadOnlyList<CallerInfo> Callers { get; }
    string CallStackAsString { get; }

    static Result Success { get; }
    static Result Fail(string error, Exception? exception = null, ...);
    static Result Propagate(Result inner, ...);
    static Error Error(string error, Exception? exception = null);

    // Implicit conversions in:
    //   Exception → Result
    //   Error → Result
    //   ResultSuccess → Result
}

// Typed result
sealed record Result<TValue> : IResult<Result<TValue>, TValue>
{
    bool Succeeded { get; }
    TValue? Value { get; }
    // ... same error fields as Result

    static Result<TValue> Success(TValue value);
    static Result<TValue> Propagate(Result<TValue> inner, ...);
    Result ToResult();
    Result<TOther> ToResult<TOther>();

    // Implicit conversions in:
    //   TValue → Result<TValue>
    //   Exception → Result<TValue>
    //   Error → Result<TValue>
}

// Error value type
readonly record struct Error(string ErrorMessage, Exception? Exception = null)
{
    // Implicit: string → Error
    // Implicit: Exception → Error
}

// Propagation chain entry
sealed record CallerInfo
{
    string MethodName { get; init; }
    string ClassName { get; init; }
    string FilePath { get; init; }
    int LineNumber { get; init; }
}
```

### Static abstract interfaces

`IResult<TSelf>` and `IResult<TSelf,TValue>` use C# 11 static abstract members — they enforce the contract on sealed record types at compile time without runtime polymorphism.

---

## JOSYN.Foundation.PropertyBag

**Purpose:** Culture-aware serialization of C# records to/from INI or JSON string format.
Used for job argument and result transfer over the JIP wire.

### Key API

```csharp
static class PropertyBag : IPropertyBag
{
    // Serialize to INI (default)
    static Result<string> Serialize<TRecord>(TRecord record)
        where TRecord : class;

    // Serialize with explicit format
    static Result<string> Serialize<TRecord>(TRecord record,
        DictionaryToStringSerializer serializeToString)
        where TRecord : class;

    // Serialize with runtime type
    static Result<string> Serialize(object record, Type recordType,
        DictionaryToStringSerializer serializeToString);

    // Deserialize
    static Result<TRecord> Deserialize<TRecord>(string raw)
        where TRecord : class;
    static Result<object> Deserialize(string raw, Type recordType);
    static Result<object[]> Deserialize(string raw, ParameterInfo[] parameters);
}
```

### Format serializers

```csharp
IniDictionarySerializer.Serialize   // key=value lines
JsonDictionarySerializer.Serialize  // JSON object
```

Format is auto-detected on deserialization (INI vs JSON).

### Supported property types

`string`, `char`, `bool`, numeric types (`byte` → `ulong`, `float`/`double`/`decimal`),
`DateTime`, `DateTimeOffset`, `DateOnly`, `TimeOnly`, `TimeSpan`, `Guid`, any `enum`.
All support `T?` nullable variants.

### Culture

`JosynCulture.Default` = `new CultureInfo("de-DE")` — affects decimal separators and date formatting.
Thread culture must be set to `de-DE` for serialization round-trips to be correct.

---

## JOSYN.Foundation.JIP

**Purpose:** Named-pipe IPC transport + a JSON-based convention layer on top.

### Two layers

```
Transport layer (JOSYN.Foundation.JIP)
    PipesClient, PipesServer, PipesProtocol, ServerStartArguments, ClientPipes, ServerPipes

Convention layer (JOSYN.Foundation.JIP.Jip)
    JipClient, JipServer, JipDispatcher, JipProtocol, Request, Response
```

### Transport layer

```csharp
// Client side
static class PipesClient : IPipesClient
{
    static Task<Result<ClientPipes>> ConnectAsync(Guid sessionKey);
    static Task<Result<byte[]>> SendRequestAsync(byte[] requestBytes, ClientPipes pipes);
    static Task<Result<string>> SendRequestAsync(string request, ClientPipes pipes);
    static Task<Result> DisconnectAsync(ClientPipes pipes, bool sendShutdownRequest = false);
}

// Server side
static class PipesServer : IPipesServer
{
    static Task<Result> RunAsync(ServerStartArguments args);
}

// Configuration
record ServerStartArguments
{
    required Func<string, Task<string>> HandleStringRequest { get; init; }
    required Func<string, Exception, Task> HandleErrorNotification { get; init; }
    Guid SessionKey { get; init; }
}
```

Pipe naming: `JOSYN-REQ-<guid>` (request) + `JOSYN-RSP-<guid>` (response).
Wire protocol: int32 LE length prefix + UTF-8 bytes.
Session isolation: each session uses a unique GUID.
Single-in-flight: a busy guard prevents concurrent requests on the same pipes.

### Convention layer (JIP)

```csharp
// Client: sends JSON requests
static class JipClient : IJipClient
{
    static Task<Result<string?>> SendAsync(ClientPipes pipes, string what, string? data = null);
}

// Server: wraps handler
static class JipServer : IJipServer
{
    static Func<string, Task<string>> WrapHandler(
        Func<Request, Task<Result<string?>>> handler);
}

// Dispatcher: fluent handler registration
sealed class JipDispatcher : IJipDispatcher
{
    IJipDispatcher RegisterHandler(string what, Func<string?, Task<Result<string?>>> handler);
    // RegisterAll<TInterface>(implementation) — auto-discovers methods via reflection
}

// Wire types
record Request(string What, string? Data = null);
record Response(bool Succeeded, string? Data = null, string? Error = null);
```

---

## Build & Package

Each sub-package lives in its own sub-directory with its own `.slnx`, `nuget.config`, and `.local-build/` scripts.

```
.local-build/build.cmd [Release|Debug]
.local-build/test.cmd
.local-build/pack.cmd              ← outputs to ../../local-packages/
```

License: MIT | Company: HAEVG AG | Target: net10.0
