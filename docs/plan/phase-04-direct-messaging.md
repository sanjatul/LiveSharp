# Phase 04 — Direct Messaging

| | |
|---|---|
| **Version at exit** | `0.3.0` (tag `v0.3.0`) |
| **Depends on** | Phase 03 (**and its approved review gate**) |
| **Branch** | `feat/phase-04-direct-messaging` |
| **Tasks** | `P04.T1` … `P04.T6` |
| **Read before starting** | `docs/architecture/wire-protocol.md`, `docs/architecture/connection-lifecycle.md`, `EXECUTION_PLAN.md` §6 |

> Do not start this phase unless `DEVELOPMENT_LOG.md` records that the Phase 03 review gate was
> passed.

---

## Goal

The first user-visible feature: user A sends a message to user B and user B receives it, on every
device B is connected from, with A's other devices seeing their own sent message. Plus the first
example application.

This phase also establishes the **feature package template** every later feature package follows:
a package, a service interface, operation handlers, an optional store with a `Null` default, an
`ILiveSharpBuilder` extension method, a docs page, and an example.

---

## Non-goals — DO NOT implement in this phase

- **No delivery or read receipts.** `Sent`/`Delivered`/`Read` is Phase 08. Do not add a state field
  "for later" — Phase 08 designs it properly with per-recipient tracking in groups.
- **No groups.** `SendToGroupAsync` on the chat service does not exist yet. Phase 05.
- **No presence.** "Is the recipient online" is answered by `IConnectionRegistry.IsOnlineAsync`, not
  by a presence service. Phase 06.
- **No typing indicators.** Phase 07.
- **No message history API, pagination, or persistence provider.** `IMessageStore` is defined with a
  `NullMessageStore` default so the feature works with no database (Spec §54). The EF Core provider
  and the history query API are Phase 14.
- **No authorization.** Anyone connected can message anyone. This is a **known, documented gap**
  closed in Phase 09. The docs page and the example README must both carry the warning.
- **No editing, deletion, reactions, attachments, or threading.** None of these are in the Spec's
  scope for pre-1.0.
- **No Blazor, React, or Next.js example.** Only the MVC example. Phases 05–07 add the others.

---

## Tasks

### P04.T1 — `LiveSharp.Chat` package, message model, and ID generation

**Deliverables**

```
src/LiveSharp.Chat/LiveSharp.Chat.csproj
src/LiveSharp.Chat/PublicAPI.{Shipped,Unshipped}.txt
src/LiveSharp.Chat/AssemblyInfo.cs
src/LiveSharp.Chat/ChatMessage.cs
src/LiveSharp.Chat/ConversationId.cs
src/LiveSharp.Chat/IMessageIdGenerator.cs
src/LiveSharp.Chat/MessageIdGenerator.cs
tests/LiveSharp.Chat.Tests/…
```

**Public API (exact):**

```csharp
namespace LiveSharp.Chat;

/// <summary>A single chat message. Immutable once created.</summary>
public sealed record ChatMessage
{
    /// <summary>Server-assigned, time-sortable identifier. Unique across the deployment.</summary>
    public required string Id { get; init; }

    /// <summary>Identifies the conversation this message belongs to. For direct messages this is a
    /// deterministic pair key; see <see cref="ConversationId"/>.</summary>
    public required string ConversationId { get; init; }

    public required string SenderId { get; init; }

    /// <summary>The recipient user identifier for a direct message.</summary>
    public required string RecipientId { get; init; }

    /// <summary>Message text. Never logged above Trace.</summary>
    public required string Content { get; init; }

    /// <summary>When the server accepted the message, from <see cref="TimeProvider"/>.</summary>
    public required DateTimeOffset SentAt { get; init; }

    /// <summary>Optional caller-supplied identifier used for idempotency and client-side matching.</summary>
    public string? ClientMessageId { get; init; }

    /// <summary>Opaque application metadata. Size-limited; never interpreted by LiveSharp.</summary>
    public IReadOnlyDictionary<string, string> Metadata { get; init; }
}

/// <summary>Builds deterministic conversation identifiers.</summary>
public static class ConversationId
{
    /// <summary>Returns the same identifier for (a,b) and (b,a).</summary>
    public static string ForDirect(string userA, string userB);

    public const string DirectPrefix = "dm:";
}

public interface IMessageIdGenerator
{
    string NewId();
}
```

**Requirements**

- `MessageIdGenerator` uses **`Guid.CreateVersion7()`** formatted with `"n"`. UUIDv7 is
  time-ordered, so message IDs sort chronologically without a separate sequence, and it needs no
  dependency (available in .NET 9+). Document this choice in the XML docs — a future maintainer will
  otherwise "improve" it into a random GUID and break ordering.
- `ConversationId.ForDirect` sorts the two user IDs with `StringComparer.Ordinal` and joins as
  `dm:{lower}|{upper}`. This must be deterministic and symmetric or A and B end up in two different
  conversations — the single most common bug in direct-messaging implementations. Test symmetry
  explicitly.
- `ForDirect` must reject null/empty inputs and reject `userA == userB` **unless** self-messaging is
  explicitly supported. Decide now: **self-messaging is supported** (a "note to self" conversation is
  a real feature and blocking it costs nothing), so `ForDirect(a, a)` returns `dm:a|a`. Document it.
- User IDs may contain `|`. Either escape it or use a length-prefixed encoding — a user ID of
  `"a|b"` must not be able to collide with another pair. Pick one, document it, and test the
  collision case.
- `Metadata` defaults to an empty frozen dictionary, never `null`.
- `Content` is `required` and non-nullable. An empty message is rejected by the validator, not the
  model.

**Tests**

| Test | Asserts |
|---|---|
| `Message_id_generator_produces_unique_ids` | 10 000 ids, no duplicates |
| `Message_id_generator_produces_lexicographically_sortable_ids` | generated in order → sorted order matches |
| `Message_ids_are_url_safe_and_have_no_dashes` | `"n"` format pinned |
| `Conversation_id_is_symmetric` | `ForDirect(a,b) == ForDirect(b,a)` |
| `Conversation_id_is_deterministic_across_calls` | |
| `Conversation_id_carries_the_direct_prefix` | |
| `Conversation_id_for_self_messaging_is_supported` | `ForDirect(a,a)` |
| `Conversation_id_rejects_null_or_empty_inputs` | both parameters, named in the exception |
| `Conversation_ids_do_not_collide_when_user_ids_contain_the_separator` | `("a\|b","c")` vs `("a","b\|c")` — **required** |
| `Chat_message_defaults_metadata_to_empty` | |
| `Chat_message_is_immutable` | reflection: no public setters |

---

### P04.T2 — Validation and limits

**Deliverables**

```
src/LiveSharp.Chat/Configuration/ChatOptions.cs
src/LiveSharp.Chat/Configuration/ChatOptionsValidator.cs
src/LiveSharp.Chat/Validation/IMessageValidator.cs
src/LiveSharp.Chat/Validation/DefaultMessageValidator.cs
tests/LiveSharp.Chat.Tests/Validation/DefaultMessageValidatorTests.cs
```

**Public API (exact):**

```csharp
namespace LiveSharp.Chat;

public sealed class ChatOptions
{
    public const string ConfigurationSectionName = "LiveSharp:Chat";

    /// <summary>Maximum message content length in UTF-16 characters. Default 4096.</summary>
    public int MaxContentLength { get; set; } = 4096;

    /// <summary>Maximum number of metadata entries. Default 16.</summary>
    public int MaxMetadataEntries { get; set; } = 16;

    /// <summary>Maximum combined length of all metadata keys and values. Default 2048.</summary>
    public int MaxMetadataLength { get; set; } = 2048;

    /// <summary>What to do when the recipient has no live connection. Default DropSilently.</summary>
    public OfflineMessageBehavior OfflineBehavior { get; set; } = OfflineMessageBehavior.DropSilently;

    /// <summary>When true, the sender's other connections also receive the message. Default true.</summary>
    public bool EchoToSender { get; set; } = true;
}

public enum OfflineMessageBehavior
{
    /// <summary>Store (if a store is configured) and report success. The message is not delivered live.</summary>
    DropSilently = 0,

    /// <summary>Report failure with <see cref="RealtimeErrorCode.RecipientOffline"/>.</summary>
    Fail = 1,
}

/// <summary>Validates an outgoing message before it is accepted.</summary>
public interface IMessageValidator
{
    OperationResult Validate(SendMessageRequest request, RealtimeOperationContext context);
}
```

**Requirements**

- `OfflineMessageBehavior` has exactly **two** values. A `Queue` option is tempting and wrong here —
  durable queueing requires persistence, which is Phase 14. Adding an enum value later is
  non-breaking; shipping a `Queue` value that silently drops is a lie.
- The default is `DropSilently`, because with `NullMessageStore` and no presence, `Fail` makes the
  library feel broken on the very first tutorial where the recipient is not yet connected. Document
  the trade-off honestly: with the default, a caller receives `Success` for a message nobody saw.
  Phase 08's delivery receipts are what make this observable.
- `DefaultMessageValidator` checks, in order, returning the first failure:
  1. `Content` null/empty/whitespace → `InvalidPayload`, "Message content must not be empty."
  2. `Content.Length > MaxContentLength` → `PayloadTooLarge`, with the actual and allowed lengths.
  3. `RecipientId` null/empty → `InvalidPayload`.
  4. Metadata entry count → `InvalidPayload`.
  5. Combined metadata length → `PayloadTooLarge`.
  6. Any metadata key null/empty → `InvalidPayload`.
- Validation failure messages **must not echo the content back**. They may state lengths. Echoing
  content into an error string that is logged is how message bodies end up in log aggregators.
- Content is **not** sanitised or HTML-encoded. LiveSharp transports text; encoding is the rendering
  layer's job and doing it here would corrupt legitimate content. Say this explicitly in
  `docs/security/` and in the example view, which must encode on render.

**Tests**

| Test | Asserts |
|---|---|
| `Accepts_a_normal_message` | |
| `Rejects_null_content` / `empty` / `whitespace_only` | table-driven |
| `Rejects_content_above_the_limit` | limit + 1 |
| `Accepts_content_exactly_at_the_limit` | boundary |
| `Rejects_an_empty_recipient_id` | |
| `Rejects_too_many_metadata_entries` | limit + 1 |
| `Rejects_metadata_above_the_combined_length_limit` | |
| `Rejects_an_empty_metadata_key` | |
| `Accepts_content_containing_html_and_does_not_alter_it` | pins the no-sanitisation contract |
| `Accepts_content_containing_emoji_and_surrogate_pairs` | length counted in UTF-16 units, documented |
| `Error_message_never_contains_the_message_content` | **security test** |
| `Error_message_reports_the_actual_and_allowed_lengths` | diagnosability |
| `Chat_options_defaults_are_valid` | |
| `Chat_options_reject_a_non_positive_max_content_length` | |
| `Chat_options_reject_a_max_content_length_exceeding_the_core_payload_limit` | cross-option validation with an actionable message |

---

### P04.T3 — `IMessageStore` and `NullMessageStore`

**Deliverables**

```
src/LiveSharp.Chat/Storage/IMessageStore.cs
src/LiveSharp.Chat/Storage/NullMessageStore.cs
src/LiveSharp.Chat/Storage/InMemoryMessageStore.cs
tests/LiveSharp.Chat.Tests/Storage/MessageStoreContractTests.cs
tests/LiveSharp.Chat.Tests/Storage/NullMessageStoreTests.cs
tests/LiveSharp.Chat.Tests/Storage/InMemoryMessageStoreTests.cs
```

**Public API (exact):**

```csharp
namespace LiveSharp.Chat.Storage;

/// <summary>Optional persistence for chat messages. The default implementation stores nothing.</summary>
public interface IMessageStore
{
    /// <summary>Persists an accepted message. Called after validation and before delivery.</summary>
    ValueTask AddAsync(ChatMessage message, CancellationToken cancellationToken = default);

    /// <summary>Returns a previously stored message, or null. Used for idempotency checks.</summary>
    ValueTask<ChatMessage?> FindByClientMessageIdAsync(string senderId, string clientMessageId, CancellationToken cancellationToken = default);
}
```

**Requirements**

- Exactly **two** methods. A history/query API (`GetConversationAsync`, pagination, cursors) is
  Phase 14, where an implementation exists that can do it efficiently. Adding query methods now
  guarantees a signature designed without a real database in mind.
- `NullMessageStore` is the registered default (`TryAddSingleton`), making persistence genuinely
  optional per Spec §54. `AddAsync` is a no-op; `FindByClientMessageIdAsync` returns `null`.
- `InMemoryMessageStore` exists for the example app and for tests: bounded (`MaxMessages`, default
  10 000, FIFO eviction), thread-safe. It must be documented as **"development and testing only —
  not durable, not distributed, bounded"** so nobody ships it. Add that sentence to the XML docs.
- Write a **shared contract test suite** (`MessageStoreContractTests`, abstract, with a factory
  method) and run it against both implementations. Phase 14's EF Core provider inherits the same
  suite. Building this now costs one extra hour and makes Phase 14 dramatically safer.
- Store failures must not lose the message silently: if `AddAsync` throws, the send fails with
  `InternalError` and logs at `Error`. Delivering a message that was not stored produces a
  conversation that disagrees with itself after a refresh. Document this ordering decision:
  **store first, then deliver.**

**Tests — `MessageStoreContractTests` (abstract, run twice)**

| Test | Asserts |
|---|---|
| `Add_then_find_by_client_message_id_returns_the_message` | (skipped for `NullMessageStore` via a capability flag) |
| `Find_by_an_unknown_client_message_id_returns_null` | |
| `Find_is_scoped_to_the_sender` | same client message ID, different senders |
| `Add_is_safe_to_call_concurrently` | `ConcurrencyHarness` |
| `All_methods_honour_a_pre_cancelled_token` | |

**Tests — implementation-specific**

| Test | Asserts |
|---|---|
| `Null_store_add_is_a_no_op` | |
| `Null_store_find_always_returns_null` | |
| `In_memory_store_evicts_the_oldest_message_beyond_the_cap` | cap + 1 |
| `In_memory_store_eviction_also_clears_the_client_message_id_index` | **leak test** |
| `In_memory_store_is_thread_safe_under_parallel_adds` | |

---

### P04.T4 — `IChatService`, the send handler, and DI

**Deliverables**

```
src/LiveSharp.Chat/IChatService.cs
src/LiveSharp.Chat/ChatService.cs
src/LiveSharp.Chat/Protocol/SendMessageRequest.cs
src/LiveSharp.Chat/Protocol/SendMessageResponse.cs
src/LiveSharp.Chat/Protocol/MessageReceivedPayload.cs
src/LiveSharp.Chat/Handlers/SendMessageHandler.cs
src/LiveSharp.Chat/DependencyInjection/LiveSharpChatBuilderExtensions.cs
src/LiveSharp.Chat/Diagnostics/ChatLog.cs
tests/LiveSharp.Chat.Tests/ChatServiceTests.cs
tests/LiveSharp.Chat.Tests/Handlers/SendMessageHandlerTests.cs
```

Also extend, in `LiveSharp.Abstractions`:

```csharp
public static partial class RealtimeOperations
{
    public static class Chat
    {
        public const string SendMessage = "chat.message.send";
    }
}

public static partial class RealtimeEvents
{
    public static class Chat
    {
        public const string MessageReceived = "chat.message.received";
    }
}
```

…and register `SendMessageRequest`, `SendMessageResponse`, and `MessageReceivedPayload` in
`LiveSharpJsonSerializerContext`. The Phase 03 guard test will fail if you forget.

**Public API (exact):**

```csharp
namespace LiveSharp.Chat;

public sealed record SendMessageRequest
{
    public required string RecipientId { get; init; }
    public required string Content { get; init; }
    public string? ClientMessageId { get; init; }
    public IReadOnlyDictionary<string, string>? Metadata { get; init; }
}

public sealed record SendMessageResponse(string MessageId, DateTimeOffset SentAt);

/// <summary>Payload of the <c>chat.message.received</c> event.</summary>
public sealed record MessageReceivedPayload(ChatMessage Message);

public interface IChatService
{
    /// <summary>Validates, stores, and delivers a direct message on behalf of <paramref name="senderId"/>.</summary>
    Task<OperationResult<ChatMessage>> SendDirectAsync(
        string senderId,
        SendMessageRequest request,
        RealtimeOperationContext? context = null,
        CancellationToken cancellationToken = default);
}

public static class LiveSharpChatBuilderExtensions
{
    public static ILiveSharpBuilder AddChat(this ILiveSharpBuilder builder);
    public static ILiveSharpBuilder AddChat(this ILiveSharpBuilder builder, Action<ChatOptions> configure);
}
```

**`SendDirectAsync` sequence — implement exactly in this order:**

1. Validate via `IMessageValidator`. Failure → return it unchanged.
2. **Idempotency:** if `ClientMessageId` is present, `FindByClientMessageIdAsync(senderId, id)`. A
   hit returns `Success(existingMessage)` **without** re-delivering. A client that retries after a
   flaky reconnect must not produce a duplicate. With `NullMessageStore` this is a no-op and
   duplicates are possible — document that idempotency requires a real store.
3. Build the `ChatMessage`: `Id` from `IMessageIdGenerator`, `ConversationId` from
   `ConversationId.ForDirect`, `SentAt` from `TimeProvider`.
4. `IMessageStore.AddAsync`. Throws → log `Error`, return `Failure(InternalError, …)`.
5. Resolve recipient liveness with `IConnectionRegistry.IsOnlineAsync`.
   - Offline **and** `OfflineBehavior == Fail` → return `Failure(RecipientOffline, …)`. Note: the
     message is already stored. Document that stored-but-not-delivered is the intended semantic; the
     alternative (compensating delete) is worse.
   - Offline **and** `DropSilently` → skip recipient delivery, continue to step 7.
6. `IRealtimeTransport.SendToUserAsync(recipientId, envelope)` with
   `RealtimeEvents.Chat.MessageReceived`.
7. If `EchoToSender`, deliver the same envelope to the sender. **Exclude the originating
   connection** when `context` is non-null, using `SendToUsersAsync` is insufficient — send to the
   sender's user and let the client de-duplicate on `ClientMessageId`, **or** enumerate the sender's
   connections and skip the originator. Choose the second: it is deterministic and does not push
   de-duplication onto every client author. Document the choice.
8. Return `Success(message)`.

**`SendMessageHandler`**

```csharp
internal sealed class SendMessageHandler : IOperationHandler<SendMessageRequest, SendMessageResponse>
{
    public static string Operation => RealtimeOperations.Chat.SendMessage;
}
```

The handler is a **five-line adapter**: pull `context.Connection.UserId`, call
`IChatService.SendDirectAsync`, map to `SendMessageResponse`. All logic is in the service, so the
service is usable from a controller, a background job, or a webhook — not only from a hub. Add a
test that calls `IChatService` with `context: null` and succeeds, proving the service is not
hub-coupled.

**`AddChat()` requirements**

- Registers `ChatService` (singleton — it is stateless and takes no scoped dependencies),
  `DefaultMessageValidator` (singleton, `TryAdd`), `MessageIdGenerator` (singleton, `TryAdd`),
  `NullMessageStore` as `IMessageStore` (singleton, `TryAdd`), the options with validation, and the
  handler via `AddOperationHandler<SendMessageHandler, SendMessageRequest, SendMessageResponse>()`.
- Idempotent. Test double registration.
- Throws `LiveSharpConfigurationException` with an actionable message if no `IRealtimeTransport` is
  registered — i.e. `AddChat()` without `AddSignalR()`. This is the most likely first-run mistake.

**Tests — `ChatServiceTests`** (using `InMemoryTransport` and `FakeTimeProvider`)

| Test | Asserts |
|---|---|
| `Send_delivers_to_every_recipient_connection` | 3 connections → 3 deliveries |
| `Send_returns_the_created_message_with_a_generated_id` | |
| `Send_stamps_sent_at_from_the_time_provider` | |
| `Send_sets_a_symmetric_conversation_id` | matches `ForDirect` |
| `Send_echoes_to_the_senders_other_connections` | 2 sender connections, originator excluded |
| `Send_does_not_echo_to_the_originating_connection` | **the duplicate-message test** |
| `Send_does_not_echo_when_echo_is_disabled` | |
| `Send_to_self_delivers_once_per_connection_without_duplication` | edge case worth pinning |
| `Send_stores_the_message_before_delivering_it` | call order asserted on a substituted store |
| `Send_fails_with_internal_error_when_the_store_throws` | and delivers nothing |
| `Send_to_an_offline_recipient_succeeds_and_delivers_nothing_by_default` | |
| `Send_to_an_offline_recipient_fails_when_configured_to_fail` | error code |
| `Send_to_an_offline_recipient_still_stores_the_message` | documented semantic |
| `Send_returns_the_validation_failure_unchanged` | code and message preserved |
| `Send_with_a_repeated_client_message_id_returns_the_original_and_does_not_redeliver` | delivery count unchanged |
| `Send_with_a_repeated_client_message_id_against_the_null_store_delivers_twice` | pins the documented limitation |
| `Send_works_with_a_null_operation_context` | proves the service is not hub-coupled |
| `Send_never_logs_message_content_by_default` | **security test** |
| `Send_logs_content_at_trace_when_sensitive_logging_is_enabled` | |
| `Concurrent_sends_from_one_user_all_deliver_exactly_once` | `ConcurrencyHarness` |
| `Concurrent_sends_produce_strictly_increasing_message_ids` | UUIDv7 ordering under load |
| `Send_honours_a_pre_cancelled_token` | |

**Tests — `SendMessageHandlerTests`**

| Test | Asserts |
|---|---|
| `Handler_operation_name_matches_the_protocol_constant` | |
| `Handler_uses_the_connection_user_id_as_the_sender` | **security test**: a client-supplied `senderId` in the payload must be impossible — assert the request record has no `SenderId` member at all |
| `Handler_maps_a_service_failure_to_the_same_error_code` | |
| `Handler_maps_a_successful_send_to_the_response_shape` | |

> `Handler_uses_the_connection_user_id_as_the_sender` is the most important security test in this
> phase. `SendMessageRequest` must **not** have a `SenderId` property. Sender identity comes only from
> the authenticated connection, never from the payload.

---

### P04.T5 — Integration tests and client support

**Deliverables**

```
tests/LiveSharp.IntegrationTests/Chat/DirectMessagingIntegrationTests.cs
clients/dotnet/LiveSharp.Client/Chat/LiveSharpChatClientExtensions.cs
clients/js/packages/client/src/chat.ts
clients/js/packages/client/src/__tests__/chat.test.ts
```

**Client convenience layers** — thin, typed wrappers so the DX matches Spec §47:

```csharp
// .NET
public static class LiveSharpChatClientExtensions
{
    public static Task<SendMessageResponse> SendMessageAsync(
        this ILiveSharpClient client, string recipientId, string content,
        string? clientMessageId = null, CancellationToken cancellationToken = default);

    public static IDisposable OnMessageReceived(
        this ILiveSharpClient client, Func<ChatMessage, Task> handler);
}
```

```ts
// TypeScript — a mixin/helper, not a subclass, so tree-shaking still works
export interface SendMessageOptions { clientMessageId?: string; metadata?: Record<string, string>; }
export function sendMessage(client: LiveSharpClient, recipientId: string, content: string, options?: SendMessageOptions): Promise<SendMessageResponse>;
export function onMessageReceived(client: LiveSharpClient, handler: (message: ChatMessage) => void | Promise<void>): Unsubscribe;
```

Add the chat operation/event constants to `protocol.ts`; the Phase 03 contract test will fail
otherwise.

**Integration tests** — two real connections over both transports:

| Test | Asserts |
|---|---|
| `User_a_sends_and_user_b_receives_over_websockets_and_long_polling` | `[Theory]` over transports |
| `Message_arrives_on_every_device_user_b_is_connected_from` | 2 B connections |
| `Sender_second_device_receives_the_echo` | |
| `Sender_originating_connection_does_not_receive_a_duplicate` | |
| `A_third_user_does_not_receive_the_message` | **isolation test** |
| `Sending_to_an_offline_user_returns_success_by_default` | |
| `Sending_an_oversized_message_returns_payload_too_large` | |
| `Sending_an_empty_message_returns_invalid_payload` | |
| `Retrying_with_the_same_client_message_id_delivers_once` | with `InMemoryMessageStore` registered |
| `Message_content_round_trips_unicode_emoji_and_html_unchanged` | |
| `Received_message_id_matches_the_send_response_id` | |
| `A_message_sent_during_a_reconnect_window_is_not_lost_or_duplicated` | drop B's transport, send, let B reconnect; assert the documented behaviour explicitly (with `DropSilently` + `NullMessageStore` the message **is** lost — the test must pin that truth, and the docs must state it) |
| `Ten_users_sending_concurrently_each_receive_only_their_own_messages` | fan-out isolation under load |

> The reconnect-window test must assert the **actual** behaviour, not the desired one. If the message
> is lost, the test says so and `docs/chat/direct-messaging.md` says so. Phase 08 and Phase 14 are
> what fix it. Writing an aspirational test here and marking it `Skip` is how a library ships a lie.

---

### P04.T6 — The MVC example application

**Deliverables**

```
examples/LiveSharp.Examples.slnx
examples/Directory.Build.props
examples/AspNetCoreMvc/LiveSharp.Examples.Mvc.csproj
examples/AspNetCoreMvc/Program.cs
examples/AspNetCoreMvc/Controllers/HomeController.cs
examples/AspNetCoreMvc/Controllers/AccountController.cs
examples/AspNetCoreMvc/Views/…
examples/AspNetCoreMvc/wwwroot/js/chat.js
examples/AspNetCoreMvc/README.md
examples/README.md
```

**Structure requirements**

- `examples/LiveSharp.Examples.slnx` is a **separate solution** referencing the library via
  `ProjectReference` (ADR-010, deviation D5). `LiveSharp.slnx` must not contain example projects —
  add an architecture test asserting no `src/**` or `tests/**` project references `examples/**`.
- `examples/Directory.Build.props` sets `IsPackable=false`, `GenerateDocumentationFile=false`,
  `TreatWarningsAsErrors=false` (examples optimise for readability, not zero-warning purity), and
  `NoWarn` for doc warnings.
- Examples are **not** built by the main CI `verify` job. Add a separate `examples` CI job that
  builds `examples/LiveSharp.Examples.slnx` so breakage is caught without slowing the main gate.

**Application requirements**

- Cookie authentication with a **hardcoded demo user list** (`alice`, `bob`, `carol`, no passwords)
  and a login page that is a dropdown. This is the minimum needed to have two distinct identities in
  two browser tabs. Put a red banner in the UI and a bold warning in the README:
  **"Demo authentication only. Never do this."**
- `Program.cs` must be short enough to read in one screen. The LiveSharp part must be exactly:

  ```csharp
  builder.Services.AddLiveSharp()
      .AddSignalR()
      .AddChat();
  // ...
  app.MapLiveSharp();
  ```

  If it takes more than these lines, the API failed the Spec §48 test and should be fixed in the
  library rather than papered over in the example.
- Register `InMemoryMessageStore` explicitly, with a comment explaining that the default is
  `NullMessageStore` and why the example overrides it.
- `chat.js` uses `@livesharp/client` from a CDN-style ESM import or a small bundled copy — do **not**
  set up a Node build pipeline for the MVC example. The MVC example's job is to show the simplest
  possible integration. A build step defeats that.
- The view **must HTML-encode message content on render** and carry a comment stating that LiveSharp
  deliberately does not sanitise, so encoding is the application's job. This is the example that
  teaches the security model.
- UI: user picker, message list, input box, connection-state indicator. No CSS framework, no build
  tooling, under 150 lines of JavaScript.

**`examples/AspNetCoreMvc/README.md`** must contain: what it demonstrates, how to run
(`dotnet run`), how to test with two browsers, the exact LiveSharp code with line references, a
**"What this example does NOT do"** section (no authorization — anyone can message anyone; no
receipts; no groups; no presence; demo auth), and a link to the phase that adds each missing piece.

**`examples/README.md`** — the index: which example demonstrates what, and which are planned.

**Tests**

Examples are not unit-tested, but add a smoke test to the `examples` CI job:
`dotnet build examples/LiveSharp.Examples.slnx` must succeed. In Phase 12, Playwright covers the
example UIs.

---

## Dependency justification

No new library dependencies. `LiveSharp.Chat` references `LiveSharp.Core` and
`LiveSharp.AspNetCore` by project reference only.

The MVC example adds no NuGet packages beyond the ASP.NET Core shared framework. If a temptation
arises to add a UI or JS build toolchain to the MVC example, resist it — that is what the React and
Next.js examples are for.

---

## Documentation deltas

- `docs/chat/direct-messaging.md` — **new**. Quick start, the full options table, the send sequence
  as numbered steps, multi-device behaviour, the echo-to-sender rule, idempotency and its dependence
  on a real store, offline behaviour and the stored-but-not-delivered semantic, and an explicit
  **Limitations** section: no authorization yet (Phase 09), no receipts (Phase 08), no history
  (Phase 14), messages sent while a recipient is disconnected are lost with the default
  configuration.
- `docs/chat/README.md` — index the chat pages.
- `docs/security/README.md` — **new content**: LiveSharp does not sanitise or encode message content;
  applications must encode on render. No authorization is enforced in `0.3.0`. Content is not
  encrypted end to end.
- `docs/configuration/README.md` — the `ChatOptions` table.
- `docs/getting-started/README.md` — extend to a working chat quick start, replacing the heartbeat
  example.
- `docs/troubleshooting/README.md` — "messages are duplicated on the sender's screen",
  "`AddChat` throws at startup", "recipient never receives anything" (user-ID resolution),
  "messages disappear after a refresh" (no store configured).
- `README.md` — flip direct messaging → `Available`; groups/presence/typing/receipts/calls remain
  `Planned`. Replace the quick start with the real one.

## Example deltas

- `examples/AspNetCoreMvc` — new, as specified.
- `examples/README.md` — new.

## CHANGELOG entry

```markdown
### Added
- `LiveSharp.Chat`: direct user-to-user messaging via the `chat.message.send` operation and the
  `chat.message.received` event.
- Time-sortable message identifiers based on UUIDv7.
- Deterministic, symmetric direct-conversation identifiers.
- Configurable message validation limits, offline behaviour, and sender echo.
- Optional `IMessageStore` with a no-op default, plus a bounded in-memory implementation for
  development.
- Client-supplied message identifiers for idempotent retries (requires a message store).
- Chat helpers for the .NET and TypeScript clients.
- ASP.NET Core MVC example application.

### Security
- Sender identity is taken from the authenticated connection and can never be supplied by the
  client payload.
- Message content is never written to logs unless sensitive payload logging is explicitly enabled.
```

---

## Exit criteria

- [ ] Two browser tabs in the MVC example exchange messages in real time.
- [ ] `SendMessageRequest` has no `SenderId` member, proven by a test.
- [ ] `Send_does_not_echo_to_the_originating_connection` passes.
- [ ] `Conversation_ids_do_not_collide_when_user_ids_contain_the_separator` passes.
- [ ] `Send_stores_the_message_before_delivering_it` passes.
- [ ] `Send_never_logs_message_content_by_default` passes.
- [ ] `Send_works_with_a_null_operation_context` passes — the service is not hub-coupled.
- [ ] The reconnect-window test pins the **actual** behaviour and the docs state the same thing.
- [ ] `MessageStoreContractTests` runs against both store implementations.
- [ ] `LiveSharp.slnx` contains no example project; the architecture test proves it.
- [ ] The `examples` CI job builds `examples/LiveSharp.Examples.slnx`.
- [ ] Chat constants exist in `protocol.ts` and the contract test passes.
- [ ] The `Program.cs` LiveSharp registration is three chained calls plus `MapLiveSharp()`.
- [ ] `docs/chat/direct-messaging.md` has a Limitations section naming every gap and its phase.
- [ ] `README.md` marks only direct messaging as `Available`.
- [ ] `scripts/verify.ps1` green; tag `v0.3.0`.

## Verify

```powershell
pwsh scripts/verify.ps1
dotnet build examples/LiveSharp.Examples.slnx
dotnet run --project examples/AspNetCoreMvc     # then open two browsers, log in as alice and bob
git tag v0.3.0
```

## Next

`docs/plan/phase-05-groups.md`, task `P05.T1`.
