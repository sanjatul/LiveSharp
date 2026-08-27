# Phase 11 — Audio Calling

| | |
|---|---|
| **Version at exit** | `0.10.0` (tag `v0.10.0`) |
| **Depends on** | Phase 10 |
| **Branch** | `feat/phase-11-audio-calling` |
| **Tasks** | `P11.T1` … `P11.T6` |
| **Read before starting** | `docs/webrtc/signaling.md`, `docs/webrtc/perfect-negotiation.md`, `docs/plan/phase-08-message-state.md` (for the state-machine test pattern) |

---

## Goal

A new package, `LiveSharp.Calls`, that turns the Phase 10 signalling primitive into a product feature:
one-to-one audio calls with ringing, accept, reject, cancel, hang-up, timeouts, busy detection, and
multi-device ringing that resolves to exactly one answering device.

`LiveSharp.Calls` owns **call state**. It does not touch media and does not duplicate signalling — a
call *has* a signalling session, and the browser does the WebRTC.

```
call.invite ──▶ ringing on every one of Bob's devices
                       │
              Bob's phone accepts
                       │
        other devices told to stop ringing
                       │
      signalling session created ──▶ perfect negotiation ──▶ audio flows peer to peer
```

---

## Scope: one-to-one only

`0.10.0` supports exactly two participants. Conference calling requires an SFU to avoid N² uplink,
and an SFU is explicitly out of scope for 1.0 (ADR-006). Say this plainly in
`docs/calls/README.md` — "group calls" is the most common feature request for a library like this and
the honest answer is "not before 1.0, and probably never in-process".

Design so that adding participants later is not blocked: `Call` has a participant collection rather
than two string fields, even though it will always hold two in this phase. That costs nothing now and
avoids a breaking change if a mesh topology is ever added for three or four participants.

---

## Non-goals — DO NOT implement in this phase

- **No video.** Phase 12. `CallMediaKind` defines the `Video` flag because it is in the wire contract
  and adding a flag value later is fine while changing the field's type is not — but no video code path
  exists.
- **No screen sharing.** Phase 13.
- **No group or conference calls.** See above.
- No call recording, transcription, or server-side media of any kind.
- No call history query API, no missed-call list, no call log pagination. `ICallStore` records calls so
  Phase 14 can persist and query them; the query API is Phase 14's job.
- No PSTN, SIP, DTMF, hold, transfer, park, or conference bridging.
- No hold/resume. Mute is in scope (it is client-side state the peer needs to know about); hold is a
  media-renegotiation feature that belongs after video.
- No distributed call routing. Single node; Phase 15.
- No ringtones, notification sounds, or push notifications. Those are application concerns, though the
  docs should point out that a real product needs push notifications for calls to a backgrounded app.

---

## Tasks

### P11.T1 — Model and the call state machine

**Deliverables**

```
src/LiveSharp.Calls/LiveSharp.Calls.csproj
src/LiveSharp.Calls/PublicAPI.{Shipped,Unshipped}.txt
src/LiveSharp.Calls/AssemblyInfo.cs
src/LiveSharp.Calls/Call.cs
src/LiveSharp.Calls/CallParticipant.cs
src/LiveSharp.Calls/CallState.cs
src/LiveSharp.Calls/CallMediaKind.cs
src/LiveSharp.Calls/CallEndReason.cs
src/LiveSharp.Calls/CallStateMachine.cs
tests/LiveSharp.Calls.Tests/CallStateMachineTests.cs
```

**Public API (exact):**

```csharp
namespace LiveSharp.Calls;

/// <summary>Lifecycle state of a call. Values are part of the wire contract.</summary>
public enum CallState
{
    /// <summary>The caller has requested the call; the server has not yet notified the callee.</summary>
    Initiating   = 0,

    /// <summary>The callee has at least one device ringing.</summary>
    Ringing      = 1,

    /// <summary>A callee device accepted. Signalling has not completed.</summary>
    Accepted     = 2,

    /// <summary>Signalling is in progress.</summary>
    Connecting   = 3,

    /// <summary>Media is flowing, as reported by the participants.</summary>
    Connected    = 4,

    /// <summary>Ended normally by a participant.</summary>
    Ended        = 5,

    /// <summary>The callee explicitly declined.</summary>
    Rejected     = 6,

    /// <summary>The callee was already in a call.</summary>
    Busy         = 7,

    /// <summary>The caller withdrew before it was answered.</summary>
    Cancelled    = 8,

    /// <summary>Signalling or media setup failed.</summary>
    Failed       = 9,

    /// <summary>Nobody answered within the ring timeout.</summary>
    Timeout      = 10,

    /// <summary>A participant's connection was lost after the call connected.</summary>
    Disconnected = 11,
}

[Flags]
public enum CallMediaKind
{
    None        = 0,
    Audio       = 1,
    Video       = 2,
    ScreenShare = 4,
}

public enum CallEndReason
{
    Unspecified          = 0,
    CallerHungUp         = 1,
    CalleeHungUp         = 2,
    CallerCancelled      = 3,
    CalleeRejected       = 4,
    CalleeBusy           = 5,
    RingTimeout          = 6,
    ConnectingTimeout    = 7,
    SignalingFailed      = 8,
    CallerDisconnected   = 9,
    CalleeDisconnected   = 10,
    ServerShutdown       = 11,
    Superseded           = 12,
}

public sealed record CallParticipant
{
    public required string UserId { get; init; }

    /// <summary>The connection that accepted, once one has. Null while ringing.</summary>
    public string? ConnectionId { get; init; }

    public bool IsMuted { get; init; }
    public CallMediaKind Media { get; init; }
}

public sealed record Call
{
    public required string CallId { get; init; }
    public required string CallerId { get; init; }
    public required string CalleeId { get; init; }
    public required CallState State { get; init; }
    public required CallMediaKind RequestedMedia { get; init; }
    public required DateTimeOffset CreatedAt { get; init; }
    public required DateTimeOffset UpdatedAt { get; init; }

    public DateTimeOffset? AnsweredAt { get; init; }
    public DateTimeOffset? EndedAt { get; init; }
    public CallEndReason? EndReason { get; init; }

    /// <summary>Created when the call is accepted, not when it is invited.</summary>
    public string? SignalingSessionId { get; init; }

    /// <summary>Always two participants in this version.</summary>
    public required IReadOnlyList<CallParticipant> Participants { get; init; }

    public IReadOnlyDictionary<string, string> Metadata { get; init; }

    public bool IsActive { get; }        // Initiating..Connected
    public bool IsTerminal { get; }      // Ended..Disconnected
    public bool IsParticipant(string userId);
    public string? GetPeerOf(string userId);
}
```

**Requirements — the state machine**

`CallStateMachine` is a pure static type with an explicit transition table, mirroring Phase 08's
`DeliveryStateMachine`:

```csharp
public static class CallStateMachine
{
    public static bool CanTransition(CallState from, CallState to);
    public static bool IsTerminal(CallState state);
    public static bool IsActive(CallState state);

    /// <summary>Returns the end reason implied by a transition, or null when it must be supplied.</summary>
    public static CallEndReason? ImpliedEndReason(CallState from, CallState to);
}
```

Legal transitions — the complete table, to be encoded exactly:

| From | To |
|---|---|
| `Initiating` | `Ringing`, `Cancelled`, `Busy`, `Failed`, `Timeout` |
| `Ringing` | `Accepted`, `Rejected`, `Cancelled`, `Timeout`, `Busy`, `Failed` |
| `Accepted` | `Connecting`, `Failed`, `Ended`, `Disconnected` |
| `Connecting` | `Connected`, `Failed`, `Ended`, `Disconnected` |
| `Connected` | `Ended`, `Disconnected`, `Failed` |
| any terminal | *(nothing)* |

- 12 states means 144 pairs. The table test must assert **all 144**, not just the legal ones. Every
  illegal transition must be rejected. This is the highest-value test suite in the phase: every user
  complaint about calling ("it said connected but there was no audio", "the call rang after I hung up",
  "I could accept a cancelled call") is one wrong cell.
- Terminal states are absorbing. `CanTransition(Ended, Connected)` is false. There is no resurrection —
  a reconnecting participant joins a **new** call, and the docs must say so.
- `Busy` is reachable from both `Initiating` and `Ringing`: from `Initiating` when the callee is already
  in a call at invite time, and from `Ringing` when a second call arrives (the second call goes `Busy`,
  the first is untouched).
- `Timeout` from `Initiating` covers "the callee has no live connection to ring at all". Do **not** add
  a separate `Unreachable` state; `Timeout` with `CallEndReason.RingTimeout` is sufficient, and one
  fewer state is one fewer row of table. Document the mapping.
- `ImpliedEndReason` exists so callers cannot set a contradictory pair (state `Rejected` with reason
  `CallerHungUp`). Where the transition determines the reason, it returns it; where it does not (e.g.
  `Connected → Ended` could be either party), it returns `null` and the caller must supply one. Test
  that every legal terminal transition either implies a reason or is documented as requiring one.

**Tests**

| Test | Asserts |
|---|---|
| `Every_state_pair_has_a_defined_transition_result` | all 144, no exception |
| `Illegal_transitions_are_rejected` | explicit list, table-driven |
| `Terminal_states_are_absorbing` | 7 terminal states × 12 targets, all false |
| `Ringing_can_be_reached_only_from_initiating` | |
| `Accepted_can_be_reached_only_from_ringing` | |
| `Connected_can_be_reached_only_from_connecting` | |
| `Busy_is_reachable_from_initiating_and_ringing` | |
| `Cancelled_is_not_reachable_after_accepted` | **the "cancel a call in progress" bug** |
| `Rejected_is_not_reachable_after_accepted` | |
| `Disconnected_is_not_reachable_before_accepted` | ringing loss is `Timeout`, not `Disconnected` |
| `Is_active_and_is_terminal_partition_every_state` | no state is both or neither |
| `Implied_end_reason_is_consistent_for_every_legal_terminal_transition` | |
| `Implied_end_reason_is_null_only_where_documented` | |
| `Call_is_participant_is_true_for_caller_and_callee` | |
| `Call_get_peer_of_returns_the_other_party` | both directions |
| `Media_kind_flags_combine_as_expected` | `Audio \| Video` |
| `State_and_reason_numeric_values_are_pinned` | wire contract |
| `State_machine_is_pure` | repeated calls, no allocation, no time |

---

### P11.T2 — Store, options, and `ICallService`

**Deliverables**

```
src/LiveSharp.Calls/Storage/ICallStore.cs
src/LiveSharp.Calls/Storage/InMemoryCallStore.cs
src/LiveSharp.Calls/Configuration/CallOptions.cs
src/LiveSharp.Calls/Configuration/CallOptionsValidator.cs
src/LiveSharp.Calls/ICallService.cs
src/LiveSharp.Calls/CallService.cs
src/LiveSharp.Calls/Diagnostics/CallLog.cs
tests/LiveSharp.Calls.Tests/Storage/CallStoreContractTests.cs
tests/LiveSharp.Calls.Tests/CallServiceTests.cs
```

**Public API (exact):**

```csharp
namespace LiveSharp.Calls;

public interface ICallService
{
    Task<OperationResult<Call>> InviteAsync(string callerId, InviteRequest request, RealtimeOperationContext? context = null, CancellationToken cancellationToken = default);
    Task<OperationResult<Call>> AcceptAsync(string calleeId, string callId, string connectionId, CancellationToken cancellationToken = default);
    Task<OperationResult<Call>> RejectAsync(string calleeId, string callId, CancellationToken cancellationToken = default);
    Task<OperationResult<Call>> CancelAsync(string callerId, string callId, CancellationToken cancellationToken = default);
    Task<OperationResult<Call>> HangUpAsync(string actingUserId, string callId, CancellationToken cancellationToken = default);

    /// <summary>Reported by a participant once media is flowing.</summary>
    Task<OperationResult<Call>> ReportConnectedAsync(string actingUserId, string callId, CancellationToken cancellationToken = default);

    Task<OperationResult<Call>> SetMuteAsync(string actingUserId, string callId, bool muted, CancellationToken cancellationToken = default);
    Task<OperationResult<Call>> GetCallAsync(string actingUserId, string callId, CancellationToken cancellationToken = default);
    Task<OperationResult<IReadOnlyList<Call>>> GetActiveCallsAsync(string userId, CancellationToken cancellationToken = default);
}

public sealed record InviteRequest
{
    public required string CalleeId { get; init; }
    public CallMediaKind Media { get; init; } = CallMediaKind.Audio;
    public IReadOnlyDictionary<string, string>? Metadata { get; init; }
}
```

```csharp
namespace LiveSharp.Calls.Storage;

public interface ICallStore
{
    ValueTask AddAsync(Call call, CancellationToken cancellationToken = default);
    ValueTask<Call?> FindAsync(string callId, CancellationToken cancellationToken = default);
    ValueTask<bool> TryUpdateAsync(Call call, DateTimeOffset expectedUpdatedAt, CancellationToken cancellationToken = default);
    ValueTask<IReadOnlyList<Call>> GetActiveCallsForUserAsync(string userId, CancellationToken cancellationToken = default);
    ValueTask<int> GetActiveCallCountForUserAsync(string userId, CancellationToken cancellationToken = default);
    ValueTask<IReadOnlyList<Call>> GetStuckAsync(CallState state, DateTimeOffset olderThan, int limit, CancellationToken cancellationToken = default);
    ValueTask<int> PruneTerminalAsync(DateTimeOffset olderThan, int limit, CancellationToken cancellationToken = default);
}
```

**Public API (exact) — options:**

```csharp
public sealed class CallOptions
{
    public const string ConfigurationSectionName = "LiveSharp:Calls";

    /// <summary>How long a call rings before timing out. Default 45 seconds.</summary>
    public TimeSpan RingTimeout { get; set; } = TimeSpan.FromSeconds(45);

    /// <summary>How long signalling may take after acceptance. Default 30 seconds.</summary>
    public TimeSpan ConnectingTimeout { get; set; } = TimeSpan.FromSeconds(30);

    /// <summary>Grace period for a lost connection during a connected call before ending it. Default 20 seconds.</summary>
    public TimeSpan ReconnectGracePeriod { get; set; } = TimeSpan.FromSeconds(20);

    /// <summary>Maximum simultaneous active calls per user. Default 1.</summary>
    public int MaxConcurrentCallsPerUser { get; set; } = 1;

    /// <summary>Maximum call duration before the server ends it. Default 4 hours. Null disables.</summary>
    public TimeSpan? MaxCallDuration { get; set; } = TimeSpan.FromHours(4);

    /// <summary>How long terminal calls are retained in the store. Default 24 hours.</summary>
    public TimeSpan TerminalCallRetention { get; set; } = TimeSpan.FromHours(24);

    /// <summary>Timeout sweep interval. Default 5 seconds.</summary>
    public TimeSpan SweepInterval { get; set; } = TimeSpan.FromSeconds(5);

    /// <summary>Media kinds the server will accept in an invite. Default Audio.</summary>
    public CallMediaKind AllowedMedia { get; set; } = CallMediaKind.Audio;
}
```

**Requirements**

- `TryUpdateAsync` with `expectedUpdatedAt` — the same optimistic-concurrency pattern as Phases 06 and
  10. Call state is written from six concurrent sources (both participants, the ring-timeout sweep, the
  connecting-timeout sweep, disconnect handlers, the duration sweep). Last-write-wins here means a
  call can transition out of a terminal state, which is exactly what `P11.T1`'s absorbing states are
  supposed to prevent. **Every** state change must be a read → validate transition → `TryUpdateAsync`
  loop, retried up to 3 times, then `Conflict`.
- `AllowedMedia` defaults to `Audio` only, so an invite requesting `Video` in `0.10.0` is rejected with
  a clear message rather than half-working. Phase 12 changes the default. This is how a flags enum
  ships ahead of its implementation honestly.
- `MaxCallDuration` exists because a browser tab left open in a call forever holds a TURN allocation
  and a call slot. Four hours is arbitrary but finite; document that it is a safety valve, not a
  product limit.
- `ReportConnectedAsync` is called by a participant when its `RTCPeerConnection` reaches
  `connected`. The server cannot observe media, so `Connected` is participant-reported — exactly the
  same reasoning as Phase 08's `Delivered`. Reference that decision explicitly in the docs so the
  pattern reads as deliberate consistency rather than repeated hedging. Either participant reporting is
  sufficient to move `Connecting → Connected`.
- Validator rules: all durations positive; `SweepInterval < RingTimeout` and `< ConnectingTimeout`;
  `MaxConcurrentCallsPerUser >= 1`; `MaxCallDuration`, when set, `> ConnectingTimeout`;
  `TerminalCallRetention > TimeSpan.Zero`; `AllowedMedia != None` with a message explaining that
  `None` disables calling entirely and `Enabled = false` is not a thing — remove the option instead.
- `CallStoreContractTests` abstract suite, same pattern as every prior store.
- Never log call metadata or participant user IDs above `Debug`. A call log at `Information` that names
  both parties reconstructs a call-detail record in your log aggregator, which has real privacy and
  regulatory weight. Log call IDs and states; document that applications wanting CDRs should subscribe
  to call events and store them deliberately.

**Tests — store contract**

| Test | Asserts |
|---|---|
| `Add_then_find_returns_the_call` | |
| `Find_unknown_call_returns_null` | |
| `Try_update_with_a_matching_token_succeeds` | |
| `Try_update_with_a_stale_token_fails_and_does_not_write` | **the concurrency primitive** |
| `Get_active_calls_excludes_terminal_calls` | |
| `Get_active_call_count_matches_get_active_calls` | |
| `Get_stuck_returns_only_calls_in_the_requested_state_older_than_the_cutoff` | |
| `Get_stuck_does_not_return_calls_at_exactly_the_cutoff` | boundary |
| `Get_stuck_honours_the_limit` | |
| `Prune_terminal_removes_only_terminal_calls` | active calls survive |
| `Prune_terminal_honours_the_limit` | |
| `Concurrent_updates_result_in_exactly_one_winner_per_token` | `ConcurrencyHarness` |
| `All_methods_honour_a_pre_cancelled_token` | |

**Tests — service, happy and unhappy paths**

| Test | Asserts |
|---|---|
| `Invite_creates_a_call_in_initiating_then_ringing` | |
| `Invite_to_a_user_with_no_live_connection_ends_as_timeout_with_ring_timeout` | |
| `Invite_to_yourself_is_rejected` | |
| `Invite_requesting_disallowed_media_is_rejected_naming_the_allowed_kinds` | video in 0.10.0 |
| `Invite_when_the_caller_is_already_at_the_call_limit_is_rejected_with_conflict` | |
| `Invite_when_the_callee_is_already_in_a_call_ends_as_busy` | and the existing call is untouched |
| `Invite_does_not_create_a_signaling_session` | **required** — see `P11.T4` |
| `Accept_moves_the_call_to_accepted_and_records_the_answering_connection` | |
| `Accept_stamps_answered_at_from_the_time_provider` | |
| `Accept_by_the_caller_is_forbidden` | |
| `Accept_by_a_third_party_is_forbidden` | |
| `Accept_of_a_cancelled_call_is_rejected_with_conflict` | |
| `Accept_of_an_already_accepted_call_is_rejected_with_conflict` | |
| `Reject_moves_the_call_to_rejected_with_callee_rejected` | |
| `Reject_by_the_caller_is_forbidden` | |
| `Cancel_by_the_caller_moves_the_call_to_cancelled` | |
| `Cancel_by_the_callee_is_forbidden` | |
| `Cancel_after_acceptance_is_rejected_with_conflict` | |
| `Hang_up_by_either_participant_ends_the_call_with_the_right_reason` | caller vs callee |
| `Hang_up_by_a_third_party_is_forbidden` | |
| `Hang_up_of_a_terminal_call_is_idempotent_and_returns_the_existing_call` | double hang-up is normal |
| `Report_connected_from_either_participant_moves_connecting_to_connected` | |
| `Report_connected_before_acceptance_is_rejected` | |
| `Report_connected_twice_is_idempotent` | |
| `Set_mute_updates_only_the_calling_participant` | |
| `Set_mute_by_a_third_party_is_forbidden` | |
| `Set_mute_on_a_terminal_call_is_rejected` | |
| `Get_call_by_a_non_participant_is_forbidden_and_indistinguishable_from_unknown` | no oracle |
| `Store_conflict_is_retried_and_eventually_succeeds` | |
| `Store_conflict_exhausted_returns_conflict` | |
| `Participant_user_ids_are_never_logged_above_debug` | **security test** |
| `All_methods_honour_a_pre_cancelled_token` | |

---

### P11.T3 — Multi-device ringing

> ## Exactly one device may win
> Bob has a laptop, a phone, and a tablet. All three ring. He taps accept on the phone while reaching
> for the laptop. If both accepts succeed, there are two answering connections, two signalling sessions,
> and a call that will never connect properly.
>
> This is a compare-and-swap problem, not a "check then set" problem, and a naive implementation passes
> every single-threaded test.

**Deliverables**

```
src/LiveSharp.Calls/Ringing/CallRinger.cs
src/LiveSharp.Calls/Ringing/CallAcceptCoordinator.cs
tests/LiveSharp.Calls.Tests/Ringing/MultiDeviceRingingTests.cs
tests/LiveSharp.Calls.Tests/Ringing/AcceptRaceTests.cs
```

**Requirements**

1. **Ring every device.** `InviteAsync` resolves the callee's live connections from
   `IConnectionRegistry` and sends `call.incoming` to each. If there are none, the call ends as
   `Timeout`/`RingTimeout` immediately — do not leave it `Initiating` for 45 seconds.
2. **The first accept wins.** `CallAcceptCoordinator` serialises accepts **per call ID** using a
   striped `SemaphoreSlim` — the same technique as Phase 02's connection limit and Phase 05's member
   limit, and for the same reason: never a global lock. Inside the gate: re-read the call, validate
   `Ringing → Accepted` via `CallStateMachine`, `TryUpdateAsync`. A loser sees the call already in
   `Accepted` and returns `Conflict` with a message the client can render as "answered on another
   device" rather than an error.
3. **Losers must be told to stop ringing.** After a successful accept, send `call.state.changed` to
   **every** callee connection **except** the accepting one, and also to every caller connection. A
   device that keeps ringing after the call was answered elsewhere is the single most noticeable bug in
   this feature. Test it with three callee connections.
4. **A device disconnecting mid-ring does not end the call** unless it was the last one. Re-resolve
   live connections on each callee disconnect; if zero remain and the call is still `Ringing`, end it
   as `Timeout`. If some remain, do nothing.
5. **A device connecting mid-ring joins the ring.** If Bob's phone comes online while the laptop is
   ringing, ring the phone too. Implement in `CallLifecycleHandler.OnConnectedAsync` by checking for
   active `Ringing` calls where the connecting user is the callee. Cap the work (a user cannot have
   more than `MaxConcurrentCallsPerUser` active calls, so this is inherently bounded) and log at
   `Debug`.
6. **Caller side.** The caller may have several devices too. `call.state.changed` goes to all of them,
   so the tab that did not initiate still shows the call. The accepting *caller* connection is not a
   concept — the caller's `ConnectionId` is recorded from the invite's originating connection when
   `context` is non-null.

**Tests**

| Test | Asserts |
|---|---|
| `Invite_rings_every_callee_connection` | 3 connections → 3 `call.incoming` |
| `Invite_rings_no_caller_connection_with_call_incoming` | callers get state changes, not incoming |
| `Invite_with_zero_callee_connections_ends_immediately_as_timeout` | not left ringing |
| `Accept_notifies_every_other_callee_connection_to_stop_ringing` | **the stuck-ringing test** |
| `Accept_notifies_every_caller_connection` | |
| `Accept_does_not_send_a_stop_notification_to_the_accepting_connection` | it already knows |
| `Second_accept_from_another_device_returns_conflict` | |
| `Second_accept_conflict_message_indicates_it_was_answered_elsewhere` | renderable by a client |
| `Concurrent_accepts_from_three_devices_produce_exactly_one_winner` | **`ConcurrencyHarness`, 32 iterations** |
| `Concurrent_accepts_record_exactly_one_answering_connection` | |
| `Concurrent_accept_and_reject_resolve_to_exactly_one_terminal_or_accepted_state` | |
| `Concurrent_accept_and_cancel_never_produce_an_accepted_and_cancelled_call` | |
| `A_callee_device_disconnecting_mid_ring_leaves_the_call_ringing` | 2 of 3 remain |
| `The_last_callee_device_disconnecting_mid_ring_ends_the_call_as_timeout` | |
| `A_callee_device_connecting_mid_ring_starts_ringing` | |
| `A_callee_device_connecting_after_acceptance_does_not_start_ringing` | |
| `The_caller_disconnecting_mid_ring_cancels_the_call` | with `CallerDisconnected` |
| `Accept_coordination_does_not_serialise_unrelated_calls` | two calls accepted concurrently, both fast |

---

### P11.T4 — Timeouts, disconnect handling, glare, and signalling integration

**Deliverables**

```
src/LiveSharp.Calls/CallLifecycleHandler.cs
src/LiveSharp.Calls/CallTimeoutSweepService.cs
src/LiveSharp.Calls/CallPruneService.cs
src/LiveSharp.Calls/Security/ICallAuthorizationService.cs
tests/LiveSharp.Calls.Tests/CallTimeoutSweepServiceTests.cs
tests/LiveSharp.Calls.Tests/CallLifecycleHandlerTests.cs
tests/LiveSharp.Calls.Tests/Security/CallAuthorizationTests.cs
```

**Requirements — the signalling session is created on accept, not on invite**

An unanswered invite must not consume signalling or TURN resources. Bob's phone ringing for 45 seconds
should cost a `Call` row and three WebSocket messages, not a signalling session, a TURN allocation, and
a candidate buffer.

So: `AcceptAsync` calls `ISignalingService.CreateSessionAsync(callerId, calleeId)` and stores the
resulting `SignalingSessionId` on the `Call`, then transitions `Accepted → Connecting` and pushes
`call.state.changed` carrying the session ID and the ICE servers. Both clients then run perfect
negotiation on that session, with the **caller as the impolite peer** — which falls out naturally
because `CreateSessionAsync(callerId, …)` makes the caller the session initiator, and Phase 10 already
defined the initiator as impolite. That alignment is not an accident; state it in the docs so nobody
"simplifies" the argument order.

Failure to create the session → `Accepted → Failed` with `SignalingFailed`, notify both parties, and
log at `Error`.

Ending a call in **any** terminal state must close the signalling session if one exists. A leaked
signalling session holds a TURN allocation. Put the close in one place that every terminal path routes
through — not duplicated across five methods — and test it from each terminal state.

**Requirements — timeouts**

`CallTimeoutSweepService : BackgroundService` at `SweepInterval`, three sweeps per tick:

| Sweep | Query | Action |
|---|---|---|
| Ring timeout | `GetStuckAsync(Ringing, now - RingTimeout)` | → `Timeout` / `RingTimeout`, notify both |
| Connecting timeout | `GetStuckAsync(Connecting, now - ConnectingTimeout)` and `GetStuckAsync(Accepted, …)` | → `Failed` / `ConnectingTimeout` |
| Max duration | `GetStuckAsync(Connected, now - MaxCallDuration)` | → `Ended` / `Unspecified`, notify both |

Same resilience contract as every prior background service. `Initiating` is also swept (with the ring
timeout) to catch a call that never reached `Ringing` because of a crash mid-invite.

**Requirements — disconnect and the reconnect grace period**

`CallLifecycleHandler.OnDisconnectedAsync`, when the disconnecting user's **last** connection goes:

- Call in `Initiating`/`Ringing`: caller lost → `Cancelled`/`CallerDisconnected`; callee lost → handled
  by `P11.T3` rule 4.
- Call in `Accepted`/`Connecting`/`Connected`: **do not end immediately.** Mark a grace deadline
  (`now + ReconnectGracePeriod`) in the call's metadata and let the sweep end it as
  `Disconnected` if the user has not reconnected by then. A five-second WiFi handover should not drop a
  call; the media path is peer-to-peer and often survives the signalling connection dropping entirely.

  This is the single most user-visible quality difference between a naive and a good implementation.
  Notify the surviving peer with a `call.state.changed` that says "peer connection lost, waiting" — do
  not silently do nothing, or the surviving user stares at a frozen UI.
- Reconnection within the grace period: `CallLifecycleHandler.OnConnectedAsync` clears the deadline and
  notifies the peer. The reconnected client must re-run negotiation if its `RTCPeerConnection` died;
  document that the application decides, using `RTCPeerConnection.connectionState`.

**Requirements — glare**

Alice invites Bob at the same instant Bob invites Alice. Two calls exist, each will make the other
`Busy`, and both can end up `Busy` — a deadlock where neither call connects and both users see a
failure.

Resolution rule: **the call whose caller has the lower ordinal user ID wins.** The other becomes
`Busy`/`Superseded`. It is arbitrary but it is deterministic, symmetric, and requires no extra round
trip — both servers (or both code paths on one server) reach the same conclusion from data they already
have. Alternatives considered and rejected: earliest `CreatedAt` (clock ties and equal timestamps are
common at millisecond resolution), random (non-deterministic, untestable), let both fail (the current
naive behaviour, and unacceptable).

Implement in `InviteAsync`'s busy-detection path: when the callee's existing active call is one where
*this caller* is the callee, apply the tie-break instead of returning `Busy`.

**Requirements — authorization**

```csharp
namespace LiveSharp.Calls.Security;

public interface ICallAuthorizationService
{
    ValueTask<bool> CanCallUserAsync(string callerId, string calleeId, CallMediaKind media, CancellationToken cancellationToken = default);
}
```

Default delegates to `ISignalingAuthorizationService.CanEstablishSessionAsync` (Phase 10), which
delegates to `CanSendToUserAsync` (Phase 09). Three layers, each overridable, each with a sensible
default, none silently permissive because Phase 09's root default logs an error at startup. The
`media` parameter exists so "may voice call" and "may video call" can differ — a real requirement in
regulated environments, and it costs one parameter now versus a breaking change in Phase 12.

Checked in `InviteAsync` **before** any store write and before ringing anybody.

**Tests**

| Test | Asserts |
|---|---|
| `Accept_creates_a_signaling_session_and_records_its_id` | |
| `Accept_makes_the_caller_the_signaling_initiator` | **the politeness alignment** |
| `Accept_notifies_both_parties_with_the_session_id_and_ice_servers` | |
| `Signaling_session_creation_failure_fails_the_call_and_notifies_both` | |
| `Every_terminal_transition_closes_the_signaling_session` | **table-driven over all 7 terminal states** |
| `A_call_that_never_accepted_closes_no_signaling_session` | nothing to leak |
| `Ring_timeout_ends_a_ringing_call_as_timeout` | `FakeTimeProvider` |
| `Ring_timeout_does_not_end_a_call_at_exactly_the_timeout` | boundary |
| `Initiating_calls_are_swept_by_the_ring_timeout` | crash recovery |
| `Connecting_timeout_fails_a_connecting_call` | |
| `Connecting_timeout_also_sweeps_accepted_calls` | |
| `Max_duration_ends_a_long_connected_call` | |
| `Null_max_duration_never_ends_a_call` | |
| `Sweep_survives_a_throwing_store` | |
| `Sweep_survives_a_throwing_signaling_service` | |
| `Caller_last_disconnect_during_ringing_cancels_the_call` | |
| `Participant_last_disconnect_during_a_connected_call_does_not_end_it_immediately` | **the grace period** |
| `Participant_last_disconnect_notifies_the_peer_that_the_connection_was_lost` | |
| `Grace_period_expiry_ends_the_call_as_disconnected` | |
| `Reconnection_within_the_grace_period_clears_the_deadline_and_notifies_the_peer` | |
| `Reconnection_after_the_grace_period_does_not_revive_a_terminal_call` | absorbing states hold |
| `Non_last_disconnect_during_a_connected_call_does_nothing` | multi-device |
| `Simultaneous_mutual_invites_resolve_deterministically_by_lower_caller_id` | **the glare test**, run both argument orders |
| `Simultaneous_mutual_invites_never_leave_both_calls_busy` | |
| `Glare_loser_ends_as_busy_with_superseded` | |
| `Invite_denied_by_the_call_authorization_service_is_forbidden` | |
| `Invite_denied_only_for_video_media_still_permits_audio` | the `media` parameter earns its place |
| `Authorization_is_checked_before_any_store_write_or_ringing` | |
| `Prune_service_removes_terminal_calls_past_retention` | |

---

### P11.T5 — Operations, handlers, and `AddCalls()`

**Deliverables**

```
src/LiveSharp.Calls/Protocol/*.cs
src/LiveSharp.Calls/Handlers/*.cs                          (seven handlers)
src/LiveSharp.Calls/DependencyInjection/LiveSharpCallsBuilderExtensions.cs
tests/LiveSharp.Calls.Tests/Handlers/…
tests/LiveSharp.Calls.Tests/RegistrationTests.cs
tests/LiveSharp.IntegrationTests/Calls/CallIntegrationTests.cs
```

Operations and events — add to `RealtimeOperations`/`RealtimeEvents`, register every DTO in
`LiveSharpJsonSerializerContext`, mirror in `protocol.ts`, and add every operation to the Phase 09
`KnownOperationPolicies` manifest:

```
call.invite            InvokeAsync      call.incoming
call.accept            InvokeAsync      call.state.changed
call.reject            InvokeAsync      call.ended
call.cancel            InvokeAsync
call.hangup            InvokeAsync
call.connected         NotifyAsync
call.mute.set          InvokeAsync
call.query             InvokeAsync
```

**Requirements**

- Every operation is `InvokeAsync` except `call.connected`. A user tapping "accept" must know whether
  it worked; there is no self-healing retry for a call. `call.connected` is telemetry-ish and
  self-corrects (the other participant reports it too, and the connecting timeout catches genuine
  failures), so it is fire-and-forget.
- `call.ended` is a **separate event** from `call.state.changed` even though `Ended` is a state. Clients
  overwhelmingly want a single "tear down the UI, stop the audio element, release the microphone" hook,
  and making them string-match on state values to find it is poor DX. Emit **both**: `call.state.changed`
  for every transition (so a client can build a full state display) and `call.ended` for terminal
  states (so the common case is one handler). Document the redundancy as deliberate.
- `AcceptRequest` carries no `ConnectionId` — it comes from `context.Connection.ConnectionId`. Every
  request DTO carries no actor identity. Reflection tests, as in every prior phase.
- Rate limits configured in this phase: `call.invite` 10/60s, `call.accept` 20/60s,
  `call.reject` 20/60s, `call.cancel` 20/60s, `call.hangup` 20/60s, `call.connected` 60/60s,
  `call.mute.set` 60/60s, `call.query` 30/60s. The invite limit is the important one — it is the
  spam/harassment vector.
- `AddCalls()` requires `AddSignaling()` (validated at startup, order-independent). It does **not**
  require chat, groups, or presence. Add a test proving a calling-only application composes — someone
  will build exactly that.
- **Presence interaction, done carefully.** If `AddPresence()` is registered, `InviteAsync` should
  consider the callee's presence: `DoNotDisturb` → end the call as `Busy`/`CalleeBusy` without ringing.
  But `LiveSharp.Calls` must not depend on `LiveSharp.Presence` (that would force presence on every
  calling consumer). Use the same solution as Phase 06's group bridge: an optional
  `ICallAdmissionPolicy` in `LiveSharp.Calls`, with a presence-aware implementation contributed by a
  bridge — or, simpler and better here, define `ICallAdmissionPolicy` in `LiveSharp.Calls` with a
  permissive default and let the **application** implement it. Reason to prefer the latter: whether DND
  blocks calls is a product decision (an emergency-services app wants DND overridden), so it should not
  be a package default at all. Document the two-line implementation in `docs/calls/README.md`.
  Do **not** create a `LiveSharp.Calls.Presence` bridge package for this.

**Tests**

| Test | Asserts |
|---|---|
| `Every_call_request_dto_lacks_an_actor_identity_member` | **security**, reflection |
| `Accept_handler_uses_the_connection_id_from_the_context` | |
| `Connected_handler_maps_failures_to_silent_success` | `NotifyAsync` contract |
| `Terminal_transitions_emit_both_state_changed_and_ended` | the deliberate redundancy |
| `Non_terminal_transitions_emit_only_state_changed` | |
| `AddCalls_registers_the_service_store_ringer_coordinator_sweeps_and_lifecycle_handler` | lifetimes |
| `AddCalls_registers_all_eight_operation_handlers` | |
| `AddCalls_is_idempotent` | |
| `AddCalls_without_AddSignaling_fails_at_startup_naming_AddSignaling` | |
| `AddCalls_before_AddSignaling_still_works` | order independence |
| `AddCalls_composes_without_chat_presence_or_groups` | **the decoupling proof** |
| `Calls_assembly_references_signaling_but_not_chat_or_presence` | architecture test |
| `A_custom_call_admission_policy_can_reject_an_invite` | |
| `The_default_call_admission_policy_permits_everything` | |
| `Every_call_operation_appears_in_the_known_policies_manifest` | Phase 09 gate |

**Integration tests (both transports)** — real connections, real signalling, **no media assertions**:

| Test | Asserts |
|---|---|
| `Full_happy_path_invite_ring_accept_negotiate_connected_hangup` | the whole lifecycle |
| `Callee_receives_call_incoming_on_every_device` | |
| `Accepting_on_one_device_stops_ringing_on_the_others` | |
| `Caller_receives_state_changes_throughout` | |
| `Reject_ends_the_call_and_notifies_the_caller` | |
| `Cancel_before_answer_stops_ringing_on_every_device` | |
| `Hang_up_notifies_the_peer_with_call_ended` | |
| `Ring_timeout_ends_the_call_and_notifies_both` | real short timeout |
| `Invite_to_an_offline_user_returns_a_timeout_call_immediately` | |
| `Invite_to_a_busy_user_returns_a_busy_call_and_does_not_disturb_the_active_call` | |
| `A_third_user_receives_nothing_about_the_call` | **isolation** |
| `The_signaling_session_created_on_accept_is_usable_for_offer_answer_exchange` | ties Phases 10 and 11 together |
| `The_signaling_session_is_closed_when_the_call_ends` | **the TURN leak test** |
| `Mute_state_change_reaches_the_peer` | |
| `Twenty_concurrent_calls_between_forty_users_all_complete_independently` | isolation under load |

---

### P11.T6 — Browser call helper and example applications

**Deliverables**

```
clients/dotnet/LiveSharp.Client/Calls/LiveSharpCallClientExtensions.cs
clients/js/packages/client/src/calls.ts
clients/js/packages/client/src/audio-call.ts
clients/js/packages/client/src/__tests__/calls.test.ts
clients/js/packages/client/src/__tests__/audio-call.test.ts
examples/AspNetCoreMvc/wwwroot/js/call.js
examples/React/src/features/calls/…
docs/calls/*.md
```

**Public API (exact) — TypeScript:**

```ts
export interface CallInfo {
  callId: string;
  callerId: string;
  calleeId: string;
  state: 'initiating' | 'ringing' | 'accepted' | 'connecting' | 'connected'
       | 'ended' | 'rejected' | 'busy' | 'cancelled' | 'failed' | 'timeout' | 'disconnected';
  requestedMedia: { audio: boolean; video: boolean; screenShare: boolean };
  signalingSessionId?: string;
  participants: Array<{ userId: string; isMuted: boolean }>;
}

export function invite(client: LiveSharpClient, calleeId: string, options?: { metadata?: Record<string, string> }): Promise<CallInfo>;
export function accept(client: LiveSharpClient, callId: string): Promise<CallInfo>;
export function reject(client: LiveSharpClient, callId: string): Promise<CallInfo>;
export function cancel(client: LiveSharpClient, callId: string): Promise<CallInfo>;
export function hangUp(client: LiveSharpClient, callId: string): Promise<CallInfo>;
export function setMute(client: LiveSharpClient, callId: string, muted: boolean): Promise<CallInfo>;

export function onIncomingCall(client: LiveSharpClient, handler: (call: CallInfo) => void): Unsubscribe;
export function onCallStateChanged(client: LiveSharpClient, handler: (call: CallInfo) => void): Unsubscribe;
export function onCallEnded(client: LiveSharpClient, handler: (call: CallInfo) => void): Unsubscribe;

export interface AudioCallHandle {
  readonly callId: string;
  readonly peerConnection: RTCPeerConnection;
  readonly localStream: MediaStream;
  readonly remoteStream: MediaStream;
  setMuted(muted: boolean): Promise<void>;
  hangUp(): Promise<void>;
}

/**
 * Acquires the microphone, attaches an RTCPeerConnection to the call's signalling
 * session, and reports connectivity to the server. Returns the raw peer connection
 * and streams so applications retain full control.
 */
export function startAudioCall(client: LiveSharpClient, call: CallInfo): Promise<AudioCallHandle>;
```

**Requirements — the browser helper**

- `startAudioCall` orchestrates: `getIceServers()` → `getUserMedia({ audio: true })` →
  `attachPeerConnection(client, { sessionId, polite })` from Phase 10 → add the audio track → wire
  `ontrack` into `remoteStream` → on `connectionState === 'connected'`, `notify('call.connected')`.
- Politeness is derived from the call, not passed in: `polite = (myUserId !== call.callerId)`. The
  application must never compute it.
- **`getUserMedia` must be called only after the user has accepted.** Requesting the microphone while
  a call is merely ringing triggers a permission prompt during an incoming call, which is both a
  terrible experience and, in some browsers, an automatic denial. Enforce it: `startAudioCall` rejects
  with an actionable error if `call.state` is not `accepted` or `connecting`. Test it.
- Errors from `getUserMedia` (`NotAllowedError`, `NotFoundError`, `NotReadableError`) must be mapped to
  distinct, actionable error messages — "microphone permission denied", "no microphone found",
  "microphone is in use by another application". A generic failure here is the top support question for
  any calling feature. Then hang up the call with a reason so the peer is not left waiting.
- `hangUp()` must stop **every** local track (`track.stop()`), close the `RTCPeerConnection`, remove
  every listener, unsubscribe from every LiveSharp event, and call `call.hangup`. Failing to
  `track.stop()` leaves the browser's microphone-in-use indicator lit after the call ends — visible,
  alarming, and a guaranteed bug report. Explicit test.
- `AudioCallHandle` exposes the raw `RTCPeerConnection` and both `MediaStream`s. Helper, not
  abstraction; same contract as Phase 10.
- No DOM at module scope; actionable error on use outside a browser. Extend the Phase 03 SSR test.

**Requirements — examples**

MVC and React both get a genuinely usable call UI, because a calling example that is hard to use
teaches nothing:

- Incoming-call toast with caller identity, Accept and Decline buttons.
- In-call bar: peer identity, elapsed timer, mute toggle, hang-up.
- Visible state readout: call state, `RTCPeerConnection.connectionState`, and the selected candidate
  pair type from `getStats()` (host / srflx / relay) — carried over from Phase 10, and the thing that
  tells a developer whether TURN is working.
- An audible ring is acceptable in the MVC example if it can be done with a short data-URI tone and is
  muted by default; do not add an audio asset dependency.
- Both READMEs: how to test with two browsers, what to do when it does not connect (the TURN
  checklist), permission requirements, and that the example is 1:1 only.

**Tests**

| Test | Asserts |
|---|---|
| `call constants match artifacts/protocol-names.json` | contract test |
| `invite resolves with the call info` | |
| `politeness is derived from the caller id` | both directions |
| `startAudioCall rejects when the call is still ringing` | **the permission-prompt rule** |
| `startAudioCall requests only audio` | constraints asserted |
| `startAudioCall reports connected when the peer connection connects` | |
| `getUserMedia NotAllowedError maps to a permission message and hangs up` | |
| `getUserMedia NotFoundError maps to a no-device message` | |
| `getUserMedia NotReadableError maps to a device-in-use message` | |
| `hangUp stops every local track` | **the microphone-indicator test** |
| `hangUp closes the peer connection and unsubscribes` | leak test |
| `hangUp is idempotent` | |
| `onCallEnded fires for every terminal state` | table-driven |
| `calls module imports cleanly in node with no DOM globals` | SSR safety |
| `startAudioCall throws an actionable error outside a browser` | on use, not import |

---

## Dependency justification

| Package | Why | Alternative? | Licence | Notes |
|---|---|---|---|---|
| *(none)* | `LiveSharp.Calls` uses only the BCL plus project references to `LiveSharp.Core`, `LiveSharp.AspNetCore`, and `LiveSharp.Signaling`. | — | — | The call state machine, the accept coordinator, and the sweeps are all plain C#. Any package here would be a red flag. |

Explicitly rejected: `SIPSorcery` — still not needed, because the server does not participate in media.
`Stateless` or any workflow/state-machine library — the transition table is a `bool[12,12]` and a switch;
a library would add a dependency, obscure the table, and make the exhaustive test harder to write.

---

## Documentation deltas

- `docs/calls/README.md` — **new**. What calling is and is not (1:1 only, no SFU, no recording, no
  PSTN), architecture, the `AddSignaling().AddCalls()` quick start, the `ICallAdmissionPolicy`
  two-liner for DND, and a note that production apps need push notifications for backgrounded clients.
- `docs/calls/lifecycle.md` — **new and the reference page for this phase**. The state diagram, the
  **complete 144-cell transition table** rendered as a matrix, every `CallEndReason` and what produces
  it, and the timeout table.
- `docs/calls/audio.md` — **new**. End-to-end client walkthrough, `getUserMedia` timing and why,
  permission-error handling, mute semantics, track cleanup, and the multi-device accept behaviour from
  the client's point of view.
- `docs/calls/troubleshooting.md` — **new**. "Rings but never connects" (TURN), "one-way audio"
  (usually a track not added, or a firewall), "microphone indicator stays on after hanging up"
  (`track.stop()`), "call answered on one device but another keeps ringing", "call drops after 45
  seconds" (ring timeout vs a stuck accept), "permission prompt appears during ringing",
  "calls fail after 4 hours" (`MaxCallDuration`), "second call gets busy" (`MaxConcurrentCallsPerUser`).
- `docs/webrtc/perfect-negotiation.md` — add the note that the call's caller is the signalling
  initiator and therefore the impolite peer.
- `docs/security/authorization.md` — the eight call operations, and `ICallAuthorizationService` with the
  `media` parameter rationale.
- `docs/security/rate-limiting.md` — call operation limits, with `call.invite` flagged as the
  harassment vector.
- `docs/security/README.md` — call metadata and participant identifiers are not logged above `Debug`;
  applications wanting call-detail records must collect them deliberately from call events.
- `docs/configuration/README.md`, `docs/configuration/service-lifetimes.md` — `CallOptions` and the new
  services.
- `README.md` — audio calls → `Available`; video and screen sharing remain `Planned`.

## Example deltas

- `examples/AspNetCoreMvc` — audio calling with the full call UI.
- `examples/React` — audio calling with the full call UI plus the WebRTC state readout.
- Both READMEs — testing instructions and the TURN checklist.

## CHANGELOG entry

```markdown
### Added
- `LiveSharp.Calls`: one-to-one audio calling with a fully specified twelve-state lifecycle.
- Multi-device ringing: every device rings, the first to accept wins, and the rest are told to stop.
- Ring, connecting, and maximum-duration timeouts, plus a reconnection grace period so a brief network
  interruption does not end a connected call.
- Deterministic glare resolution for simultaneous mutual invites.
- `ICallAuthorizationService` and `ICallAdmissionPolicy` for per-media call permission and admission
  decisions.
- `startAudioCall` browser helper handling microphone acquisition, negotiation, and cleanup, returning
  the raw `RTCPeerConnection` and media streams.
- Audio calling in the MVC and React examples, including WebRTC diagnostics.

### Notes
- Calls are one-to-one. Conference calling requires a selective forwarding unit and is out of scope.
- `Connected` is reported by the participants, not inferred by the server, for the same reason message
  delivery is client-acknowledged.
- The signalling session is created when a call is accepted, so unanswered calls consume no TURN
  resources.
```

---

## Exit criteria

- [ ] `Every_state_pair_has_a_defined_transition_result` covers all **144** pairs.
- [ ] `Terminal_states_are_absorbing` passes for all 7 terminal states.
- [ ] `Cancelled_is_not_reachable_after_accepted` passes.
- [ ] `Concurrent_accepts_from_three_devices_produce_exactly_one_winner` passes, without a global lock.
- [ ] `Accept_notifies_every_other_callee_connection_to_stop_ringing` passes.
- [ ] `Invite_does_not_create_a_signaling_session` passes.
- [ ] `Every_terminal_transition_closes_the_signaling_session` passes from all 7 terminal states.
- [ ] `Participant_last_disconnect_during_a_connected_call_does_not_end_it_immediately` passes.
- [ ] `Simultaneous_mutual_invites_never_leave_both_calls_busy` passes in both argument orders.
- [ ] `Invite_requesting_disallowed_media_is_rejected_naming_the_allowed_kinds` passes — video is
      honestly unavailable in `0.10.0`.
- [ ] `AddCalls_composes_without_chat_presence_or_groups` passes.
- [ ] `Calls_assembly_references_signaling_but_not_chat_or_presence` passes.
- [ ] `startAudioCall rejects when the call is still ringing` passes.
- [ ] `hangUp stops every local track` passes.
- [ ] All three `getUserMedia` error mappings pass.
- [ ] `Participant_user_ids_are_never_logged_above_debug` passes.
- [ ] `docs/calls/lifecycle.md` contains the complete 144-cell transition matrix.
- [ ] Two browsers complete an audio call in both the MVC and React examples, and the candidate-pair
      readout is visible.
- [ ] Every call operation is in the Phase 09 `KnownOperationPolicies` manifest.
- [ ] Call constants in `protocol.ts`; contract test green.
- [ ] `scripts/verify.ps1` green; tag `v0.10.0`.

## Verify

```powershell
pwsh scripts/verify.ps1
dotnet test tests/LiveSharp.Calls.Tests -c Release
dotnet test tests/LiveSharp.Calls.Tests -c Release --filter "FullyQualifiedName~StateMachine|FullyQualifiedName~AcceptRace"
dotnet test tests/LiveSharp.IntegrationTests -c Release --filter "FullyQualifiedName~Call"
pnpm --dir clients/js -r test
git tag v0.10.0
```

## Next

`docs/plan/phase-12-video-calling.md`, task `P12.T1`.
