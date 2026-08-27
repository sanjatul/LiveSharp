# Phase 08 — Message Delivery State

| | |
|---|---|
| **Version at exit** | `0.7.0` (tag `v0.7.0`) |
| **Depends on** | Phase 07 |
| **Branch** | `feat/phase-08-message-state` |
| **Tasks** | `P08.T1` … `P08.T5` |
| **Read before starting** | `docs/plan/phase-04-direct-messaging.md`, `docs/plan/phase-05-groups.md`, `docs/architecture/observers.md` |

---

## Goal

`Sent → Delivered → Read`, per recipient, for direct and group messages — with idempotent
acknowledgements, tolerance for out-of-order and duplicate acks, and batching so that an active
conversation does not generate O(n²) receipt traffic.

This phase also closes the honesty gap Phase 04 opened: with `OfflineMessageBehavior.DropSilently`,
`SendDirectAsync` returns `Success` for a message nobody saw. Receipts are what make that observable
to the application instead of a silent lie.

---

## Decision: `Delivered` is client-acknowledged, not server-inferred

The tempting implementation marks a message `Delivered` when `IRealtimeTransport.SendToUserAsync`
returns without throwing. That is wrong and must not be built.

A successful transport send means "the socket accepted the frame". It does not mean the browser tab
was alive, the JavaScript handler ran, or the message reached the UI. Marking `Delivered` on send
produces a double-tick for messages the recipient never saw — the exact failure mode users notice and
lose trust over.

So: the **client** sends `chat.receipt.delivered` when its handler has processed the message, and
`chat.receipt.read` when the application declares it read. The server records `Sent` on acceptance and
nothing more.

Consequence to document honestly: a client that never acks leaves messages at `Sent` forever. That is
correct — `Sent` is the truth. Both shipped clients ack automatically on handler completion so the
common case works with no application code.

Write this up in `docs/chat/delivery-state.md` under "Why Delivered is not inferred".

---

## Non-goals — DO NOT implement in this phase

- No unread counts, no "last read position", no conversation-level read markers. Those need the
  Phase 14 history/pagination API to be meaningful; building them on top of receipt rows alone
  produces an API that cannot be implemented efficiently against a database.
- No persistence provider. `InMemoryMessageStateStore` only; Phase 14.
- No authorization. **A client can currently ack a message addressed to someone else.** This is a real
  vulnerability, documented here, pinned by a test, and closed in Phase 09 by
  `IRealtimeAuthorizationService.CanActOnMessageAsync`. It must be called out in
  `docs/chat/delivery-state.md`, `docs/security/README.md`, and the changelog.
- No end-to-end encryption, no message editing/deletion, no reactions.
- No delivery retries or store-and-forward. A message not delivered live is not redelivered until
  Phase 14 provides history.
- No distributed receipt aggregation. Phase 15.
- No `Failed` transitions driven by transport errors — `Failed` exists in the enum as a terminal state
  the application can set, but LiveSharp does not infer it in this phase (inferring it has the same
  problem as inferring `Delivered`).

---

## Tasks

### P08.T1 — State model and state machine

**Deliverables**

```
src/LiveSharp.Chat/DeliveryState/MessageDeliveryState.cs
src/LiveSharp.Chat/DeliveryState/MessageReceipt.cs
src/LiveSharp.Chat/DeliveryState/MessageDeliverySummary.cs
src/LiveSharp.Chat/DeliveryState/DeliveryStateMachine.cs
tests/LiveSharp.Chat.Tests/DeliveryState/DeliveryStateMachineTests.cs
```

**Public API (exact):**

```csharp
namespace LiveSharp.Chat;

public enum MessageDeliveryState
{
    /// <summary>Accepted by the server but not yet handed to any transport.</summary>
    Pending   = 0,

    /// <summary>Handed to the transport. The recipient may or may not have received it.</summary>
    Sent      = 1,

    /// <summary>The recipient's client acknowledged receipt.</summary>
    Delivered = 2,

    /// <summary>The recipient's application declared the message read.</summary>
    Read      = 3,

    /// <summary>Terminal failure. Set explicitly by the application; never inferred by LiveSharp.</summary>
    Failed    = 4,
}

/// <summary>One recipient's state for one message.</summary>
public sealed record MessageReceipt(
    string MessageId,
    string UserId,
    MessageDeliveryState State,
    DateTimeOffset UpdatedAt);

/// <summary>Aggregate view of a message's delivery across all recipients.</summary>
public sealed record MessageDeliverySummary
{
    public required string MessageId { get; init; }
    public required int RecipientCount { get; init; }
    public required int DeliveredCount { get; init; }
    public required int ReadCount { get; init; }
    public required int FailedCount { get; init; }

    /// <summary>The weakest state across all recipients: Read only when every recipient has read.</summary>
    public required MessageDeliveryState AggregateState { get; init; }
}
```

**State machine rules — implement in `DeliveryStateMachine` as a pure static table**

```csharp
public static class DeliveryStateMachine
{
    /// <summary>True when a receipt may move from <paramref name="from"/> to <paramref name="to"/>.</summary>
    public static bool CanTransition(MessageDeliveryState from, MessageDeliveryState to);

    /// <summary>Applies a transition, returning the resulting state. Never regresses.</summary>
    public static MessageDeliveryState Advance(MessageDeliveryState current, MessageDeliveryState requested);

    public static bool IsTerminal(MessageDeliveryState state);
}
```

1. **Monotonic.** `Pending < Sent < Delivered < Read`. A transition to a lower state is **not an
   error** — it is a no-op. `Advance(Read, Delivered)` returns `Read`. Out-of-order acks are normal
   (two devices, one network); rejecting them with an error would make every client implement retry
   logic for a non-problem.
2. **Skipping forward is legal.** `Advance(Sent, Read)` returns `Read`. A `read` ack arriving before
   the `delivered` ack must promote straight to `Read` and must **not** require a synthetic
   `Delivered` step. Test it — this is the single most common real-world ordering.
3. **`Failed` is terminal.** Any state may move to `Failed`; nothing may move out of it, including
   `Read`. `Advance(Failed, Read)` returns `Failed`.
4. **`Read` is terminal except for `Failed`.**
5. `Advance` is total: every `(from, to)` pair returns a defined value. No exceptions, no `default`
   throw. A wire client can send any enum value.
6. Reject out-of-range enum values at the protocol boundary (`P08.T3`), not here.

**Tests — exhaustive and table-driven**

| Test | Asserts |
|---|---|
| `Every_state_pair_has_a_defined_transition_result` | all 25 combinations, no exception |
| `Advance_never_regresses` | for all pairs, result ≥ `from` in ordinal terms except `Failed` |
| `Advance_from_sent_to_read_skips_delivered` | **rule 2** |
| `Advance_from_read_to_delivered_stays_read` | **rule 1**, no error |
| `Advance_from_delivered_to_delivered_is_idempotent` | |
| `Advance_to_failed_is_always_allowed` | from every state |
| `Advance_out_of_failed_is_never_allowed` | to every state |
| `Advance_out_of_read_is_only_allowed_to_failed` | |
| `Can_transition_matches_advance_for_every_pair` | the two APIs cannot disagree |
| `Is_terminal_is_true_only_for_read_and_failed` | |
| `Numeric_enum_values_are_pinned` | wire and future persistence contract |
| `State_machine_is_pure` | no allocations, no time, no I/O — assert via repeated calls |

> The 25-combination table test is not busywork. Delivery state is the feature users complain about
> loudest when it is wrong, and every wrong behaviour is one wrong table cell. Write the table once,
> assert it exhaustively, and this feature never regresses.

---

### P08.T2 — Receipt store and summary aggregation

**Deliverables**

```
src/LiveSharp.Chat/DeliveryState/Storage/IMessageStateStore.cs
src/LiveSharp.Chat/DeliveryState/Storage/InMemoryMessageStateStore.cs
src/LiveSharp.Chat/Configuration/MessageStateOptions.cs
src/LiveSharp.Chat/Configuration/MessageStateOptionsValidator.cs
tests/LiveSharp.Chat.Tests/DeliveryState/Storage/MessageStateStoreContractTests.cs
tests/LiveSharp.Chat.Tests/DeliveryState/Storage/InMemoryMessageStateStoreTests.cs
```

**Public API (exact):**

```csharp
namespace LiveSharp.Chat.Storage;

public interface IMessageStateStore
{
    /// <summary>Creates receipt rows in <see cref="MessageDeliveryState.Sent"/> for each recipient.
    /// Idempotent: existing rows are left untouched.</summary>
    ValueTask InitialiseAsync(string messageId, IReadOnlyCollection<string> recipientIds, DateTimeOffset at, CancellationToken cancellationToken = default);

    /// <summary>Advances one receipt. Returns the state before and after, so the caller can decide
    /// whether a change is worth broadcasting.</summary>
    ValueTask<ReceiptTransition> AdvanceAsync(string messageId, string userId, MessageDeliveryState requested, DateTimeOffset at, CancellationToken cancellationToken = default);

    ValueTask<IReadOnlyList<MessageReceipt>> GetReceiptsAsync(string messageId, CancellationToken cancellationToken = default);
    ValueTask<MessageDeliverySummary?> GetSummaryAsync(string messageId, CancellationToken cancellationToken = default);
    ValueTask<IReadOnlyDictionary<string, MessageDeliverySummary>> GetSummariesAsync(IReadOnlyCollection<string> messageIds, CancellationToken cancellationToken = default);

    /// <summary>Removes receipts for messages older than the cutoff. Returns the number removed.</summary>
    ValueTask<int> PruneAsync(DateTimeOffset olderThan, int limit, CancellationToken cancellationToken = default);
}

/// <summary>Result of an advance: whether a row existed, and the states either side.</summary>
public readonly record struct ReceiptTransition(
    bool Found,
    MessageDeliveryState Previous,
    MessageDeliveryState Current)
{
    public bool Changed => Found && Previous != Current;
}
```

**Public API (exact) — options:**

```csharp
public sealed class MessageStateOptions
{
    public const string ConfigurationSectionName = "LiveSharp:Chat:DeliveryState";

    public bool EnableDeliveryReceipts { get; set; } = true;
    public bool EnableReadReceipts { get; set; } = true;

    /// <summary>Receipt updates for one message are coalesced within this window. Default 500ms.</summary>
    public TimeSpan ReceiptBatchWindow { get; set; } = TimeSpan.FromMilliseconds(500);

    /// <summary>Maximum message ids accepted in one receipt operation. Default 200.</summary>
    public int MaxReceiptsPerBatch { get; set; } = 200;

    /// <summary>Receipts older than this are pruned. Default 7 days.</summary>
    public TimeSpan ReceiptRetention { get; set; } = TimeSpan.FromDays(7);

    /// <summary>How often the prune sweep runs. Default 1 hour.</summary>
    public TimeSpan PruneInterval { get; set; } = TimeSpan.FromHours(1);

    /// <summary>Maximum tracked messages in the in-memory store. Default 100000.</summary>
    public int MaxTrackedMessages { get; set; } = 100_000;
}
```

**Requirements**

- `AdvanceAsync` returns `ReceiptTransition` rather than `bool`. The caller needs three distinct
  answers — no such receipt, receipt unchanged (duplicate ack), receipt advanced — and collapsing them
  loses the information that decides whether to broadcast. A duplicate ack must produce **zero**
  traffic; without `Previous`, every duplicate ack rebroadcasts.
- `AdvanceAsync` for a `(messageId, userId)` with no row returns `Found = false` and does **not**
  create one. Creating rows on ack would let any client manufacture receipt state for arbitrary
  message IDs — an unbounded-key attack. Rows are created only by `InitialiseAsync`, which is called
  by the send path with the authoritative recipient list.
- `InitialiseAsync` is idempotent so a retried send (Phase 04 idempotency) does not reset receipts.
- **Aggregate rule** for `MessageDeliverySummary.AggregateState`: the **minimum** state across all
  recipients, with `Failed` counted separately rather than dragging the aggregate down. So a group
  message is `Read` only when every recipient has read it, and `Delivered` when every recipient has at
  least `Delivered`. Expose the counts too, because "3 of 12 read" is what a real UI shows and an
  aggregate alone forces the client to fetch all receipts. Document the rule; the alternative
  (maximum) means one keen reader marks a message read for everybody.
- A message with `RecipientCount == 0` (a group whose only member is the sender) summarises as `Sent`,
  not `Read`. An empty-set minimum is a trap; special-case it and test it.
- `GetSummariesAsync` exists so a client rendering 50 messages makes one call, not 50.
- `InMemoryMessageStateStore`: `ConcurrentDictionary<string, ImmutableDictionary<string, MessageReceipt>>`
  keyed by message ID, plus a bounded FIFO of message IDs for `MaxTrackedMessages` eviction. Eviction
  must clear the receipt map too — the Phase 04 `InMemoryMessageStore` eviction leak test has an exact
  analogue here.
- `PruneAsync` needs a time-ordered key to be efficient. Message IDs are UUIDv7 (Phase 04), so they
  sort chronologically — prune by comparing the ID's embedded timestamp or by keeping a
  `SortedDictionary` of insertion times. Document which, because Phase 14's SQL implementation will
  want an index on the same thing.
- `MessageStateStoreContractTests` abstract suite, same pattern as Phases 04–06.

**Tests — contract suite**

| Test | Asserts |
|---|---|
| `Initialise_creates_a_sent_receipt_per_recipient` | |
| `Initialise_twice_does_not_reset_advanced_receipts` | idempotency |
| `Initialise_with_no_recipients_creates_nothing` | |
| `Advance_for_an_uninitialised_message_reports_not_found` | **and creates nothing** |
| `Advance_for_an_unknown_user_on_a_known_message_reports_not_found` | |
| `Advance_from_sent_to_delivered_reports_changed` | |
| `Advance_from_delivered_to_delivered_reports_unchanged` | **the duplicate-ack test** |
| `Advance_from_read_to_delivered_reports_unchanged_and_stays_read` | |
| `Advance_from_sent_to_read_reports_changed_with_previous_sent` | |
| `Advance_stamps_updated_at_from_the_supplied_time` | |
| `Advance_does_not_stamp_updated_at_when_unchanged` | so a duplicate ack does not bump retention |
| `Get_receipts_returns_one_row_per_recipient` | |
| `Get_receipts_for_an_unknown_message_returns_empty` | |
| `Summary_aggregate_is_the_minimum_recipient_state` | mixed states |
| `Summary_is_read_only_when_every_recipient_has_read` | |
| `Summary_with_zero_recipients_is_sent_not_read` | **the empty-set trap** |
| `Summary_counts_failed_recipients_separately` | aggregate not dragged to Failed |
| `Summary_for_an_unknown_message_returns_null` | |
| `Get_summaries_returns_an_entry_only_for_known_messages` | documented: sparse, unlike presence |
| `Prune_removes_receipts_older_than_the_cutoff` | |
| `Prune_does_not_remove_receipts_at_exactly_the_cutoff` | boundary |
| `Prune_honours_the_limit` | |
| `Concurrent_advances_for_different_users_all_land` | `ConcurrencyHarness` |
| `Concurrent_advances_for_one_user_converge_on_the_highest_state` | racing delivered/read |
| `All_methods_honour_a_pre_cancelled_token` | |

**Tests — in-memory specific**

| Test | Asserts |
|---|---|
| `Evicts_the_oldest_message_beyond_the_tracked_limit` | cap + 1 |
| `Eviction_clears_the_receipt_map_for_the_evicted_message` | **leak test** |
| `Options_defaults_are_valid` | |
| `Prune_interval_not_less_than_the_retention_is_rejected` | actionable message |
| `Max_receipts_per_batch_above_the_core_payload_budget_is_rejected` | cross-option validation |
| `Receipt_batch_window_of_zero_is_valid_and_disables_batching` | degenerate config is legal |

---

### P08.T3 — Receipt service, batching, and send-path integration

**Deliverables**

```
src/LiveSharp.Chat/DeliveryState/IMessageStateService.cs
src/LiveSharp.Chat/DeliveryState/MessageStateService.cs
src/LiveSharp.Chat/DeliveryState/ReceiptBatcher.cs
src/LiveSharp.Chat/DeliveryState/MessageStateInitialiser.cs
src/LiveSharp.Chat/DeliveryState/ReceiptPruneService.cs
src/LiveSharp.Chat/Diagnostics/DeliveryStateLog.cs
tests/LiveSharp.Chat.Tests/DeliveryState/…
```

**Public API (exact):**

```csharp
namespace LiveSharp.Chat;

public interface IMessageStateService
{
    Task<OperationResult<int>> MarkDeliveredAsync(string userId, IReadOnlyCollection<string> messageIds, CancellationToken cancellationToken = default);
    Task<OperationResult<int>> MarkReadAsync(string userId, IReadOnlyCollection<string> messageIds, CancellationToken cancellationToken = default);
    Task<OperationResult> MarkFailedAsync(string userId, string messageId, CancellationToken cancellationToken = default);

    Task<OperationResult<IReadOnlyList<MessageReceipt>>> GetReceiptsAsync(string actingUserId, string messageId, CancellationToken cancellationToken = default);
    Task<IReadOnlyDictionary<string, MessageDeliverySummary>> GetSummariesAsync(IReadOnlyCollection<string> messageIds, CancellationToken cancellationToken = default);
}
```

**Requirements — send-path integration via the Phase 07 observer**

`MessageStateInitialiser : IChatMessageObserver` (the seam introduced in Phase 07 — reuse it, do not
add a second hook). On `OnMessageSentAsync`:

- Direct message → recipients = `[message.RecipientId]`.
- Group message → recipients = group members **excluding the sender**, from `IGroupStore`.
- `IMessageStateStore.InitialiseAsync(message.Id, recipients, timeProvider.GetUtcNow())`.

The observer contract says implementations must not throw, and Phase 04 says the store write precedes
delivery. But the observer fires **after** delivery, so a receipt-initialisation failure means the
message was delivered with no receipt rows and every subsequent ack reports `Found = false`. Accept
that, log at `Error`, and document it: receipts are best-effort metadata, and losing them must never
fail a message that was already delivered. The alternative — initialising receipts before delivery —
adds a store round trip to the critical path of every message for the benefit of a cosmetic feature.

**Requirements — batching**

`ReceiptBatcher` coalesces `chat.receipt.updated` broadcasts per message within
`ReceiptBatchWindow`, using a `TimeProvider` timer:

- Keyed by message ID. A second change for the same message inside the window replaces the pending
  summary — receipts are a level, not an edge, exactly as with presence in Phase 06.
- Flush immediately when a message reaches a terminal aggregate (`Read` by all, or `Failed`), so the
  final double-tick is not delayed.
- Bound the pending map; above `10 × MaxTrackedMessages / 100` entries, flush and warn.
- Reuse the shape of `PresenceChangeCoalescer` from Phase 06. If the two end up structurally
  identical, extract a shared internal `Coalescer<TKey, TValue>` into `LiveSharp.Core` — but only
  after both exist and the duplication is proven, not preemptively.

**Requirements — broadcast targeting**

`chat.receipt.updated` goes to the **message sender** and to the acking user's own other connections.
It does **not** go to other recipients: in a group of 50, telling all 50 that recipient 37 has read
the message is 50× traffic for information only the sender's UI shows. Document this; a client that
wants full receipt visibility calls `chat.receipt.query`.

**Requirements — the authorization gap**

`MarkDeliveredAsync`/`MarkReadAsync` currently check only that a receipt row exists for
`(messageId, userId)` — which does mean a client can only ack messages it was actually a recipient of,
because rows are created from the authoritative recipient list. That is a meaningful accidental
defence and should be stated as such.

What is **not** defended: the acting user ID comes from the connection, so a client cannot ack *as*
someone else. So the real remaining gap is narrower than Phase 04 feared: a client can ack messages it
legitimately received, at times it chooses. That is acceptable.

`GetReceiptsAsync` is the actual gap: it currently returns receipts to any caller who knows a message
ID. Restrict it now to the message sender and the message's recipients — computable from the receipt
rows plus `IMessageStore` — and return `Forbidden` otherwise. Phase 09 replaces the hand-rolled check
with `CanActOnMessageAsync`. Add a pinning test for what is and is not allowed today.

**Requirements — prune service**

`ReceiptPruneService : BackgroundService` at `PruneInterval` calling `PruneAsync` with a bounded limit.
Same resilience rules as the Phase 02 reaper and Phase 06 sweep. Log at `Information` only when rows
were removed.

**Tests**

| Test | Asserts |
|---|---|
| `Initialiser_creates_receipts_for_a_direct_message_recipient` | |
| `Initialiser_creates_receipts_for_every_group_member_except_the_sender` | |
| `Initialiser_creates_nothing_for_a_group_with_only_the_sender` | |
| `A_failing_initialiser_does_not_fail_the_send` | and logs at `Error` |
| `Mark_delivered_advances_and_broadcasts_to_the_sender` | |
| `Mark_delivered_broadcasts_to_the_ackers_other_connections` | |
| `Mark_delivered_does_not_broadcast_to_other_recipients` | **the fan-out rule** |
| `Mark_delivered_twice_broadcasts_once` | **the duplicate-ack traffic test** |
| `Mark_read_before_delivered_advances_straight_to_read` | |
| `Mark_read_when_delivery_receipts_are_disabled_still_works` | independent toggles |
| `Mark_delivered_when_delivery_receipts_are_disabled_is_a_no_op_success` | |
| `Mark_for_an_unknown_message_id_reports_zero_updated_and_succeeds` | not an error — clients replay acks after reconnect |
| `Mark_with_a_batch_above_the_limit_returns_invalid_payload` | limit + 1 |
| `Mark_with_an_empty_batch_succeeds_with_zero` | |
| `Mark_with_duplicate_ids_in_one_batch_counts_once` | |
| `Get_receipts_by_the_sender_succeeds` | |
| `Get_receipts_by_a_recipient_succeeds` | |
| `Get_receipts_by_an_unrelated_user_is_forbidden` | **security** |
| `Get_receipts_for_an_unknown_message_is_indistinguishable_from_forbidden` | no existence oracle |
| `Batcher_coalesces_two_changes_for_one_message_into_one_broadcast` | `FakeTimeProvider` |
| `Batcher_flushes_immediately_on_a_terminal_aggregate` | |
| `Batcher_does_not_coalesce_across_different_messages` | |
| `Batcher_flush_on_dispose_loses_nothing` | |
| `A_zero_batch_window_broadcasts_immediately` | |
| `Prune_service_removes_expired_receipts` | |
| `Prune_service_survives_a_throwing_store` | |
| `Group_of_fifty_recipients_acking_produces_at_most_one_broadcast_per_batch_window` | **the O(n²) test** |
| `All_methods_honour_a_pre_cancelled_token` | |

---

### P08.T4 — Operations, handlers, `AddMessageState()`, and clients

**Deliverables**

```
src/LiveSharp.Chat/DeliveryState/Protocol/ReceiptRequest.cs
src/LiveSharp.Chat/DeliveryState/Protocol/ReceiptResponse.cs
src/LiveSharp.Chat/DeliveryState/Protocol/ReceiptUpdatedPayload.cs
src/LiveSharp.Chat/DeliveryState/Protocol/ReceiptQueryRequest.cs
src/LiveSharp.Chat/DeliveryState/Handlers/MarkDeliveredHandler.cs
src/LiveSharp.Chat/DeliveryState/Handlers/MarkReadHandler.cs
src/LiveSharp.Chat/DeliveryState/Handlers/QueryReceiptsHandler.cs
src/LiveSharp.Chat/DependencyInjection/LiveSharpMessageStateBuilderExtensions.cs
clients/dotnet/LiveSharp.Client/Chat/LiveSharpReceiptClientExtensions.cs
clients/js/packages/client/src/receipts.ts
tests/…
tests/LiveSharp.IntegrationTests/Chat/DeliveryStateIntegrationTests.cs
```

Operations and events — add to `RealtimeOperations`/`RealtimeEvents`, register DTOs in
`LiveSharpJsonSerializerContext`, mirror in `protocol.ts`:

```
chat.receipt.delivered   (NotifyAsync — batched, fire-and-forget)
chat.receipt.read        (InvokeAsync — the application wants confirmation)
chat.receipt.query       (InvokeAsync)
                                            chat.receipt.updated
```

**Requirements**

- `chat.receipt.delivered` is `NotifyAsync`: it is automatic client plumbing, high-frequency, and
  self-healing on the next batch. `chat.receipt.read` is `InvokeAsync`: it is an explicit application
  action and the developer wants to know it landed. Justify the asymmetry in the docs — it will look
  inconsistent otherwise.
- `ReceiptRequest` carries `IReadOnlyList<string> MessageIds` and **no** `UserId`. Acting user comes
  from the connection. Assert the absence by reflection, as in Phases 04–07.
- `ReceiptRequest` must **not** carry a `State`. Two named operations are clearer than one operation
  with a state parameter, and they cannot be used to force a message to `Read` on someone's behalf or
  to drive a message to `Failed` from a hostile client.
- Reject out-of-range enum values wherever a state does cross the wire (`chat.receipt.updated` payload
  is server-authored, so only `MarkFailedAsync` is exposed — keep it server-side only in this phase and
  do **not** add a `chat.receipt.failed` operation).
- Auto-ack in both clients: after a `chat.message.received` handler chain completes without throwing,
  the client queues the message ID for `chat.receipt.delivered` and flushes on a short client-side
  timer (default 250 ms) or at 50 IDs. If a handler throws, **do not** ack — the message did not reach
  the application. This is what makes `Delivered` mean something. Make auto-ack disableable
  (`autoAcknowledgeDelivery: false`) for applications that want to control it.
- `AddMessageState()` requires `AddChat()`; group receipts additionally require `AddGroups()` and
  degrade to direct-only (with a startup `Information` log) if absent.

**Tests**

| Test | Asserts |
|---|---|
| `Receipt_request_has_no_user_id_or_state_member` | **security**, reflection |
| `Mark_delivered_handler_uses_the_connection_user_id` | |
| `Mark_read_handler_returns_the_updated_count` | |
| `Query_handler_forbids_an_unrelated_caller` | |
| `AddMessageState_registers_the_service_store_batcher_initialiser_and_prune_service` | lifetimes |
| `AddMessageState_registers_the_initialiser_as_a_chat_message_observer` | reuses the Phase 07 seam |
| `AddMessageState_without_AddChat_fails_at_startup_naming_AddChat` | |
| `AddMessageState_without_AddGroups_logs_and_supports_direct_messages_only` | |
| `AddChat_alone_registers_no_receipt_operations` | opt-in is real |
| `Client_auto_acks_delivery_after_the_handler_completes` | |
| `Client_does_not_ack_when_a_handler_throws` | **the meaning of Delivered** |
| `Client_batches_multiple_message_ids_into_one_ack` | |
| `Client_auto_ack_can_be_disabled` | |
| `receipt constants match artifacts/protocol-names.json` | contract test |
| `receipts module imports cleanly with no DOM globals` | |

**Integration tests (both transports)**

| Test | Asserts |
|---|---|
| `Sender_sees_delivered_after_the_recipient_client_receives` | end to end, no application code |
| `Sender_sees_read_after_the_recipient_marks_read` | |
| `Read_arriving_before_delivered_results_in_read` | force the ordering |
| `A_second_delivered_ack_produces_no_second_broadcast` | |
| `Group_message_summary_reports_partial_read_counts` | 3 members, 1 reads |
| `Group_message_summary_reaches_read_only_when_all_have_read` | |
| `Receipts_replayed_after_a_reconnect_are_accepted_without_error` | client replays its pending acks |
| `Offline_recipient_message_stays_at_sent` | **the Phase 04 honesty fix, made observable** |
| `A_recipient_cannot_query_receipts_for_an_unrelated_message` | security |
| `Fifty_recipients_reading_within_one_window_produce_one_broadcast_to_the_sender` | scale |

---

### P08.T5 — Example applications and documentation

**Deliverables**

```
examples/AspNetCoreMvc/…              receipt ticks in the message list
examples/React/…                      receipt ticks + read-on-visible
docs/chat/delivery-state.md
```

**Requirements**

- MVC example: single tick for `Sent`, double tick for `Delivered`, coloured double tick for `Read`,
  driven by `chat.receipt.updated`. Include a deliberately visible "sent but not delivered" case by
  messaging an offline user, so the example teaches the semantics rather than hiding them.
- React example: additionally demonstrate `IntersectionObserver`-driven `chat.receipt.read` — mark read
  only when the message is actually on screen. This is the correct pattern and the one applications get
  wrong (marking read on message arrival, which defeats the feature). Comment it as such.
- Both example READMEs gain a "Delivery state" section explaining what each tick means and explicitly
  that `Delivered` is client-acknowledged.

**`docs/chat/delivery-state.md`** must contain:
- The state diagram and the full transition table (copied from `DeliveryStateMachine`).
- **"Why Delivered is not inferred"** — the reasoning from the top of this file.
- The aggregate rule for groups, with a worked 12-recipient example.
- The batching model and the `chat.receipt.updated` targeting rule.
- The `NotifyAsync`/`InvokeAsync` asymmetry and why.
- Client auto-ack behaviour and how to disable it.
- **Limitations**: no unread counts or read positions (Phase 14), receipts are best-effort metadata
  and a receipt-initialisation failure does not fail the message, `GetReceiptsAsync` authorization is
  hand-rolled until Phase 09, node-local until Phase 15, `Failed` is never inferred, receipts are
  pruned after `ReceiptRetention`.

---

## Dependency justification

No new packages. Reuses `IChatMessageObserver` (Phase 07), `IGroupStore` (Phase 05),
`IMessageStore` (Phase 04), `IRealtimeTransport`, and `TimeProvider`.

If `ReceiptBatcher` and `PresenceChangeCoalescer` prove structurally identical, extract an internal
`Coalescer<TKey, TValue>` into `LiveSharp.Core` — as a refactor with its own commit, after both exist.
Do not create it speculatively in this phase.

---

## Documentation deltas

- `docs/chat/delivery-state.md` — **new**, as specified above.
- `docs/chat/direct-messaging.md` — replace the "no receipts" limitation with a link, and update the
  offline-behaviour section to explain that `Sent` is now observable.
- `docs/chat/groups.md` — per-recipient receipts and the aggregate rule.
- `docs/security/README.md` — receipt query authorization is hand-rolled until Phase 09; acking is
  constrained by receipt-row existence; receipts reveal read timing to the sender (a privacy property
  worth stating).
- `docs/configuration/README.md` — `MessageStateOptions`, including the interaction between
  `ReceiptBatchWindow`, `MaxReceiptsPerBatch`, and `Limits.MaxPayloadBytes`.
- `docs/configuration/service-lifetimes.md` — service, store, batcher, initialiser, prune service.
- `docs/troubleshooting/README.md` — "messages stay on one tick" (client not acking, or handler
  throwing), "read receipts never arrive" (`EnableReadReceipts`), "receipt events are delayed"
  (`ReceiptBatchWindow`), "receipts disappear after a week" (`ReceiptRetention`), "group message never
  reaches Read" (one member never opens it — by design).
- `README.md` — message delivery state → `Available`.

## Example deltas

- `examples/AspNetCoreMvc` — receipt ticks including a visible sent-but-not-delivered case.
- `examples/React` — receipt ticks plus `IntersectionObserver` read marking.
- Both READMEs — "Delivery state" section.

## CHANGELOG entry

```markdown
### Added
- Message delivery state: `Sent`, `Delivered`, `Read`, and `Failed`, tracked per recipient for both
  direct and group messages.
- `chat.receipt.delivered` / `chat.receipt.read` / `chat.receipt.query` operations and the
  `chat.receipt.updated` event.
- Idempotent, out-of-order-tolerant receipt acknowledgement with a monotonic state machine.
- Receipt batching so an active conversation does not produce quadratic receipt traffic.
- Automatic delivery acknowledgement in the .NET and TypeScript clients, sent only after the
  application's message handler completes successfully.
- Receipt retention with a background prune sweep.

### Notes
- `Delivered` is acknowledged by the recipient's client, never inferred from a successful transport
  send. A client that does not acknowledge leaves messages at `Sent`.

### Security
- Receipt queries are restricted to a message's sender and recipients.
- Acknowledgements can only advance receipts that already exist, so a client cannot manufacture
  receipt state for arbitrary message identifiers.
```

---

## Exit criteria

- [ ] `Every_state_pair_has_a_defined_transition_result` covers all 25 combinations.
- [ ] `Advance_from_sent_to_read_skips_delivered` and `Advance_from_read_to_delivered_stays_read` pass.
- [ ] `Advance_for_an_uninitialised_message_reports_not_found` **and creates nothing**.
- [ ] `Summary_with_zero_recipients_is_sent_not_read` passes.
- [ ] `Mark_delivered_twice_broadcasts_once` passes.
- [ ] `Mark_delivered_does_not_broadcast_to_other_recipients` passes.
- [ ] `Client_does_not_ack_when_a_handler_throws` passes on both clients.
- [ ] `Offline_recipient_message_stays_at_sent` passes — the Phase 04 gap is now observable.
- [ ] `Fifty_recipients_reading_within_one_window_produce_one_broadcast_to_the_sender` passes.
- [ ] `Receipt_request_has_no_user_id_or_state_member` passes.
- [ ] `Get_receipts_by_an_unrelated_user_is_forbidden` passes.
- [ ] `Eviction_clears_the_receipt_map_for_the_evicted_message` passes.
- [ ] `MessageStateStoreContractTests` runs against `InMemoryMessageStateStore`.
- [ ] `docs/chat/delivery-state.md` contains "Why Delivered is not inferred".
- [ ] The React example marks read on visibility, not on arrival.
- [ ] Receipt constants in `protocol.ts`; contract test green.
- [ ] `scripts/verify.ps1` green; tag `v0.7.0`.

## Verify

```powershell
pwsh scripts/verify.ps1
dotnet test tests/LiveSharp.Chat.Tests -c Release --filter "FullyQualifiedName~DeliveryState"
dotnet test tests/LiveSharp.IntegrationTests -c Release --filter "FullyQualifiedName~DeliveryState"
pnpm --dir clients/js -r test
git tag v0.7.0
```

## Next

`docs/plan/phase-09-auth.md`, task `P09.T1`.

> Phase 09 is the security phase. It closes every gap Phases 03–08 deliberately left open, and it
> contains a **breaking change**: anonymous connections stop being allowed by default. Read the whole
> phase file before writing any code.
