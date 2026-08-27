# Phase 07 — Typing Indicators

| | |
|---|---|
| **Version at exit** | `0.6.0` (tag `v0.6.0`) |
| **Depends on** | Phase 06 |
| **Branch** | `feat/phase-07-typing` |
| **Tasks** | `P07.T1` … `P07.T4` |
| **Read before starting** | `docs/plan/phase-05-groups.md`, `docs/presence/README.md` |

---

## Goal

"Alice is typing…" — for direct conversations and groups, with server-side auto-expiry so a client
that dies mid-sentence does not leave a permanent ghost, and with throttling so a keystroke does not
become a network round trip.

Small feature, three real engineering problems: **unbounded state**, **resource-per-user timers**, and
**traffic amplification**. Everything in this phase exists to solve one of those three.

---

## Non-goals — DO NOT implement in this phase

- **No persistence, ever.** Typing state is ephemeral by definition. There is no `ITypingStore`, no
  Phase 14 provider, and no reason to have one. If someone later asks for typing history, the answer
  is no.
- No distributed typing state. Single node. Phase 15 decides whether typing is even worth
  replicating (it probably is not — a cross-node typing indicator that arrives 200 ms late is worse
  than none).
- No receipts (Phase 08) or authorization (Phase 09). A user can currently signal typing into any
  conversation ID they can name; the gap is documented and closed in Phase 09.
- No "recording audio" / "uploading file" activity variants. One activity: typing. Adding a
  `TypingActivity` enum with one member is speculative; adding a second member later is additive.
- No per-user typing preferences or opt-out. Application concern.

---

## Tasks

### P07.T1 — Model, tracker, and options

**Deliverables**

```
src/LiveSharp.Chat/Typing/TypingState.cs
src/LiveSharp.Chat/Typing/ITypingTracker.cs
src/LiveSharp.Chat/Typing/InMemoryTypingTracker.cs
src/LiveSharp.Chat/Configuration/TypingOptions.cs
src/LiveSharp.Chat/Configuration/TypingOptionsValidator.cs
src/LiveSharp.Chat/Diagnostics/TypingLog.cs
tests/LiveSharp.Chat.Tests/Typing/InMemoryTypingTrackerTests.cs
tests/LiveSharp.Chat.Tests/Typing/TypingTrackerConcurrencyTests.cs
```

**Public API (exact):**

```csharp
namespace LiveSharp.Chat;

/// <summary>One user's active typing indication in one conversation.</summary>
public sealed record TypingState(
    string ConversationId,
    string UserId,
    DateTimeOffset StartedAt,
    DateTimeOffset ExpiresAt);

public sealed class TypingOptions
{
    public const string ConfigurationSectionName = "LiveSharp:Chat:Typing";

    /// <summary>A typing indication expires this long after the last start. Default 5 seconds.</summary>
    public TimeSpan TypingTimeout { get; set; } = TimeSpan.FromSeconds(5);

    /// <summary>Minimum interval between broadcast typing notifications for one user and conversation.
    /// Default 2 seconds.</summary>
    public TimeSpan MinNotificationInterval { get; set; } = TimeSpan.FromSeconds(2);

    /// <summary>How often expired indications are swept. Default 1 second.</summary>
    public TimeSpan SweepInterval { get; set; } = TimeSpan.FromSeconds(1);

    /// <summary>Maximum conversations one user may be typing in simultaneously. Default 8.</summary>
    public int MaxConcurrentConversationsPerUser { get; set; } = 8;

    /// <summary>Maximum tracked typing entries server-wide. Above this, new indications are dropped.
    /// Default 100000.</summary>
    public int MaxTrackedEntries { get; set; } = 100_000;

    /// <summary>When true, sending a message clears the sender's typing state. Default true.</summary>
    public bool ClearOnMessageSent { get; set; } = true;
}
```

```csharp
namespace LiveSharp.Chat.Typing;

/// <summary>Tracks who is currently typing where. In-memory and node-local by design.</summary>
public interface ITypingTracker
{
    /// <summary>Records or refreshes a typing indication. Returns the resulting state, or a failure
    /// when a limit is reached.</summary>
    OperationResult<TypingStartOutcome> Start(string conversationId, string userId);

    /// <summary>Clears an indication. Returns true if one existed.</summary>
    bool Stop(string conversationId, string userId);

    /// <summary>Clears every indication for a user, across all conversations.</summary>
    IReadOnlyList<string> StopAll(string userId);

    IReadOnlyList<string> GetTypingUsers(string conversationId);

    /// <summary>Removes expired entries and returns what was removed, for broadcasting.</summary>
    IReadOnlyList<TypingState> SweepExpired();

    int Count { get; }
}

/// <summary>Result of a start, telling the caller whether a broadcast is warranted.</summary>
public sealed record TypingStartOutcome(TypingState State, bool ShouldNotify);
```

**Requirements**

- The tracker is **synchronous**. It is a node-local in-memory map on a hot path (potentially every
  few keystrokes from every user); wrapping it in `ValueTask` buys nothing and costs clarity. This is
  a deliberate departure from the async store pattern in Phases 04–06, and the reason must be in the
  XML docs — otherwise someone will "fix" it for consistency.
- Two indexes, both `ConcurrentDictionary`, both with empty-key removal:
  `_byConversation : conversationId -> ImmutableDictionary<userId, TypingState>` and
  `_byUser : userId -> ImmutableHashSet<conversationId>`. The reverse index makes `StopAll` on
  disconnect O(conversations-for-user) instead of a full scan.
- **`ShouldNotify` is the throttle.** `Start` always refreshes `ExpiresAt`, but returns
  `ShouldNotify = false` when the previous broadcast for this (conversation, user) pair was within
  `MinNotificationInterval`. Refreshing without broadcasting is exactly right: the indicator stays
  alive on the recipients' screens because their own client-side timer has not fired, and the wire
  stays quiet. Store `LastNotifiedAt` alongside the state.
- Limits, both enforced in `Start`:
  - `MaxConcurrentConversationsPerUser` exceeded → `Failure(Conflict, …)` naming the limit. A client
    typing in 500 conversations is broken or hostile.
  - `MaxTrackedEntries` exceeded → `Failure(RateLimited, …)` and a `Warning` log, at most once per
    `SweepInterval` (do not log per rejected keystroke). This is the server-wide backstop.
- `SweepExpired` returns what it removed so the sweep service can broadcast `typing.stopped`. A
  sweep that silently drops state leaves every client showing a stuck indicator until its own local
  timeout — which is why client-side timeouts must also exist, and the docs must say so.

> ## One timer, not one timer per typist
> The obvious implementation gives each typing user a `TimeProvider` timer that fires on expiry. With
> 10 000 concurrent typists that is 10 000 timer registrations churning every five seconds — a
> measurable allocation and scheduling load for a cosmetic feature.
>
> Use **one** `PeriodicTimer` in `TypingSweepService` (`P07.T2`) scanning for expiries at
> `SweepInterval`. The cost is up to `SweepInterval` of extra latency on the "stopped" event, which is
> invisible at 1 s. Put this reasoning in a comment; the per-user-timer version looks more elegant and
> someone will propose it.

**Tests**

| Test | Asserts |
|---|---|
| `Start_records_state_with_the_configured_expiry` | `FakeTimeProvider` |
| `Start_returns_should_notify_true_the_first_time` | |
| `Start_again_inside_the_min_interval_returns_should_notify_false` | **the throttle** |
| `Start_again_inside_the_min_interval_still_extends_the_expiry` | **required** — throttled but alive |
| `Start_again_after_the_min_interval_returns_should_notify_true` | |
| `Start_at_exactly_the_min_interval_returns_should_notify_true` | boundary |
| `Stop_returns_true_then_false` | |
| `Stop_all_clears_every_conversation_for_the_user_and_returns_them` | |
| `Stop_all_for_a_user_who_is_not_typing_returns_empty` | |
| `Get_typing_users_excludes_expired_entries` | reads must not show stale state even before a sweep |
| `Get_typing_users_for_an_unknown_conversation_returns_empty` | |
| `Sweep_removes_expired_entries_and_returns_them` | |
| `Sweep_does_not_remove_entries_at_exactly_their_expiry` | boundary |
| `Sweep_of_an_empty_tracker_returns_empty` | |
| `Empty_conversation_key_is_removed_after_the_last_typist_stops` | leak test 1 |
| `Empty_user_key_is_removed_after_the_last_conversation_stops` | leak test 2 |
| `Sweeping_also_removes_empty_index_keys` | leak test 3 — the one people miss |
| `Start_beyond_the_per_user_conversation_limit_returns_conflict` | limit + 1 |
| `Start_beyond_the_server_wide_entry_limit_returns_rate_limited` | |
| `Server_wide_limit_rejection_is_logged_at_most_once_per_sweep_interval` | log-flood guard |
| `Count_reflects_starts_stops_and_sweeps` | |
| `Concurrent_starts_and_stops_leave_both_indexes_consistent` | `ConcurrencyHarness` |
| `Concurrent_starts_never_exceed_the_per_user_limit` | 32 workers, limit 4 |
| `Concurrent_sweep_and_start_never_lose_a_live_entry` | the classic sweep race |
| `Typing_options_defaults_are_valid` | |
| `Min_notification_interval_not_less_than_the_typing_timeout_is_rejected` | a throttle longer than the timeout means the indicator never refreshes — actionable message |
| `Sweep_interval_not_less_than_the_typing_timeout_is_rejected` | |

---

### P07.T2 — `ITypingService`, sweep service, and lifecycle integration

**Deliverables**

```
src/LiveSharp.Chat/Typing/ITypingService.cs
src/LiveSharp.Chat/Typing/TypingService.cs
src/LiveSharp.Chat/Typing/TypingSweepService.cs
src/LiveSharp.Chat/Typing/TypingLifecycleHandler.cs
tests/LiveSharp.Chat.Tests/Typing/TypingServiceTests.cs
tests/LiveSharp.Chat.Tests/Typing/TypingSweepServiceTests.cs
tests/LiveSharp.Chat.Tests/Typing/TypingLifecycleHandlerTests.cs
```

**Public API (exact):**

```csharp
namespace LiveSharp.Chat;

public interface ITypingService
{
    Task<OperationResult> StartAsync(string userId, string conversationId, RealtimeOperationContext? context = null, CancellationToken cancellationToken = default);
    Task<OperationResult> StopAsync(string userId, string conversationId, RealtimeOperationContext? context = null, CancellationToken cancellationToken = default);
    Task<IReadOnlyList<string>> GetTypingUsersAsync(string conversationId, CancellationToken cancellationToken = default);
}
```

**Requirements — recipient resolution**

A conversation ID is either `dm:{a}|{b}` (Phase 04) or `grp:{groupId}` (Phase 05). Typing must reach
the right people without `ITypingService` gaining a hard dependency on `IGroupService`:

- `dm:` — parse the pair, verify the acting user is one of the two → otherwise `Forbidden`, and
  deliver to the **other** participant plus the sender's own other connections. Reuse the Phase 04
  `ConversationId` helpers; add `ConversationId.TryParseDirect(id, out a, out b)` rather than
  re-implementing the split, including the separator-escaping rule.
- `grp:` — deliver via `IRealtimeTransport.SendToGroupExceptAsync` on
  `GroupTransportName.For(groupId)`, excluding the originating connection. Membership is **not**
  checked here in this phase (Phase 09); the gap is documented and gets a pinning test.
- Any other prefix → `InvalidPayload` naming the two supported forms. Do not silently accept
  arbitrary conversation IDs; that is an unbounded-key attack on the tracker.

**Requirements — service**

- `StartAsync`: parse and authorise the conversation form, `ITypingTracker.Start`, and broadcast
  `typing.started` **only when `ShouldNotify`**. A throttled start returns `Success` with no traffic.
- `StopAsync`: `ITypingTracker.Stop`; broadcast `typing.stopped` only if an entry existed. A stop for
  a user who was not typing is a no-op success — clients send redundant stops constantly.
- A `Conflict`/`RateLimited` result from the tracker must **not** be surfaced as an error to a
  `NotifyAsync` caller. See `P07.T3`.
- Never log the conversation ID above `Debug` and never log it at all for `dm:` forms, because a
  direct conversation ID contains two user IDs. Add a test.

**Requirements — sweep service**

`TypingSweepService : BackgroundService`, one `PeriodicTimer` from `TimeProvider` at `SweepInterval`:
call `SweepExpired`, group the removed states by conversation, and broadcast `typing.stopped` for
each. Same resilience rules as the Phase 02 reaper: per-conversation try/catch, bounded work, `Debug`
summary, `Information` only when something was swept, survives a throwing transport, stops promptly.

**Requirements — lifecycle**

`TypingLifecycleHandler : IConnectionLifecycleHandler`:
- `OnConnectedAsync`: nothing.
- `OnDisconnectedAsync`: if this was the user's **last** connection, `StopAll(userId)` and broadcast
  `typing.stopped` for each conversation. If the user still has other connections, do nothing — they
  may genuinely still be typing on their phone. Getting this wrong (stopping on any disconnect) makes
  multi-device typing flicker; getting it wrong the other way (never stopping) leaves ghosts.
  Relies on the Phase 02 guarantee that registry cleanup precedes disconnect handlers.

**Tests — service**

| Test | Asserts |
|---|---|
| `Start_in_a_direct_conversation_notifies_the_other_participant` | |
| `Start_in_a_direct_conversation_notifies_the_senders_other_connections` | |
| `Start_does_not_notify_the_originating_connection` | |
| `Start_in_a_direct_conversation_the_user_is_not_part_of_is_forbidden` | **security** |
| `Start_in_a_group_conversation_notifies_the_group_except_the_sender` | |
| `Start_with_an_unrecognised_conversation_id_form_returns_invalid_payload` | table-driven: `""`, `"x"`, `"dm:"`, `"grp:"`, `"foo:bar"` |
| `Start_throttled_by_the_tracker_notifies_nobody_but_returns_success` | **the throttle end to end** |
| `Stop_notifies_when_an_entry_existed` | |
| `Stop_for_a_user_who_was_not_typing_notifies_nobody_and_returns_success` | |
| `Get_typing_users_excludes_the_caller` | a client should not render itself as typing |
| `Conversation_id_is_never_logged_for_direct_conversations` | **security test** |
| `Group_typing_from_a_non_member_currently_succeeds` | **pins the documented Phase 09 gap** |
| `All_methods_honour_a_pre_cancelled_token` | |

**Tests — sweep and lifecycle**

| Test | Asserts |
|---|---|
| `Sweep_broadcasts_typing_stopped_for_expired_entries` | `FakeTimeProvider` |
| `Sweep_broadcasts_once_per_conversation_not_once_per_user` | fan-out efficiency |
| `Sweep_with_nothing_expired_broadcasts_nothing` | |
| `A_throwing_transport_does_not_stop_the_sweep_service` | |
| `Sweep_stops_promptly_on_the_stopping_token` | |
| `Last_disconnect_stops_all_typing_for_the_user_and_broadcasts` | |
| `Non_last_disconnect_leaves_typing_state_intact` | **the multi-device rule** |
| `Disconnect_broadcast_failure_does_not_fail_the_disconnect` | |

---

### P07.T3 — Operations, `ClearOnMessageSent`, and `AddTyping()`

**Deliverables**

```
src/LiveSharp.Chat/Typing/Protocol/TypingRequest.cs
src/LiveSharp.Chat/Typing/Protocol/TypingChangedPayload.cs
src/LiveSharp.Chat/Typing/Handlers/TypingStartHandler.cs
src/LiveSharp.Chat/Typing/Handlers/TypingStopHandler.cs
src/LiveSharp.Chat/Typing/Handlers/GetTypingUsersHandler.cs
src/LiveSharp.Chat/Typing/TypingMessageSentObserver.cs
src/LiveSharp.Chat/IChatMessageObserver.cs
src/LiveSharp.Chat/DependencyInjection/LiveSharpTypingBuilderExtensions.cs
tests/LiveSharp.Chat.Tests/Typing/Handlers/…
tests/LiveSharp.Chat.Tests/Typing/RegistrationTests.cs
tests/LiveSharp.IntegrationTests/Chat/TypingIntegrationTests.cs
```

Operations and events — add to `RealtimeOperations`/`RealtimeEvents`, register DTOs in
`LiveSharpJsonSerializerContext`, mirror in `protocol.ts`:

```
typing.start     (NotifyAsync)      typing.started
typing.stop      (NotifyAsync)      typing.stopped
typing.query     (InvokeAsync)
```

**Requirements — fire-and-forget**

`typing.start` and `typing.stop` are sent through the client's `notify()` / `NotifyAsync`, not
`invoke()`. Reasons to state in the docs:
- A user typing for 30 seconds generates ~15 starts; 15 round trips with correlation IDs and
  responses per user per message is pure overhead for information that self-heals.
- A lost typing notification is corrected by the next one or by expiry. There is nothing to retry.

Consequence, and the reason this needs care: a `NotifyAsync` handler's failure is invisible to the
client. So a throttled or rate-limited typing start must be a **silent success**, never an error.
Returning `rate_limited` per keystroke would fill client logs with noise about a working system. The
handler maps `Conflict`/`RateLimited` from the service to a `Debug` log and success. Test this
explicitly — it looks like swallowing errors and needs a comment saying it is deliberate.

**Requirements — `ClearOnMessageSent` without coupling**

Sending a message should stop the sender's typing indicator. The wrong way is for `ChatService` to
depend on `ITypingService` — chat would then require typing, and typing is opt-in.

Introduce a minimal observer seam in `LiveSharp.Chat`:

```csharp
namespace LiveSharp.Chat;

/// <summary>Notified after a message is accepted and delivered. Implementations must not throw.</summary>
public interface IChatMessageObserver
{
    ValueTask OnMessageSentAsync(ChatMessage message, CancellationToken cancellationToken = default);
}
```

`ChatService` resolves `IEnumerable<IChatMessageObserver>` (empty by default, zero cost) and invokes
them after successful delivery, sequentially, with per-observer try/catch and `Error` logging — the
same error-isolation contract as `IConnectionLifecycleHandler`. `AddTyping()` registers
`TypingMessageSentObserver`, which calls `StopAsync` when `ClearOnMessageSent` is true.

This is a real abstraction with a real second consumer already visible: Phase 08's delivery-state
service wants the same hook. Note that in the XML docs so Phase 08 reuses it rather than adding
another one.

**Requirements — `AddTyping()`**

```csharp
public static class LiveSharpTypingBuilderExtensions
{
    public static ILiveSharpBuilder AddTyping(this ILiveSharpBuilder builder);
    public static ILiveSharpBuilder AddTyping(this ILiveSharpBuilder builder, Action<TypingOptions> configure);
}
```

Requires `AddChat()` (validated at startup, order-independent, same pattern as `AddGroups()`).
Group typing works only if `AddGroups()` is also registered — if it is not, a `grp:` conversation ID
returns `NotFound` rather than failing obscurely inside the transport. Test it.

**Tests — handlers, registration, integration (both transports)**

| Test | Asserts |
|---|---|
| `Typing_start_handler_uses_the_connection_user_id` | reflection: no `UserId` in `TypingRequest` |
| `Typing_start_handler_maps_a_rate_limited_result_to_silent_success` | **the deliberate swallow** |
| `Typing_start_handler_logs_a_throttled_start_at_debug` | |
| `Typing_query_handler_returns_the_current_typists` | |
| `AddTyping_registers_the_tracker_service_sweep_and_lifecycle_handler` | lifetimes |
| `AddTyping_registers_the_message_sent_observer_only_when_clear_on_send_is_enabled` | |
| `AddTyping_without_AddChat_fails_at_startup_naming_AddChat` | |
| `AddTyping_without_AddGroups_returns_not_found_for_group_conversations` | |
| `AddChat_alone_registers_no_typing_operations` | opt-in is real |
| `Chat_service_with_no_observers_registered_does_no_extra_work` | zero-cost default |
| `A_throwing_observer_does_not_fail_the_send` | error isolation |
| `Sending_a_message_broadcasts_typing_stopped_for_the_sender` | end to end |
| `Sending_a_message_does_not_broadcast_typing_stopped_when_disabled` | |
| `Alice_typing_is_visible_to_bob` | integration, both transports |
| `Alice_typing_expires_automatically_without_a_stop` | real short timeout; documented |
| `Alice_typing_stops_when_alice_disconnects` | |
| `Alice_typing_survives_alice_closing_one_of_two_devices` | |
| `Rapid_typing_start_calls_produce_at_most_one_notification_per_interval` | **the amplification test**: 20 starts in 2 s → 1 broadcast |
| `Group_typing_reaches_every_member_except_the_sender` | |
| `Two_users_typing_in_one_conversation_both_appear` | |

---

### P07.T4 — Next.js example and client support

**Deliverables**

```
examples/NextJs/package.json
examples/NextJs/next.config.ts
examples/NextJs/app/layout.tsx
examples/NextJs/app/page.tsx
examples/NextJs/app/chat/ChatClient.tsx
examples/NextJs/lib/livesharp.ts
examples/NextJs/README.md
clients/dotnet/LiveSharp.Client/Chat/LiveSharpTypingClientExtensions.cs
clients/js/packages/client/src/typing.ts
clients/js/packages/client/src/__tests__/typing.test.ts
```

**Next.js requirements — this example exists to prove the SSR story**

Spec §60 asks specifically about client/server boundaries and browser-only APIs. Demonstrate, with
comments explaining each:

- `@livesharp/client` imported in a `"use client"` component only. The example must **also** import it
  in a server component to prove the Phase 03 SSR-safety guarantee holds — if that import crashes the
  build, the guarantee was fiction and Phase 03 must be fixed, not the example.
- Connection created in a `useEffect`, torn down in the cleanup. Not at module scope, and not in a
  `useState` initialiser (which runs twice in React strict mode and creates two connections — call
  this out in a comment, it is the mistake everyone makes).
- The `on()` unsubscribe function returned from `useEffect`.
- `next.config.ts` rewrite/proxy to the ASP.NET Core host, with the WebSocket note.
- Auth token retrieval shown as a placeholder with a comment pointing at Phase 09.
- `app/chat/ChatClient.tsx` implements: message list, send box that calls `typing.start` on input
  (client-side throttled to match `MinNotificationInterval`, demonstrating that clients should throttle
  too) and `typing.stop` on blur/send, and a typing indicator row driven by `typing.started`/
  `typing.stopped` with a **client-side timeout** as a safety net.
- `README.md` must include a "Common mistakes" section: module-scope connection, strict-mode double
  connection, missing unsubscribe, proxy without WebSocket support, importing the client in a server
  component and assuming it will connect.

**Client typing helpers** — on both clients: `StartTypingAsync`/`StopTypingAsync`,
`OnTypingStarted`/`OnTypingStopped`. The TS helper should additionally expose a
`createTypingSignaller(client, conversationId, { minInterval })` returning `{ keystroke(), stop() }`
that does client-side throttling, so example and application authors do not each re-implement it.
Server-side throttling is the security boundary; client-side throttling is the bandwidth optimisation.
Both are needed; say so.

Typing constants added to `protocol.ts`; the Phase 03 contract test enforces parity.

**Tests**

| Test | Asserts |
|---|---|
| `typing constants match artifacts/protocol-names.json` | contract test |
| `createTypingSignaller emits at most one start per min interval` | fake timers |
| `createTypingSignaller stop always emits` | stop is never throttled |
| `createTypingSignaller cleans up its timer on stop` | leak |
| `typing module imports cleanly with no DOM globals` | SSR safety |
| `on typing started delivers and unsubscribes` | |

`examples/NextJs` joins the `pnpm` workspace and the `examples` CI job (build only — Playwright
arrives in Phase 12).

---

## Dependency justification

No new .NET packages. Typing lives in `LiveSharp.Chat` and reuses the transport, the tracker is
BCL-only.

`examples/NextJs` adds `next`, `react`, `react-dom`, `typescript` — dev-only, in the existing
`clients/js`-adjacent tooling. No UI library, no state manager.

`IChatMessageObserver` is a new abstraction rather than a package: justified because Phase 08 needs
the same hook, and the alternative (chat depending on typing) inverts the opt-in relationship.

---

## Documentation deltas

- `docs/chat/typing.md` — **new**. Quick start, `TypingOptions` table, the two-layer throttle model
  (client-side for bandwidth, server-side for safety), why `NotifyAsync` and what that means for error
  visibility, the auto-expiry contract and why clients also need a local timeout, multi-device
  behaviour, and a **Limitations** section: node-local (Phase 15), no membership check on group
  conversation IDs (Phase 09), never persisted by design.
- `docs/chat/README.md` — index.
- `docs/architecture/observers.md` — **new, short**. The `IChatMessageObserver` seam, its
  error-isolation contract, and the rule that observers must be cheap and must not throw.
- `docs/configuration/README.md` — `TypingOptions`, with the four interacting durations and a worked
  correct/incorrect example (mirroring the Phase 06 presence heartbeat page).
- `docs/troubleshooting/README.md` — "typing indicator sticks forever" (no client timeout, or
  `SweepInterval` misconfigured), "typing never appears" (`AddTyping` missing, or wrong conversation
  ID form), "typing flickers with two devices", "typing floods the network" (client not throttling).
- `README.md` — typing indicators → `Available`.

## Example deltas

- `examples/NextJs` — new.
- `examples/AspNetCoreMvc`, `examples/BlazorServer`, `examples/BlazorWasm`, `examples/React` — typing
  indicators added to each.
- `examples/README.md` — updated.

## CHANGELOG entry

```markdown
### Added
- Typing indicators for direct and group conversations, with server-side auto-expiry and throttling.
- `typing.start` / `typing.stop` fire-and-forget operations and `typing.started` / `typing.stopped`
  events.
- `IChatMessageObserver` extension point, used to clear typing state when a message is sent.
- `createTypingSignaller` client helper providing client-side throttling.
- Next.js example application demonstrating client/server boundaries and SSR-safe usage.

### Notes
- Typing state is intentionally never persisted and is node-local until distributed support lands.
```

---

## Exit criteria

- [ ] `Rapid_typing_start_calls_produce_at_most_one_notification_per_interval` passes.
- [ ] `Start_again_inside_the_min_interval_still_extends_the_expiry` passes.
- [ ] `Sweeping_also_removes_empty_index_keys` passes.
- [ ] `Non_last_disconnect_leaves_typing_state_intact` passes.
- [ ] `Typing_start_handler_maps_a_rate_limited_result_to_silent_success` passes.
- [ ] `Concurrent_sweep_and_start_never_lose_a_live_entry` passes.
- [ ] `Conversation_id_is_never_logged_for_direct_conversations` passes.
- [ ] `Group_typing_from_a_non_member_currently_succeeds` passes and the gap is documented.
- [ ] `Chat_service_with_no_observers_registered_does_no_extra_work` passes.
- [ ] Exactly **one** background timer serves all typing expiry; no per-user timers exist.
- [ ] `@livesharp/client` still imports cleanly with no DOM globals, and the Next.js example proves
      it by importing in a server component.
- [ ] `docs/architecture/observers.md` documents the observer contract.
- [ ] Typing constants in `protocol.ts`; contract test green.
- [ ] `scripts/verify.ps1` green; `examples` CI job green; tag `v0.6.0`.

## Verify

```powershell
pwsh scripts/verify.ps1
dotnet test tests/LiveSharp.Chat.Tests -c Release --filter "FullyQualifiedName~Typing"
dotnet test tests/LiveSharp.IntegrationTests -c Release --filter "FullyQualifiedName~Typing"
pnpm --dir clients/js -r test
pnpm --dir examples/NextJs build
git tag v0.6.0
```

## Next

`docs/plan/phase-08-message-state.md`, task `P08.T1`.
