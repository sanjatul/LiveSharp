# Phase 02 — Connection Infrastructure

| | |
|---|---|
| **Version at exit** | `0.1.0` (tag `v0.1.0`) |
| **Depends on** | Phase 01 |
| **Branch** | `feat/phase-02-connections` |
| **Tasks** | `P02.T1` … `P02.T6` |
| **Read before starting** | `docs/plan/phase-01-abstractions.md`, `EXECUTION_PLAN.md` §5.2 |

---

## Goal

Implement the connection lifecycle: who is connected, on which connections, since when, and what
happens when they go away — **entirely without a web server**. At the end of this phase, connection
tracking, group membership tracking, lifecycle notification, stale-connection reaping, and an
in-memory transport are all implemented and tested with `FakeTimeProvider`.

This is what makes Phase 03 a thin adapter rather than a rewrite. If Phase 03 has to add logic,
that logic belongs here.

Ship `0.1.0` at the end of this phase — the first tagged version.

---

## Non-goals — DO NOT implement in this phase

- No SignalR. No `IHubContext`. No hub. No HTTP. No `Microsoft.AspNetCore.*` reference.
- No presence semantics. Tracking that a connection exists is **not** presence. `Online`/`Away`/
  `Busy`, presence subscriptions, and change broadcasting are Phase 06.
- No messages of any kind. The in-memory transport moves `RealtimeEnvelope` values; it does not know
  what a chat message is.
- No authentication. `IUserResolver` gets a default implementation, but connection authorization,
  policies, and token handling are Phase 09.
- No distributed registry. `NodeId` stays `null`. Redis is Phase 15.
- No group *service* (create/administer/roles). Only the low-level `IGroupRegistry` membership map.
  Group semantics are Phase 05.

---

## Tasks

### P02.T1 — `InMemoryConnectionRegistry`

**Deliverables**

```
src/LiveSharp.Core/Connections/InMemoryConnectionRegistry.cs
tests/LiveSharp.Core.Tests/Connections/InMemoryConnectionRegistryTests.cs
tests/LiveSharp.Core.Tests/Connections/ConnectionRegistryConcurrencyTests.cs
```

**Implementation requirements**

Two indexes, both `ConcurrentDictionary`, kept consistent:

```
_byConnectionId : ConcurrentDictionary<string, RealtimeConnection>
_byUserId       : ConcurrentDictionary<string, ImmutableHashSet<string>>   // userId -> connectionIds
```

- `_byUserId` values are **immutable sets updated with `AddOrUpdate`**, so
  `GetUserConnectionsAsync` can return a snapshot without locking or copying under a lock. A
  `ConcurrentBag`/`List` behind a lock would serialise every fan-out.
- Ordering guarantee: `AddAsync` writes `_byConnectionId` **first**, then `_byUserId`.
  `RemoveAsync` removes from `_byUserId` **first**, then `_byConnectionId`. This means a concurrent
  reader may briefly see a connection that is not yet in the user index (harmless — a message is
  not delivered) but never a user-index entry pointing at a connection that no longer exists (which
  would produce a delivery attempt to a dead connection). Document this ordering in a code comment
  with the reason; it will look arbitrary otherwise.
- Removing the last connection for a user must remove the user's key entirely. Otherwise a busy
  server accumulates one empty set per user who ever connected — an unbounded leak. Test it.
- `AddAsync` with an existing `ConnectionId` **replaces** it and logs a warning. A transport must
  not produce duplicate IDs; if it does, silently keeping the stale record is worse.
- `RemoveAsync` returns `false` for an unknown connection ID without throwing — disconnect handlers
  legitimately race and double-remove.
- `TouchAsync` updates `LastSeenAt` via `AddOrUpdate` with `record with { LastSeenAt = now }`.
  Reading `_byConnectionId` and writing back non-atomically loses concurrent updates.
- `GetConnectionCountAsync` returns `_byConnectionId.Count`. Do **not** maintain a separate counter;
  it will drift.
- All methods complete synchronously — return `ValueTask.FromResult`/`ValueTask.CompletedTask`. No
  `async` state machines on this path.
- `MaxConnectionsPerUser` is **not** enforced here. The registry is a data structure. Enforcement
  needs to reject or disconnect, which is `P02.T5`'s `ConnectionLifecycleService`. Keeping policy
  out of the registry is what lets Phase 15 swap in a Redis implementation without reimplementing
  policy.

**Tests — `InMemoryConnectionRegistryTests`**

| Test | Asserts |
|---|---|
| `Add_then_find_returns_the_connection` | |
| `Find_unknown_connection_returns_null` | |
| `Add_duplicate_connection_id_replaces_and_warns` | log record captured |
| `Remove_returns_true_for_a_known_connection` | |
| `Remove_returns_false_for_an_unknown_connection` | no throw |
| `Remove_twice_returns_false_the_second_time` | |
| `Get_user_connections_returns_all_connections_for_that_user` | three connections |
| `Get_user_connections_returns_empty_for_an_unknown_user` | empty, not null |
| `Is_online_is_true_while_any_connection_remains` | remove 2 of 3 → still online |
| `Is_online_is_false_after_the_last_connection_is_removed` | |
| `Removing_the_last_user_connection_removes_the_user_index_entry` | reflection or an internal accessor; **required** — this is the leak test |
| `Touch_updates_last_seen_at_from_the_time_provider` | `FakeTimeProvider` |
| `Touch_on_an_unknown_connection_is_a_no_op` | no throw |
| `Touch_preserves_every_other_member` | `ConnectedAt`, `Metadata`, `UserId` |
| `Connection_count_reflects_adds_and_removes` | |
| `Returned_user_connection_list_is_a_snapshot` | mutating the registry afterwards does not change the returned list |
| `All_methods_honour_a_pre_cancelled_token` | `AssertCancellation.PropagatesAsync` for each method |

**Tests — `ConnectionRegistryConcurrencyTests`** (use `ConcurrencyHarness`)

| Test | Asserts |
|---|---|
| `Parallel_adds_of_distinct_connections_all_land` | 32 workers × 500 adds → count is 16 000 |
| `Parallel_add_and_remove_of_the_same_user_leaves_a_consistent_index` | after the storm, every ID in `_byUserId` exists in `_byConnectionId` |
| `Parallel_removes_report_exactly_one_true_per_connection` | sum of `true` results equals the connection count |
| `Parallel_touch_and_remove_does_not_resurrect_a_removed_connection` | the classic read-modify-write bug |
| `Parallel_add_remove_leaves_no_orphaned_user_entries` | after all removals, the user index is empty |
| `Concurrent_reads_during_writes_never_return_a_torn_snapshot` | every returned connection has a non-null `UserId` matching the queried user |

---

### P02.T2 — `InMemoryGroupRegistry`

**Deliverables**

```
src/LiveSharp.Core/Connections/InMemoryGroupRegistry.cs
tests/LiveSharp.Core.Tests/Connections/InMemoryGroupRegistryTests.cs
```

**Implementation requirements**

Bidirectional index, same immutable-set technique:

```
_byGroup      : ConcurrentDictionary<string, ImmutableHashSet<string>>   // group -> connectionIds
_byConnection : ConcurrentDictionary<string, ImmutableHashSet<string>>   // connectionId -> groups
```

- The reverse index exists solely so `RemoveConnectionFromAllAsync` is O(groups-for-connection)
  rather than O(all groups). Disconnect is a hot path on a server churning connections; scanning
  every group per disconnect is a quadratic trap.
- Both indexes must be updated together, and empty sets must be removed from both. Two leak tests
  required (one per index).
- Group name comparison is **case-sensitive and ordinal**. Pin it with a test. Case-insensitive
  group names look friendly and then behave inconsistently with the transport's own group semantics
  and with Redis keys in Phase 15.
- Adding a connection to a group twice is idempotent — no duplicate, no error.
- `RemoveConnectionFromAllAsync` must be safe to call for a connection in zero groups.

**Tests**

| Test | Asserts |
|---|---|
| `Add_puts_the_connection_in_the_group` | |
| `Add_twice_is_idempotent` | one member |
| `Remove_returns_true_then_false` | |
| `Get_groups_for_connection_returns_every_group` | |
| `Get_connections_in_group_returns_every_member` | |
| `Get_connections_in_an_unknown_group_returns_empty` | |
| `Remove_connection_from_all_clears_both_indexes` | |
| `Remove_connection_from_all_for_a_group_less_connection_is_a_no_op` | |
| `Empty_group_key_is_removed_after_the_last_member_leaves` | leak test 1 |
| `Empty_connection_key_is_removed_after_the_last_group_leaves` | leak test 2 |
| `Group_names_are_case_sensitive` | `Team` ≠ `team` |
| `Parallel_join_and_leave_leaves_both_indexes_mutually_consistent` | `ConcurrencyHarness`; every edge appears in both directions or neither |
| `All_methods_honour_a_pre_cancelled_token` | |

---

### P02.T3 — `DefaultUserResolver`

**Deliverables**

```
src/LiveSharp.Core/Connections/DefaultUserResolver.cs
tests/LiveSharp.Core.Tests/Connections/DefaultUserResolverTests.cs
```

**Implementation requirements**

Resolve in this order, first non-empty wins:

1. `ClaimTypes.NameIdentifier` (`http://schemas.xmlsoap.org/ws/2005/05/identity/claims/nameidentifier`)
2. `"sub"` — raw JWT subject, present when `MapInboundClaims = false`
3. `ClaimTypes.Name`
4. `principal.Identity?.Name`

Return `null` if none resolve or if `principal.Identity?.IsAuthenticated != true`.

- Add a `ClaimTypeUserResolver(string claimType)` alongside it so a consumer with a custom claim
  needs one line, not a class. Registered via
  `AddLiveSharp().UseUserIdClaim("tenant_user_id")`.
- Trim whitespace and treat whitespace-only as absent. A user ID of `" "` will produce
  hard-to-diagnose routing failures.
- Do **not** log the resolved user ID above `Debug`, and never log the principal's claims.

**Tests**

| Test | Asserts |
|---|---|
| `Resolves_name_identifier_claim` | |
| `Falls_back_to_sub_when_name_identifier_is_absent` | |
| `Falls_back_to_name_claim` | |
| `Falls_back_to_identity_name` | |
| `Returns_null_for_an_unauthenticated_principal` | even when claims are present |
| `Returns_null_when_no_candidate_claim_exists` | |
| `Returns_null_for_a_whitespace_only_claim_value` | |
| `Trims_surrounding_whitespace` | |
| `Prefers_name_identifier_over_sub_when_both_exist` | ordering pinned |
| `Claim_type_resolver_uses_the_configured_claim` | |
| `Claim_type_resolver_rejects_a_null_or_empty_claim_type_at_construction` | |
| `Does_not_log_claim_values_above_debug` | `CapturingLoggerProvider` |

---

### P02.T4 — `InMemoryTransport`

**Deliverables**

```
src/LiveSharp.Core/Transport/InMemoryTransport.cs
src/LiveSharp.Core/Transport/InMemoryTransportSink.cs
tests/LiveSharp.Core.Tests/Transport/InMemoryTransportTests.cs
```

This is not a toy. It is the implementation that proves `IRealtimeTransport` is a genuine seam
(deviation D1's justification depends on it), and it is what every feature test from Phase 04
onward uses to assert "the right envelope went to the right people" without a web host.

**Implementation requirements**

```csharp
namespace LiveSharp.Transport;

/// <summary>Records deliveries in memory. Intended for tests and in-process scenarios.</summary>
public sealed class InMemoryTransport : IRealtimeTransport
{
    public InMemoryTransport(IConnectionRegistry connections, IGroupRegistry groups, TimeProvider timeProvider, ILogger<InMemoryTransport> logger);

    /// <summary>Every delivery in order, one entry per receiving connection.</summary>
    public IReadOnlyList<RecordedDelivery> Deliveries { get; }

    /// <summary>Connections asked to disconnect, with the supplied reason.</summary>
    public IReadOnlyList<RecordedDisconnect> Disconnects { get; }

    public void Clear();

    /// <summary>Waits until at least <paramref name="count"/> deliveries have been recorded.</summary>
    public Task WaitForDeliveriesAsync(int count, TimeSpan timeout, CancellationToken cancellationToken = default);
}

public sealed record RecordedDelivery(string ConnectionId, string UserId, RealtimeEnvelope Envelope, DateTimeOffset At);
public sealed record RecordedDisconnect(string ConnectionId, string? Reason, DateTimeOffset At);
```

- Fan-out resolves through `IConnectionRegistry`/`IGroupRegistry`, so tests exercise the same
  resolution path production uses. One `RecordedDelivery` per receiving connection — a user with
  three connections yields three entries. This is exactly the multi-device behaviour that later
  phases must get right, so make it visible from the start.
- `SendToUserAsync` for an offline user records nothing and does **not** throw. Offline handling is
  a feature-level decision (Phase 04's `OfflineMessageBehavior`), not a transport error.
- `SendToGroupExceptAsync` must exclude by **connection** ID, not user ID. Test that a sender's
  other connections still receive.
- `Deliveries` and `Disconnects` are append-only and thread-safe. Return snapshots, not live views —
  a test enumerating while a background fan-out appends must not throw.
- `WaitForDeliveriesAsync` uses a `TaskCompletionSource` signalled on append, with a real timeout
  (not `FakeTimeProvider`, since tests await it on the real clock). Without this every downstream
  test degenerates into `Task.Delay(100)` and the suite becomes flaky.
- `Clear()` resets both lists and any pending waiters.

**Tests**

| Test | Asserts |
|---|---|
| `Send_to_connection_records_one_delivery` | |
| `Send_to_an_unknown_connection_records_nothing_and_does_not_throw` | |
| `Send_to_user_records_one_delivery_per_connection` | 3 connections → 3 deliveries |
| `Send_to_an_offline_user_records_nothing_and_does_not_throw` | |
| `Send_to_users_deduplicates_repeated_user_ids` | same user twice → one delivery per connection |
| `Send_to_group_records_one_delivery_per_member_connection` | |
| `Send_to_group_except_excludes_only_the_named_connections` | sender's second connection still receives |
| `Send_to_an_empty_group_records_nothing` | |
| `Disconnect_records_the_connection_and_reason` | |
| `Deliveries_are_recorded_in_send_order` | |
| `Deliveries_returns_a_snapshot_safe_to_enumerate_during_concurrent_sends` | `ConcurrencyHarness` |
| `Clear_resets_deliveries_and_disconnects` | |
| `Wait_for_deliveries_completes_when_the_count_is_reached` | |
| `Wait_for_deliveries_throws_TimeoutException_with_the_observed_count_in_the_message` | diagnosability |
| `Delivery_timestamp_comes_from_the_time_provider` | `FakeTimeProvider` |
| `Parallel_sends_record_every_delivery_exactly_once` | 32 × 200 sends → 6 400 entries |
| `All_methods_honour_a_pre_cancelled_token` | |

---

### P02.T5 — Connection lifecycle service and hooks

**Deliverables**

```
src/LiveSharp.Abstractions/IConnectionLifecycleHandler.cs
src/LiveSharp.Core/Connections/IConnectionLifecycleService.cs
src/LiveSharp.Core/Connections/ConnectionLifecycleService.cs
src/LiveSharp.Core/Connections/ConnectionRejectedReason.cs
tests/LiveSharp.Core.Tests/Connections/ConnectionLifecycleServiceTests.cs
```

This is the orchestrator Phase 03's hub calls into. **All policy lives here**, so the hub stays a
five-line adapter.

**Public API (exact):**

```csharp
namespace LiveSharp;

/// <summary>Notified when a connection is established or torn down. Implement to react to presence-adjacent events.</summary>
public interface IConnectionLifecycleHandler
{
    ValueTask OnConnectedAsync(IRealtimeConnection connection, CancellationToken cancellationToken = default);
    ValueTask OnDisconnectedAsync(IRealtimeConnection connection, DisconnectReason reason, CancellationToken cancellationToken = default);
}

public enum DisconnectReason
{
    ClientClosed = 0,
    ServerClosed = 1,
    TransportError = 2,
    IdleTimeout = 3,
    ConnectionLimit = 4,
    Shutdown = 5,
}
```

```csharp
namespace LiveSharp.Connections;

public interface IConnectionLifecycleService
{
    ValueTask<OperationResult<RealtimeConnection>> ConnectAsync(ConnectionRequest request, CancellationToken cancellationToken = default);
    ValueTask DisconnectAsync(string connectionId, DisconnectReason reason, CancellationToken cancellationToken = default);
}

public sealed record ConnectionRequest
{
    public required string ConnectionId { get; init; }
    public required string UserId { get; init; }
    public string? NodeId { get; init; }
    public IReadOnlyDictionary<string, string>? Metadata { get; init; }
}
```

**`ConnectAsync` sequence — implement exactly in this order:**

1. Validate `ConnectionId`/`UserId` non-empty → `Failure(InvalidPayload, …)`.
2. Read existing connections for the user.
3. If at `MaxConnectionsPerUser`:
   - `RejectNewest` → return `Failure(Conflict, "…")` **without** registering. Log at `Information`
     with the user's connection count (not the user ID above `Debug`).
   - `DisconnectOldest` → pick the connection with the smallest `ConnectedAt`, call
     `DisconnectAsync(oldest, DisconnectReason.ConnectionLimit)`, then continue.
4. Register in `IConnectionRegistry` with `ConnectedAt = LastSeenAt = timeProvider.GetUtcNow()`.
5. Invoke every `IConnectionLifecycleHandler.OnConnectedAsync`.
6. Return `Success(connection)`.

**`DisconnectAsync` sequence:**

1. `FindAsync`; if absent, return silently (idempotent — double-disconnect is normal).
2. `IGroupRegistry.RemoveConnectionFromAllAsync`.
3. `IConnectionRegistry.RemoveAsync`.
4. Invoke every `IConnectionLifecycleHandler.OnDisconnectedAsync`.

**Handler invocation rules — this is where real systems break:**

- Handlers are invoked **sequentially in registration order**, not concurrently. Phase 06's presence
  handler and Phase 05's group handler have order-sensitive expectations, and parallel invocation
  makes failures non-deterministic.
- A handler that **throws** is logged at `Error` and **does not prevent the remaining handlers from
  running**, and does not fail `ConnectAsync`/`DisconnectAsync`. A misbehaving third-party handler
  must not be able to leak connections by aborting cleanup. Aggregate nothing; log each.
- Registry cleanup happens **before** disconnect handlers run, so a handler that queries "is this
  user still online?" gets the post-disconnect answer. Document this; it is load-bearing for
  Phase 06.
- Handlers receive `CancellationToken.None` for the **disconnect** path. Cleanup must complete even
  when the connection's token is already cancelled — which it always is on a real disconnect. Getting
  this wrong means presence never clears. Test it explicitly.

**Tests**

| Test | Asserts |
|---|---|
| `Connect_registers_the_connection` | |
| `Connect_stamps_connected_at_and_last_seen_at_from_the_time_provider` | equal on connect |
| `Connect_rejects_an_empty_connection_id` / `…_user_id` | error code |
| `Connect_invokes_lifecycle_handlers_in_registration_order` | order recorded |
| `Connect_at_the_limit_with_RejectNewest_fails_and_does_not_register` | count unchanged |
| `Connect_at_the_limit_with_DisconnectOldest_disconnects_the_oldest_and_registers` | oldest chosen by `ConnectedAt` |
| `Connect_below_the_limit_succeeds` | boundary: limit − 1 |
| `Connect_exactly_at_the_limit_minus_one_succeeds_and_at_the_limit_fails` | off-by-one guard |
| `Disconnect_removes_the_connection` | |
| `Disconnect_removes_the_connection_from_every_group` | |
| `Disconnect_invokes_handlers_after_registry_cleanup` | handler observes `IsOnlineAsync == false` |
| `Disconnect_of_an_unknown_connection_is_a_silent_no_op` | no handler invocation, no throw |
| `Disconnect_twice_invokes_handlers_once` | idempotency |
| `Disconnect_passes_the_supplied_reason_to_handlers` | table-driven over every `DisconnectReason` |
| `A_throwing_connect_handler_does_not_fail_the_connection` | result still `Succeeded` |
| `A_throwing_connect_handler_does_not_prevent_later_handlers` | handler 3 ran after handler 2 threw |
| `A_throwing_disconnect_handler_does_not_prevent_registry_cleanup` | connection gone |
| `A_throwing_handler_is_logged_at_error_with_the_handler_type_name` | diagnosability |
| `Disconnect_handlers_run_even_when_the_ambient_token_is_cancelled` | **critical** |
| `Concurrent_connects_for_one_user_never_exceed_the_limit` | `ConcurrencyHarness`, 32 workers racing on a limit of 5 |
| `Concurrent_connect_and_disconnect_leaves_a_consistent_registry` | |

> `Concurrent_connects_for_one_user_never_exceed_the_limit` will fail on a naive
> read-then-write implementation. Fix it with a per-user gate (e.g. a striped
> `SemaphoreSlim` keyed by user ID, or a `ConcurrentDictionary` compare-and-swap loop) — **not** a
> global lock, which would serialise all connects on the server. Document the chosen approach.

---

### P02.T6 — Stale connection reaper

**Deliverables**

```
src/LiveSharp.Core/Connections/ConnectionReaperService.cs
tests/LiveSharp.Core.Tests/Connections/ConnectionReaperServiceTests.cs
```

**Implementation requirements**

- `BackgroundService` using `timeProvider.CreateTimer` (or a `PeriodicTimer` created from the
  `TimeProvider`) with period `ConnectionOptions.ReaperInterval`. **Never** `Task.Delay` on the
  system clock — the whole test suite for this class depends on `FakeTimeProvider`.
- Each tick: enumerate connections, select those where
  `now - LastSeenAt > ConnectionOptions.IdleTimeout`, and for each call
  `IConnectionLifecycleService.DisconnectAsync(id, DisconnectReason.IdleTimeout)`.
- One slow or throwing disconnect must not abort the sweep or kill the background service.
  Wrap each per-connection disconnect in try/catch, log at `Warning`, continue.
- Log a single summary per tick at `Debug` (`scanned`, `reaped`, `elapsed`) and at `Information`
  only when `reaped > 0`. A reaper logging every tick at `Information` is noise that trains
  operators to ignore the log.
- Cap the work per tick (`MaxReapsPerTick`, internal const, e.g. 1 000) so a pathological state
  cannot make one tick run for minutes. Log at `Warning` when the cap is hit.
- Respect `stoppingToken`: stop promptly, and do **not** treat shutdown cancellation as an error.
- The service must be resilient to `IConnectionRegistry` throwing (relevant once Phase 15 puts
  Redis behind it): catch, log, and try again next tick rather than crashing the host.

**Tests**

| Test | Asserts |
|---|---|
| `Does_not_reap_a_fresh_connection` | advance less than the timeout |
| `Reaps_a_connection_past_the_idle_timeout` | advance past it |
| `Does_not_reap_at_exactly_the_idle_timeout` | boundary is strictly greater-than |
| `Reaping_uses_the_IdleTimeout_disconnect_reason` | handler observes the reason |
| `Touching_a_connection_prevents_reaping` | touch mid-way, advance, still present |
| `Reaps_multiple_stale_connections_in_one_tick` | |
| `A_throwing_disconnect_does_not_stop_the_sweep` | remaining stale connections still reaped |
| `A_throwing_registry_does_not_stop_the_service` | next tick still runs |
| `Reaps_nothing_when_the_registry_is_empty` | no logs above debug |
| `Logs_at_information_only_when_something_was_reaped` | `CapturingLoggerProvider` |
| `Honours_the_max_reaps_per_tick_cap_and_warns` | 1 001 stale connections → 1 000 reaped, warning logged |
| `Stops_promptly_on_the_stopping_token` | `StopAsync` completes within a real 1 s |
| `Shutdown_cancellation_is_not_logged_as_an_error` | |
| `Uses_the_configured_reaper_interval` | two ticks over two intervals |

---

## Dependency justification

No new packages. `Microsoft.Extensions.Hosting.Abstractions` (added in Phase 01) provides
`BackgroundService`. `FakeTimeProvider` (Phase 01 test dependency) provides deterministic time.

If `Microsoft.Extensions.Diagnostics.Abstractions` from Phase 01 is still unused at the end of this
phase, **remove it** and reintroduce it in Phase 16.

---

## Documentation deltas

- `docs/architecture/connection-lifecycle.md` — **new**. The connect and disconnect sequences as
  numbered steps, the handler ordering and error-isolation contract, the registry index-ordering
  rationale, and the "cleanup before handlers" guarantee. This page is the reference every later
  phase cites.
- `docs/configuration/README.md` — document `Connections.*` options with defaults, ranges, and the
  operational meaning of `IdleTimeout` vs `ReaperInterval`.
- `docs/configuration/service-lifetimes.md` — add the registries (singleton), the lifecycle service
  (singleton), the reaper (hosted service), the transport (singleton).
- `docs/troubleshooting/README.md` — add "users appear online after disconnecting" and "connections
  are dropped unexpectedly" with the option and log-event pointers.
- `README.md` — flip the feature-status table: connection management → `Preview`.

## Example deltas

None. Examples begin in Phase 04.

## CHANGELOG entry

```markdown
### Added
- In-memory connection registry with per-user indexing and multi-connection support.
- In-memory group membership registry with a reverse index for O(1) disconnect cleanup.
- Connection lifecycle service enforcing per-user connection limits with configurable behaviour.
- `IConnectionLifecycleHandler` extension point with error isolation between handlers.
- Background reaper that disconnects idle connections, driven by `TimeProvider`.
- Default claims-based user resolver with a configurable claim type.
- In-memory transport for testing and in-process scenarios.
```

---

## Exit criteria

- [ ] `LiveSharp.Core` still has no `Microsoft.AspNetCore.*` reference (architecture test green).
- [ ] Every registry has a passing parallel-stress test and a passing empty-key leak test.
- [ ] `Concurrent_connects_for_one_user_never_exceed_the_limit` passes without a global lock.
- [ ] `Disconnect_handlers_run_even_when_the_ambient_token_is_cancelled` passes.
- [ ] The reaper is fully tested with `FakeTimeProvider`; **zero** `Task.Delay` on the system clock
      anywhere in `src/`.
- [ ] `docs/architecture/connection-lifecycle.md` exists and matches the implementation.
- [ ] `PublicAPI.Unshipped.txt` updated for both projects.
- [ ] `scripts/verify.ps1` green on Windows and Linux.
- [ ] Tag `v0.1.0`; MinVer produces `0.1.0`; `dotnet pack` yields
      `LiveSharp.Abstractions.0.1.0.nupkg` and `LiveSharp.Core.0.1.0.nupkg` with XML docs, README,
      licence expression, and a `.snupkg`.
- [ ] `CHANGELOG.md` has a real `## [0.1.0] - <date>` section, not just `Unreleased`.

## Verify

```powershell
pwsh scripts/verify.ps1 -SkipJs -SkipE2E
dotnet test LiveSharp.slnx -c Release --filter "FullyQualifiedName~Concurrency"
git tag v0.1.0
dotnet pack LiveSharp.slnx -c Release -o artifacts/packages
Get-ChildItem artifacts/packages
```

## Next

`docs/plan/phase-03-signalr-transport.md`, task `P03.T1`.

> **Reminder:** Phase 03 ends at the mandatory human review gate (`EXECUTION_PLAN.md` §9). The wire
> protocol is frozen there.
