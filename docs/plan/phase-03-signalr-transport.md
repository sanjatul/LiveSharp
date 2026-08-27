# Phase 03 — SignalR Transport & Clients

| | |
|---|---|
| **Version at exit** | `0.2.0` (tag `v0.2.0`) |
| **Depends on** | Phase 02 |
| **Branch** | `feat/phase-03-signalr-transport` |
| **Tasks** | `P03.T1` … `P03.T7` |
| **Read before starting** | `EXECUTION_PLAN.md` §6 (wire protocol — normative), `docs/architecture/connection-lifecycle.md`, `docs/architecture/decisions/ADR-004`, `ADR-005` |

> ## MANDATORY REVIEW GATE
> **This phase ends with a human review.** Do not begin Phase 04 until the Phase 03 PR is approved.
> The wire protocol and transport are the two decisions that are expensive to reverse. Everything
> from Phase 04 to Phase 18 assumes the shapes frozen here.

---

## Goal

Put a real network under the abstractions. Create `LiveSharp.AspNetCore` (the SignalR hub, the
transport implementation, and `MapLiveSharp`), the .NET client, and the TypeScript client. At the
end of this phase a browser and a .NET process can both connect, invoke an operation, receive an
event, survive a reconnect, and be rejected on a protocol mismatch.

No feature exists yet — the only operation is `connection.heartbeat`. That is deliberate: the
transport must be proven in isolation before chat semantics can hide its bugs.

---

## Non-goals — DO NOT implement in this phase

- No chat, messages, groups service, presence, or typing. The only operation is
  `connection.heartbeat` and the only events are `connection.ready` / `connection.closing`.
- No authentication or authorization enforcement. The hub reads `Context.User` if present and the
  resolver maps it, but **anonymous connections are allowed** in this phase. `[Authorize]`, policies,
  and rate limiting are Phase 09. Say so in the docs so nobody ships this.
- No `@livesharp/react`. That is Phase 17. `@livesharp/client` must be framework-agnostic.
- No MessagePack. JSON only; MessagePack is Phase 16.
- No Redis backplane, no `NodeId` population. Phase 15.
- No Blazor/React/Next.js example apps. Phase 04 onward. A minimal console/test harness is fine.

---

## Tasks

### P03.T1 — `LiveSharp.AspNetCore` project and protocol types

**Deliverables**

```
src/LiveSharp.AspNetCore/LiveSharp.AspNetCore.csproj
src/LiveSharp.AspNetCore/PublicAPI.Shipped.txt
src/LiveSharp.AspNetCore/PublicAPI.Unshipped.txt
src/LiveSharp.AspNetCore/AssemblyInfo.cs
src/LiveSharp.Abstractions/Protocol/OperationResponse.cs
src/LiveSharp.Abstractions/Protocol/HeartbeatRequest.cs
src/LiveSharp.Abstractions/Protocol/ConnectionReadyPayload.cs
src/LiveSharp.Abstractions/Protocol/LiveSharpJsonSerializerContext.cs
tests/LiveSharp.AspNetCore.Tests/…
tests/LiveSharp.IntegrationTests/…
```

**`.csproj`** — uses a framework reference, **not** package references:

```xml
<ItemGroup>
  <FrameworkReference Include="Microsoft.AspNetCore.App" />
  <ProjectReference Include="../LiveSharp.Core/LiveSharp.Core.csproj" />
</ItemGroup>
```

This is the mechanical justification for deviation **D1**: SignalR server-side types come from the
shared framework at zero NuGet cost, so a separate `LiveSharp.SignalR` package would add a package
without removing a dependency. Put that sentence in a comment in the `.csproj`.

**Public API (exact):**

```csharp
namespace LiveSharp.Protocol;

/// <summary>Result of a client-to-server operation. Serialized to the caller.</summary>
public sealed record OperationResponse(
    bool Success,
    string CorrelationId,
    JsonElement? Data = null,
    string? Code = null,
    string? Message = null)
{
    public static OperationResponse FromResult(OperationResult<JsonElement?> result, string correlationId);
}

public sealed record HeartbeatRequest;

/// <summary>Sent to a client immediately after its connection is accepted.</summary>
public sealed record ConnectionReadyPayload(
    string ConnectionId,
    string UserId,
    int ProtocolVersion,
    DateTimeOffset ServerTime);
```

**`LiveSharpJsonSerializerContext`** — a `[JsonSerializable]`-annotated partial context. Every
protocol type is registered here. Options: `PropertyNamingPolicy = JsonNamingPolicy.CamelCase`,
`DefaultIgnoreCondition = WhenWritingNull`, `NumberHandling = Strict`.

> **Every later phase must add its request/response types to this context.** A type that is missing
> will fail at runtime with a reflection-disabled serializer, not at compile time. Add a test that
> reflects over all types implementing the protocol marker and asserts each resolves in the context —
> that turns a runtime surprise into a build failure. Do this now; it pays for itself six times.

**Tests**

| Test | Asserts |
|---|---|
| `OperationResponse_from_a_successful_result_carries_data_and_no_code` | |
| `OperationResponse_from_a_failed_result_carries_code_and_message_and_no_data` | |
| `Protocol_types_serialize_with_camel_case_property_names` | golden string per type |
| `Protocol_types_round_trip_through_the_source_generated_context` | |
| `Every_protocol_type_is_registered_in_the_serializer_context` | reflection; the guard test above |
| `Serializer_context_does_not_fall_back_to_reflection` | construct with `JsonSerializerOptions { TypeInfoResolver = LiveSharpJsonSerializerContext.Default }` and assert an unregistered type throws |

---

### P03.T2 — `LiveSharpHub`

**Deliverables**

```
src/LiveSharp.AspNetCore/Hubs/LiveSharpHub.cs
src/LiveSharp.AspNetCore/Hubs/HubConnectionContextAccessor.cs
src/LiveSharp.AspNetCore/Handlers/HeartbeatHandler.cs
tests/LiveSharp.AspNetCore.Tests/Hubs/LiveSharpHubTests.cs
```

**Public API (exact):**

```csharp
namespace LiveSharp.Hubs;

/// <summary>The single LiveSharp hub. All client-to-server traffic flows through two methods.</summary>
public sealed class LiveSharpHub : Hub
{
    public Task<OperationResponse> InvokeAsync(string operation, JsonElement? payload);
    public Task NotifyAsync(string operation, JsonElement? payload);
    public override Task OnConnectedAsync();
    public override Task OnDisconnectedAsync(Exception? exception);
}
```

**The hub must be thin.** Every branch below is a delegation, not logic. If a code path in the hub
exceeds a few lines, it belongs in `LiveSharp.Core`.

**`OnConnectedAsync`:**

1. Parse the protocol version from `Context.GetHttpContext()?.Request.Query[LiveSharpProtocol.VersionQueryParameter]`.
   Missing → assume `1`. Unparseable or unsupported → `Context.Abort()` after attempting to send a
   `connection.closing` envelope with `RealtimeErrorCode.ProtocolMismatch`, then return **without**
   registering. Log at `Warning`.
2. Resolve the user ID via `IUserResolver` from `Context.User`. When `null` (anonymous, allowed in
   this phase), synthesise `$"anon:{Context.ConnectionId}"` and record
   `Metadata["livesharp.anonymous"] = "true"`. Document this loudly as a Phase 03-only behaviour that
   Phase 09 removes.
3. Build a `ConnectionRequest` with `ConnectionId = Context.ConnectionId`, the resolved user ID, and
   metadata (`userAgent` truncated to 256 chars, `transport`, `protocolVersion`). **Never** put
   headers, cookies, tokens, or query strings other than the version into metadata.
4. `IConnectionLifecycleService.ConnectAsync`. On failure → send `connection.closing` with the error
   code, then `Context.Abort()`.
5. On success → send `connection.ready` with `ConnectionReadyPayload`.

**`OnDisconnectedAsync`:**

1. Map to a `DisconnectReason`: `exception is null` → `ClientClosed`; otherwise `TransportError`.
   `IHostApplicationLifetime.ApplicationStopping` already fired → `Shutdown`.
2. `IConnectionLifecycleService.DisconnectAsync(Context.ConnectionId, reason, CancellationToken.None)`.
   **`CancellationToken.None` is mandatory** — `Context.ConnectionAborted` is already cancelled here,
   and passing it means cleanup never runs. This is the single most common SignalR cleanup bug.

**`InvokeAsync`:**

1. Generate `correlationId = Guid.CreateVersion7().ToString("n")`. Sortable, no dependency,
   time-ordered, which makes log correlation trivial.
2. Validate `operation`: non-empty, `<= Limits.MaxOperationNameLength` → respond
   `Failure(InvalidPayload)`. Do not throw; a client must not be able to generate hub exceptions.
3. Validate payload size against `Limits.MaxPayloadBytes` → `Failure(PayloadTooLarge)`. Measure the
   raw UTF-8 byte length of the `JsonElement` (`GetRawText()` length is a close enough upper bound;
   prefer writing to a pooled `ArrayBufferWriter` and reading `WrittenCount`).
4. `IConnectionRegistry.FindAsync(Context.ConnectionId)`. Absent → `Failure(NotFound, "Connection is
   not registered.")`. This happens legitimately when a message races a disconnect.
5. `IConnectionRegistry.TouchAsync`.
6. Build `RealtimeOperationContext` with `Services = Context.GetHttpContext()!.RequestServices` —
   **no**, use the hub's own DI scope: inject `IServiceProvider` into the hub constructor, which in
   SignalR is the hub-invocation scope. Document which scope handlers get and add a test proving two
   invocations on the same connection get different scoped instances.
7. `IOperationRouter.RouteAsync` → `OperationResponse.FromResult(result, correlationId)`.
8. Wrap everything in a try/catch that logs at `Error` with the correlation ID and returns
   `Failure(InternalError, "An internal error occurred. Correlation ID: {id}")`. **The exception
   text never crosses the wire.**

**`NotifyAsync`:** identical to `InvokeAsync` but returns `Task`, discards the result, and logs a
failed result at `Debug`. Used by high-frequency operations (typing, heartbeat) where the round trip
is waste.

**Hub configuration** — set in `MapLiveSharp` (`P03.T4`), not here: `MaximumReceiveMessageSize`
aligned with `Limits.MaxPayloadBytes`, `ClientTimeoutInterval`, `KeepAliveInterval`,
`HandshakeTimeout`, `EnableDetailedErrors = false` by default (and a warning logged if a consumer
turns it on outside Development).

**Tests — `LiveSharpHubTests`** (unit tests with a substituted `HubCallerContext` and `IHubCallerClients`)

| Test | Asserts |
|---|---|
| `OnConnected_registers_the_connection_via_the_lifecycle_service` | |
| `OnConnected_sends_connection_ready_with_the_protocol_version` | |
| `OnConnected_with_an_unsupported_protocol_version_aborts_without_registering` | `Abort` called, registry empty |
| `OnConnected_with_a_missing_protocol_version_assumes_version_one` | |
| `OnConnected_with_an_unparseable_protocol_version_aborts` | `"abc"`, `""`, `"1.0"`, `"-1"` |
| `OnConnected_with_an_anonymous_principal_synthesises_an_anon_user_id` | prefix + metadata flag |
| `OnConnected_when_the_lifecycle_service_rejects_aborts_and_sends_the_error_code` | connection-limit rejection |
| `OnConnected_metadata_contains_no_headers_cookies_or_tokens` | **security test**; assert the metadata key set exactly |
| `OnConnected_truncates_a_long_user_agent` | 4 KiB user agent → 256 chars |
| `OnDisconnected_maps_a_null_exception_to_ClientClosed` | |
| `OnDisconnected_maps_an_exception_to_TransportError` | |
| `OnDisconnected_passes_CancellationToken_None` | **critical**; substituted lifecycle service asserts `!ct.CanBeCanceled` |
| `Invoke_routes_to_the_router_and_returns_the_response` | |
| `Invoke_generates_a_unique_sortable_correlation_id` | two calls → different, ordered ids |
| `Invoke_returns_invalid_payload_for_an_empty_operation_name` | |
| `Invoke_returns_invalid_payload_for_an_over_long_operation_name` | |
| `Invoke_returns_payload_too_large_above_the_configured_limit` | limit + 1 byte |
| `Invoke_accepts_a_payload_exactly_at_the_limit` | boundary |
| `Invoke_returns_not_found_when_the_connection_is_not_registered` | |
| `Invoke_touches_the_connection` | `LastSeenAt` advanced |
| `Invoke_converts_a_router_exception_to_internal_error_without_leaking_detail` | exception message absent from the response |
| `Invoke_includes_the_correlation_id_in_the_internal_error_message` | operability |
| `Invoke_logs_a_router_exception_at_error_with_the_correlation_id` | |
| `Notify_returns_without_a_response_payload` | |
| `Notify_logs_a_failed_result_at_debug_and_does_not_throw` | |
| `Hub_is_sealed` | inheritance is not a supported extension point — the router is |

---

### P03.T3 — `SignalRRealtimeTransport` and the user-id provider

**Deliverables**

```
src/LiveSharp.AspNetCore/Transport/SignalRRealtimeTransport.cs
src/LiveSharp.AspNetCore/Transport/LiveSharpUserIdProvider.cs
tests/LiveSharp.AspNetCore.Tests/Transport/SignalRRealtimeTransportTests.cs
```

**Implementation requirements**

`SignalRRealtimeTransport(IHubContext<LiveSharpHub> hubContext, IConnectionRegistry connections, IGroupRegistry groups, ILogger<SignalRRealtimeTransport> logger)`.

Every method maps to exactly one `IHubContext` call:

| `IRealtimeTransport` | SignalR |
|---|---|
| `SendToConnectionAsync` | `Clients.Client(id).SendAsync("Receive", envelope, ct)` |
| `SendToUserAsync` | `Clients.User(userId).SendAsync("Receive", envelope, ct)` |
| `SendToUsersAsync` | `Clients.Users(userIds).SendAsync(...)` after de-duplicating |
| `SendToGroupAsync` | `Clients.Group(name).SendAsync(...)` |
| `SendToGroupExceptAsync` | `Clients.GroupExcept(name, excluded).SendAsync(...)` |
| `AddToGroupAsync` | `Groups.AddToGroupAsync(connectionId, name, ct)` **and** `IGroupRegistry.AddAsync` |
| `RemoveFromGroupAsync` | `Groups.RemoveFromGroupAsync(...)` **and** `IGroupRegistry.RemoveAsync` |
| `DisconnectAsync` | send `connection.closing`, then there is no `IHubContext` abort primitive — see below |

**Three sharp edges to handle explicitly:**

1. **Group membership is dual-written.** SignalR owns its own group map (needed for
   `Clients.Group`), and `IGroupRegistry` owns ours (needed for reconnect rehydration in Phase 05 and
   for cross-node routing in Phase 15). Write SignalR **first**, then the registry; if the registry
   write throws, the SignalR membership is harmlessly extra. Document the ordering and the
   reconciliation strategy. A test must assert both are written.

2. **`DisconnectAsync` has no `IHubContext` primitive.** `IHubContext` cannot abort a connection.
   Implement it by maintaining an internal `ConcurrentDictionary<string, HubCallerContext>` populated
   in `LiveSharpHub.OnConnectedAsync` and cleared in `OnDisconnectedAsync`, exposed through an
   internal `IHubConnectionAborter`. Send `connection.closing` with the reason **first**, then call
   `context.Abort()`. If the connection is unknown (e.g. it lives on another node — Phase 15),
   log at `Debug` and return successfully. Do not throw: callers treat disconnect as best-effort.
   Note in the code comment that Phase 15 replaces this with a backplane message.

3. **`SendToUserAsync` depends on `IUserIdProvider`.** SignalR's `Clients.User(x)` resolves through
   `IUserIdProvider`, not through `IConnectionRegistry`. If the two disagree, messages vanish
   silently. `LiveSharpUserIdProvider : IUserIdProvider` must delegate to the **same**
   `IUserResolver`, and must apply the same anonymous fallback as the hub. Add an integration test
   that sends to a user by ID and asserts arrival — a unit test cannot catch this class of mismatch.

**Also required**

- `"Receive"` is the single client method name. Declare it as
  `internal const string ClientReceiveMethod = "Receive"` in one place; a string literal duplicated
  across the transport and both clients is a guaranteed future bug.
- Wrap SignalR exceptions in `LiveSharpTransportException` with the operation and target in the
  message, preserving the inner exception.
- Log at `Trace` only, and never the payload unless `Diagnostics.AllowSensitivePayloadLogging`.

**Tests**

| Test | Asserts |
|---|---|
| `Send_to_connection_calls_the_client_proxy_with_the_Receive_method` | substituted `IHubContext` |
| `Send_to_user_calls_the_user_proxy` | |
| `Send_to_users_deduplicates_the_user_list` | |
| `Send_to_group_calls_the_group_proxy` | |
| `Send_to_group_except_passes_the_exclusion_list_through` | |
| `Add_to_group_writes_signalr_first_then_the_registry` | call order asserted |
| `Add_to_group_still_leaves_signalr_membership_when_the_registry_throws` | |
| `Remove_from_group_updates_both_signalr_and_the_registry` | |
| `Disconnect_sends_connection_closing_before_aborting` | order asserted |
| `Disconnect_of_an_unknown_connection_succeeds_silently` | |
| `A_signalr_exception_is_wrapped_in_LiveSharpTransportException` | inner exception preserved |
| `Payload_is_not_logged_when_sensitive_logging_is_disabled` | **security test** |
| `Payload_is_logged_at_trace_when_sensitive_logging_is_enabled` | |
| `User_id_provider_delegates_to_the_user_resolver` | |
| `User_id_provider_applies_the_same_anonymous_fallback_as_the_hub` | identical string for the same connection |
| `Cancellation_is_forwarded_to_every_SendAsync_call` | |

---

### P03.T4 — `MapLiveSharp`, options, and SignalR wiring

**Deliverables**

```
src/LiveSharp.AspNetCore/DependencyInjection/LiveSharpAspNetCoreBuilderExtensions.cs
src/LiveSharp.AspNetCore/DependencyInjection/LiveSharpEndpointRouteBuilderExtensions.cs
src/LiveSharp.AspNetCore/Configuration/LiveSharpHubOptions.cs
tests/LiveSharp.AspNetCore.Tests/DependencyInjection/RegistrationTests.cs
```

**Public API (exact):**

```csharp
namespace LiveSharp;

public static class LiveSharpAspNetCoreBuilderExtensions
{
    /// <summary>Registers the SignalR-based transport and the LiveSharp hub.</summary>
    public static ILiveSharpBuilder AddSignalR(this ILiveSharpBuilder builder);
    public static ILiveSharpBuilder AddSignalR(this ILiveSharpBuilder builder, Action<LiveSharpHubOptions> configure);
}

public static class LiveSharpEndpointRouteBuilderExtensions
{
    /// <summary>Maps the LiveSharp hub at the configured path (default <c>/livesharp</c>).</summary>
    public static HubEndpointConventionBuilder MapLiveSharp(this IEndpointRouteBuilder endpoints);
    public static HubEndpointConventionBuilder MapLiveSharp(this IEndpointRouteBuilder endpoints, string path);
}

public sealed class LiveSharpHubOptions
{
    public string Path { get; set; } = LiveSharpProtocol.DefaultPath;
    public TimeSpan ClientTimeoutInterval { get; set; } = TimeSpan.FromSeconds(30);
    public TimeSpan KeepAliveInterval { get; set; } = TimeSpan.FromSeconds(15);
    public TimeSpan HandshakeTimeout { get; set; } = TimeSpan.FromSeconds(15);
    public bool EnableDetailedErrors { get; set; }
}
```

**Target developer experience** — the whole point of the project (Spec §47). This must work:

```csharp
builder.Services.AddLiveSharp().AddSignalR();
// ...
app.MapLiveSharp();
```

**Requirements**

- `AddSignalR()` calls ASP.NET Core's `services.AddSignalR()` internally with `TryAdd`-style
  idempotency so a consumer who already called `AddSignalR()` for their own hubs is not broken. Test
  both orders: LiveSharp first, then theirs; and theirs first, then LiveSharp.
- Register `SignalRRealtimeTransport` as the `IRealtimeTransport` singleton, replacing any existing
  registration (`Replace`, not `TryAdd` — if a consumer explicitly registered `InMemoryTransport`
  they get what they asked for; document precedence).
- Register `LiveSharpUserIdProvider` as `IUserIdProvider` with `TryAdd`, and **log a warning at
  startup if a different `IUserIdProvider` is already registered**, because that silently breaks
  `SendToUserAsync`. A warning here saves someone a week.
- Validate `LiveSharpHubOptions`: `Path` must start with `/`; `KeepAliveInterval` must be less than
  `ClientTimeoutInterval` (SignalR's own recommendation is a factor of two — enforce
  `ClientTimeoutInterval >= 2 * KeepAliveInterval` with an actionable message).
- Propagate `Limits.MaxPayloadBytes` into `HubOptions.MaximumReceiveMessageSize`. Two independent
  size limits that can disagree is a support nightmare.
- Log a `Warning` at startup when `EnableDetailedErrors` is true and the environment is not
  Development.
- `MapLiveSharp` throws `LiveSharpConfigurationException` with a fix-it message if `AddSignalR()`
  was never called. The failure mode for a missing registration must be a clear startup exception,
  not a 404 the developer debugs for an hour.

**Tests**

| Test | Asserts |
|---|---|
| `AddSignalR_registers_the_signalr_transport_as_the_realtime_transport` | |
| `AddSignalR_is_idempotent` | descriptor count stable |
| `AddSignalR_after_a_consumer_called_AddSignalR_does_not_duplicate_signalr_services` | |
| `AddSignalR_before_a_consumer_calls_AddSignalR_still_works` | both orders |
| `AddSignalR_registers_the_user_id_provider` | |
| `AddSignalR_warns_when_another_user_id_provider_is_already_registered` | log captured |
| `Max_payload_bytes_propagates_to_the_hub_maximum_receive_message_size` | |
| `Path_not_starting_with_a_slash_is_rejected_with_guidance` | |
| `Client_timeout_less_than_twice_the_keep_alive_is_rejected_with_guidance` | |
| `Detailed_errors_outside_development_logs_a_warning` | |
| `MapLiveSharp_without_AddSignalR_throws_a_configuration_exception_naming_the_fix` | message contains `AddSignalR` |
| `MapLiveSharp_maps_the_default_path` | endpoint data source inspected |
| `MapLiveSharp_with_an_explicit_path_overrides_the_option` | |
| `MapLiveSharp_returns_a_convention_builder_usable_for_RequireAuthorization` | forward-compatibility with Phase 09 |

---

### P03.T5 — Integration tests over a real SignalR connection

**Deliverables**

```
tests/LiveSharp.IntegrationTests/LiveSharp.IntegrationTests.csproj
tests/LiveSharp.IntegrationTests/Infrastructure/LiveSharpTestApplication.cs
tests/LiveSharp.IntegrationTests/Infrastructure/TestHubConnectionFactory.cs
tests/LiveSharp.IntegrationTests/ConnectionLifecycleIntegrationTests.cs
tests/LiveSharp.IntegrationTests/OperationRoutingIntegrationTests.cs
tests/LiveSharp.IntegrationTests/ReconnectIntegrationTests.cs
```

This task is the reason Phase 03 is de-risked. Unit tests with a substituted `IHubContext` cannot
catch the `IUserIdProvider` mismatch, the serializer-context gap, the group dual-write, or the
`CancellationToken.None` disconnect bug. Only a real connection can.

**The `TestServer` recipe — encode this exactly; it is the documented risk from `EXECUTION_PLAN.md` §12:**

```csharp
// WebApplicationFactory<TEntryPoint> or a minimal WebApplicationFactory over a test Program.
var server = factory.Server;

var connection = new HubConnectionBuilder()
    .WithUrl(new Uri(server.BaseAddress, "livesharp?lsv=1"), options =>
    {
        // Required: route HTTP negotiation through the in-memory TestServer.
        options.HttpMessageHandlerFactory = _ => server.CreateHandler();

        // Required for the WebSocket transport: TestServer needs its own WebSocket client.
        options.WebSocketFactory = async (context, cancellationToken) =>
        {
            var wsClient = server.CreateWebSocketClient();
            var uri = new UriBuilder(context.Uri) { Scheme = "ws" }.Uri;
            return await wsClient.ConnectAsync(uri, cancellationToken);
        };

        options.Transports = transportType; // parameterised per test
    })
    .WithAutomaticReconnect()
    .Build();
```

`TestHubConnectionFactory` wraps this and is parameterised over
`HttpTransportType.WebSockets` and `HttpTransportType.LongPolling`. **Every routing test runs
against both** via `[Theory]` + `MemberData`. Server-Sent Events may be added but is not required.

`LiveSharpTestApplication` is a `WebApplicationFactory` that:
- overrides `TimeProvider` with `FakeTimeProvider` where a test needs it (but **not** for SignalR's
  own keep-alive timers — SignalR uses its own internals and will not respect it; do not fight this,
  use real short intervals for timeout tests instead and document why),
- installs `CapturingLoggerProvider`,
- allows a test to register extra operation handlers,
- exposes the `IConnectionRegistry` and the `IRealtimeTransport` for assertions,
- is `IAsyncDisposable` and asserts no leaked connections at teardown.

**Tests**

| Test | Asserts |
|---|---|
| `Client_connects_and_receives_connection_ready` | both transports; payload matches the server's connection ID |
| `Server_registry_contains_the_connection_after_connect` | |
| `Server_registry_is_empty_after_the_client_disconnects_cleanly` | **the `CancellationToken.None` regression test** |
| `Server_registry_is_empty_after_the_client_is_killed_abruptly` | dispose the handler mid-flight |
| `Client_with_an_unsupported_protocol_version_is_rejected` | `lsv=999`; connect throws or closes |
| `Client_without_a_protocol_version_connects_successfully` | backward-compatibility default |
| `Invoke_of_the_heartbeat_operation_returns_success` | both transports |
| `Invoke_of_an_unknown_operation_returns_unknown_operation` | no exception thrown to the client |
| `Invoke_with_a_malformed_payload_returns_invalid_payload` | |
| `Invoke_with_an_oversized_payload_returns_payload_too_large_or_closes_the_connection` | assert whichever SignalR does and pin it |
| `Invoke_response_never_contains_a_stack_trace` | register a throwing handler; assert the response |
| `Notify_of_the_heartbeat_operation_does_not_return_a_value` | |
| `Send_to_user_reaches_every_connection_of_that_user` | **the `IUserIdProvider` regression test**; 2 connections, same user |
| `Send_to_user_does_not_reach_a_different_user` | |
| `Send_to_group_reaches_only_group_members` | after `AddToGroupAsync` |
| `Add_to_group_writes_both_the_signalr_group_and_the_registry` | |
| `Two_invocations_receive_different_scoped_handler_dependencies` | DI scope proof |
| `Automatic_reconnect_produces_a_new_connection_id_registered_on_the_server` | stop the server connection, let the client reconnect |
| `Stale_state_from_the_previous_connection_id_is_removed_after_reconnect` | old ID absent from the registry |
| `Connection_limit_rejection_closes_the_client_connection_with_the_reason` | limit 1, connect twice |
| `Concurrent_invocations_from_one_connection_all_receive_correct_correlated_responses` | 100 parallel invokes, correlation IDs matched |
| `Fifty_concurrent_clients_all_connect_and_are_registered` | smoke-scale sanity |

---

### P03.T6 — `LiveSharp.Client` (.NET)

**Deliverables**

```
clients/dotnet/LiveSharp.Client/LiveSharp.Client.csproj
clients/dotnet/LiveSharp.Client/LiveSharpClient.cs
clients/dotnet/LiveSharp.Client/ILiveSharpClient.cs
clients/dotnet/LiveSharp.Client/LiveSharpClientOptions.cs
clients/dotnet/LiveSharp.Client/LiveSharpClientException.cs
clients/dotnet/LiveSharp.Client/PublicAPI.{Shipped,Unshipped}.txt
tests/LiveSharp.Client.Tests/…
```

Add this project to `LiveSharp.slnx` (it is a shipping package, unlike `examples/**`).

**Public API (exact):**

```csharp
namespace LiveSharp.Client;

public interface ILiveSharpClient : IAsyncDisposable
{
    ConnectionState State { get; }
    string? ConnectionId { get; }

    event EventHandler<ConnectionState>? StateChanged;

    Task ConnectAsync(CancellationToken cancellationToken = default);
    Task DisconnectAsync(CancellationToken cancellationToken = default);

    /// <summary>Invokes a server operation and returns the typed response.</summary>
    Task<TResponse> InvokeAsync<TRequest, TResponse>(string operation, TRequest request, CancellationToken cancellationToken = default);

    /// <summary>Invokes a server operation that returns no payload.</summary>
    Task InvokeAsync<TRequest>(string operation, TRequest request, CancellationToken cancellationToken = default);

    /// <summary>Sends an operation without awaiting a result.</summary>
    Task NotifyAsync<TRequest>(string operation, TRequest request, CancellationToken cancellationToken = default);

    /// <summary>Subscribes to a server event. Dispose the returned handle to unsubscribe.</summary>
    IDisposable On<TPayload>(string eventName, Func<TPayload, Task> handler);
}

public sealed class LiveSharpClient : ILiveSharpClient
{
    public LiveSharpClient(LiveSharpClientOptions options, ILogger<LiveSharpClient>? logger = null);
}

public sealed class LiveSharpClientOptions
{
    public required Uri Url { get; set; }
    public Func<Task<string?>>? AccessTokenProvider { get; set; }
    public bool AutomaticReconnect { get; set; } = true;
    public IReadOnlyList<TimeSpan>? ReconnectDelays { get; set; }
    public JsonSerializerOptions? JsonSerializerOptions { get; set; }
    public Action<IHttpConnectionOptions>? ConfigureHttpConnection { get; set; }
}
```

**Requirements**

- Appends `?lsv={LiveSharpProtocol.Version}` to the URL automatically, preserving any existing query
  string. A developer must never have to know the parameter exists.
- One `HubConnection.On<RealtimeEnvelope>("Receive", …)` internally, fanning out to subscriptions
  registered via `On<TPayload>(eventName, handler)`. A `ConcurrentDictionary<string,
  ImmutableList<Subscription>>` keyed by event name.
- `On` returns an `IDisposable` that unsubscribes. Subscriptions leaking is the classic real-time
  memory leak (Spec §51); the disposal path needs a test.
- A subscriber that throws is logged and **does not** prevent other subscribers from running or tear
  down the connection.
- An event with no subscriber is logged at `Debug`, not `Warning` — forward compatibility means a
  newer server sends events an older client does not know.
- `InvokeAsync` translates a failed `OperationResponse` into `LiveSharpClientException` carrying
  `ErrorCode`, `Message`, and `CorrelationId`. Developers should be able to `catch (LiveSharpClientException e) when (e.ErrorCode == RealtimeErrorCode.RateLimited)`.
- `StateChanged` fires for `Connecting`/`Connected`/`Reconnecting`/`Disconnected`, mapped from
  `HubConnection`'s `Reconnecting`/`Reconnected`/`Closed` events.
- `ConfigureHttpConnection` is the deliberate escape hatch for advanced users (Spec §70: simple API
  **and** advanced API). Document it as such.
- `DisposeAsync` is idempotent, unsubscribes everything, and disposes the `HubConnection`.
- **Do not** re-expose `HubConnection` as a public property. Once it is public, LiveSharp can never
  change transports. Provide `ConfigureHttpConnection` instead and say why in the XML docs.

**Tests** — unit tests against a substituted invoker plus integration tests reusing
`LiveSharpTestApplication` from `P03.T5`.

| Test | Asserts |
|---|---|
| `Connect_appends_the_protocol_version_query_parameter` | |
| `Connect_preserves_an_existing_query_string` | `?tenant=a` → `?tenant=a&lsv=1` |
| `State_transitions_are_reported_in_order` | Connecting → Connected → Disconnected |
| `On_receives_a_matching_event` | |
| `On_ignores_a_non_matching_event` | |
| `Multiple_subscribers_to_one_event_all_run` | |
| `A_throwing_subscriber_does_not_prevent_other_subscribers` | |
| `A_throwing_subscriber_does_not_close_the_connection` | |
| `Disposing_the_subscription_handle_stops_delivery` | leak test |
| `Disposing_the_subscription_handle_twice_is_safe` | |
| `An_event_with_no_subscriber_is_logged_at_debug_not_warning` | |
| `Invoke_returns_the_deserialized_response` | |
| `Invoke_throws_LiveSharpClientException_carrying_the_error_code_and_correlation_id` | |
| `Notify_does_not_await_a_response` | |
| `Dispose_async_is_idempotent` | |
| `Dispose_async_unsubscribes_all_handlers` | |
| `Access_token_provider_is_invoked_on_connect` | |
| `Client_reconnects_and_resubscribes_after_a_transport_drop` | integration |
| `Events_received_after_a_reconnect_still_reach_subscribers` | **the resubscription regression test** |
| `Public_surface_does_not_expose_HubConnection` | reflection — the architectural guarantee |

---

### P03.T7 — `@livesharp/client` (TypeScript)

**Deliverables**

```
clients/js/package.json                       (pnpm workspace root, private)
clients/js/pnpm-workspace.yaml
clients/js/tsconfig.base.json
clients/js/.npmrc
clients/js/eslint.config.js
clients/js/packages/client/package.json       (name: @livesharp/client)
clients/js/packages/client/tsconfig.json
clients/js/packages/client/tsup.config.ts
clients/js/packages/client/src/index.ts
clients/js/packages/client/src/client.ts
clients/js/packages/client/src/protocol.ts
clients/js/packages/client/src/errors.ts
clients/js/packages/client/src/types.ts
clients/js/packages/client/src/__tests__/*.test.ts
clients/js/packages/client/README.md
```

**Package configuration**

- ESM-first with a CJS fallback, `"type": "module"`, `exports` map with `types`/`import`/`require`,
  `sideEffects: false`, `files: ["dist"]`.
- `@microsoft/signalr` as a **dependency** (not a peer). A peer dependency here just makes every
  consumer type the same install command; there is one supported version and LiveSharp owns it.
  Record this reasoning in the ADR-004 write-up.
- Build with `tsup` (esbuild) producing `dist/index.js`, `dist/index.cjs`, `dist/index.d.ts`.
- Test with `vitest`. Lint with `eslint` + `typescript-eslint`.
- `"version"` is kept **in lockstep with the NuGet packages** — a release script reads the MinVer
  version and writes it into `package.json`. No compatibility matrix to maintain (ADR-010).

**Public API (exact):**

```ts
export const PROTOCOL_VERSION = 1;

export type ConnectionState = 'disconnected' | 'connecting' | 'connected' | 'reconnecting';

export interface LiveSharpClientOptions {
  url: string;
  accessTokenFactory?: () => string | Promise<string>;
  automaticReconnect?: boolean | number[];
  logLevel?: 'none' | 'error' | 'warning' | 'information' | 'debug' | 'trace';
  transport?: 'webSockets' | 'serverSentEvents' | 'longPolling';
}

export interface OperationResponse<TData = unknown> {
  success: boolean;
  correlationId: string;
  data?: TData;
  code?: string;
  message?: string;
}

export class LiveSharpError extends Error {
  readonly code: string;
  readonly correlationId?: string;
}

export type Unsubscribe = () => void;

export class LiveSharpClient {
  constructor(options: LiveSharpClientOptions);

  readonly state: ConnectionState;
  readonly connectionId: string | undefined;

  connect(): Promise<void>;
  disconnect(): Promise<void>;

  invoke<TResponse = void, TRequest = unknown>(operation: string, payload?: TRequest): Promise<TResponse>;
  notify<TRequest = unknown>(operation: string, payload?: TRequest): Promise<void>;

  on<TPayload = unknown>(event: string, handler: (payload: TPayload) => void | Promise<void>): Unsubscribe;
  onStateChange(handler: (state: ConnectionState) => void): Unsubscribe;
}

/** Operation and event name constants, mirroring the .NET RealtimeOperations/RealtimeEvents. */
export const Operations: { connection: { heartbeat: string } };
export const Events: { connection: { ready: string; closing: string } };
```

**Requirements**

- `invoke` rejects with `LiveSharpError` when `success === false`. `code` and `correlationId` must be
  preserved so a consumer can branch on the error, exactly as in the .NET client.
- `on` returns an unsubscribe function (not an object) — idiomatic in JS and impossible to forget to
  call in a React `useEffect` cleanup.
- Handler errors are caught, logged, and isolated. One bad handler must not break the connection.
- Appends `lsv` to the URL automatically, preserving existing query parameters.
- The default `automaticReconnect` uses SignalR's default backoff; `number[]` passes custom delays
  through.
- **No DOM APIs at module scope.** No `window`, `document`, or `navigator` touched on import — this
  package must be importable in a Next.js server component and in Node without crashing. Add a test
  that imports the module in the `node` vitest environment with no DOM globals. This is the single
  most common failure mode for a browser SDK used with SSR.
- `protocol.ts` holds the constants and is the file a code generator would replace in Phase 17.
  Until then, a **contract test** guards drift: the .NET test suite writes
  `artifacts/protocol-names.json` from reflection over `RealtimeOperations`/`RealtimeEvents`, and a
  vitest test asserts the TS constants match it exactly. Without this, the two sides silently
  diverge within two phases.

**Tests** (vitest)

| Test | Asserts |
|---|---|
| `appends the protocol version to the url` | |
| `preserves an existing query string` | |
| `reports state transitions in order` | |
| `invoke resolves with the response data on success` | |
| `invoke rejects with LiveSharpError carrying code and correlationId on failure` | |
| `notify resolves without a value` | |
| `on delivers matching events` | |
| `on ignores non-matching events` | |
| `on returns a working unsubscribe function` | |
| `multiple handlers for one event all run` | |
| `a throwing handler does not prevent other handlers` | |
| `a throwing handler does not disconnect` | |
| `an event with no handler does not throw` | |
| `disconnect is idempotent` | |
| `module imports cleanly in a node environment with no DOM globals` | **SSR safety** |
| `exported operation and event constants match artifacts/protocol-names.json` | **contract test** |

**`scripts/verify.ps1`** — activate the JS gate in this task:

```
pnpm --dir clients/js install --frozen-lockfile
pnpm --dir clients/js -r build
pnpm --dir clients/js -r test
pnpm --dir clients/js -r lint
```

Add a `js` job to `.github/workflows/ci.yml` using `pnpm/action-setup` + `actions/setup-node` with
the pnpm cache, and enable the `npm` Dependabot ecosystem for `clients/js`.

---

## Dependency justification

| Package | Why | Alternative? | Licence | Notes |
|---|---|---|---|---|
| `Microsoft.AspNetCore.App` (FrameworkReference) | SignalR server, endpoint routing, DI integration. | None. | MIT | Zero NuGet cost; the mechanical basis for deviation D1. |
| `Microsoft.AspNetCore.SignalR.Client` `10.0.11` | The .NET client wraps it. | Hand-rolling the SignalR protocol is absurd. | MIT | `clients/dotnet` only; never referenced by `src/`. |
| `Microsoft.AspNetCore.Mvc.Testing` `10.0.11` | `WebApplicationFactory`/`TestServer` per Spec §18. | No. | MIT | Test-only. |
| `@microsoft/signalr` | The TS client wraps it. | No. | MIT | Direct dependency, not peer — one supported version, owned by LiveSharp. |
| `typescript`, `tsup`, `vitest`, `eslint`, `typescript-eslint`, `@types/node` | Build, test, and lint the TS package. | No. | MIT / Apache-2.0 | Dev-only; not shipped. |

Explicitly rejected: `Microsoft.AspNetCore.SignalR.Protocols.MessagePack` (Phase 16 — do not add a
protocol before there is a measurement justifying it), `Polly` (SignalR's own reconnect policy is
sufficient and configurable), any React/framework package in `@livesharp/client`.

---

## Documentation deltas

- `docs/architecture/wire-protocol.md` — **new and important**. The full normative protocol: the two
  hub methods, the envelope and response shapes, the naming convention, version negotiation, the
  error-code table, and worked JSON examples of a successful invoke, a failed invoke, and a
  server-pushed event. This is the document a third-party client author implements against.
- `docs/getting-started/README.md` — first real content: install, `AddLiveSharp().AddSignalR()`,
  `MapLiveSharp()`, connect from .NET, connect from TypeScript. With an explicit banner: **"No
  authentication is enforced yet — see Phase 09. Do not expose this to the internet."**
- `docs/configuration/README.md` — the `LiveSharpHubOptions` table, and how it interacts with
  SignalR's own `HubOptions`.
- `docs/troubleshooting/README.md` — real entries: "messages sent to a user never arrive"
  (`IUserIdProvider` conflict), "`MapLiveSharp` throws at startup", "connection rejected with
  `protocol_mismatch`", "client connects then immediately closes", "handler changes are not picked up
  (scoped lifetime)".
- `docs/api/README.md` — link the two client surfaces.
- `clients/js/packages/client/README.md` — the npm package README, standalone and complete.
- `README.md` — flip the status table: SignalR transport → `Preview`, .NET client → `Preview`,
  TypeScript client → `Preview`. Add the first genuinely runnable quick-start.

## Example deltas

None yet — but `docs/getting-started` snippets must be **copied from a compiling test** so they
cannot rot. Add a test project or a `#region`-extracted snippet mechanism now; retrofitting it once
there are twenty snippets is much harder.

## CHANGELOG entry

```markdown
### Added
- `LiveSharp.AspNetCore`: SignalR-based transport, `LiveSharpHub`, `AddSignalR()`, `MapLiveSharp()`.
- Wire protocol v1: `InvokeAsync`/`NotifyAsync` operations and a single `Receive` event channel,
  with query-string protocol version negotiation.
- `LiveSharp.Client` for .NET, with automatic reconnect and typed event subscriptions.
- `@livesharp/client` for TypeScript/JavaScript, framework-agnostic and SSR-safe.
- Integration test harness exercising real SignalR connections over WebSockets and long polling.

### Security
- Detailed hub errors are disabled by default; internal exception details are never returned to
  clients.
```

---

## Exit criteria

- [ ] Both clients connect, invoke `connection.heartbeat`, and receive `connection.ready`.
- [ ] Every routing integration test passes over **WebSockets and LongPolling**.
- [ ] `Send_to_user_reaches_every_connection_of_that_user` passes — the `IUserIdProvider` wiring is
      proven end to end, not assumed.
- [ ] `Server_registry_is_empty_after_the_client_disconnects_cleanly` passes — the
      `CancellationToken.None` disconnect bug is proven absent.
- [ ] `Events_received_after_a_reconnect_still_reach_subscribers` passes on both clients.
- [ ] `module imports cleanly in a node environment with no DOM globals` passes.
- [ ] The protocol-name contract test passes: TS constants match the .NET reflection output.
- [ ] `Every_protocol_type_is_registered_in_the_serializer_context` passes.
- [ ] `Public_surface_does_not_expose_HubConnection` passes.
- [ ] `scripts/verify.ps1` runs the JS gate and is green on Windows and Linux.
- [ ] CI has a `js` job; Dependabot covers `clients/js`.
- [ ] `docs/architecture/wire-protocol.md` is complete enough for a third party to write a client.
- [ ] Tag `v0.2.0`; `dotnet pack` produces three packages; `pnpm pack` produces a valid tarball.
- [ ] **The Phase 03 PR is reviewed and approved by a human before Phase 04 begins.**

## Verify

```powershell
pwsh scripts/verify.ps1
dotnet test tests/LiveSharp.IntegrationTests -c Release
pnpm --dir clients/js -r test
git tag v0.2.0
```

## Next

**STOP.** Request human review of the wire protocol and transport. Only after approval:
`docs/plan/phase-04-direct-messaging.md`, task `P04.T1`.
