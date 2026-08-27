# Phase 14 — Persistence

| | |
|---|---|
| **Version at exit** | `0.13.0` (tag `v0.13.0`) |
| **Depends on** | Phase 13 |
| **Branch** | `feat/phase-14-persistence` |
| **Tasks** | `P14.T1` … `P14.T6` |
| **Read before starting** | every `*StoreContractTests` written in Phases 04–11, and `docs/architecture/overview.md` |

---

## Goal

A new optional package, `LiveSharp.EntityFrameworkCore`, implementing the store interfaces defined
across Phases 04–11 against a real database, plus the **history query API** those phases deliberately
deferred.

The core packages must remain EF-free. A consumer who wants in-memory real-time messaging with no
database must still get exactly that, and an architecture test proves it.

This phase is where the store-contract-test discipline pays off: five abstract contract suites already
exist, and the EF implementations inherit all of them. If they pass, the provider is correct by
construction against everything the rest of the library expects.

---

## What was deferred to here, and why it was right to defer it

| Deferred | From | Reason it needed a database first |
|---|---|---|
| Message history and pagination | `P04.T3` | A cursor API designed without a query planner in mind is a cursor API that cannot be indexed |
| Group persistence | `P05.T1` | — |
| Receipt persistence and read positions | `P08.T2` | Unread counts need an efficient `COUNT(*) WHERE state < Read`, which is meaningless in a dictionary |
| Call history | `P11.T2` | — |

Presence and typing are **not** persisted. Presence is derived state that is wrong the moment a process
dies; typing is ephemeral by definition (`P07` non-goals). Persisting either would be storing a cache in
a database. State this explicitly so nobody adds it later "for completeness".

---

## Non-goals — DO NOT implement in this phase

- No coupling of any core package to EF Core. Architecture test enforced.
- No presence or typing persistence. See above.
- No signalling session persistence. Sessions are sub-minute-lived; a database write per SDP exchange is
  latency for no benefit.
- No Dapper, raw-ADO, MongoDB, or Cosmos providers. One provider, done properly. Others can be built by
  consumers against the same interfaces — that is what the interfaces are for.
- No full-text search over messages. That needs a search engine, not an `ORM LIKE '%x%'`.
- No message editing, deletion, or soft-delete. Not in the pre-1.0 feature surface.
- No automatic migration execution at startup. `Database.Migrate()` on boot is a production footgun in
  multi-instance deployments. Provide the migrations; the operator runs them.
- No multi-tenancy schema, sharding, or partitioning strategy.
- No archival/cold storage tiering.

---

## Tasks

### P14.T1 — The history query API

**Deliverables**

```
src/LiveSharp.Abstractions/Paging/CursorPage.cs
src/LiveSharp.Abstractions/Paging/PageRequest.cs
src/LiveSharp.Chat/Storage/IMessageHistoryStore.cs
src/LiveSharp.Chat/Storage/InMemoryMessageStore.cs            (implements history)
src/LiveSharp.Chat/IMessageHistoryService.cs
src/LiveSharp.Chat/MessageHistoryService.cs
src/LiveSharp.Chat/Protocol/*.cs                              (history operations)
src/LiveSharp.Chat/Handlers/GetHistoryHandler.cs
tests/LiveSharp.Chat.Tests/Storage/MessageHistoryContractTests.cs
```

Design this **before** touching EF, so the interface is shaped by what a database can do efficiently
rather than by what a dictionary makes easy.

**Public API (exact):**

```csharp
namespace LiveSharp;

/// <summary>An opaque forward-only cursor request.</summary>
public sealed record PageRequest
{
    /// <summary>Maximum items to return. Clamped to the configured maximum.</summary>
    public int Limit { get; init; } = 50;

    /// <summary>Opaque cursor from a previous page. Null starts from the newest item.</summary>
    public string? Cursor { get; init; }
}

/// <summary>A page of results plus the cursor for the next page.</summary>
public sealed record CursorPage<T>
{
    public required IReadOnlyList<T> Items { get; init; }

    /// <summary>Cursor for the next page, or null when there are no more items.</summary>
    public string? NextCursor { get; init; }

    public bool HasMore => NextCursor is not null;

    public static CursorPage<T> Empty { get; }
}
```

```csharp
namespace LiveSharp.Chat.Storage;

public interface IMessageHistoryStore
{
    /// <summary>Messages in a conversation, newest first.</summary>
    ValueTask<CursorPage<ChatMessage>> GetConversationAsync(string conversationId, PageRequest page, CancellationToken cancellationToken = default);

    ValueTask<ChatMessage?> FindByIdAsync(string messageId, CancellationToken cancellationToken = default);

    /// <summary>Conversations a user participates in, most recently active first, with the latest message.</summary>
    ValueTask<CursorPage<ConversationSummary>> GetConversationsForUserAsync(string userId, PageRequest page, CancellationToken cancellationToken = default);

    /// <summary>Messages in a conversation the user has not yet read.</summary>
    ValueTask<int> GetUnreadCountAsync(string conversationId, string userId, CancellationToken cancellationToken = default);

    ValueTask<int> PruneAsync(DateTimeOffset olderThan, int limit, CancellationToken cancellationToken = default);
}

public sealed record ConversationSummary
{
    public required string ConversationId { get; init; }
    public required DateTimeOffset LastActivityAt { get; init; }
    public ChatMessage? LastMessage { get; init; }
    public int UnreadCount { get; init; }
}
```

**Requirements — cursor design, which is the whole point of this task**

- **Keyset pagination, not offset.** `OFFSET 10000` degrades linearly and returns duplicates or skips when
  rows are inserted between pages — which in a chat application happens constantly. This is not a
  preference; offset pagination is wrong for a live message feed.
- The cursor encodes the last-seen message ID. Because message IDs are UUIDv7 (Phase 04) they are
  monotonically sortable, so the query is `WHERE ConversationId = @c AND Id < @cursor ORDER BY Id DESC
  LIMIT @n` — a single index seek. Document that this is *why* UUIDv7 was chosen in Phase 04; the
  decision only pays off here.
- The cursor must be **opaque** to callers: Base64Url of a small versioned payload, not a raw ID. Callers
  who parse cursors build dependencies on the internal ordering, and then it can never change. Include a
  version byte so the format can evolve. Reject a malformed or wrong-version cursor with
  `RealtimeErrorCode.InvalidPayload` and a message saying to restart pagination — never throw, and never
  silently reset to page one (which produces an infinite loop in a client that keeps paging).
- `Limit` clamped to `ChatOptions.MaxHistoryPageSize` (new, default 100). A client requesting 100 000 must
  get 100, not an error — clamping is friendlier than failing and the client gets `NextCursor` anyway.
- `GetConversationsForUserAsync` is the expensive one: it needs "conversations this user is in" which for
  direct messages is derivable from the conversation ID (`dm:a|b` contains the user ID) but is a `LIKE`
  query, and for groups requires a join. Solve it with an explicit `ConversationParticipant` projection
  table maintained on write, not with a clever query. Denormalising here is correct: reads vastly
  outnumber writes, and the alternative is an unindexable pattern match. Document the trade-off.
- `GetUnreadCountAsync` requires the receipt table from Phase 08. If `AddMessageState()` is not
  registered, return `0` and log at `Debug` once — do not throw, and do not silently return a wrong
  number without saying why.
- `IMessageHistoryStore` is **separate** from `IMessageStore`. `IMessageStore` (two methods, write path)
  stays as it is; history is an additional capability a store may implement. `NullMessageStore` does not
  implement it, and `MessageHistoryService` returns a clear `Failure(NotFound, "No message history store
  is configured. Register LiveSharp.EntityFrameworkCore or an implementation of IMessageHistoryStore.")`
  Splitting the interfaces keeps the no-database path genuinely free of history concepts.
- `InMemoryMessageStore` implements `IMessageHistoryStore` too, so the contract suite runs against both
  and the examples get history without a database.

**Operations** — `chat.history.get`, `chat.conversations.get`, `chat.unread.get`. All `InvokeAsync`, all
authorized: the caller must be a participant of the conversation
(`IRealtimeAuthorizationService.CanAccessGroupAsync(Read)` for groups, participant check for direct).
Add to the Phase 09 `KnownOperationPolicies` manifest and the rate-limit table (history 30/60s —
pagination is bursty).

**Tests — `MessageHistoryContractTests` (abstract, run against in-memory and EF)**

| Test | Asserts |
|---|---|
| `Get_conversation_returns_newest_first` | |
| `Get_conversation_respects_the_limit` | |
| `Get_conversation_clamps_an_excessive_limit` | 100 000 → max |
| `Get_conversation_paginates_through_every_message_exactly_once` | 250 messages, 3 pages, no duplicates, no gaps |
| `Get_conversation_returns_a_null_next_cursor_on_the_last_page` | |
| `Get_conversation_of_an_empty_conversation_returns_empty_with_no_cursor` | |
| `Inserting_messages_between_pages_does_not_duplicate_or_skip_results` | **the keyset-pagination test** |
| `A_malformed_cursor_is_rejected_with_invalid_payload` | table-driven: `""`, `"abc"`, wrong version, truncated |
| `A_cursor_from_a_different_conversation_returns_no_results_rather_than_leaking` | **security test** |
| `Find_by_id_returns_the_message` | |
| `Find_by_id_of_an_unknown_message_returns_null` | |
| `Get_conversations_for_user_returns_most_recently_active_first` | |
| `Get_conversations_for_user_includes_the_last_message` | |
| `Get_conversations_for_user_excludes_conversations_the_user_is_not_in` | **security test** |
| `Get_conversations_for_user_paginates_correctly` | |
| `Get_unread_count_counts_only_messages_below_read` | |
| `Get_unread_count_excludes_the_users_own_messages` | you have read what you sent |
| `Get_unread_count_returns_zero_without_a_state_store` | documented degradation |
| `Prune_removes_messages_older_than_the_cutoff` | |
| `Prune_honours_the_limit` | |
| `All_methods_honour_a_pre_cancelled_token` | |

---

### P14.T2 — The EF Core package and schema

**Deliverables**

```
src/LiveSharp.EntityFrameworkCore/LiveSharp.EntityFrameworkCore.csproj
src/LiveSharp.EntityFrameworkCore/PublicAPI.{Shipped,Unshipped}.txt
src/LiveSharp.EntityFrameworkCore/LiveSharpDbContext.cs
src/LiveSharp.EntityFrameworkCore/Entities/*.cs
src/LiveSharp.EntityFrameworkCore/Configurations/*.cs
src/LiveSharp.EntityFrameworkCore/Configuration/LiveSharpEfOptions.cs
tests/LiveSharp.EntityFrameworkCore.Tests/…
```

**Public API (exact):**

```csharp
namespace LiveSharp.EntityFrameworkCore;

/// <summary>LiveSharp's persistence context. May be used standalone or inherited by an application context.</summary>
public class LiveSharpDbContext : DbContext
{
    public LiveSharpDbContext(DbContextOptions<LiveSharpDbContext> options);

    public DbSet<ChatMessageEntity> Messages => Set<ChatMessageEntity>();
    public DbSet<ConversationParticipantEntity> ConversationParticipants => Set<ConversationParticipantEntity>();
    public DbSet<MessageReceiptEntity> MessageReceipts => Set<MessageReceiptEntity>();
    public DbSet<GroupEntity> Groups => Set<GroupEntity>();
    public DbSet<GroupMemberEntity> GroupMembers => Set<GroupMemberEntity>();
    public DbSet<CallEntity> Calls => Set<CallEntity>();
    public DbSet<CallParticipantEntity> CallParticipants => Set<CallParticipantEntity>();

    /// <summary>Applies LiveSharp's entity configuration. Call from an inheriting context's OnModelCreating.</summary>
    public static void ApplyLiveSharpConfiguration(ModelBuilder modelBuilder, LiveSharpEfOptions? options = null);
}
```

**Requirements**

- `LiveSharpDbContext` is **not sealed**, and `ApplyLiveSharpConfiguration` is public static, so a consumer
  can put LiveSharp's tables in their existing `AppDbContext` — which is what most real applications
  want, because a second `DbContext` means a second connection pool and no shared transactions. This is
  the one place in the library where inheritance is a supported extension point; mark it with the
  `[DesignedForInheritance]` attribute the Phase 01 architecture test requires.
- `LiveSharpEfOptions`: `TableNamePrefix` (default `LiveSharp`), `Schema` (default null), and
  `UseUtcDateTimeOffset`. Prefixing avoids colliding with an application's own `Messages` or `Groups`
  table — near-certain otherwise.
- Entities are **separate types** from the domain records. `ChatMessageEntity` is not `ChatMessage`.
  Mapping costs a little code and buys: mutable entities for EF change tracking, storage-appropriate
  types, indexes and column facets without polluting the public model, and freedom to change the schema
  without a public API break. Write explicit mapping methods, not AutoMapper.
- Indexes, which are the actual deliverable of this task:

| Table | Index | Serves |
|---|---|---|
| `Messages` | `(ConversationId, Id DESC)` | `GetConversationAsync` — the primary read path |
| `Messages` | `(SenderId, ClientMessageId)` unique filtered | Phase 04 idempotency lookup |
| `Messages` | `(SentAt)` | `PruneAsync` |
| `ConversationParticipants` | `(UserId, LastActivityAt DESC)` | `GetConversationsForUserAsync` |
| `ConversationParticipants` | `(ConversationId, UserId)` primary key | participation check |
| `MessageReceipts` | `(MessageId, UserId)` primary key | `AdvanceAsync` |
| `MessageReceipts` | `(UserId, State)` filtered `State < 3` | unread counts |
| `GroupMembers` | `(GroupId, UserId)` primary key | membership |
| `GroupMembers` | `(UserId)` | `GetGroupIdsForUserAsync` and Phase 05 rehydration |
| `Calls` | `(State, UpdatedAt)` | Phase 11 timeout sweeps |
| `CallParticipants` | `(UserId, CallId)` | call history |

Every index must be justified by a named query in a comment. An index with no query is write cost for
nothing; a query with no index is a table scan waiting for production traffic.

- `Metadata` dictionaries persist as JSON columns (`ToJson()` / `OwnsMany` with JSON, whichever the EF 10
  provider supports cleanly). Do **not** create a key-value side table — metadata is read whole or not at
  all.
- Concurrency: the `TryUpdateAsync`/`TrySetAsync` optimistic pattern from Phases 06, 08, 10 and 11 maps to
  a concurrency token on `UpdatedAt`. Configure `IsConcurrencyToken()` and translate
  `DbUpdateConcurrencyException` into the `false` return the contract expects — **never** let it escape,
  because every caller is written to retry on `false`, not to catch EF exceptions.
- Enums persist as `int` with the pinned numeric values from their phases. Never as strings: the pinning
  tests exist to make the integers a contract, and string storage bloats indexes.

**Tests**

| Test | Asserts |
|---|---|
| `Model_builds_without_error_on_sqlite_and_postgres` | |
| `Every_declared_index_exists_in_the_model` | reflection over the model, matched to an expected list |
| `Table_names_carry_the_configured_prefix` | |
| `Custom_schema_is_applied` | |
| `An_inheriting_context_can_apply_the_configuration` | the extension point works |
| `An_inheriting_context_can_add_its_own_entities_alongside` | |
| `Metadata_round_trips_as_json` | including empty and unicode |
| `Enums_persist_as_their_pinned_integer_values` | |
| `Concurrency_token_conflict_returns_false_rather_than_throwing` | **the contract translation** |
| `Entity_and_domain_mapping_round_trips_every_property` | reflection: no property silently unmapped |

---

### P14.T3 — Store implementations

**Deliverables**

```
src/LiveSharp.EntityFrameworkCore/Stores/EfMessageStore.cs
src/LiveSharp.EntityFrameworkCore/Stores/EfMessageHistoryStore.cs
src/LiveSharp.EntityFrameworkCore/Stores/EfMessageStateStore.cs
src/LiveSharp.EntityFrameworkCore/Stores/EfGroupStore.cs
src/LiveSharp.EntityFrameworkCore/Stores/EfCallStore.cs
tests/LiveSharp.EntityFrameworkCore.Tests/Stores/*.cs
```

**Requirements**

- Each store inherits the corresponding abstract contract suite from its feature package's test project.
  That requires the contract suites to be **published in a shared test library** rather than living inside
  `tests/LiveSharp.Chat.Tests`. Extract them into `tests/LiveSharp.StoreContracts` (non-packable) in this
  task and update the Phase 04–11 test projects to reference it. This is a mechanical refactor and it is
  the single highest-value thing in the phase: it makes the EF provider's correctness a matter of running
  existing tests.
- `DbContext` lifetime: stores take `IDbContextFactory<LiveSharpDbContext>`, not an injected `DbContext`.
  `DbContext` is not thread-safe, and LiveSharp's stores are singletons called concurrently from many
  connections. An injected scoped `DbContext` in a singleton store is a guaranteed
  `InvalidOperationException` under load — the classic EF-in-a-real-time-app failure. Document this
  loudly and register with `AddDbContextFactory`.
- Every query must be `AsNoTracking()` for reads. Tracking thousands of messages per second is pure
  allocation.
- Every write path must handle the unique-constraint violation for idempotency
  (`ClientMessageId`) by treating it as "already exists" and returning the existing row, not by
  pre-checking. Check-then-insert races under concurrent retries; insert-and-catch does not. Translating
  a provider-specific unique-violation error is unpleasant — use EF's `DbUpdateException` with an inner
  provider exception, and document that the translation is tested per provider.
- `EfMessageStateStore.AdvanceAsync` must not read-then-write. Use a single `ExecuteUpdateAsync` with a
  `WHERE State < @requested` predicate so the monotonic rule is enforced by the database and concurrent
  acks cannot regress state. Return the transition by reading the affected count plus one follow-up read.
  This is the one place where a raw set-based update genuinely beats change tracking, and Phase 08's
  contract tests will prove the semantics are preserved.
- `EfCallStore.GetStuckAsync` must use the `(State, UpdatedAt)` index and a bounded `Take`.
- `ConversationParticipantEntity` maintenance: `EfMessageStore.AddAsync` upserts participant rows for the
  sender and recipients and bumps `LastActivityAt`. This is the denormalisation from `P14.T1`. It must be
  in the **same transaction** as the message insert, or a crash between them silently hides a conversation
  from a user's list.
- No `Database.EnsureCreated()` and no `Migrate()` anywhere in library code.

**Tests** — beyond the inherited contract suites:

| Test | Asserts |
|---|---|
| `Stores_use_a_context_factory_not_an_injected_context` | reflection on constructor parameters |
| `Concurrent_calls_to_one_store_instance_do_not_share_a_context` | `ConcurrencyHarness`, 32 workers |
| `Read_queries_do_not_track_entities` | assert `ChangeTracker.Entries()` is empty after a read |
| `Duplicate_client_message_id_insert_returns_the_existing_message` | insert-and-catch path |
| `Concurrent_duplicate_client_message_id_inserts_produce_one_row` | the race |
| `Advance_uses_a_single_set_based_update` | captured SQL contains no `SELECT` before the `UPDATE` |
| `Concurrent_advances_cannot_regress_state` | database-enforced monotonicity |
| `Message_insert_and_participant_upsert_share_a_transaction` | kill mid-way via an interceptor, assert neither landed |
| `Get_conversation_query_uses_the_conversation_id_index` | captured SQL / execution plan on SQLite is weak — assert on generated SQL shape instead and note the limitation |
| `No_library_code_calls_migrate_or_ensure_created` | source or IL scan |

---

### P14.T4 — Migrations, DI, and provider testing

**Deliverables**

```
src/LiveSharp.EntityFrameworkCore/Migrations/…                  (initial migration)
src/LiveSharp.EntityFrameworkCore/DependencyInjection/LiveSharpEfBuilderExtensions.cs
tests/LiveSharp.EntityFrameworkCore.Tests/Providers/SqliteTests.cs
tests/LiveSharp.EntityFrameworkCore.Tests/Providers/PostgresTests.cs
docs/configuration/persistence.md
```

**Public API (exact):**

```csharp
namespace LiveSharp;

public static class LiveSharpEfBuilderExtensions
{
    /// <summary>Registers Entity Framework Core implementations of the LiveSharp stores.</summary>
    public static ILiveSharpBuilder AddEntityFrameworkStores(
        this ILiveSharpBuilder builder,
        Action<DbContextOptionsBuilder> configureDbContext,
        Action<LiveSharpEfOptions>? configureOptions = null);

    /// <summary>Registers stores against a context the application already configured.</summary>
    public static ILiveSharpBuilder AddEntityFrameworkStores<TContext>(this ILiveSharpBuilder builder, Action<LiveSharpEfOptions>? configureOptions = null)
        where TContext : DbContext;
}
```

**Requirements**

- Two overloads because there are two real scenarios: LiveSharp owns its context, or the application does.
  The generic overload requires `TContext` to expose the LiveSharp `DbSet`s — enforce with a startup
  validation that produces a clear error naming the missing sets, not a runtime `NullReferenceException`.
- Registration must **replace** the in-memory stores (`Replace`, not `TryAdd`), and log at `Information`
  which stores were replaced. A consumer who calls `AddEntityFrameworkStores()` and still gets
  `NullMessageStore` because of registration order will lose an afternoon.
- Registration is additive per store: if only chat is registered, do not register `EfCallStore` for an
  `ICallStore` nobody asked for. Detect which feature packages are present by checking for their marker
  services, and log which stores were wired.
- **Migrations are shipped in the package** as compiled `Migration` classes, generated for a
  provider-agnostic model where possible. Provider-specific column types (JSON, `timestamptz`) mean a
  truly single migration set is unlikely; ship SQL scripts per provider in
  `src/LiveSharp.EntityFrameworkCore/Migrations/Scripts/{provider}/` and document
  `dotnet ef migrations script` as the supported path for consumers with their own context. Be honest in
  the docs: a library shipping EF migrations for a context the consumer may inherit is genuinely awkward,
  and the recommended approach for production is that the consumer generates and reviews their own
  migrations.
- Test against **SQLite** (fast, in the unit test loop) and **PostgreSQL via Testcontainers** (real
  provider behaviour: JSON columns, `timestamptz`, unique-violation error codes, concurrency). SQLite
  alone would let provider-specific bugs through; Postgres alone would make the test suite slow and
  require Docker locally. Run the full contract suites on both. Tag the Postgres suite so it can be
  skipped locally (`-SkipContainers`) but always runs in CI.
- SQL Server is **not** tested. Say so. Adding it is a matter of adding a Testcontainers module and a
  provider script directory, and a contributor can do it; claiming support without testing it would be
  dishonest.

**Tests**

| Test | Asserts |
|---|---|
| `AddEntityFrameworkStores_replaces_the_in_memory_message_store` | |
| `AddEntityFrameworkStores_logs_which_stores_it_replaced` | |
| `AddEntityFrameworkStores_does_not_register_a_call_store_when_calls_are_not_added` | |
| `AddEntityFrameworkStores_before_AddChat_still_wires_the_message_store` | order independence |
| `Generic_overload_with_a_context_missing_the_live_sharp_sets_fails_at_startup_naming_them` | |
| `AddEntityFrameworkStores_registers_a_context_factory` | |
| `Every_contract_suite_passes_on_sqlite` | inherited suites |
| `Every_contract_suite_passes_on_postgres` | inherited suites, Testcontainers |
| `Initial_migration_creates_every_table_and_index_on_postgres` | |
| `Migration_is_idempotent_when_applied_twice` | |

---

### P14.T5 — Retention, call history, and background maintenance

**Deliverables**

```
src/LiveSharp.Chat/Retention/MessageRetentionService.cs
src/LiveSharp.Chat/Configuration/ChatOptions.cs                 (modified)
src/LiveSharp.Calls/ICallHistoryService.cs
src/LiveSharp.Calls/CallHistoryService.cs
src/LiveSharp.Calls/Storage/ICallHistoryStore.cs
src/LiveSharp.Calls/Protocol/*.cs
src/LiveSharp.Calls/Handlers/GetCallHistoryHandler.cs
tests/…
```

**Requirements**

- `ChatOptions.MessageRetention` (`TimeSpan?`, default `null` = keep forever) and
  `MessageRetentionSweepInterval` (default 6 hours). `MessageRetentionService : BackgroundService` calls
  `IMessageHistoryStore.PruneAsync` in bounded batches. Default `null` because silently deleting a
  consumer's chat history would be an appalling default. Log at `Warning` on the first sweep when
  retention is configured, stating how many messages it will delete — a mis-set retention should be
  loud before it is destructive.
- Retention must delete receipts and participant rows for pruned messages, or the tables grow forever
  while messages shrink. Cascade delete at the schema level is the right tool here; verify it works on
  both providers.
- `ICallHistoryService.GetHistoryAsync(userId, PageRequest)` returning `CursorPage<Call>`, plus
  `GetMissedAsync`. Operation `call.history.get`, authorized to the calling user only — a user may read
  their own call history and nobody else's, and there is no admin surface in 1.0.
- Call history is the one place where storing participant identities is the **point**, so the Phase 11
  "do not log participant IDs" rule does not extend to persistence. Note the distinction in
  `docs/security/README.md`: LiveSharp does not *log* call metadata, but it does *store* it when a call
  store is configured, and that store is a call-detail record with the associated regulatory weight.
  Applications in regulated environments must know this.

**Tests**

| Test | Asserts |
|---|---|
| `Retention_disabled_by_default_deletes_nothing` | |
| `Retention_sweep_deletes_messages_past_the_cutoff` | |
| `Retention_sweep_cascades_to_receipts_and_participants` | both providers |
| `Retention_sweep_is_bounded_per_tick` | |
| `Retention_sweep_logs_a_warning_on_first_run_with_the_affected_count` | |
| `Retention_sweep_survives_a_throwing_store` | |
| `Call_history_returns_the_users_calls_newest_first` | |
| `Call_history_paginates_correctly` | |
| `Call_history_excludes_calls_the_user_was_not_in` | **security test** |
| `Missed_calls_returns_only_timeout_and_rejected_calls_where_the_user_was_the_callee` | |
| `Call_history_for_another_user_is_forbidden` | |

---

### P14.T6 — Examples, documentation, and the architecture guarantee

**Deliverables**

```
examples/AspNetCoreMvc/…                                        (SQLite + history)
examples/React/…                                                (infinite scroll + unread badges)
tests/LiveSharp.ArchitectureTests/PersistenceIsolationTests.cs
docs/configuration/persistence.md
docs/chat/history.md
```

**Requirements — the architecture guarantee**

Add explicit tests that persistence is genuinely optional:

| Test | Asserts |
|---|---|
| `No_core_assembly_references_entity_framework` | `Abstractions`, `Core`, `AspNetCore`, `Chat`, `Presence`, `Signaling`, `Calls` |
| `An_application_with_no_store_registered_starts_and_sends_messages` | full in-memory smoke test |
| `An_application_with_no_history_store_returns_a_clear_error_from_history_operations` | the message names the package to install |
| `LiveSharp_EntityFrameworkCore_is_not_referenced_by_the_metapackage` | opt-in stays opt-in |

**Requirements — examples**

- MVC example switches to SQLite with `AddEntityFrameworkStores`, ships a `.gitignore`d database file,
  and demonstrates loading the last 50 messages on page load. Its README must show the `dotnet ef
  database update` step explicitly — the example is where a developer learns that migrations are their
  responsibility.
- React example demonstrates **infinite scroll** with the cursor API: load newest 50, fetch older on
  scroll-to-top, and preserve scroll position while prepending (the part everyone gets wrong). Plus
  unread badges from `chat.unread.get`. This is the example that makes the cursor API's shape make sense.
- Keep at least one example (Blazor WASM) with **no** persistence at all, so the no-database path stays
  exercised and visible.

**Requirements — documentation**

`docs/configuration/persistence.md`: the two registration overloads and when to use each, the
`IDbContextFactory` requirement and why, the inherited-context pattern, migrations (including the honest
"generate your own for production" recommendation), tested providers (SQLite, PostgreSQL) and untested
ones, connection resiliency (`EnableRetryOnFailure` and its interaction with the optimistic-concurrency
retry loops — they must not multiply), the index list and the queries they serve, and retention.

`docs/chat/history.md`: the cursor model, why keyset and not offset, opaque cursors and the
restart-pagination error, page-size clamping, unread counts and their dependency on `AddMessageState()`,
the `ConversationParticipant` denormalisation, and a worked infinite-scroll implementation.

---

## Dependency justification

| Package | Why | Alternative? | Licence | Notes |
|---|---|---|---|---|
| `Microsoft.EntityFrameworkCore.Relational` | The provider's model, migrations, and set-based operations. | Dapper (no migrations, hand-written SQL per provider — more code, more provider drift), raw ADO (worse). | MIT | **Only** in `LiveSharp.EntityFrameworkCore`. An architecture test proves no other shipped assembly references it. |
| `Microsoft.EntityFrameworkCore.Sqlite` | Test provider, fast inner loop. | — | MIT | Test-only. |
| `Npgsql.EntityFrameworkCore.PostgreSQL` | Test provider, real relational behaviour. | — | PostgreSQL licence | Test-only. |
| `Testcontainers.PostgreSql` | Real PostgreSQL in CI without a shared fixture database. | A CI service container (less portable, no local parity). | MIT | Test-only. |
| `Microsoft.EntityFrameworkCore.Design` | Generating the migrations. | — | MIT | `PrivateAssets=all`, build-time only. |

Explicitly rejected: AutoMapper (explicit mapping is clearer and allocation-free), any repository
abstraction over EF (the store interfaces already are that abstraction — a second layer would be pure
ceremony), `EFCore.BulkExtensions` (no measured need; revisit in Phase 16 if pruning proves slow).

---

## Documentation deltas

- `docs/configuration/persistence.md` — **new**, as specified.
- `docs/chat/history.md` — **new**, as specified.
- `docs/calls/history.md` — **new**. Call history, missed calls, and the call-detail-record privacy note.
- `docs/chat/delivery-state.md` — replace the "no unread counts" limitation with a link to history.
- `docs/chat/direct-messaging.md` — remove the "no history" limitation; explain that messages sent while
  a recipient is offline are now retrievable through history when a store is configured, which finally
  resolves the Phase 04 reconnect-window honesty note.
- `docs/security/README.md` — persistence stores message content, group rosters, receipts (read timing),
  and call-detail records; retention is off by default; this is a data-protection consideration.
- `docs/architecture/overview.md` — the persistence boundary and the "core is EF-free" guarantee.
- `docs/troubleshooting/README.md` — "`InvalidOperationException: A second operation started on this
  context`" (the injected-context mistake), "history returns an error" (no store registered), "cursor
  rejected" (restart pagination), "messages disappear" (retention), "migrations not applied", "unread
  counts are always zero" (`AddMessageState` missing), "slow history queries" (missing index / wrong
  provider migration).
- `README.md` — persistence → `Available` with the tested-providers list.

## Example deltas

- `examples/AspNetCoreMvc` — SQLite persistence, history on load, migration instructions.
- `examples/React` — infinite scroll with scroll-position preservation, unread badges.
- `examples/BlazorWasm` — deliberately left with **no** persistence, documented as the in-memory path.

## CHANGELOG entry

```markdown
### Added
- `LiveSharp.EntityFrameworkCore`: optional Entity Framework Core persistence for messages, groups,
  message receipts, and calls, tested against SQLite and PostgreSQL.
- Message history with opaque keyset cursors: `chat.history.get`, `chat.conversations.get`, and
  `chat.unread.get`.
- Call history and missed-call queries.
- Configurable message retention with a bounded background prune sweep, disabled by default.
- Shared store contract test suites, so any third-party store implementation can prove its correctness
  against the same tests LiveSharp's own implementations pass.

### Notes
- Presence and typing state are deliberately not persisted; both are derived or ephemeral.
- Stores resolve a `DbContext` per operation from `IDbContextFactory`, because LiveSharp's stores are
  singletons invoked concurrently.
- LiveSharp ships migration scripts, but generating and reviewing migrations against your own context is
  the recommended production path.
- Enabling a call store creates call-detail records. Consider your data-protection obligations.
```

---

## Exit criteria

- [ ] `No_core_assembly_references_entity_framework` passes for all seven core/feature assemblies.
- [ ] `An_application_with_no_store_registered_starts_and_sends_messages` passes.
- [ ] Every store contract suite from Phases 04–11 passes against **both** SQLite and PostgreSQL.
- [ ] `Inserting_messages_between_pages_does_not_duplicate_or_skip_results` passes.
- [ ] `A_cursor_from_a_different_conversation_returns_no_results_rather_than_leaking` passes.
- [ ] `Stores_use_a_context_factory_not_an_injected_context` passes.
- [ ] `Concurrent_calls_to_one_store_instance_do_not_share_a_context` passes.
- [ ] `Concurrent_advances_cannot_regress_state` passes — monotonicity is database-enforced.
- [ ] `Message_insert_and_participant_upsert_share_a_transaction` passes.
- [ ] `Every_declared_index_exists_in_the_model` passes, and every index has a justifying comment.
- [ ] `Concurrency_token_conflict_returns_false_rather_than_throwing` passes.
- [ ] `No_library_code_calls_migrate_or_ensure_created` passes.
- [ ] `Retention_sweep_cascades_to_receipts_and_participants` passes on both providers.
- [ ] Contract suites live in `tests/LiveSharp.StoreContracts` and are referenced by every store test
      project.
- [ ] The React example's infinite scroll preserves scroll position while prepending.
- [ ] `examples/BlazorWasm` still runs with no database.
- [ ] `docs/configuration/persistence.md` is honest about the migrations story and the untested providers.
- [ ] History and call-history operations are in the `KnownOperationPolicies` manifest and the rate-limit
      table.
- [ ] `scripts/verify.ps1` green, including the Postgres container suite in CI; tag `v0.13.0`.

## Verify

```powershell
pwsh scripts/verify.ps1
dotnet test tests/LiveSharp.EntityFrameworkCore.Tests -c Release
dotnet test tests/LiveSharp.ArchitectureTests -c Release
dotnet ef migrations list --project src/LiveSharp.EntityFrameworkCore
dotnet run --project examples/AspNetCoreMvc      # after: dotnet ef database update
git tag v0.13.0
```

## Next

`docs/plan/phase-15-distributed.md`, task `P15.T1`.
