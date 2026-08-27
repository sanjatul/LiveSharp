# Phase 01 — Core Abstractions

| | |
|---|---|
| **Version at exit** | `0.1.0` (with Phase 02) |
| **Depends on** | Phase 00 |
| **Branch** | `feat/phase-01-abstractions` |
| **Tasks** | `P01.T1` … `P01.T6` |
| **Read before starting** | `EXECUTION_PLAN.md` §3, §5, §6; `docs/architecture/decisions/ADR-004`, `ADR-007`, `ADR-008`, `ADR-009` |

---

## Goal

Create `LiveSharp.Abstractions` and `LiveSharp.Core`: the contracts and the configuration/DI
foundation that every later phase builds on. This is the highest-leverage phase in the project —
these names and shapes appear in every subsequent file, and changing them later is a breaking
change.

The single most important structural property established here: **`LiveSharp.Core` contains no
ASP.NET Core reference**. All domain logic is unit-testable with no web host, no SignalR, and no
network. Phase 03 plugs a transport in behind an interface defined here.

---

## Non-goals — DO NOT implement in this phase

- No connection registry implementation. `P01` defines `IConnectionRegistry`; **Phase 02
  implements it**.
- No transport implementation of any kind. Not even in-memory — that is `P02.T4`.
- No hub, no endpoint mapping, no `MapLiveSharp`, no SignalR reference anywhere.
- No chat, message, presence, typing, group, call, or signalling types. Those belong to their own
  phases and adding "just the interface now" produces speculative abstractions that will be wrong
  (Spec §53: "Do not create all of these interfaces unless there is a concrete requirement").
- No filter/middleware pipeline. Phase 09 introduces it when there is a real cross-cutting need.
  Design the router so adding it later is additive.
- No stores (`IMessageStore`, `IPresenceStore`, `ICallStore`). Each arrives with its feature.

> If you find yourself writing an interface whose only implementation would be a `Null*` no-op and
> whose only consumer does not exist yet, stop — it belongs to a later phase.

---

## Tasks

### P01.T1 — Projects, DI wiring of the test harness, architecture tests

**Deliverables**

```
src/LiveSharp.Abstractions/LiveSharp.Abstractions.csproj
src/LiveSharp.Abstractions/PublicAPI.Shipped.txt        (empty)
src/LiveSharp.Abstractions/PublicAPI.Unshipped.txt
src/LiveSharp.Abstractions/AssemblyInfo.cs              (InternalsVisibleTo tests)
src/LiveSharp.Core/LiveSharp.Core.csproj
src/LiveSharp.Core/PublicAPI.Shipped.txt                (empty)
src/LiveSharp.Core/PublicAPI.Unshipped.txt
src/LiveSharp.Core/AssemblyInfo.cs
src/BannedSymbols.txt                                   (shared, linked by src projects)
tests/LiveSharp.Abstractions.Tests/…
tests/LiveSharp.Core.Tests/…
tests/LiveSharp.ArchitectureTests/…
tests/LiveSharp.TestKit/…                               (shared test helpers, not packable)
```

Delete `tests/LiveSharp.BuildSmokeTests` from Phase 00 and record the deletion.

**`LiveSharp.Abstractions.csproj`** package references — and nothing else, ever:

```
Microsoft.Extensions.DependencyInjection.Abstractions
Microsoft.Extensions.Logging.Abstractions
Microsoft.Extensions.Options
```

**`LiveSharp.Core.csproj`** — `ProjectReference` to `Abstractions`, plus:

```
Microsoft.Extensions.Options.ConfigurationExtensions
Microsoft.Extensions.Hosting.Abstractions      (for BackgroundService in Phase 02)
Microsoft.Extensions.Diagnostics.Abstractions
```

**`src/BannedSymbols.txt`** — mechanically enforces ADR-008 and §5.2:

```
P:System.DateTime.UtcNow; use TimeProvider.GetUtcNow() — see ADR-008
P:System.DateTime.Now; use TimeProvider.GetUtcNow() — see ADR-008
P:System.DateTimeOffset.UtcNow; use TimeProvider.GetUtcNow() — see ADR-008
P:System.DateTimeOffset.Now; use TimeProvider.GetUtcNow() — see ADR-008
M:System.Diagnostics.Stopwatch.StartNew; use TimeProvider.GetTimestamp() — see ADR-008
M:System.Threading.Tasks.Task.Delay(System.TimeSpan); use the TimeProvider overload — see ADR-008
M:System.Threading.Tasks.Task.Run``1(System.Func{``0}); prefer explicit scheduling
P:System.Threading.Tasks.Task`1.Result; do not block on async work
M:System.Threading.Tasks.Task.Wait; do not block on async work
```

**`tests/LiveSharp.TestKit`** — shared, non-packable helpers used by every later test project.
Create these now so later phases do not each invent their own:

| Helper | Purpose |
|---|---|
| `LiveSharpTestHost` | builds a `ServiceProvider` with `AddLiveSharp`, a `FakeTimeProvider`, and a capturing logger; exposes `GetRequiredService<T>()` and `IAsyncDisposable` |
| `CapturingLoggerProvider` / `LogRecord` | records structured log entries so tests can assert on `EventId`, level, and the **absence** of sensitive values |
| `FakeTimeProvider` re-export + `AdvanceAsync(TimeSpan)` | advances time and yields so timer callbacks run before assertions |
| `ConcurrencyHarness.RunParallelAsync(int workers, int iterations, Func<int,int,Task>)` | standard parallel-stress driver used by every registry/store test |
| `AssertCancellation.PropagatesAsync(Func<CancellationToken,Task>)` | asserts a pre-cancelled token throws `OperationCanceledException` |

**`tests/LiveSharp.ArchitectureTests`** — reflection-based guard rails for `EXECUTION_PLAN.md` §3.
Written with plain reflection over `Assembly.GetReferencedAssemblies()`; no third-party
architecture-testing package (dependency policy — the BCL is sufficient).

Required tests:

| Test | Asserts |
|---|---|
| `Abstractions_references_only_allowed_assemblies` | referenced assembly simple names ⊆ { `System.*`, `netstandard`, `Microsoft.Extensions.DependencyInjection.Abstractions`, `Microsoft.Extensions.Logging.Abstractions`, `Microsoft.Extensions.Options`, `Microsoft.Extensions.Primitives` } |
| `Core_does_not_reference_AspNetCore` | no referenced assembly starts with `Microsoft.AspNetCore` |
| `Core_does_not_reference_EntityFramework_Redis_or_SIPSorcery` | negative name checks |
| `No_src_assembly_references_a_client_or_example_assembly` | negative name checks |
| `All_public_types_in_shipping_assemblies_are_documented` | every public type/member has XML docs (read the generated `.xml` and diff against reflected members) |
| `All_public_non_static_classes_are_sealed_or_explicitly_marked_open` | fails unless the type carries `[DesignedForInheritance]` (an internal attribute defined in TestKit) — forces a conscious decision |
| `Public_async_methods_accept_a_CancellationToken` | every public method returning `Task`/`Task<T>`/`ValueTask`/`ValueTask<T>` has a final `CancellationToken` parameter, unless annotated `[NoCancellation]` with a reason |

These tests are cheap to write once and prevent a whole class of drift for 17 phases. Keep them
green in every later phase.

---

### P01.T2 — Identity, connection, and envelope contracts

**Namespace:** `LiveSharp`

**Deliverables** — `src/LiveSharp.Abstractions/`

```
RealtimeConnection.cs
IRealtimeConnection.cs
ConnectionState.cs
IUserResolver.cs
RealtimeEnvelope.cs
```

**Public API (exact):**

```csharp
namespace LiveSharp;

/// <summary>Describes a single live client connection owned by one authenticated user.</summary>
public interface IRealtimeConnection
{
    string ConnectionId { get; }
    string UserId { get; }
    DateTimeOffset ConnectedAt { get; }
    DateTimeOffset LastSeenAt { get; }
    string? NodeId { get; }
    IReadOnlyDictionary<string, string> Metadata { get; }
}

/// <summary>Immutable snapshot of a connection. Safe to hand out and to cache.</summary>
public sealed record RealtimeConnection(
    string ConnectionId,
    string UserId,
    DateTimeOffset ConnectedAt,
    DateTimeOffset LastSeenAt,
    string? NodeId = null,
    IReadOnlyDictionary<string, string>? Metadata = null) : IRealtimeConnection;

public enum ConnectionState
{
    Disconnected = 0,
    Connecting = 1,
    Connected = 2,
    Reconnecting = 3,
}

/// <summary>Maps an authenticated principal onto the stable user identifier LiveSharp routes on.</summary>
public interface IUserResolver
{
    string? ResolveUserId(ClaimsPrincipal principal);
}

/// <summary>A single server-to-client message. <see cref="Event"/> values come from <see cref="RealtimeEvents"/>.</summary>
public sealed record RealtimeEnvelope(
    string Event,
    JsonElement Payload,
    string? CorrelationId = null);
```

**Design notes to honour**

- `Metadata` is `IReadOnlyDictionary<string, string>` — **not** `object`. It crosses the wire and
  crosses process boundaries in Phase 15; anything else invites serialization surprises. Default to
  an empty frozen dictionary, never `null`, so consumers never null-check.
- `LastSeenAt` lives on the connection, not in a side table, because Phase 02's reaper and Phase 06's
  presence TTL both need it and duplicating it invites divergence.
- `NodeId` is nullable and unused until Phase 15. It is included now because adding a property to a
  public record later is technically non-breaking but forces every construction site to change.
  Document it as "reserved for scale-out; `null` on a single node".
- `ClaimsPrincipal` is `System.Security.Claims` — part of the BCL, so `Abstractions` stays free of
  ASP.NET Core. Verify with the architecture test.
- Deliberately **no** `IRealtimeConnection.SendAsync`. Sending is the transport's job
  (`IRealtimeTransport`); putting it on the connection would make every connection snapshot a live
  handle and destroy the ability to cache or replicate them.

**Tests** — `LiveSharp.Abstractions.Tests`

| Test | Asserts |
|---|---|
| `RealtimeConnection_defaults_metadata_to_empty` | `Metadata` is non-null and empty when omitted |
| `RealtimeConnection_value_equality_ignores_reference_identity` | two identical records are equal |
| `RealtimeConnection_with_expression_preserves_other_members` | `with { LastSeenAt = x }` |
| `ConnectionState_values_are_stable` | explicit numeric values pinned — these cross the wire |
| `RealtimeEnvelope_requires_non_empty_event` | guard throws `ArgumentException` naming the parameter |

---

### P01.T3 — Error model: `OperationResult`, error codes, exceptions

Implements ADR-007. This is the single most reused shape in the codebase — get it right.

**Deliverables** — `src/LiveSharp.Abstractions/`

```
OperationResult.cs
RealtimeErrorCode.cs
Exceptions/LiveSharpException.cs
Exceptions/LiveSharpConfigurationException.cs
Exceptions/LiveSharpTransportException.cs
Exceptions/LiveSharpOperationException.cs
```

**Public API (exact):**

```csharp
namespace LiveSharp;

/// <summary>Outcome of an operation that can fail for expected, non-exceptional reasons.</summary>
public readonly record struct OperationResult
{
    public bool Succeeded { get; }
    public string? ErrorCode { get; }
    public string? ErrorMessage { get; }

    public static OperationResult Success();
    public static OperationResult Failure(string errorCode, string errorMessage);
}

public readonly record struct OperationResult<TValue>
{
    public bool Succeeded { get; }
    public TValue? Value { get; }
    public string? ErrorCode { get; }
    public string? ErrorMessage { get; }

    public static OperationResult<TValue> Success(TValue value);
    public static OperationResult<TValue> Failure(string errorCode, string errorMessage);

    public bool TryGetValue([NotNullWhen(true)] out TValue? value);

    public static implicit operator OperationResult(OperationResult<TValue> result);
}

/// <summary>Machine-readable error codes surfaced to clients. Values are part of the wire contract.</summary>
public static class RealtimeErrorCode
{
    public const string UnknownOperation   = "unknown_operation";
    public const string InvalidPayload     = "invalid_payload";
    public const string Unauthenticated    = "unauthenticated";
    public const string Forbidden          = "forbidden";
    public const string NotFound           = "not_found";
    public const string RateLimited        = "rate_limited";
    public const string PayloadTooLarge    = "payload_too_large";
    public const string RecipientOffline   = "recipient_offline";
    public const string Conflict           = "conflict";
    public const string ProtocolMismatch   = "protocol_mismatch";
    public const string InternalError      = "internal_error";
}
```

**Rules**

- `OperationResult` is a `readonly record struct` — these are allocated per operation on a hot
  path; a class would add an allocation per message.
- Accessing `Value` on a failed result must throw `InvalidOperationException` with a message that
  names the error code, so a misuse in a consumer's code is immediately diagnosable. Prefer
  `TryGetValue` in library code.
- `Failure` must reject null/empty `errorCode` and `errorMessage` — a failure with no reason is
  worse than an exception.
- `ErrorMessage` is developer-facing and **must never contain user content, tokens, or PII**,
  because it is returned over the wire. Document this on the member and add a test that a helper
  for building failures does not interpolate payload data.
- Which to use, documented in `docs/architecture/overview.md`:
  - `throw` — misconfiguration, programmer error, invariant violation, transport death.
  - `OperationResult` — anything the remote caller could legitimately cause: not found,
    forbidden, offline, rate limited, invalid input.

**Exception hierarchy**

```csharp
public class LiveSharpException : Exception                       // base, catch-all for consumers
public sealed class LiveSharpConfigurationException : LiveSharpException
public sealed class LiveSharpTransportException : LiveSharpException
public sealed class LiveSharpOperationException : LiveSharpException  // carries ErrorCode
```

Each: three standard constructors (`()`, `(string)`, `(string, Exception)`).
`LiveSharpOperationException` additionally exposes `string ErrorCode { get; }` and a
`FromResult(OperationResult)` factory, so a client that prefers exceptions can convert.
`LiveSharpException` is the **only** non-sealed type here — the base is designed for catching, the
leaves are not designed for inheritance.

**Tests**

| Test | Asserts |
|---|---|
| `Success_result_has_no_error` | `Succeeded`, null code and message |
| `Failure_result_carries_code_and_message` | round-trip |
| `Failure_rejects_null_or_empty_code` / `…_message` | `ArgumentException` naming the parameter |
| `Value_on_failed_result_throws_with_error_code_in_message` | message contains the code |
| `TryGetValue_returns_false_and_null_on_failure` | |
| `TryGetValue_returns_true_and_value_on_success` | `NotNullWhen` honoured |
| `Implicit_conversion_to_non_generic_preserves_failure` | code and message preserved |
| `Default_struct_is_a_failure_not_a_success` | **critical** — `default(OperationResult)` must not be mistaken for success. Ensure the internal flag is inverted or a state enum is used so the default value is `Failure` with code `internal_error` |
| `LiveSharpOperationException_FromResult_carries_code` | |
| `All_error_code_constants_are_unique_and_snake_case` | reflection over `RealtimeErrorCode` |

> `Default_struct_is_a_failure_not_a_success` is not a nice-to-have. A struct whose default value
> means "success" will eventually let an uninitialised field silently authorise something.

---

### P01.T4 — Transport, registry, and handler contracts

These are the seams. No implementations in this phase.

**Deliverables** — `src/LiveSharp.Abstractions/`

```
IRealtimeTransport.cs
IConnectionRegistry.cs
IGroupRegistry.cs
IOperationHandler.cs
RealtimeOperationContext.cs
```

**Public API (exact):**

```csharp
namespace LiveSharp;

/// <summary>Delivers envelopes to connected clients. Implemented once per transport technology.</summary>
public interface IRealtimeTransport
{
    ValueTask SendToConnectionAsync(string connectionId, RealtimeEnvelope envelope, CancellationToken cancellationToken = default);
    ValueTask SendToUserAsync(string userId, RealtimeEnvelope envelope, CancellationToken cancellationToken = default);
    ValueTask SendToUsersAsync(IReadOnlyCollection<string> userIds, RealtimeEnvelope envelope, CancellationToken cancellationToken = default);
    ValueTask SendToGroupAsync(string groupName, RealtimeEnvelope envelope, CancellationToken cancellationToken = default);
    ValueTask SendToGroupExceptAsync(string groupName, IReadOnlyCollection<string> excludedConnectionIds, RealtimeEnvelope envelope, CancellationToken cancellationToken = default);
    ValueTask AddToGroupAsync(string connectionId, string groupName, CancellationToken cancellationToken = default);
    ValueTask RemoveFromGroupAsync(string connectionId, string groupName, CancellationToken cancellationToken = default);
    ValueTask DisconnectAsync(string connectionId, string? reason = null, CancellationToken cancellationToken = default);
}

/// <summary>Tracks which connections exist and which user owns each one.</summary>
public interface IConnectionRegistry
{
    ValueTask AddAsync(RealtimeConnection connection, CancellationToken cancellationToken = default);
    ValueTask<bool> RemoveAsync(string connectionId, CancellationToken cancellationToken = default);
    ValueTask<RealtimeConnection?> FindAsync(string connectionId, CancellationToken cancellationToken = default);
    ValueTask<IReadOnlyList<RealtimeConnection>> GetUserConnectionsAsync(string userId, CancellationToken cancellationToken = default);
    ValueTask<bool> IsOnlineAsync(string userId, CancellationToken cancellationToken = default);
    ValueTask TouchAsync(string connectionId, CancellationToken cancellationToken = default);
    ValueTask<int> GetConnectionCountAsync(CancellationToken cancellationToken = default);
}

/// <summary>Tracks group membership independently of the transport's own group mechanism.</summary>
public interface IGroupRegistry
{
    ValueTask AddAsync(string groupName, string connectionId, CancellationToken cancellationToken = default);
    ValueTask<bool> RemoveAsync(string groupName, string connectionId, CancellationToken cancellationToken = default);
    ValueTask RemoveConnectionFromAllAsync(string connectionId, CancellationToken cancellationToken = default);
    ValueTask<IReadOnlyList<string>> GetGroupsForConnectionAsync(string connectionId, CancellationToken cancellationToken = default);
    ValueTask<IReadOnlyList<string>> GetConnectionsInGroupAsync(string groupName, CancellationToken cancellationToken = default);
}

/// <summary>Everything a handler needs about the caller. Constructed by the transport adapter.</summary>
public sealed class RealtimeOperationContext
{
    public required string Operation { get; init; }
    public required IRealtimeConnection Connection { get; init; }
    public required string CorrelationId { get; init; }
    public ClaimsPrincipal? User { get; init; }
    public IServiceProvider Services { get; init; }
    public CancellationToken CancellationToken { get; init; }
}

/// <summary>Handles one client-to-server operation. Registered by feature packages.</summary>
public interface IOperationHandler
{
    static abstract string Operation { get; }
}

public interface IOperationHandler<TRequest, TResponse> : IOperationHandler
{
    ValueTask<OperationResult<TResponse>> HandleAsync(TRequest request, RealtimeOperationContext context);
}

/// <summary>Handler for operations with no response payload.</summary>
public interface IOperationHandler<TRequest> : IOperationHandler
{
    ValueTask<OperationResult> HandleAsync(TRequest request, RealtimeOperationContext context);
}
```

**Design notes to honour**

- `IRealtimeTransport` is intentionally **write-only**. Inbound flow is the router's concern.
  Keeping them separate is what allows the in-memory transport in Phase 02 to be 60 lines.
- `ValueTask` throughout: the in-memory implementations complete synchronously, and this interface
  is called once per delivered message.
- `SendToGroupExceptAsync` exists because "send to group, but not back to the sender's own
  connections" is needed by Phase 05 and retrofitting it would be a breaking addition to an
  interface consumers may have implemented.
- `TouchAsync` updates `LastSeenAt`. Phase 02's reaper and Phase 06's presence both need it.
- `IOperationHandler.Operation` is a **`static abstract`** property, so the operation name is
  compile-time data available to the router without instantiating the handler and without an
  attribute + reflection scan.
- `RealtimeOperationContext.CancellationToken` mirrors the connection abort token. Handlers must
  pass it down; the architecture test's `[NoCancellation]` allowance covers `HandleAsync` because
  the token travels in the context rather than as a parameter — document that explicitly.
- `IServiceProvider Services` on the context is the request scope, so handlers can resolve scoped
  services without capturing the root provider. Document the lifetime clearly.

**Tests** — contract-shape tests only, since there are no implementations yet.

| Test | Asserts |
|---|---|
| `IRealtimeTransport_methods_all_accept_a_cancellation_token` | reflection |
| `IConnectionRegistry_methods_all_accept_a_cancellation_token` | reflection |
| `Operation_context_requires_operation_connection_and_correlation_id` | `required` members enforced |
| `Handler_operation_name_is_available_statically` | a test handler in the test assembly exposes `Operation` without instantiation |
| `Handler_interfaces_are_covariant_where_intended` | compile-time assertion via a test handler implementing both shapes |

---

### P01.T5 — Options, validation, and `AddLiveSharp`

**Deliverables** — `src/LiveSharp.Core/`

```
Configuration/LiveSharpOptions.cs
Configuration/ConnectionOptions.cs
Configuration/LimitsOptions.cs
Configuration/DiagnosticsOptions.cs
Configuration/LiveSharpOptionsValidator.cs
DependencyInjection/LiveSharpServiceCollectionExtensions.cs
DependencyInjection/ILiveSharpBuilder.cs
DependencyInjection/LiveSharpBuilder.cs
Diagnostics/LiveSharpLog.cs
```

**Public API (exact):**

```csharp
namespace LiveSharp;

public sealed class LiveSharpOptions
{
    public const string ConfigurationSectionName = "LiveSharp";

    public ConnectionOptions Connections { get; } = new();
    public LimitsOptions Limits { get; } = new();
    public DiagnosticsOptions Diagnostics { get; } = new();
}

public sealed class ConnectionOptions
{
    /// <summary>Maximum simultaneous connections per user. Default 5. Zero or negative is invalid.</summary>
    public int MaxConnectionsPerUser { get; set; } = 5;

    /// <summary>What to do when a user exceeds <see cref="MaxConnectionsPerUser"/>. Default RejectNewest.</summary>
    public ConnectionLimitBehavior LimitBehavior { get; set; } = ConnectionLimitBehavior.RejectNewest;

    /// <summary>A connection with no activity for this long is considered stale. Default 2 minutes.</summary>
    public TimeSpan IdleTimeout { get; set; } = TimeSpan.FromMinutes(2);

    /// <summary>How often the reaper scans for stale connections. Default 30 seconds.</summary>
    public TimeSpan ReaperInterval { get; set; } = TimeSpan.FromSeconds(30);
}

public enum ConnectionLimitBehavior { RejectNewest = 0, DisconnectOldest = 1 }

public sealed class LimitsOptions
{
    /// <summary>Maximum inbound operation payload size in bytes. Default 32 KiB.</summary>
    public int MaxPayloadBytes { get; set; } = 32 * 1024;

    /// <summary>Maximum length of an operation name. Default 128.</summary>
    public int MaxOperationNameLength { get; set; } = 128;
}

public sealed class DiagnosticsOptions
{
    /// <summary>When true, message payloads may be logged at Trace level. Never enable in production.</summary>
    public bool AllowSensitivePayloadLogging { get; set; }
}

public interface ILiveSharpBuilder
{
    IServiceCollection Services { get; }
}

public static class LiveSharpServiceCollectionExtensions
{
    public static ILiveSharpBuilder AddLiveSharp(this IServiceCollection services);
    public static ILiveSharpBuilder AddLiveSharp(this IServiceCollection services, Action<LiveSharpOptions> configure);
    public static ILiveSharpBuilder AddLiveSharp(this IServiceCollection services, IConfiguration configuration);
}
```

**Rules**

- Nested option objects are **get-only properties initialised inline**, not settable. This makes
  `options.Connections.IdleTimeout = x` the only way to configure them, which keeps
  `IOptionsMonitor` change notification coherent and prevents a consumer nulling a whole section.
- `ILiveSharpBuilder` is the extension point every feature package hangs off
  (`AddLiveSharp().AddChat()`). It exposes `IServiceCollection` and nothing else — resist adding
  convenience members; feature packages add extension methods on the interface.
- `AddLiveSharp` must be idempotent. Calling it twice must not double-register services or throw.
  Use `TryAdd*` throughout and add a test.
- Register `TimeProvider.System` with `TryAddSingleton` so tests and consumers can substitute it.
- Wire up `services.AddOptions<LiveSharpOptions>().Bind(section).ValidateOnStart()` and register
  `LiveSharpOptionsValidator` as `IValidateOptions<LiveSharpOptions>`.
- **Document every service lifetime** (Spec §9) in `docs/configuration/service-lifetimes.md` as a
  table: service, lifetime, thread-safety, why.

**Validation messages must be actionable.** Not `"Invalid value"` but:

```
LiveSharp: Connections.MaxConnectionsPerUser must be greater than zero (was -1).
Set services.AddLiveSharp(o => o.Connections.MaxConnectionsPerUser = 5) or remove the override.
```

Validator rules:

| Property | Rule |
|---|---|
| `Connections.MaxConnectionsPerUser` | `> 0` |
| `Connections.IdleTimeout` | `> TimeSpan.Zero` |
| `Connections.ReaperInterval` | `> TimeSpan.Zero` **and** `< IdleTimeout` (a reaper slower than the timeout cannot enforce it) |
| `Limits.MaxPayloadBytes` | `> 0` and `<= 1 MiB` (larger belongs in blob storage, not a hub message — say so in the message) |
| `Limits.MaxOperationNameLength` | `> 0` and `<= 512` |

Aggregate **all** failures into one `ValidateOptionsResult.Fail(IEnumerable<string>)`. A consumer
with three misconfigured values should learn all three on the first run, not one per restart.

**`LiveSharpLog.cs`** — `[LoggerMessage]` source-generated logging with a documented `EventId`
range. Reserve ranges now so later phases do not collide:

| Range | Area |
|---|---|
| `1000–1099` | Core / configuration / DI |
| `1100–1199` | Connections & lifecycle (Phase 02) |
| `1200–1299` | Transport & routing (Phase 03) |
| `1300–1399` | Chat (Phase 04) |
| `1400–1499` | Groups (Phase 05) |
| `1500–1599` | Presence (Phase 06) |
| `1600–1699` | Typing (Phase 07) |
| `1700–1799` | Delivery state (Phase 08) |
| `1800–1899` | Auth & authorization (Phase 09) |
| `1900–1999` | Signalling (Phase 10) |
| `2000–2099` | Calls (Phases 11–13) |
| `2100–2199` | Persistence (Phase 14) |
| `2200–2299` | Distributed (Phase 15) |

Record this table in `docs/troubleshooting/log-events.md` and keep it current — it is what a
developer greps when something breaks in production.

**Tests** — `LiveSharp.Core.Tests`

| Test | Asserts |
|---|---|
| `AddLiveSharp_registers_TimeProvider_as_singleton` | lifetime asserted via `ServiceDescriptor` |
| `AddLiveSharp_is_idempotent` | calling twice yields the same descriptor count |
| `AddLiveSharp_returns_builder_exposing_the_same_service_collection` | reference equality |
| `AddLiveSharp_with_configure_delegate_applies_values` | resolved `IOptions<LiveSharpOptions>` |
| `AddLiveSharp_with_IConfiguration_binds_the_LiveSharp_section` | nested keys bound |
| `Options_validation_runs_on_start` | `IHost.StartAsync` throws `OptionsValidationException` |
| `Validation_reports_all_failures_at_once` | three bad values → three messages |
| `Validation_message_names_the_property_and_the_offending_value` | string assertions per rule |
| `Reaper_interval_not_less_than_idle_timeout_is_rejected` | the non-obvious rule |
| `MaxPayloadBytes_above_one_mebibyte_is_rejected_with_guidance` | message mentions an alternative |
| `Default_options_are_valid` | defaults pass validation — a config system whose defaults fail is broken |
| `Nested_option_sections_are_not_settable` | reflection: no public setter on `Connections`/`Limits`/`Diagnostics` |
| `Every_logger_message_event_id_is_within_its_declared_range` | reflection over `LiveSharpLog` |
| `No_logger_message_template_contains_a_payload_or_token_placeholder` | template names checked against a deny-list (`payload`, `content`, `token`, `password`, `sdp`, `candidate`) |

---

### P01.T6 — Operation names, event names, and the operation router

**Deliverables**

```
src/LiveSharp.Abstractions/Protocol/RealtimeOperations.cs
src/LiveSharp.Abstractions/Protocol/RealtimeEvents.cs
src/LiveSharp.Abstractions/Protocol/LiveSharpProtocol.cs
src/LiveSharp.Core/Routing/IOperationRouter.cs
src/LiveSharp.Core/Routing/OperationRouter.cs
src/LiveSharp.Core/Routing/OperationRegistration.cs
src/LiveSharp.Core/DependencyInjection/LiveSharpBuilderExtensions.cs
```

**Public API (exact):**

```csharp
namespace LiveSharp.Protocol;

public static class LiveSharpProtocol
{
    /// <summary>Current wire protocol version. Incremented only by an accepted ADR.</summary>
    public const int Version = 1;

    /// <summary>Query string parameter clients use to declare their protocol version.</summary>
    public const string VersionQueryParameter = "lsv";

    /// <summary>Default endpoint path for the LiveSharp hub.</summary>
    public const string DefaultPath = "/livesharp";
}

/// <summary>Client-to-server operation names. Feature packages add nested classes here.</summary>
public static class RealtimeOperations
{
    public static class Connection
    {
        public const string Heartbeat = "connection.heartbeat";
    }
}

/// <summary>Server-to-client event names.</summary>
public static class RealtimeEvents
{
    public static class Connection
    {
        public const string Ready   = "connection.ready";
        public const string Closing = "connection.closing";
    }
}
```

Only the `connection.*` names exist in this phase. Each later phase adds its own nested class.

```csharp
namespace LiveSharp.Routing;

public interface IOperationRouter
{
    bool CanHandle(string operation);
    IReadOnlyCollection<string> RegisteredOperations { get; }

    ValueTask<OperationResult<JsonElement?>> RouteAsync(
        string operation,
        JsonElement? payload,
        RealtimeOperationContext context);
}
```

**`OperationRouter` requirements**

- Built from an immutable `FrozenDictionary<string, OperationRegistration>` created once at
  registration time. Lookup is on the hot path for every inbound message.
- `OperationRegistration` holds the operation name, the request/response `Type`s, and a
  pre-compiled delegate that deserializes the payload, resolves the handler from
  `context.Services`, invokes it, and serializes the response. No reflection per call.
- Deserialization uses the source-generated `JsonSerializerContext`. A payload that fails to
  deserialize returns `Failure(RealtimeErrorCode.InvalidPayload, …)` — it never throws out of
  `RouteAsync`.
- Unknown operation → `Failure(RealtimeErrorCode.UnknownOperation, …)`. Never throw; a malicious
  client must not be able to generate exceptions at will.
- A handler throwing → logged with the correlation ID at `Error`, and converted to
  `Failure(RealtimeErrorCode.InternalError, "An internal error occurred. Correlation ID: …")`. The
  exception detail **never** crosses the wire (Spec §24).
- `OperationCanceledException` when `context.CancellationToken` is cancelled propagates rather than
  becoming an `InternalError` — a disconnecting client is not a server error.
- Registering two handlers for the same operation throws `LiveSharpConfigurationException` at
  startup naming both handler types. Fail loudly at composition time, never silently last-wins.
- Operation names are validated at registration: non-empty, `<= MaxOperationNameLength`, matching
  `^[a-z][a-z0-9]*(\.[a-z][a-z0-9]*)+$`. A typo becomes a startup error, not a runtime mystery.

```csharp
namespace LiveSharp;

public static class LiveSharpBuilderExtensions
{
    public static ILiveSharpBuilder AddOperationHandler<THandler, TRequest, TResponse>(this ILiveSharpBuilder builder)
        where THandler : class, IOperationHandler<TRequest, TResponse>;

    public static ILiveSharpBuilder AddOperationHandler<THandler, TRequest>(this ILiveSharpBuilder builder)
        where THandler : class, IOperationHandler<TRequest>;
}
```

Handlers register as **scoped** — they may depend on scoped services and hold no state between
invocations. Document this and assert it in a test.

**Tests** — `LiveSharp.Core.Tests`

| Test | Asserts |
|---|---|
| `Router_dispatches_to_the_registered_handler` | handler receives the deserialized request |
| `Router_returns_unknown_operation_for_an_unregistered_name` | error code, no throw |
| `Router_returns_invalid_payload_for_malformed_json` | error code, no throw |
| `Router_returns_invalid_payload_when_a_required_member_is_missing` | |
| `Router_passes_the_operation_context_through_unchanged` | connection, correlation ID, user |
| `Router_converts_a_handler_exception_to_internal_error` | wire response carries no exception text |
| `Router_logs_a_handler_exception_with_the_correlation_id` | via `CapturingLoggerProvider` |
| `Router_does_not_leak_exception_details_to_the_client` | asserts the exception message string is absent from the response |
| `Router_propagates_cancellation_rather_than_reporting_internal_error` | pre-cancelled token |
| `Duplicate_operation_registration_throws_at_startup_naming_both_handlers` | both type names in the message |
| `Invalid_operation_name_is_rejected_at_registration` | table-driven: `""`, `"Chat.Send"`, `"chat"`, `"chat..send"`, `"chat.send!"`, 200-char name |
| `Handlers_are_registered_as_scoped` | `ServiceDescriptor.Lifetime` |
| `Handler_resolves_scoped_dependencies_from_the_context_scope` | two invocations get two instances |
| `RegisteredOperations_reflects_every_registration` | |
| `Router_lookup_is_case_sensitive` | `chat.message.send` ≠ `Chat.Message.Send` — pin the behaviour deliberately |
| `Concurrent_routing_of_1000_operations_produces_1000_correct_results` | `ConcurrencyHarness` |
| `All_operation_and_event_name_constants_match_the_naming_convention` | reflection over `RealtimeOperations`/`RealtimeEvents` |
| `Operation_and_event_names_are_globally_unique` | reflection; guards against a later phase reusing a name |

---

## Dependency justification

| Package | Why | BCL alternative? | Licence | Notes |
|---|---|---|---|---|
| `Microsoft.Extensions.DependencyInjection.Abstractions` | `IServiceCollection` for `ILiveSharpBuilder`. | No. | MIT | Abstractions-only, already in every ASP.NET Core app. |
| `Microsoft.Extensions.Logging.Abstractions` | `ILogger<T>` per Spec §23. | No. | MIT | Abstractions-only; no logging implementation is taken. |
| `Microsoft.Extensions.Options` | `IOptions<T>`/`IValidateOptions<T>` per Spec §10. | No. | MIT | |
| `Microsoft.Extensions.Options.ConfigurationExtensions` | `Bind` for the `IConfiguration` overload. | No. | MIT | `Core` only, not `Abstractions`. |
| `Microsoft.Extensions.Hosting.Abstractions` | `BackgroundService` base for the Phase 02 reaper. | No. | MIT | `Core` only. |
| `Microsoft.Extensions.Diagnostics.Abstractions` | Metrics seam for Phase 16 without a later dependency change. | Partly (`System.Diagnostics.Metrics`). | MIT | If it is not used by the end of this phase, **remove it** and add it in Phase 16 instead. |

No other packages. In particular: no MediatR (the router is 150 lines and MediatR would put a
third party on the hot path), no AutoMapper, no FluentValidation (options validation is a dozen
comparisons), no architecture-testing package.

---

## Documentation deltas

- `docs/architecture/overview.md` — fill in the real type names, the inbound/outbound flow, and the
  exceptions-vs-`OperationResult` decision table.
- `docs/configuration/README.md` — every option, its default, its valid range, and what breaks if
  it is wrong.
- `docs/configuration/service-lifetimes.md` — **new**; the service/lifetime/thread-safety table.
- `docs/troubleshooting/log-events.md` — **new**; the `EventId` range table.
- `docs/api/README.md` — note that generated API reference arrives in Phase 17.

## Example deltas

None. Examples begin in Phase 04.

## CHANGELOG entry

```markdown
### Added
- `LiveSharp.Abstractions`: connection, transport, registry, and operation-handler contracts.
- `LiveSharp.Core`: strongly typed options with start-up validation, `AddLiveSharp` DI entry point,
  and the operation router.
- `OperationResult` error model and the LiveSharp exception hierarchy.
- Wire protocol constants and version negotiation groundwork (`LiveSharpProtocol.Version = 1`).
- Architecture tests enforcing package dependency direction.
```

---

## Exit criteria

- [ ] `LiveSharp.Abstractions` references only the three allowed `Microsoft.Extensions.*`
      abstractions packages — proven by an architecture test, not by inspection.
- [ ] `LiveSharp.Core` has no `Microsoft.AspNetCore.*` reference — proven by an architecture test.
- [ ] `default(OperationResult)` is a failure, with a test proving it.
- [ ] Every public member has XML documentation that explains behaviour rather than restating the
      name.
- [ ] Every public async method takes a `CancellationToken`, proven by an architecture test.
- [ ] `BannedSymbols.txt` is active: adding `DateTime.UtcNow` to a `src/` file fails the build.
      Verify this by temporarily adding it.
- [ ] Options validation reports all failures at once with property names and remedies.
- [ ] `PublicAPI.Unshipped.txt` is populated for both projects and committed.
- [ ] `scripts/verify.ps1` green; `tests/LiveSharp.BuildSmokeTests` deleted.
- [ ] `docs/configuration/service-lifetimes.md` documents every registration.

## Verify

```powershell
pwsh scripts/verify.ps1 -SkipJs -SkipE2E
dotnet test tests/LiveSharp.ArchitectureTests -c Release
```

## Next

`docs/plan/phase-02-connections.md`, task `P02.T1`.
