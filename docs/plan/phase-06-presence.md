# Phase 06 — Presence

| | |
|---|---|
| **Version at exit** | `0.5.0` (tag `v0.5.0`) |
| **Depends on** | Phase 05 |
| **Branch** | `feat/phase-06-presence` |
| **Tasks** | `P06.T1` … `P06.T6` |
| **Read before starting** | `docs/architecture/connection-lifecycle.md`, `docs/plan/phase-05-groups.md`, `EXECUTION_PLAN.md` §3 |

---

## Goal

Answer "who is available, and how do I find out when that changes" — with correct aggregation across
a user's devices, bounded broadcast cost, TTL-based expiry for half-open connections, and a
subscription model that does not leak presence to people who should not see it.

Phase 02 already tracks *connections*. Presence is not connection tracking: a user with three
connections is one presence; a user who set themselves to Busy is Busy even though their socket is
live; a user whose laptop lid closed is Online until a heartbeat expires.

---

## Package decision: a new `LiveSharp.Presence` package

Presence is required by Phase 11 (calling needs to know whether the callee is reachable and whether
they are already Busy) and is genuinely useful in applications with no chat at all — a collaborative
editor, a dashboard, a support-agent availability board.

Putting presence in `LiveSharp.Chat` would force `LiveSharp.Calls` to depend on chat, which is
exactly the coupling Spec §7 forbids and Spec §12 explicitly calls out ("Calling must be
architecturally separated from chat").

So: **`LiveSharp.Presence` depends on `Abstractions` + `Core` + `AspNetCore` only.** It must not
reference `LiveSharp.Chat`. The default subscription source needs group membership, which lives in
chat — resolve that with an optional integration (`P06.T3`), not a package reference. Add an
architecture test asserting `LiveSharp.Presence` has no reference to `LiveSharp.Chat`.

Write this up as `docs/architecture/decisions/ADR-012-presence-package.md` (Context / Decision /
Alternatives / Consequences). The negative consequence to state honestly: a consumer wanting chat +
presence installs two packages and calls two builder methods.

---

## Non-goals — DO NOT implement in this phase

- No typing indicators. Typing is per-conversation and ephemeral; presence is per-user and stateful.
  Phase 07.
- No receipts. Phase 08.
- No authorization policies. `IPresenceSubscriptionSource` is the *scoping* boundary and it is real,
  but there is no policy engine and no `IRealtimeAuthorizationService` yet. Phase 09.
- No distributed presence. Single node only; `NodeId` stays `null`. Phase 15.
- No presence history, session logs, "last active 3 days ago" analytics, or aggregate reporting.
- No custom user-defined presence state machines or arbitrary status enums. Five states, fixed.
- No persistence provider. In-memory only; Phase 14 decides whether presence is even worth persisting
  (it probably is not — presence is derived state).

---

## Tasks

### P06.T1 — Package, model, store

**Deliverables**

```
src/LiveSharp.Presence/LiveSharp.Presence.csproj
src/LiveSharp.Presence/PublicAPI.{Shipped,Unshipped}.txt
src/LiveSharp.Presence/AssemblyInfo.cs
src/LiveSharp.Presence/PresenceStatus.cs
src/LiveSharp.Presence/UserPresence.cs
src/LiveSharp.Presence/PresenceSource.cs
src/LiveSharp.Presence/Storage/IPresenceStore.cs
src/LiveSharp.Presence/Storage/InMemoryPresenceStore.cs
src/LiveSharp.Presence/Configuration/PresenceOptions.cs
src/LiveSharp.Presence/Configuration/PresenceOptionsValidator.cs
src/LiveSharp.Presence/Diagnostics/PresenceLog.cs
tests/LiveSharp.Presence.Tests/…
tests/LiveSharp.Presence.Tests/Storage/PresenceStoreContractTests.cs
```

**Public API (exact):**

```csharp
namespace LiveSharp.Presence;

public enum PresenceStatus
{
    Offline      = 0,
    Online       = 1,
    Away         = 2,
    Busy         = 3,
    DoNotDisturb = 4,
}

/// <summary>How a status was arrived at. Determines precedence when devices disagree.</summary>
public enum PresenceSource
{
    /// <summary>Derived from connection state or inactivity. Lowest precedence.</summary>
    Derived  = 0,

    /// <summary>Set explicitly by the user on one of their devices. Wins over derived.</summary>
    Explicit = 1,
}

/// <summary>Aggregated presence for one user across all of their connections.</summary>
public sealed record UserPresence
{
    public required string UserId { get; init; }
    public required PresenceStatus Status { get; init; }
    public required PresenceSource Source { get; init; }

    /// <summary>Optional free-text status. Length-limited. Never logged above Trace.</summary>
    public string? StatusMessage { get; init; }

    /// <summary>When any of this user's connections was last active.</summary>
    public required DateTimeOffset LastSeenAt { get; init; }

    /// <summary>When the current status was set or derived.</summary>
    public required DateTimeOffset UpdatedAt { get; init; }

    /// <summary>Number of live connections. Zero implies Offline.</summary>
    public required int ConnectionCount { get; init; }

    public static UserPresence Offline(string userId, DateTimeOffset at);
}
```

```csharp
namespace LiveSharp.Presence.Storage;

public interface IPresenceStore
{
    ValueTask<UserPresence?> FindAsync(string userId, CancellationToken cancellationToken = default);
    ValueTask<IReadOnlyDictionary<string, UserPresence>> FindManyAsync(IReadOnlyCollection<string> userIds, CancellationToken cancellationToken = default);

    /// <summary>Writes presence only if <paramref name="expectedUpdatedAt"/> matches the stored value.
    /// Returns false on a mismatch so the caller can re-aggregate.</summary>
    ValueTask<bool> TrySetAsync(UserPresence presence, DateTimeOffset? expectedUpdatedAt, CancellationToken cancellationToken = default);

    ValueTask<bool> RemoveAsync(string userId, CancellationToken cancellationToken = default);

    /// <summary>Users whose <see cref="UserPresence.LastSeenAt"/> is older than <paramref name="olderThan"/>.</summary>
    ValueTask<IReadOnlyList<string>> GetStaleAsync(DateTimeOffset olderThan, int limit, CancellationToken cancellationToken = default);
}
```

**Public API (exact) — options:**

```csharp
public sealed class PresenceOptions
{
    public const string ConfigurationSectionName = "LiveSharp:Presence";

    /// <summary>How often clients should heartbeat. Advertised to clients on connect. Default 30s.</summary>
    public TimeSpan HeartbeatInterval { get; set; } = TimeSpan.FromSeconds(30);

    /// <summary>Presence not refreshed within this window expires to Offline. Default 90s.</summary>
    public TimeSpan PresenceTtl { get; set; } = TimeSpan.FromSeconds(90);

    /// <summary>Derive Away after this much inactivity. Null disables automatic Away. Default 5 minutes.</summary>
    public TimeSpan? AwayAfter { get; set; } = TimeSpan.FromMinutes(5);

    /// <summary>Suppress repeat broadcasts for the same user within this window. Default 2s.</summary>
    public TimeSpan ChangeCoalescingWindow { get; set; } = TimeSpan.FromSeconds(2);

    /// <summary>How often the expiry sweep runs. Default 15s.</summary>
    public TimeSpan SweepInterval { get; set; } = TimeSpan.FromSeconds(15);

    /// <summary>Maximum explicit subscriptions one user may hold. Default 512.</summary>
    public int MaxSubscriptionsPerUser { get; set; } = 512;

    /// <summary>Maximum recipients for one presence broadcast. Above this, the change is not fanned out. Default 1000.</summary>
    public int BroadcastFanoutLimit { get; set; } = 1000;

    /// <summary>Maximum status message length. Default 140.</summary>
    public int MaxStatusMessageLength { get; set; } = 140;

    /// <summary>Maximum users in one presence.query request. Default 100.</summary>
    public int MaxQueryBatchSize { get; set; } = 100;
}
```

**Requirements**

- `PresenceStatus` values are explicitly numbered because they cross the wire and are persisted in
  Phase 14. Never reorder. `Offline = 0` so `default` is the safe answer.
- `TrySetAsync` takes an optimistic-concurrency token (`expectedUpdatedAt`) instead of a blind write.
  Presence is written from three concurrent sources — connect handlers, heartbeats, and the expiry
  sweep — and last-write-wins produces a user who flickers Offline while holding a live connection.
  The caller retries the aggregate on `false`. Cap retries (3) and log a `Warning` on exhaustion.
- `GetStaleAsync` takes a `limit` so the sweep cannot pull an unbounded set. Phase 14/15 stores need
  this to become a bounded query.
- `FindManyAsync` returns a dictionary containing an entry for **every** requested user, synthesising
  `UserPresence.Offline` for unknown ones. A partial result forces every caller to write the same
  null-coalescing loop, and half of them will get it wrong.
- Validator rules: `PresenceTtl > HeartbeatInterval` (a TTL shorter than the heartbeat expires
  everyone — actionable message required, this is the misconfiguration people actually make);
  `SweepInterval < PresenceTtl`; `AwayAfter > HeartbeatInterval` when set;
  `ChangeCoalescingWindow >= TimeSpan.Zero`; all counts positive; `MaxQueryBatchSize <= 1000`.
- `InMemoryPresenceStore`: single `ConcurrentDictionary<string, UserPresence>`, `TrySetAsync` via
  `TryUpdate`/`TryAdd` compare-and-swap on `UpdatedAt`. Documented as development-only and
  single-node.
- `PresenceStoreContractTests` abstract suite, same pattern as Phases 04 and 05.

**Tests**

| Test | Asserts |
|---|---|
| `Offline_factory_produces_zero_connections_and_offline_status` | |
| `Presence_status_numeric_values_are_pinned` | wire contract |
| `Find_unknown_user_returns_null` | |
| `Find_many_returns_an_entry_for_every_requested_user` | unknown → `Offline` |
| `Find_many_with_an_empty_request_returns_empty` | |
| `Try_set_with_a_matching_token_succeeds` | |
| `Try_set_with_a_stale_token_fails_and_does_not_write` | **the concurrency primitive** |
| `Try_set_with_a_null_token_creates_a_new_entry` | |
| `Try_set_with_a_null_token_on_an_existing_entry_fails` | prevents accidental blind overwrite |
| `Remove_returns_true_then_false` | |
| `Get_stale_returns_only_users_older_than_the_cutoff` | boundary: exactly at the cutoff is not stale |
| `Get_stale_honours_the_limit` | |
| `Concurrent_try_set_calls_result_in_exactly_one_winner_per_token` | `ConcurrencyHarness` |
| `All_methods_honour_a_pre_cancelled_token` | |
| `Presence_options_defaults_are_valid` | |
| `Presence_ttl_not_greater_than_the_heartbeat_interval_is_rejected_with_guidance` | |
| `Sweep_interval_not_less_than_the_ttl_is_rejected` | |
| `Away_after_less_than_the_heartbeat_interval_is_rejected` | |

---

### P06.T2 — `IPresenceService`: aggregation and status precedence

**Deliverables**

```
src/LiveSharp.Presence/IPresenceService.cs
src/LiveSharp.Presence/PresenceService.cs
src/LiveSharp.Presence/PresenceAggregator.cs
tests/LiveSharp.Presence.Tests/PresenceServiceTests.cs
tests/LiveSharp.Presence.Tests/PresenceAggregatorTests.cs
```

**Public API (exact):**

```csharp
namespace LiveSharp.Presence;

public interface IPresenceService
{
    Task<UserPresence> GetAsync(string userId, CancellationToken cancellationToken = default);
    Task<IReadOnlyDictionary<string, UserPresence>> GetManyAsync(IReadOnlyCollection<string> userIds, CancellationToken cancellationToken = default);

    /// <summary>Sets an explicit status for the user. Applies to the user, not the connection.</summary>
    Task<OperationResult<UserPresence>> SetStatusAsync(string userId, PresenceStatus status, string? statusMessage = null, CancellationToken cancellationToken = default);

    /// <summary>Clears an explicit status, returning the user to derived presence.</summary>
    Task<OperationResult<UserPresence>> ClearStatusAsync(string userId, CancellationToken cancellationToken = default);

    /// <summary>Refreshes liveness for one connection. Called by the heartbeat operation.</summary>
    Task HeartbeatAsync(string userId, string connectionId, CancellationToken cancellationToken = default);
}
```

**Aggregation rules — the whole point of this task. Implement exactly and encode each as a test.**

`PresenceAggregator.Aggregate(existing, connectionCount, lastSeenAt, now, options)` is a **pure
function** with no I/O. Making it pure is what makes the 20-odd rule combinations testable without
a host.

1. `connectionCount == 0` → `Offline`, `Source = Derived`, `StatusMessage = null`. An offline user
   has no status message; leaving a stale "In a meeting" on an offline user is a bug users notice.
2. `connectionCount > 0` and an `Explicit` status exists that is **not** `Offline` → keep it.
   Explicit beats derived. A user who set Busy stays Busy while connected.
3. `connectionCount > 0` and an `Explicit` status of `Offline` exists → this is "appear offline".
   Honour it: report `Offline` with `Source = Explicit`. **But** `ConnectionCount` must still reflect
   reality internally, because Phase 11 calling needs to know the user is actually reachable.
   Expose `IsReachable` derived from `ConnectionCount > 0` as a computed property and document that
   `Status == Offline` does not imply unreachable. This distinction is the one that will bite Phase 11
   if skipped.
4. `connectionCount > 0`, no explicit status, `AwayAfter` set, `now - lastSeenAt > AwayAfter` →
   `Away`, `Derived`.
5. `connectionCount > 0`, no explicit status, otherwise → `Online`, `Derived`.
6. Between two devices setting conflicting explicit statuses, **most recently set wins** —
   compare `UpdatedAt`. Document why: any other rule (highest severity, first wins) surprises the
   user who just changed it on the device in their hand.
7. An explicit status **survives a reconnect** as long as presence has not expired via TTL. It is
   cleared by `ClearStatusAsync`, by TTL expiry, or by the last connection closing.

**Service requirements**

- `GetAsync` reads the store, then reconciles against `IConnectionRegistry` live counts before
  returning. Presence derived from a store that a crashed node never updated is stale; reconciling on
  read is cheap on one node and is the correct single-node answer. Note in the code that Phase 15
  replaces this with a distributed count.
- `SetStatusAsync` validates `statusMessage` length and rejects `PresenceStatus` values outside the
  enum (a wire client can send `99`) → `InvalidPayload`.
- Every mutation goes through the `TrySetAsync` retry loop; on 3 failures return
  `Failure(Conflict, …)` and log a `Warning`.
- All mutations that change the aggregate call `IPresenceBroadcaster.PublishAsync` (`P06.T4`).
  Mutations that do **not** change the aggregate (a heartbeat from an already-Online user) must not
  broadcast. Compare the pre- and post-aggregate and short-circuit. Without this, heartbeats alone
  generate a broadcast per user per interval — an O(users × subscribers) background load for zero
  information.
- `HeartbeatAsync` also calls `IConnectionRegistry.TouchAsync`, so there is one liveness clock rather
  than two that can disagree.

**Tests — `PresenceAggregatorTests`** (pure, table-driven, fast)

| Test | Asserts |
|---|---|
| `Zero_connections_yields_offline_derived` | |
| `Zero_connections_clears_the_status_message` | |
| `One_connection_with_no_explicit_status_yields_online` | |
| `Explicit_busy_survives_while_connected` | |
| `Explicit_status_beats_derived_away` | inactive but explicitly Online → Online |
| `Explicit_offline_while_connected_reports_offline_but_remains_reachable` | **rule 3** |
| `Inactivity_beyond_away_after_yields_away` | |
| `Inactivity_exactly_at_away_after_does_not_yield_away` | boundary |
| `Null_away_after_never_yields_away` | |
| `Most_recently_set_explicit_status_wins` | two devices |
| `Explicit_status_survives_a_reconnect` | count 1 → 0 → 1 within TTL |
| `Explicit_status_is_cleared_by_the_last_disconnect` | |
| `Aggregate_is_a_pure_function` | same inputs → same output, no store touched |

**Tests — `PresenceServiceTests`**

| Test | Asserts |
|---|---|
| `Get_for_an_unknown_user_returns_offline` | |
| `Get_reconciles_against_the_live_connection_count` | store says Online, registry says 0 → Offline |
| `Get_many_returns_an_entry_per_requested_user` | |
| `Get_many_rejects_a_batch_above_the_limit` | `InvalidPayload` |
| `Set_status_updates_and_broadcasts` | |
| `Set_status_rejects_an_out_of_range_enum_value` | wire-safety |
| `Set_status_rejects_an_over_long_status_message` | |
| `Set_status_never_logs_the_status_message_above_trace` | **security test** |
| `Clear_status_returns_the_user_to_derived_presence` | |
| `Heartbeat_touches_the_connection_registry` | |
| `Heartbeat_that_does_not_change_the_aggregate_does_not_broadcast` | **the broadcast-storm test** |
| `Heartbeat_that_changes_away_to_online_does_broadcast` | |
| `Concurrent_status_writes_converge_on_the_last_one` | `ConcurrencyHarness` |
| `Store_conflict_is_retried_and_eventually_succeeds` | substituted store fails twice |
| `Store_conflict_exhausted_returns_conflict_and_warns` | 3 failures |
| `All_methods_honour_a_pre_cancelled_token` | |

---

### P06.T3 — Subscriptions and scoping

**Deliverables**

```
src/LiveSharp.Presence/Subscriptions/IPresenceSubscriptionSource.cs
src/LiveSharp.Presence/Subscriptions/ExplicitPresenceSubscriptionSource.cs
src/LiveSharp.Presence/Subscriptions/CompositePresenceSubscriptionSource.cs
src/LiveSharp.Presence/Subscriptions/IPresenceSubscriptionStore.cs
src/LiveSharp.Presence/Subscriptions/InMemoryPresenceSubscriptionStore.cs
src/LiveSharp.Chat.Presence/…                      (see integration note)
tests/LiveSharp.Presence.Tests/Subscriptions/…
```

**Public API (exact):**

```csharp
namespace LiveSharp.Presence;

/// <summary>Determines who is entitled to be told about a user's presence changes.</summary>
public interface IPresenceSubscriptionSource
{
    /// <summary>User identifiers that should receive presence changes for <paramref name="subjectUserId"/>.</summary>
    ValueTask<IReadOnlyCollection<string>> GetObserversAsync(string subjectUserId, CancellationToken cancellationToken = default);

    /// <summary>Whether <paramref name="observerUserId"/> may read <paramref name="subjectUserId"/>'s presence.</summary>
    ValueTask<bool> CanObserveAsync(string observerUserId, string subjectUserId, CancellationToken cancellationToken = default);
}

public interface IPresenceSubscriptionStore
{
    ValueTask<bool> AddAsync(string observerUserId, string subjectUserId, CancellationToken cancellationToken = default);
    ValueTask<bool> RemoveAsync(string observerUserId, string subjectUserId, CancellationToken cancellationToken = default);
    ValueTask RemoveAllForObserverAsync(string observerUserId, CancellationToken cancellationToken = default);
    ValueTask<IReadOnlyList<string>> GetSubjectsAsync(string observerUserId, CancellationToken cancellationToken = default);
    ValueTask<IReadOnlyList<string>> GetObserversAsync(string subjectUserId, CancellationToken cancellationToken = default);
    ValueTask<int> GetSubjectCountAsync(string observerUserId, CancellationToken cancellationToken = default);
}
```

> ## This is a security boundary, not a convenience
> `GetObserversAsync` decides who receives a broadcast and `CanObserveAsync` decides who may query.
> A default that returns "everyone" turns presence into an org-chart scraper: any authenticated user
> could enumerate every other user's activity pattern. **The default must not be
> "all connected users".**

**Requirements**

- `ExplicitPresenceSubscriptionSource` is the **default**: an observer sees only subjects they
  explicitly subscribed to, and `CanObserveAsync` returns `true` only for an existing subscription or
  for `observer == subject`. Restrictive by default (Spec §25: security by default).
- `CompositePresenceSubscriptionSource` unions several sources and short-circuits `CanObserveAsync` on
  the first `true`. Registered automatically when more than one source is present.
- **Group-based subscriptions without a package dependency.** A `GroupPresenceSubscriptionSource`
  that derives observers from shared group membership needs `IGroupStore` from `LiveSharp.Chat`, and
  `LiveSharp.Presence` must not reference chat. Resolve it with a **third, tiny bridge package
  `LiveSharp.Chat.Presence`** that references both and contributes the source via
  `AddGroupPresence()`. Alternatives and why they lose:
  - Reflection/duck typing to find `IGroupStore` — fragile, untestable, violates "no excessive
    reflection" (Spec §36).
  - Presence referencing chat — the coupling Spec §12 forbids.
  - Chat referencing presence and registering the source — inverts the dependency but forces presence
    on every chat consumer.

  A three-line bridge package is the honest answer. Its downside, to be documented: a fourth package
  in the ecosystem whose only job is wiring.
- Enforce `MaxSubscriptionsPerUser` in `AddAsync` → `Conflict` naming the limit.
- `RemoveAllForObserverAsync` is called on **disconnect** only if subscriptions are
  connection-scoped. Decide: subscriptions are **user-scoped and survive disconnect**, because a
  client re-subscribing to 200 contacts on every reconnect is a thundering-herd generator. Document
  it, and provide `presence.unsubscribe.all` for clients that want the other behaviour.
- Both stores use the bidirectional `ImmutableHashSet` + empty-key-removal discipline from Phase 02.
  Two leak tests.

**Tests**

| Test | Asserts |
|---|---|
| `Default_source_denies_an_unsubscribed_observer` | **the security default** |
| `Default_source_allows_an_observer_of_their_own_presence` | |
| `Default_source_allows_a_subscribed_observer` | |
| `Subscribe_then_get_observers_includes_the_observer` | |
| `Unsubscribe_removes_the_observer` | |
| `Subscribe_twice_is_idempotent` | |
| `Subscribe_beyond_the_limit_returns_conflict_naming_the_limit` | |
| `Remove_all_for_observer_clears_both_indexes` | |
| `Empty_observer_key_is_removed` | leak test 1 |
| `Empty_subject_key_is_removed` | leak test 2 |
| `Composite_source_unions_observers_without_duplicates` | |
| `Composite_source_can_observe_is_true_if_any_source_allows` | |
| `Composite_source_can_observe_is_false_if_no_source_allows` | |
| `Group_source_returns_fellow_group_members_as_observers` | bridge package |
| `Group_source_denies_a_user_sharing_no_group` | |
| `Group_source_deduplicates_users_sharing_several_groups` | |
| `Presence_assembly_does_not_reference_the_chat_assembly` | **architecture test** |
| `Subscriptions_survive_a_disconnect` | pins the documented decision |
| `Concurrent_subscribes_never_exceed_the_limit` | `ConcurrencyHarness` |

---

### P06.T4 — Broadcasting, coalescing, and the expiry sweep

**Deliverables**

```
src/LiveSharp.Presence/Broadcasting/IPresenceBroadcaster.cs
src/LiveSharp.Presence/Broadcasting/PresenceBroadcaster.cs
src/LiveSharp.Presence/Broadcasting/PresenceChangeCoalescer.cs
src/LiveSharp.Presence/PresenceLifecycleHandler.cs
src/LiveSharp.Presence/PresenceExpirySweepService.cs
tests/LiveSharp.Presence.Tests/Broadcasting/…
tests/LiveSharp.Presence.Tests/PresenceLifecycleHandlerTests.cs
tests/LiveSharp.Presence.Tests/PresenceExpirySweepServiceTests.cs
```

**Requirements — coalescing**

- `PresenceChangeCoalescer` holds a per-user pending change and a `TimeProvider` timer of
  `ChangeCoalescingWindow`. A second change for the same user inside the window **replaces** the
  pending payload rather than queueing it — presence is a level, not an edge, so only the latest
  matters.
- A transition to a **terminal-ish** state must not be delayed behind coalescing when it is the
  final state: on `Offline`, flush immediately. A user who closes their laptop should not appear
  online for another two seconds while a coalescer waits. Document the asymmetry.
- Bound the pending map: above `10 × BroadcastFanoutLimit` entries, flush everything and log a
  `Warning`. An unbounded coalescing buffer under a connection storm is a memory leak with extra
  steps.
- Deterministically testable: every timer via `TimeProvider`.

**Requirements — broadcasting**

- `PresenceBroadcaster.PublishAsync(UserPresence)`:
  1. `IPresenceSubscriptionSource.GetObserversAsync(subject)`.
  2. If observer count `> BroadcastFanoutLimit`: **do not fan out.** Log a `Warning` with the count
     and the subject's observer total. Clients must fall back to `presence.query` polling for
     very-high-degree users. Document this ceiling prominently — silently dropping is only acceptable
     because the alternative (a 50 000-message fan-out per status flip) takes the server down. Emit a
     metric hook for Phase 16.
  3. Otherwise `IRealtimeTransport.SendToUsersAsync(observers, envelope)` with `presence.changed`.
  4. Always also send to the subject's own connections, so a user's other devices reflect the change.
- `StatusMessage` is included in the payload. That is the point of it, but note in
  `docs/security/` that presence status messages are visible to every observer.

**Requirements — lifecycle handler**

`PresenceLifecycleHandler : IConnectionLifecycleHandler`, registered by `AddPresence()`:

- `OnConnectedAsync`: re-aggregate and publish if the aggregate changed (first connection of a user →
  `Offline → Online` broadcast; second connection → no change, no broadcast).
- `OnDisconnectedAsync`: re-aggregate and publish. This **depends on the Phase 02 guarantee** that
  registry cleanup happens *before* disconnect handlers run — otherwise the live connection count
  still includes the connection being torn down and the user never goes offline. Reference
  `docs/architecture/connection-lifecycle.md` in a comment and assert the behaviour in a test, because
  if someone reorders Phase 02 this is the symptom.
- Uses `CancellationToken.None` on the disconnect path (Phase 02 already passes it; do not
  reintroduce a token).

**Requirements — expiry sweep**

`PresenceExpirySweepService : BackgroundService`, period `SweepInterval`:

- `IPresenceStore.GetStaleAsync(now - PresenceTtl, limit)`, then for each: re-aggregate. If the
  registry says zero connections, write `Offline` and broadcast. If the registry says connections
  exist but presence is stale, that is a **half-open connection**: log at `Information` and let the
  Phase 02 reaper handle the connection itself — do not disconnect from here, or two components fight
  over the same connection.
- Same resilience rules as the Phase 02 reaper: per-user try/catch, bounded work per tick, log
  summary at `Debug` and only at `Information` when something changed, survive a throwing store,
  stop promptly on `stoppingToken`.

**Tests — coalescer**

| Test | Asserts |
|---|---|
| `A_single_change_is_published_after_the_window` | `FakeTimeProvider` |
| `Two_changes_inside_the_window_publish_once_with_the_latest_value` | |
| `Changes_in_separate_windows_publish_twice` | |
| `An_offline_transition_publishes_immediately` | the asymmetry |
| `Changes_for_different_users_do_not_coalesce_together` | |
| `A_zero_coalescing_window_publishes_immediately` | degenerate config |
| `Exceeding_the_pending_cap_flushes_and_warns` | |
| `Disposing_the_coalescer_flushes_pending_changes` | no silent loss |
| `Concurrent_changes_for_one_user_publish_at_most_once_per_window` | `ConcurrencyHarness` |

**Tests — broadcaster**

| Test | Asserts |
|---|---|
| `Publishes_to_every_observer` | |
| `Publishes_to_the_subjects_own_connections` | |
| `Does_not_publish_to_a_non_observer` | **isolation / security** |
| `Skips_fan_out_above_the_limit_and_warns_with_the_count` | limit + 1 |
| `Fans_out_exactly_at_the_limit` | boundary |
| `Publishes_nothing_when_there_are_no_observers` | |
| `Includes_the_status_message_in_the_payload` | |

**Tests — lifecycle handler**

| Test | Asserts |
|---|---|
| `First_connection_broadcasts_online` | |
| `Second_connection_of_the_same_user_broadcasts_nothing` | |
| `Last_disconnect_broadcasts_offline` | |
| `Non_last_disconnect_broadcasts_nothing` | |
| `Disconnect_sees_a_connection_count_excluding_the_closing_connection` | **the Phase 02 ordering dependency** |
| `A_broadcast_failure_does_not_fail_the_connection` | |

**Tests — sweep**

| Test | Asserts |
|---|---|
| `Expires_presence_older_than_the_ttl_to_offline` | |
| `Does_not_expire_presence_within_the_ttl` | |
| `Does_not_expire_at_exactly_the_ttl` | boundary |
| `Expiry_broadcasts_the_change` | |
| `A_stale_presence_with_live_connections_logs_at_information_and_does_not_disconnect` | no fight with the Phase 02 reaper |
| `A_throwing_store_does_not_stop_the_service` | |
| `Honours_the_per_tick_limit` | |
| `Stops_promptly_on_the_stopping_token` | |

---

### P06.T5 — Operations, handlers, and `AddPresence()`

**Deliverables**

```
src/LiveSharp.Presence/Protocol/*.cs
src/LiveSharp.Presence/Handlers/SetStatusHandler.cs
src/LiveSharp.Presence/Handlers/ClearStatusHandler.cs
src/LiveSharp.Presence/Handlers/QueryPresenceHandler.cs
src/LiveSharp.Presence/Handlers/SubscribeHandler.cs
src/LiveSharp.Presence/Handlers/UnsubscribeHandler.cs
src/LiveSharp.Presence/Handlers/PresenceHeartbeatHandler.cs
src/LiveSharp.Presence/DependencyInjection/LiveSharpPresenceBuilderExtensions.cs
tests/LiveSharp.Presence.Tests/Handlers/…
tests/LiveSharp.Presence.Tests/RegistrationTests.cs
tests/LiveSharp.IntegrationTests/Presence/PresenceIntegrationTests.cs
```

Operations and events (add to `RealtimeOperations`/`RealtimeEvents`, register every DTO in
`LiveSharpJsonSerializerContext`, mirror in `protocol.ts`):

```
presence.status.set          presence.changed
presence.status.clear
presence.query
presence.subscribe
presence.unsubscribe
presence.heartbeat          (via NotifyAsync — fire-and-forget)
```

**Requirements**

- Handlers take the acting user from `context.Connection.UserId`. No request DTO may carry a
  `UserId` for the *actor*. `presence.query` and `presence.subscribe` carry *target* user IDs, which
  is correct — and every one must pass `CanObserveAsync`. Test the negative case for both.
- `presence.query` for a user the caller cannot observe returns `Offline` rather than `Forbidden`.
  Returning `Forbidden` reveals that the user exists and is being hidden; returning `Offline` reveals
  nothing. Document the deliberate choice. (Contrast with Phase 05 groups, which returns `Forbidden`
  because group membership is not a secret from members.)
- `presence.heartbeat` goes through `NotifyAsync`. A round trip per 30 s per client is pure waste, and
  a failed heartbeat is recovered by the next one. Reuse the Phase 01
  `RealtimeOperations.Connection.Heartbeat` name rather than adding a second heartbeat — one clock.
  If `AddPresence()` is registered, that handler additionally updates presence. Justify: two
  heartbeats means two intervals to configure and two ways for them to disagree.
- `AddPresence()` requires nothing but `AddLiveSharp().AddSignalR()`. It must **not** require chat.
  `AddGroupPresence()` (bridge package) requires both `AddPresence()` and `AddGroups()`.
- `ConnectionReadyPayload` gains the advertised `HeartbeatInterval` so clients do not hard-code it.
  This is an additive change to a public record — allowed, still recorded in `PublicAPI.Unshipped.txt`
  and the changelog.

**Tests — handlers and registration**

| Test | Asserts |
|---|---|
| `Set_status_handler_uses_the_connection_user_id` | **security**: no actor `UserId` in the DTO, asserted by reflection |
| `Query_handler_returns_offline_for_an_unobservable_user` | not `Forbidden` — the documented choice |
| `Query_handler_returns_real_presence_for_an_observable_user` | |
| `Query_handler_rejects_a_batch_above_the_limit` | |
| `Subscribe_handler_rejects_a_subject_the_caller_cannot_observe` | |
| `Heartbeat_handler_updates_presence_and_touches_the_connection` | |
| `AddPresence_registers_the_service_store_broadcaster_and_sweep` | lifetimes |
| `AddPresence_registers_the_presence_lifecycle_handler` | |
| `AddPresence_is_idempotent` | |
| `AddPresence_works_without_AddChat` | **the decoupling proof** |
| `AddGroupPresence_without_AddGroups_fails_at_startup_naming_AddGroups` | |
| `Connection_ready_payload_advertises_the_heartbeat_interval` | |

**Tests — integration (both transports)**

| Test | Asserts |
|---|---|
| `Subscriber_receives_presence_changed_when_the_subject_connects` | |
| `Subscriber_receives_presence_changed_when_the_subject_disconnects` | |
| `Non_subscriber_receives_nothing` | **isolation** |
| `A_users_second_device_receives_their_own_status_change` | |
| `Explicit_busy_set_on_one_device_is_visible_to_subscribers` | |
| `Explicit_status_survives_a_reconnect_within_the_ttl` | |
| `Presence_goes_offline_after_the_last_connection_closes` | |
| `Presence_goes_offline_when_heartbeats_stop_on_a_half_open_connection` | real short TTL, documented as the one place real time is used |
| `Two_devices_connecting_produce_exactly_one_online_broadcast` | coalescing end to end |
| `Query_of_an_unrelated_user_returns_offline` | security, end to end |
| `Group_members_see_each_others_presence_with_the_bridge_registered` | |

---

### P06.T6 — Blazor WebAssembly and React examples

**Deliverables**

```
examples/BlazorWasm/LiveSharp.Examples.BlazorWasm/…            (client project)
examples/BlazorWasm/LiveSharp.Examples.BlazorWasm.Server/…     (host + API)
examples/BlazorWasm/README.md
examples/React/…                                               (Vite + React + TS)
examples/React/README.md
clients/dotnet/LiveSharp.Client/Presence/LiveSharpPresenceClientExtensions.cs
clients/js/packages/client/src/presence.ts
clients/js/packages/client/src/__tests__/presence.test.ts
```

**Blazor WASM requirements**

- Uses `LiveSharp.Client` (the .NET client) — this is the case where a real client is genuinely
  needed, because the code runs in the browser. `examples/BlazorWasm/README.md` must contrast this
  with `examples/BlazorServer`'s approach and link both ways. This pair of examples is the
  documentation for Spec §58.
- Demonstrates: presence roster with live status dots, setting your own status, direct + group chat.
- `ILiveSharpClient` registered as a singleton in the WASM `Program.cs`, connected on first render,
  disposed on app teardown. Show the `AccessTokenProvider` wiring even though Phase 09 has not
  landed — with a comment that real auth arrives in `0.8.0`.

**React requirements**

- Vite + React + TypeScript, consuming `@livesharp/client` **directly** via a hand-written
  `useLiveSharp` hook in the example's own `src/hooks/`. Do **not** create `@livesharp/react` here —
  that is Phase 17, and writing the hook by hand in an example first is how we learn what the package
  should actually contain. Add a comment in the hook saying exactly that.
- Demonstrates the full subscribe → receive `presence.changed` → render loop, plus cleanup in the
  `useEffect` return (the unsubscribe function from `on()`), which is the thing React developers get
  wrong.
- Vite dev-server proxy to the ASP.NET Core host; document the WebSocket proxy configuration, because
  a misconfigured proxy silently downgrades to long polling and people file bugs about it.
- No CSS framework. `pnpm` workspace member so the root JS gate builds and lints it.

**Client extensions** — `GetPresenceAsync`, `SetStatusAsync`, `SubscribeAsync`, `UnsubscribeAsync`,
`OnPresenceChanged` on both clients; presence constants added to `protocol.ts`.

**Tests**

| Test | Asserts |
|---|---|
| `presence constants match artifacts/protocol-names.json` | contract test |
| `on presence changed delivers and unsubscribes` | |
| `set status rejects with LiveSharpError on an invalid status` | |
| `presence module imports cleanly with no DOM globals` | SSR safety |

---

## Dependency justification

No new runtime packages. `LiveSharp.Presence` references `LiveSharp.Core` and
`LiveSharp.AspNetCore` by project reference. The `LiveSharp.Chat.Presence` bridge references
`LiveSharp.Presence` and `LiveSharp.Chat`.

`examples/React` adds `vite`, `react`, `react-dom`, `@vitejs/plugin-react`, `typescript` — dev-only,
inside `clients/js`-style tooling already justified in Phase 03. No state-management library, no UI
kit, no CSS framework: the example must show LiveSharp, not a React stack.

New package count check (Spec §7): the ecosystem is now `Abstractions`, `Core`, `AspNetCore`, `Chat`,
`Presence`, `Chat.Presence`. Six packages, each with a distinct dependency profile. Record in
ADR-012 that `Chat.Presence` is the one package whose existence is pure wiring, and that it is
accepted only because the alternatives invert or widen a dependency.

---

## Documentation deltas

- `docs/presence/README.md` — **new**. Concepts, the five statuses, the aggregation rules as an
  ordered list, the `Status == Offline` vs `IsReachable` distinction, options, quick start.
- `docs/presence/subscriptions.md` — **new**. The security model: default-deny, explicit
  subscriptions, the group bridge, why `presence.query` returns `Offline` rather than `Forbidden`,
  and the `BroadcastFanoutLimit` ceiling with the recommended client fallback.
- `docs/presence/heartbeats.md` — **new**. Why heartbeats exist, how TTL interacts with
  `Connections.IdleTimeout` from Phase 02, and the configuration table for the four interacting
  durations. Include a worked example of a correct set of values and a broken one.
- `docs/architecture/decisions/ADR-012-presence-package.md` — **new**.
- `docs/security/README.md` — presence status messages are visible to all observers; presence leaks
  activity patterns; default-deny subscriptions.
- `docs/configuration/README.md`, `docs/configuration/service-lifetimes.md` — presence entries.
- `docs/troubleshooting/README.md` — "everyone shows offline" (TTL < heartbeat), "presence never goes
  offline" (Phase 02 handler ordering), "presence changes are not received" (no subscription),
  "presence stops updating for popular users" (fan-out limit), "status resets on reconnect" (TTL).
- `README.md` — presence → `Available`.

## Example deltas

- `examples/BlazorWasm` — new.
- `examples/React` — new.
- `examples/AspNetCoreMvc`, `examples/BlazorServer` — presence indicators added.
- `examples/README.md` — updated index with the Blazor Server vs WASM pointer.

## CHANGELOG entry

```markdown
### Added
- `LiveSharp.Presence`: online/away/busy/do-not-disturb presence with per-user aggregation across
  connections, explicit-over-derived status precedence, and heartbeat-driven TTL expiry.
- Default-deny presence subscriptions with an explicit subscription store.
- `LiveSharp.Chat.Presence` bridge deriving presence observers from shared group membership.
- Presence change coalescing to prevent broadcast storms from flapping connections.
- Blazor WebAssembly and React example applications.
- Presence helpers for the .NET and TypeScript clients.

### Changed
- `ConnectionReadyPayload` now advertises the server's heartbeat interval (additive).

### Security
- Presence is not visible by default: an observer must be subscribed, or share a group when the
  bridge package is installed.
- `presence.query` for an unobservable user returns Offline rather than a forbidden error, so it
  cannot be used to probe for account existence.
```

---

## Exit criteria

- [ ] `LiveSharp.Presence` has no reference to `LiveSharp.Chat` — architecture test green.
- [ ] `AddPresence_works_without_AddChat` passes.
- [ ] `Default_source_denies_an_unsubscribed_observer` passes.
- [ ] `Heartbeat_that_does_not_change_the_aggregate_does_not_broadcast` passes.
- [ ] `Disconnect_sees_a_connection_count_excluding_the_closing_connection` passes.
- [ ] `Explicit_offline_while_connected_reports_offline_but_remains_reachable` passes.
- [ ] `Skips_fan_out_above_the_limit_and_warns_with_the_count` passes.
- [ ] `Two_devices_connecting_produce_exactly_one_online_broadcast` passes end to end.
- [ ] `Presence_ttl_not_greater_than_the_heartbeat_interval_is_rejected_with_guidance` passes.
- [ ] `PresenceStoreContractTests` runs against `InMemoryPresenceStore`.
- [ ] ADR-012 exists with an honest negative-consequences section.
- [ ] `docs/presence/heartbeats.md` documents the four interacting durations with a worked example.
- [ ] Blazor WASM and React examples run; both READMEs cross-link the Server/WASM contrast.
- [ ] `examples/React` is a `pnpm` workspace member and passes the JS gate.
- [ ] Presence constants in `protocol.ts`; contract test green.
- [ ] `scripts/verify.ps1` green; tag `v0.5.0`.

## Verify

```powershell
pwsh scripts/verify.ps1
dotnet test tests/LiveSharp.Presence.Tests -c Release
dotnet test tests/LiveSharp.IntegrationTests -c Release --filter "FullyQualifiedName~Presence"
dotnet test tests/LiveSharp.ArchitectureTests -c Release
pnpm --dir clients/js -r test
git tag v0.5.0
```

## Next

`docs/plan/phase-07-typing.md`, task `P07.T1`.
