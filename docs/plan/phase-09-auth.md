# Phase 09 — Authentication & Authorization

| | |
|---|---|
| **Version at exit** | `0.8.0` (tag `v0.8.0`) |
| **Depends on** | Phase 08 |
| **Branch** | `feat/phase-09-auth` |
| **Tasks** | `P09.T1` … `P09.T7` |
| **Read before starting** | `SECURITY.md`, `docs/architecture/wire-protocol.md`, and the **Limitations** section of every doc page written in Phases 04–08 |

> ## This phase contains a breaking change
> Anonymous connections stop being allowed by default. Every consumer on `0.7.0` or earlier is
> relying on that, because it was the only thing that worked. The migration must be documented in
> `CHANGELOG.md`, `docs/security/authentication.md`, and a dedicated
> `docs/migration/0.7-to-0.8.md` before the phase is considered complete.

---

## Goal

Close every security gap Phases 03–08 deliberately left open. Until this phase, LiveSharp is a
technically impressive way to let any anonymous browser read and write anyone's messages. After this
phase it is something you could put on the internet.

Read the accumulated debt honestly:

| Gap | Opened in | Closed by |
|---|---|---|
| Anonymous connections synthesise an `anon:` user ID | `P03.T2` | `P09.T1` |
| No `[Authorize]` on the hub | `P03.T2` | `P09.T1` |
| Any connected user can message any other user | `P04.T4` | `P09.T5` |
| Any connected user can create unlimited groups | `P05.T2` | `P09.T4`, `P09.T6` |
| Group message membership is not checked for typing | `P07.T2` | `P09.T5` |
| `GetReceiptsAsync` authorization is hand-rolled | `P08.T3` | `P09.T5` |
| No rate limiting anywhere | all | `P09.T6` |
| A JWT can outlive its connection indefinitely | `P03.T2` | `P09.T7` |

Every row gets a test that flips from "pins the gap" to "pins the fix", and the corresponding
**Limitations** bullet gets deleted from its doc page. Deleting those bullets is part of the exit
criteria — a limitation that is fixed but still documented is as bad as one that is documented but
not fixed.

---

## What LiveSharp is not responsible for

State this in `SECURITY.md` and `docs/security/authentication.md` before writing code, because the
scope boundary determines the whole design:

- LiveSharp does **not** issue tokens, hash passwords, store users, or implement an identity
  provider. It consumes the host application's ASP.NET Core authentication.
- LiveSharp does **not** provide end-to-end encryption. The server sees message content. TLS protects
  it in transit; nothing protects it from the server operator.
- LiveSharp does **not** decide *who your users are*. It decides what an already-identified user may
  do inside LiveSharp's own surface.

---

## Non-goals — DO NOT implement in this phase

- No identity provider, user store, login endpoints, password handling, or token issuance.
- No end-to-end encryption or message signing.
- No distributed rate limiting. Limits are per-node; a two-node deployment doubles effective limits.
  Documented, closed in Phase 15.
- No audit-log persistence. Structured logs are emitted; shipping them somewhere is the application's
  job.
- No WebRTC or calling authorization. Phases 10 and 11 add their own resource checks on top of the
  pipeline built here.
- No CAPTCHA, IP reputation, geo-blocking, or content moderation.
- No multi-tenancy model. `IUserResolver` can encode a tenant in the user ID today; a first-class
  tenant concept would be a new ADR.

---

## Tasks

### P09.T1 — Require authentication

**Deliverables**

```
src/LiveSharp.AspNetCore/Security/LiveSharpAuthenticationOptions.cs
src/LiveSharp.AspNetCore/Security/LiveSharpAuthenticationOptionsValidator.cs
src/LiveSharp.AspNetCore/Hubs/LiveSharpHub.cs                     (modified)
src/LiveSharp.AspNetCore/DependencyInjection/…                     (modified)
docs/migration/0.7-to-0.8.md
tests/LiveSharp.AspNetCore.Tests/Security/AuthenticationRequirementTests.cs
tests/LiveSharp.IntegrationTests/Security/AnonymousRejectionTests.cs
```

**Public API (exact):**

```csharp
namespace LiveSharp;

public sealed class LiveSharpAuthenticationOptions
{
    public const string ConfigurationSectionName = "LiveSharp:Authentication";

    /// <summary>Allow unauthenticated connections. Default false. Development use only.</summary>
    public bool AllowAnonymous { get; set; }

    /// <summary>Authorization policy applied to the hub endpoint. Null uses the default policy.</summary>
    public string? HubPolicy { get; set; }

    /// <summary>Authentication schemes accepted on the hub endpoint. Empty uses the default scheme.</summary>
    public IList<string> Schemes { get; } = new List<string>();
}
```

**Requirements**

1. `[Authorize]` on `LiveSharpHub`. When `AllowAnonymous` is true, `AddSignalR()` registers a hub
   filter that bypasses it — do **not** conditionally remove the attribute, which is not possible at
   runtime, and do not have two hub classes.
2. Delete the `anon:{ConnectionId}` synthesis from `OnConnectedAsync`. When `IUserResolver` returns
   `null` for an authenticated principal, that is a **configuration error**, not an anonymous user:
   abort with `RealtimeErrorCode.Unauthenticated` and log at `Error` with a message naming
   `IUserResolver` and the claim types actually present on the principal (claim **types** only, never
   values). Someone whose `sub` claim is named something unexpected will otherwise spend a day on this.
3. When `AllowAnonymous` is true, keep the `anon:` synthesis but log a `Warning` **once per
   application start** (not per connection) stating that anonymous access is enabled and must not be
   used in production. Also refuse to combine `AllowAnonymous = true` with a non-Development
   environment unless `LiveSharpAuthenticationOptions.HubPolicy` is explicitly set to `null` and a
   second flag is set — no. Simpler and better: log an `Error` (not `Warning`) at startup when
   `AllowAnonymous` is true and `IHostEnvironment.IsDevelopment()` is false, and continue. Loud but not
   fatal; a hard failure would break someone's staging environment at 2am for a config they chose.
4. `MapLiveSharp()` applies `RequireAuthorization(policy)` when `HubPolicy` is set, and
   `RequireAuthorization()` otherwise, unless `AllowAnonymous`. Returning
   `HubEndpointConventionBuilder` from Phase 03 was forward planning for exactly this; the consumer can
   still chain their own conventions.
5. `docs/migration/0.7-to-0.8.md`: what broke, the one-line fix for people who genuinely want anonymous
   (`o.Authentication.AllowAnonymous = true`), and the correct fix (configure authentication).

**Tests**

| Test | Asserts |
|---|---|
| `Hub_carries_the_authorize_attribute` | reflection |
| `Anonymous_connection_is_rejected_by_default` | integration; connect throws |
| `Anonymous_connection_is_accepted_when_explicitly_allowed` | |
| `Allow_anonymous_logs_a_startup_warning_exactly_once` | not per connection |
| `Allow_anonymous_outside_development_logs_a_startup_error` | |
| `Authenticated_principal_with_no_resolvable_user_id_is_rejected_as_unauthenticated` | |
| `Unresolvable_user_id_error_names_the_present_claim_types` | diagnosability |
| `Unresolvable_user_id_error_does_not_log_claim_values` | **security test** |
| `Map_live_sharp_applies_require_authorization_by_default` | endpoint metadata |
| `Map_live_sharp_applies_the_configured_policy` | |
| `Map_live_sharp_omits_authorization_when_anonymous_is_allowed` | |
| `Consumer_conventions_chained_after_map_live_sharp_still_apply` | |
| `No_anon_prefix_user_id_is_ever_created_when_anonymous_is_disallowed` | grep-style assertion via the registry |

---

### P09.T2 — JWT bearer over WebSockets

**Deliverables**

```
src/LiveSharp.AspNetCore/Security/LiveSharpJwtExtensions.cs
docs/security/authentication.md
tests/LiveSharp.AspNetCore.Tests/Security/JwtQueryStringTests.cs
tests/LiveSharp.IntegrationTests/Security/JwtAuthenticationTests.cs
```

**The problem, stated plainly:** the browser `WebSocket` constructor cannot set an `Authorization`
header. SignalR's JavaScript client therefore appends the token as `?access_token=…` on the
negotiate and connect requests. ASP.NET Core's JWT bearer handler only reads the header. Without
glue, WebSocket connections are unauthenticated while long polling works — a failure mode that looks
like a transport bug and is actually an auth bug.

**Public API (exact):**

```csharp
namespace LiveSharp;

public static class LiveSharpJwtExtensions
{
    /// <summary>Reads the bearer token from the <c>access_token</c> query parameter for requests to the
    /// LiveSharp hub path, which is required for browser WebSocket connections.</summary>
    public static AuthenticationBuilder AddLiveSharpJwtQueryString(this AuthenticationBuilder builder);

    public static AuthenticationBuilder AddLiveSharpJwtQueryString(this AuthenticationBuilder builder, string authenticationScheme);
}
```

**Requirements**

- Implemented as a `PostConfigure<JwtBearerOptions>` that chains onto the existing
  `Events.OnMessageReceived` rather than replacing it — a consumer with their own handler must not have
  it silently dropped. Test that a pre-existing handler still runs.
- The path check must use the **configured** `LiveSharpHubOptions.Path`, not a hardcoded
  `/livesharp`, and must match on segment boundaries (`/livesharp` and `/livesharp/negotiate`, not
  `/livesharpadmin`). A prefix match without a boundary check leaks the query-string token acceptance
  to unrelated endpoints. Test the negative case.
- Only apply when `Request.Query["access_token"]` is present **and** no `Authorization` header exists.
  Header wins.
- The token must never be logged. Add a test that scans captured logs for the token string after a
  successful and a failed authentication.

**Security documentation is mandatory for this task**, in `docs/security/authentication.md`:

- Query-string tokens land in reverse-proxy access logs, browser history, and `Referer` headers.
  Mitigations to state: use short-lived tokens (minutes), scope them to LiveSharp, exclude the query
  string from access logs at the proxy (with an nginx and an IIS example), and never reuse the
  application's long-lived session token.
- Explain that `LiveSharp.Client` (.NET) and any non-browser client should use the
  `Authorization` header instead, and that `AccessTokenProvider` on the .NET client already does.
- Show the complete working `Program.cs`: `AddAuthentication().AddJwtBearer().AddLiveSharpJwtQueryString()`
  plus `AddLiveSharp().AddSignalR()` plus `MapLiveSharp()`.

**Tests**

| Test | Asserts |
|---|---|
| `Query_string_token_is_accepted_on_the_hub_path` | |
| `Query_string_token_is_ignored_on_an_unrelated_path` | **the boundary test**: `/livesharpadmin` |
| `Query_string_token_is_accepted_on_the_negotiate_sub_path` | |
| `Query_string_token_is_ignored_when_an_authorization_header_is_present` | header wins |
| `A_pre_existing_on_message_received_handler_still_runs` | chaining, not replacing |
| `Configured_non_default_hub_path_is_honoured` | |
| `Token_value_never_appears_in_logs_on_success` | **security test** |
| `Token_value_never_appears_in_logs_on_failure` | **security test** |
| `Websocket_connection_authenticates_with_a_query_string_token` | integration |
| `Long_polling_connection_authenticates_with_a_query_string_token` | integration |
| `Invalid_token_is_rejected_before_connection_registration` | registry stays empty |

---

### P09.T3 — The operation filter pipeline

**Deliverables**

```
src/LiveSharp.Abstractions/Filters/IOperationFilter.cs
src/LiveSharp.Abstractions/Filters/OperationFilterDelegate.cs
src/LiveSharp.Core/Routing/OperationRouter.cs                     (modified)
src/LiveSharp.Core/Filters/OperationFilterPipeline.cs
src/LiveSharp.Core/Filters/CorrelationLoggingFilter.cs
src/LiveSharp.Core/Filters/PayloadValidationFilter.cs
src/LiveSharp.Core/DependencyInjection/…                          (AddOperationFilter)
tests/LiveSharp.Core.Tests/Filters/…
```

**Public API (exact):**

```csharp
namespace LiveSharp.Filters;

public delegate ValueTask<OperationResult<JsonElement?>> OperationFilterDelegate();

/// <summary>Wraps operation invocation. Filters run in registration order, outermost first.</summary>
public interface IOperationFilter
{
    ValueTask<OperationResult<JsonElement?>> InvokeAsync(
        RealtimeOperationContext context,
        JsonElement? payload,
        OperationFilterDelegate next);
}
```

```csharp
namespace LiveSharp;

public static partial class LiveSharpBuilderExtensions
{
    /// <summary>Registers a filter. Filters execute in registration order.</summary>
    public static ILiveSharpBuilder AddOperationFilter<TFilter>(this ILiveSharpBuilder builder)
        where TFilter : class, IOperationFilter;
}
```

**Requirements**

- The pipeline is built **once** at startup into an array plus a pre-composed delegate chain, not
  rebuilt per invocation. `OperationRouter.RouteAsync` invokes the pipeline, whose innermost step is
  the existing handler dispatch. Retrofitting this must not change `IOperationHandler` or any handler
  written in Phases 04–08 — additive only. Add a test that a Phase-04-era handler works unchanged.
- Built-in filter order, fixed and documented (consumer filters append after these unless they opt to
  run earlier via a documented ordering value):

  1. `CorrelationLoggingFilter` — establishes the logging scope with the correlation ID, operation
     name, and connection ID. Outermost so everything inside is correlated.
  2. `AuthenticationFilter` (`P09.T4`) — rejects unauthenticated when required.
  3. `AuthorizationFilter` (`P09.T4`) — policy/role checks.
  4. `RateLimitingFilter` (`P09.T6`) — after authorization so limits are per-identity, not per-anon.
  5. `PayloadValidationFilter` — size and shape.

  The ordering rationale matters: rate limiting **after** authorization means an unauthenticated flood
  is rejected by cheaper checks first, and authenticated limits are attributable. Rate limiting
  *before* authentication would let an attacker exhaust a shared bucket. State this in the docs.
- A filter that throws is caught by the pipeline, logged at `Error` with the correlation ID, and turned
  into `Failure(InternalError, …)` with **no exception detail on the wire** — the same contract the
  router already has. A filter must not be able to crash the hub.
- A filter that short-circuits (returns without calling `next`) must be respected, and the remaining
  filters and the handler must not run. Test with the short-circuit at position 1, in the middle, and
  at the last position.
- Filters are registered as **singletons** and must be stateless and thread-safe. Document it and
  assert the lifetime in a test. (Scoped filters would allocate per invocation on the hottest path in
  the system for no benefit — filters can resolve scoped dependencies from `context.Services`.)
- Migrate `IMessageValidator` (Phase 04) to run inside `PayloadValidationFilter` while keeping the
  `IMessageValidator` public interface intact. Existing custom validators keep working. Test it.

**Tests**

| Test | Asserts |
|---|---|
| `Filters_run_in_registration_order` | |
| `Filters_wrap_the_handler_outermost_first` | order recorded on the way in and out |
| `A_filter_that_short_circuits_prevents_later_filters_and_the_handler` | three positions |
| `A_filter_can_modify_the_result_on_the_way_out` | |
| `A_throwing_filter_becomes_an_internal_error` | |
| `A_throwing_filter_does_not_leak_its_message_to_the_client` | **security test** |
| `A_throwing_filter_is_logged_with_the_correlation_id` | |
| `Cancellation_inside_a_filter_propagates_rather_than_becoming_internal_error` | |
| `A_handler_written_before_the_pipeline_existed_works_unchanged` | **the additive-change proof** |
| `The_pipeline_is_composed_once_not_per_invocation` | assert via a counting factory |
| `Filters_are_registered_as_singletons` | `ServiceDescriptor.Lifetime` |
| `Built_in_filters_run_in_the_documented_order` | pins the security-relevant ordering |
| `Correlation_scope_is_present_for_every_log_written_inside_the_pipeline` | |
| `Existing_message_validators_still_run_via_the_validation_filter` | |
| `Pipeline_with_no_filters_registered_behaves_identically_to_direct_dispatch` | zero-cost default |

---

### P09.T4 — Declarative operation authorization

**Deliverables**

```
src/LiveSharp.Abstractions/Security/RealtimeAuthorizeAttribute.cs
src/LiveSharp.Core/Routing/OperationRegistration.cs               (modified)
src/LiveSharp.AspNetCore/Security/AuthenticationFilter.cs
src/LiveSharp.AspNetCore/Security/AuthorizationFilter.cs
src/LiveSharp.AspNetCore/Security/OperationPolicyMap.cs
tests/LiveSharp.AspNetCore.Tests/Security/OperationAuthorizationTests.cs
```

**Public API (exact):**

```csharp
namespace LiveSharp;

/// <summary>Declares the authorization requirements for an operation handler.</summary>
[AttributeUsage(AttributeTargets.Class, AllowMultiple = false, Inherited = false)]
public sealed class RealtimeAuthorizeAttribute : Attribute
{
    public string? Policy { get; set; }
    public string? Roles { get; set; }

    /// <summary>When true, the operation is reachable without authentication. Use sparingly.</summary>
    public bool AllowAnonymous { get; set; }
}
```

```csharp
namespace LiveSharp.Security;

/// <summary>Maps operation names to authorization policies without using attributes.</summary>
public sealed class OperationPolicyMap
{
    public OperationPolicyMap RequirePolicy(string operation, string policy);
    public OperationPolicyMap RequireRoles(string operation, params string[] roles);
    public OperationPolicyMap AllowAnonymous(string operation);
}
```

**Requirements**

- The attribute is read **once at registration** and stored on `OperationRegistration` as resolved
  metadata (a nullable policy name, a parsed role array, and a boolean). No per-invocation reflection —
  this runs on every message.
- Both mechanisms coexist. Precedence: `OperationPolicyMap` (configured by the consumer) **overrides**
  the attribute (shipped by LiveSharp or a feature package), because the consumer must be able to
  tighten or loosen what a package author chose. Document the precedence and test it.
- `AuthorizationFilter` delegates to ASP.NET Core's `IAuthorizationService` for policies and to
  `ClaimsPrincipal.IsInRole` for roles. Do not reimplement policy evaluation.
- Roles is a comma-separated OR set, matching `[Authorize(Roles = "a,b")]` semantics exactly. Anything
  else surprises every ASP.NET Core developer.
- Failure → `Failure(RealtimeErrorCode.Forbidden, "…")` with a message naming the operation but
  **not** the policy name or the missing role. Leaking the policy name tells an attacker the shape of
  your authorization model. Log the policy name server-side at `Information` with the correlation ID
  so the developer can still debug it. Test both halves.
- Every operation shipped by LiveSharp gets an explicit decision recorded in a table in
  `docs/security/authorization.md`. Operations that need no policy beyond authentication are listed as
  such — an operation with no row in that table is a review failure. Add a test that reflects over all
  registered operations and fails if any is missing from a `KnownOperationPolicies` manifest, so a new
  operation added in Phase 10+ cannot silently skip the security review.

**Tests**

| Test | Asserts |
|---|---|
| `Operation_with_no_attribute_requires_authentication_only` | |
| `Operation_with_a_policy_attribute_is_allowed_when_the_policy_succeeds` | |
| `Operation_with_a_policy_attribute_is_forbidden_when_the_policy_fails` | |
| `Operation_with_roles_is_allowed_when_the_user_has_any_listed_role` | OR semantics |
| `Operation_with_roles_is_forbidden_when_the_user_has_none` | |
| `Operation_marked_allow_anonymous_is_reachable_unauthenticated` | |
| `Policy_map_overrides_the_attribute` | both tighten and loosen |
| `Forbidden_response_does_not_name_the_policy_or_role` | **security test** |
| `Forbidden_is_logged_server_side_with_the_policy_name_and_correlation_id` | diagnosability |
| `Attribute_metadata_is_resolved_at_registration_not_per_invocation` | counting reflection calls |
| `Every_registered_operation_appears_in_the_known_policies_manifest` | **the review gate** |
| `Authentication_filter_rejects_before_the_authorization_filter_runs` | ordering |

---

### P09.T5 — Resource-based authorization

**Deliverables**

```
src/LiveSharp.Abstractions/Security/IRealtimeAuthorizationService.cs
src/LiveSharp.Abstractions/Security/GroupAccess.cs
src/LiveSharp.Abstractions/Security/MessageAction.cs
src/LiveSharp.Core/Security/AllowAllAuthorizationService.cs
src/LiveSharp.Chat/Security/ChatAuthorizationExtensions.cs
tests/…                                                          (per feature package)
```

**Public API (exact):**

```csharp
namespace LiveSharp.Security;

/// <summary>Application-supplied decisions about what a user may do to a specific resource.</summary>
public interface IRealtimeAuthorizationService
{
    ValueTask<bool> CanSendToUserAsync(string senderId, string recipientId, CancellationToken cancellationToken = default);
    ValueTask<bool> CanCreateGroupAsync(string userId, CancellationToken cancellationToken = default);
    ValueTask<bool> CanAccessGroupAsync(string userId, string groupId, GroupAccess access, CancellationToken cancellationToken = default);
    ValueTask<bool> CanViewPresenceAsync(string viewerId, string subjectUserId, CancellationToken cancellationToken = default);
    ValueTask<bool> CanActOnMessageAsync(string userId, string messageId, MessageAction action, CancellationToken cancellationToken = default);
}

[Flags]
public enum GroupAccess
{
    None       = 0,
    Read       = 1,
    Send       = 2,
    Administer = 4,
}

public enum MessageAction
{
    ViewReceipts    = 0,
    Acknowledge     = 1,
    MarkRead        = 2,
}
```

> ## The default implementation must be loud, not silent
> `AllowAllAuthorizationService` returns `true` for everything — that is the only default that does not
> break every existing consumer on upgrade. But a permissive default that says nothing is how libraries
> ship insecure deployments.
>
> It must log an **`Error`** at startup, once, naming the interface and the docs page:
> *"LiveSharp is using AllowAllAuthorizationService: every user may message every other user, read any
> group they can name, and view any presence. Implement IRealtimeAuthorizationService before going to
> production. See docs/security/authorization.md."*
>
> Do not downgrade this to `Warning` to make CI output cleaner. It is `Error` on purpose.

**Requirements — closing the specific gaps**

| Gap | Enforcement point | Check |
|---|---|---|
| Any user can DM any user (`P04.T4`) | `ChatService.SendDirectAsync` | `CanSendToUserAsync` |
| Any user can create groups (`P05.T2`) | `GroupService.CreateAsync` | `CanCreateGroupAsync` |
| Group send by non-member | `ChatService.SendToGroupAsync` | membership **and** `CanAccessGroupAsync(Send)` |
| Group admin actions | `GroupService.AddMember/RemoveMember/ChangeRole/Delete` | role rules **and** `CanAccessGroupAsync(Administer)` |
| Group roster read | `GroupService.GetMembersAsync` | membership **and** `CanAccessGroupAsync(Read)` |
| Typing into a group you are not in (`P07.T2`) | `TypingService.StartAsync` | membership |
| Typing into a DM you are not part of | already enforced in `P07.T2` | keep |
| Receipt query by an unrelated user (`P08.T3`) | `MessageStateService.GetReceiptsAsync` | `CanActOnMessageAsync(ViewReceipts)` |
| Presence query for an unrelated user (`P06.T5`) | `PresenceService` / `QueryPresenceHandler` | `CanViewPresenceAsync` **in addition to** `IPresenceSubscriptionSource.CanObserveAsync` |

- The existing role rules and `IPresenceSubscriptionSource` checks are **not replaced** — both must
  pass. Two independent gates that both have to agree is the correct posture, and it means a consumer
  who implements only one of the two interfaces is still protected by the other.
- Checks live in the **service** layer, not in filters, because they need resource identity from the
  deserialized payload and because the services must stay safe when called from a controller or a
  background job (the Phase 04 "not hub-coupled" property).
- Failures return `Forbidden` with a generic message. For reads, keep the Phase 05 rule: unknown and
  forbidden are indistinguishable.
- `ValueTask<bool>` not `Task<bool>`: an in-memory implementation returns synchronously, and these are
  called on every message.
- Every check must be given a **negative test in its own feature package's test project**. Ten new
  tests spread across four projects, not one grab-bag file.

**Tests — one table, split across projects**

| Test | Asserts |
|---|---|
| `Direct_message_to_a_disallowed_recipient_is_forbidden` | `LiveSharp.Chat.Tests` |
| `Direct_message_to_an_allowed_recipient_succeeds` | |
| `Group_creation_by_a_disallowed_user_is_forbidden` | |
| `Group_send_by_a_non_member_is_forbidden` | **was passing as allowed in Phase 05** |
| `Group_send_by_a_member_the_service_disallows_is_forbidden` | both gates |
| `Group_administer_by_an_admin_the_service_disallows_is_forbidden` | |
| `Group_roster_read_by_a_disallowed_member_is_forbidden` | |
| `Typing_into_a_group_you_are_not_a_member_of_is_forbidden` | **was passing as allowed in Phase 07** |
| `Receipt_query_for_an_unrelated_message_is_forbidden_via_the_service` | **replaces the hand-rolled check** |
| `Presence_query_requires_both_the_subscription_source_and_the_authorization_service` | four combinations |
| `Allow_all_authorization_service_logs_an_error_at_startup_exactly_once` | |
| `A_custom_authorization_service_replaces_the_permissive_default` | `TryAdd` semantics |
| `Authorization_failures_are_logged_with_the_actor_and_resource_at_information` | auditability |
| `Authorization_failure_messages_do_not_reveal_resource_existence` | **security test** |
| `All_authorization_checks_honour_a_pre_cancelled_token` | |

---

### P09.T6 — Rate limiting

**Deliverables**

```
src/LiveSharp.AspNetCore/Security/RateLimitingFilter.cs
src/LiveSharp.AspNetCore/Security/LiveSharpRateLimitOptions.cs
src/LiveSharp.AspNetCore/Security/OperationRateLimitPolicy.cs
tests/LiveSharp.AspNetCore.Tests/Security/RateLimitingTests.cs
tests/LiveSharp.IntegrationTests/Security/RateLimitIntegrationTests.cs
```

**Public API (exact):**

```csharp
namespace LiveSharp.Security;

public sealed class LiveSharpRateLimitOptions
{
    public const string ConfigurationSectionName = "LiveSharp:RateLimits";

    public bool Enabled { get; set; } = true;

    /// <summary>Default limit applied to any operation without a specific policy.</summary>
    public OperationRateLimitPolicy Default { get; set; } = new() { PermitLimit = 60, Window = TimeSpan.FromSeconds(10) };

    /// <summary>Per-operation overrides, keyed by operation name.</summary>
    public IDictionary<string, OperationRateLimitPolicy> Operations { get; } = new Dictionary<string, OperationRateLimitPolicy>(StringComparer.Ordinal);

    /// <summary>Total inbound operations per connection across all operations.</summary>
    public OperationRateLimitPolicy PerConnection { get; set; } = new() { PermitLimit = 300, Window = TimeSpan.FromSeconds(10) };

    /// <summary>Maximum tracked partitions before the oldest are evicted. Default 100000.</summary>
    public int MaxPartitions { get; set; } = 100_000;
}

public sealed class OperationRateLimitPolicy
{
    public int PermitLimit { get; set; }
    public TimeSpan Window { get; set; }

    /// <summary>Requests permitted to burst above the steady rate. Default 0.</summary>
    public int QueueLimit { get; set; }

    /// <summary>Partition key source. Default User.</summary>
    public RateLimitPartition Partition { get; set; } = RateLimitPartition.User;
}

public enum RateLimitPartition { User = 0, Connection = 1 }
```

**Requirements**

- Use `System.Threading.RateLimiting` from the BCL — `PartitionedRateLimiter.Create` with a
  `SlidingWindowRateLimiter` per partition. **No new dependency.** Explicitly reject
  `AspNetCoreRateLimit` and similar: they target HTTP middleware, not hub operations.
- Two limiters evaluated per operation: the per-operation limit and the per-connection aggregate. Both
  must permit. The aggregate exists so 20 different cheap operations cannot together saturate a
  connection.
- Partition by **user** by default, not connection. A user with five devices should not get five times
  the budget for abusive behaviour. Per-connection partitioning is available for operations where
  device-level fairness matters.
- Defaults must be generous enough not to break the existing examples and tight enough to matter.
  Sensible starting points, to be stated in the docs as starting points rather than
  recommendations: `chat.message.send` 20/10s, `typing.start` 30/10s, `presence.status.set` 10/60s,
  `presence.query` 20/10s, `chat.receipt.*` 60/10s, `group.create` 5/60s, `group.member.add` 30/60s.
- **`NotifyAsync` operations must be dropped silently when limited, never errored.** `typing.start` and
  `chat.receipt.delivered` are fire-and-forget; returning `rate_limited` produces a failure the client
  cannot see and a log line per keystroke. Detect the fire-and-forget path from the router (a flag on
  `RealtimeOperationContext`, set by the hub) and short-circuit to success with a `Debug` log. This is
  the same deliberate-swallow pattern as `P07.T3`; reference it.
- Rejections for `InvokeAsync` return `Failure(RealtimeErrorCode.RateLimited, …)` including a
  `retryAfterSeconds` value in `OperationResponse.Data` so a client can back off intelligently rather
  than hammering.
- `MaxPartitions` bounds the limiter dictionary. `PartitionedRateLimiter` does not evict by default; an
  unbounded partition map keyed by user ID is a memory leak on a public deployment. Implement eviction
  (LRU or a periodic sweep of idle partitions via `TimeProvider`) and add a leak test.
- `IAsyncDisposable`: limiters hold timers. Dispose them with the filter, and test that disposal is
  idempotent.
- **Per-node only.** A two-node deployment permits twice the configured rate. Document it prominently
  in `docs/security/rate-limiting.md` and link forward to Phase 15.

**Tests**

| Test | Asserts |
|---|---|
| `Operation_within_the_limit_is_permitted` | |
| `Operation_beyond_the_limit_is_rate_limited` | limit + 1 |
| `Operation_exactly_at_the_limit_is_permitted` | boundary |
| `Limit_recovers_after_the_window_elapses` | `FakeTimeProvider` |
| `Per_operation_override_takes_precedence_over_the_default` | |
| `Per_connection_aggregate_limits_across_different_operations` | 3 operations, shared budget |
| `Two_users_have_independent_budgets` | **isolation** |
| `Two_connections_of_one_user_share_a_user_partitioned_budget` | the deliberate choice |
| `Two_connections_of_one_user_have_independent_connection_partitioned_budgets` | |
| `Notify_operations_are_dropped_silently_when_limited` | **required** |
| `Notify_operations_dropped_by_the_limiter_are_logged_at_debug` | |
| `Invoke_rejection_includes_a_retry_after_value` | |
| `Rate_limiting_disabled_permits_everything` | |
| `Rate_limiting_runs_after_authorization` | ordering: an unauthorized request does not consume budget |
| `Partition_count_is_bounded_and_idle_partitions_are_evicted` | **leak test** |
| `Disposal_releases_every_limiter_and_is_idempotent` | |
| `Concurrent_requests_from_one_user_are_limited_correctly_without_over_permitting` | `ConcurrencyHarness` |
| `A_flood_from_one_user_does_not_affect_another_users_latency` | integration, coarse assertion |

---

### P09.T7 — Token expiry, sensitive-logging audit, and examples

**Deliverables**

```
src/LiveSharp.AspNetCore/Security/TokenExpiryMonitor.cs
src/LiveSharp.AspNetCore/Security/ITokenExpiryPolicy.cs
tests/LiveSharp.AspNetCore.Tests/Security/TokenExpiryMonitorTests.cs
tests/LiveSharp.ArchitectureTests/SensitiveLoggingTests.cs
docs/security/*.md
examples/**                                                       (all five updated)
```

**Requirements — token expiry**

A WebSocket can live for hours; a JWT typically lives for minutes. ASP.NET Core authenticates the
connection **once**, at handshake. Without action, a revoked or expired token keeps its connection
alive indefinitely — which means "log out" does not log out.

- `TokenExpiryMonitor : BackgroundService` at a configurable interval (default 30 s): scan registered
  connections, read the `exp` claim captured at connect time, and disconnect those past expiry with
  `DisconnectReason.ServerClosed` and a `connection.closing` envelope carrying
  `RealtimeErrorCode.Unauthenticated`.
- Store the expiry on the connection at connect time in `RealtimeConnection.Metadata` under
  `livesharp.token.exp` as a Unix seconds string. Do **not** store the token. Re-reading the principal
  later is not possible — SignalR's `Context.User` is a snapshot — so capturing `exp` at connect is the
  only option. Document that.
- `ITokenExpiryPolicy` lets the application override behaviour: disconnect immediately, disconnect with
  a grace period, or never (for consumers using long-lived tokens deliberately). Default: disconnect
  with a 30 s grace period, so a client that refreshes promptly is not dropped mid-message.
- Clients must reconnect with a fresh token. `LiveSharpClientOptions.AccessTokenProvider` is already
  invoked per connection attempt, so `.WithAutomaticReconnect()` plus a provider that fetches a current
  token already does the right thing — verify it in an integration test and document the pattern. The
  TS client's `accessTokenFactory` behaves the same way.
- A connection with **no** `exp` claim is never expired by the monitor. Log at `Debug` once per
  connection. Cookie authentication has no `exp`; do not break it.

**Requirements — sensitive-logging audit**

One architecture test that reflects over **every** `[LoggerMessage]` method in every `src/` assembly and
fails on:
- a message template parameter whose name matches a deny-list: `token`, `accessToken`, `password`,
  `secret`, `credential`, `content`, `message` (as a body), `payload`, `sdp`, `candidate`,
  `statusMessage`, `claim`, `authorization`, `cookie`;
- a template at `Information` or above containing a `{UserId}` placeholder — user IDs are PII in many
  jurisdictions and belong at `Debug`;
- a template at `Information` or above containing a `{ConversationId}` placeholder — direct conversation
  IDs embed two user IDs.

Allow explicit opt-out via an assembly-level attribute listing approved exceptions, so the test is
enforceable rather than a thing people disable. Every exception needs a comment. This single test is
worth more than any amount of review discipline.

Also: audit and fix everything the test catches across Phases 01–08.

**Requirements — examples**

All five examples move from the hardcoded demo dropdown to **real cookie authentication with a
claims-based identity** (still a hardcoded user list — this is a sample, not an identity provider — but
with proper `ClaimsPrincipal`, `NameIdentifier`, and roles). The React and Next.js examples additionally
demonstrate JWT with `accessTokenFactory` and a token-refresh path, because that is the realistic SPA
setup and the one the query-string documentation applies to.

Each example gains a minimal `IRealtimeAuthorizationService` implementation showing a real rule (for
instance: you may only message users who share a group with you), so the interface is demonstrated
rather than merely documented. Remove the "no authorization — anyone can message anyone" warnings from
every README, and remove the red demo-auth banner where real auth now exists (keep it where the user
list is still hardcoded, with revised wording).

**Tests**

| Test | Asserts |
|---|---|
| `Connection_past_its_token_expiry_is_disconnected` | `FakeTimeProvider` |
| `Connection_within_its_token_expiry_is_not_disconnected` | |
| `Connection_within_the_grace_period_is_not_disconnected` | |
| `Connection_with_no_expiry_claim_is_never_disconnected` | cookie auth |
| `Expiry_disconnect_sends_connection_closing_with_unauthenticated` | |
| `Expiry_disconnect_uses_the_server_closed_reason` | |
| `A_custom_expiry_policy_overrides_the_default` | |
| `The_token_itself_is_never_stored_in_connection_metadata` | **security test** |
| `Monitor_survives_a_throwing_registry` | |
| `Monitor_stops_promptly_on_the_stopping_token` | |
| `Client_with_an_access_token_provider_reconnects_with_a_fresh_token` | integration, both clients |
| `No_logger_message_template_parameter_matches_the_sensitive_deny_list` | **the audit test** |
| `No_logger_message_at_information_or_above_logs_a_user_id` | |
| `No_logger_message_at_information_or_above_logs_a_conversation_id` | |
| `Approved_logging_exceptions_are_explicitly_listed_and_commented` | |

---

## Dependency justification

| Package | Why | Alternative? | Licence | Notes |
|---|---|---|---|---|
| `Microsoft.AspNetCore.Authentication.JwtBearer` | Only in the **examples** and in `LiveSharp.AspNetCore.Tests`, for the JWT integration tests. `LiveSharpJwtExtensions` needs `JwtBearerOptions`, which is in the shared framework — verify this; if it is not, the extension moves to a separate `LiveSharp.AspNetCore.Jwt` package rather than adding a package reference to the core ASP.NET Core package. | None. | MIT | **Verify shared-framework availability first.** Adding a JWT package dependency to `LiveSharp.AspNetCore` would force it on cookie-auth consumers, which violates Spec §7. If it is not in the framework, split the package and record it in ADR-004. |
| `System.Threading.RateLimiting` | Rate limiting. In the shared framework / BCL. | Hand-rolling a sliding window is possible and worse. | MIT | No package reference needed on `net10.0`. Confirm. |

Explicitly rejected: `AspNetCoreRateLimit` (HTTP middleware, wrong layer), `IdentityServer`/`OpenIddict`
(LiveSharp does not issue tokens), any policy-DSL library (ASP.NET Core authorization is sufficient).

---

## Documentation deltas

- `SECURITY.md` — substantially expanded: the security model, the threat model summary (what LiveSharp
  defends against and what it explicitly does not), the responsibility boundary from the top of this
  file, reporting process, and supported versions.
- `docs/security/authentication.md` — **new**. Cookie and JWT setups end to end, the query-string token
  problem and its mitigations with nginx/IIS examples, `IUserResolver` customisation, the
  `AllowAnonymous` escape hatch and why it is `Error`-logged, token expiry and refresh.
- `docs/security/authorization.md` — **new**. The two-layer model (declarative operations + resource
  checks), the complete operation → policy table, `IRealtimeAuthorizationService` with a worked
  implementation, and why the default is permissive-but-loud.
- `docs/security/rate-limiting.md` — **new**. Partitioning, the default table as starting points, the
  `NotifyAsync` silent-drop behaviour, `retryAfterSeconds`, and the per-node limitation.
- `docs/security/checklist.md` — **new**. A production go-live checklist: authentication configured,
  `AllowAnonymous` false, `IRealtimeAuthorizationService` implemented, rate limits reviewed,
  `EnableDetailedErrors` false, TLS enforced, access-log query-string redaction, token lifetime short,
  sensitive logging off, CORS reviewed, `MaxPayloadBytes` reviewed.
- `docs/migration/0.7-to-0.8.md` — **new**. The anonymous-connection breaking change.
- **Every** Phase 04–08 doc page: delete the Limitations bullets this phase fixed and link to the
  security pages.
- `docs/troubleshooting/README.md` — "WebSocket connects but long polling authenticates" (and vice
  versa), "connection rejected with unauthenticated after upgrading to 0.8.0", "user ID cannot be
  resolved", "operations return forbidden after implementing IRealtimeAuthorizationService",
  "clients disconnect every few minutes" (token expiry), "typing floods stop working" (rate limits).
- `README.md` — authentication and authorization → `Available`; remove the pre-alpha security warning
  from the quick start.

## Example deltas

All five examples: real cookie auth with claims, a demonstrated `IRealtimeAuthorizationService`,
rate-limit configuration, and revised READMEs. React and Next.js additionally show JWT with token
refresh.

## CHANGELOG entry

```markdown
### Added
- Operation filter pipeline (`IOperationFilter`) providing correlation logging, authentication,
  authorization, rate limiting, and payload validation as composable, ordered stages.
- `[RealtimeAuthorize]` attribute and `OperationPolicyMap` for declarative per-operation authorization.
- `IRealtimeAuthorizationService` for resource-level decisions: direct messaging, group access,
  presence visibility, and message receipt actions.
- Rate limiting built on `System.Threading.RateLimiting`, partitioned per user or per connection, with
  per-operation policies and bounded partition tracking.
- `AddLiveSharpJwtQueryString()` enabling browser WebSocket authentication with bearer tokens.
- `TokenExpiryMonitor` disconnecting connections whose access token has expired, with a configurable
  policy.
- Production security checklist and authentication, authorization, and rate-limiting guides.

### Changed
- **BREAKING** Unauthenticated connections are rejected by default. Set
  `o.Authentication.AllowAnonymous = true` to restore the previous behaviour, or configure
  authentication. See `docs/migration/0.7-to-0.8.md`.
- `IMessageValidator` now executes inside the operation filter pipeline. Custom implementations are
  unaffected.

### Security
- Group messages, group administration, and group rosters now require membership and an authorization
  decision.
- Typing indicators can no longer be sent into a group the user is not a member of.
- Message receipts can no longer be queried by users unrelated to the message.
- Presence queries require both a subscription and an authorization decision.
- Access tokens are never logged and are never stored in connection metadata.
- An automated test prevents any log message from carrying tokens, credentials, message content, SDP,
  ICE candidates, or user identifiers above `Debug` level.
- The permissive default authorization service logs an error at startup and must be replaced before
  production use.
```

---

## Exit criteria

- [ ] `Anonymous_connection_is_rejected_by_default` passes.
- [ ] Every gap in the debt table at the top of this file has a passing **fix** test, and its former
      "pins the gap" test has been inverted or deleted with a commit message explaining why.
- [ ] `Every_registered_operation_appears_in_the_known_policies_manifest` passes.
- [ ] `No_logger_message_template_parameter_matches_the_sensitive_deny_list` passes across all `src/`
      assemblies, with every exception explicitly listed and commented.
- [ ] `Notify_operations_are_dropped_silently_when_limited` passes.
- [ ] `Rate_limiting_runs_after_authorization` passes.
- [ ] `Partition_count_is_bounded_and_idle_partitions_are_evicted` passes.
- [ ] `A_handler_written_before_the_pipeline_existed_works_unchanged` passes.
- [ ] `Pipeline_with_no_filters_registered_behaves_identically_to_direct_dispatch` passes.
- [ ] `Query_string_token_is_ignored_on_an_unrelated_path` passes.
- [ ] `The_token_itself_is_never_stored_in_connection_metadata` passes.
- [ ] `Allow_all_authorization_service_logs_an_error_at_startup_exactly_once` passes.
- [ ] JWT availability in the shared framework is **verified**; if absent, the package split is done
      and ADR-004 is updated.
- [ ] `docs/security/checklist.md` exists and every item on it is actually configurable.
- [ ] `docs/migration/0.7-to-0.8.md` exists.
- [ ] Every Limitations bullet fixed by this phase is deleted from its doc page.
- [ ] All five examples authenticate properly and demonstrate `IRealtimeAuthorizationService`.
- [ ] `scripts/verify.ps1` green; tag `v0.8.0`.

## Verify

```powershell
pwsh scripts/verify.ps1
dotnet test LiveSharp.slnx -c Release --filter "FullyQualifiedName~Security"
dotnet test tests/LiveSharp.ArchitectureTests -c Release
dotnet test tests/LiveSharp.IntegrationTests -c Release --filter "FullyQualifiedName~Jwt|FullyQualifiedName~RateLimit|FullyQualifiedName~Anonymous"
git tag v0.8.0
```

## Next

`docs/plan/phase-10-webrtc-signaling.md`, task `P10.T1`.

> Phase 10 begins the WebRTC track. It depends on this phase's `IRealtimeAuthorizationService`: an
> unauthorized signalling relay is an anonymous messaging channel and a bandwidth-amplification
> vector. Do not start Phase 10 until `P09.T5` is complete.
