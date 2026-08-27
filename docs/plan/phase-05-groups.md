# Phase 05 — Groups & Channels

| | |
|---|---|
| **Version at exit** | `0.4.0` (tag `v0.4.0`) |
| **Depends on** | Phase 04 |
| **Branch** | `feat/phase-05-groups` |
| **Tasks** | `P05.T1` … `P05.T6` |
| **Read before starting** | `docs/plan/phase-04-direct-messaging.md`, `docs/architecture/connection-lifecycle.md`, `docs/architecture/wire-protocol.md` |

---

## Goal

Many-to-many messaging: named groups with membership, roles, and fan-out. Built on the
`IGroupRegistry` from Phase 02 and the chat pipeline from Phase 04, in the `LiveSharp.Chat` package.

This phase contains one problem that will silently break production if handled casually:
**transport group membership does not survive a reconnect.** `P05.T4` exists solely for it.

---

## Non-goals — DO NOT implement in this phase

- No presence (Phase 06), typing (Phase 07), or receipts (Phase 08) for groups. Phase 08 handles
  per-recipient group receipts.
- No authorization. Role checks are enforced *within* the group model, but there is no policy layer
  and no `IRealtimeAuthorizationService` yet — a connected user can still create groups freely and
  read any group they can name. Phase 09.
- No persistence provider. `InMemoryGroupStore` only; EF Core is Phase 14.
- No group discovery, search, listing of all groups, or public/private visibility model.
- No invitations, join requests, bans, mutes, or moderation beyond the three roles.
- No group avatars, topics, pinned messages, or per-group settings beyond `Metadata`.
- No group message history API. Phase 14.
- No distributed group state. Phase 15.

---

## Tasks

### P05.T1 — Group model, store, and options

**Deliverables**

```
src/LiveSharp.Chat/Groups/RealtimeGroup.cs
src/LiveSharp.Chat/Groups/GroupMember.cs
src/LiveSharp.Chat/Groups/GroupRole.cs
src/LiveSharp.Chat/Groups/GroupTransportName.cs
src/LiveSharp.Chat/Groups/Storage/IGroupStore.cs
src/LiveSharp.Chat/Groups/Storage/InMemoryGroupStore.cs
src/LiveSharp.Chat/Configuration/GroupOptions.cs
src/LiveSharp.Chat/Configuration/GroupOptionsValidator.cs
tests/LiveSharp.Chat.Tests/Groups/Storage/GroupStoreContractTests.cs
tests/LiveSharp.Chat.Tests/Groups/Storage/InMemoryGroupStoreTests.cs
```

**Public API (exact):**

```csharp
namespace LiveSharp.Chat;

/// <summary>A named set of users that can be addressed as one recipient.</summary>
public sealed record RealtimeGroup
{
    /// <summary>Server-assigned, time-sortable identifier. Used in all wire operations.</summary>
    public required string Id { get; init; }

    /// <summary>Human-readable display name. Not unique and never used for routing.</summary>
    public required string Name { get; init; }

    public required string OwnerId { get; init; }
    public required DateTimeOffset CreatedAt { get; init; }
    public IReadOnlyDictionary<string, string> Metadata { get; init; }
}

public enum GroupRole
{
    Member = 0,
    Admin  = 1,
    Owner  = 2,
}

public sealed record GroupMember(
    string GroupId,
    string UserId,
    GroupRole Role,
    DateTimeOffset JoinedAt);

/// <summary>Maps a group identifier onto the transport group name used for fan-out.</summary>
public static class GroupTransportName
{
    public const string Prefix = "lsg:";
    public static string For(string groupId);
}
```

```csharp
namespace LiveSharp.Chat.Storage;

public interface IGroupStore
{
    ValueTask AddGroupAsync(RealtimeGroup group, CancellationToken cancellationToken = default);
    ValueTask<bool> DeleteGroupAsync(string groupId, CancellationToken cancellationToken = default);
    ValueTask<RealtimeGroup?> FindGroupAsync(string groupId, CancellationToken cancellationToken = default);

    ValueTask AddMemberAsync(GroupMember member, CancellationToken cancellationToken = default);
    ValueTask<bool> RemoveMemberAsync(string groupId, string userId, CancellationToken cancellationToken = default);
    ValueTask<GroupMember?> FindMemberAsync(string groupId, string userId, CancellationToken cancellationToken = default);
    ValueTask<bool> TryUpdateRoleAsync(string groupId, string userId, GroupRole role, CancellationToken cancellationToken = default);

    ValueTask<IReadOnlyList<GroupMember>> GetMembersAsync(string groupId, CancellationToken cancellationToken = default);
    ValueTask<IReadOnlyList<string>> GetGroupIdsForUserAsync(string userId, CancellationToken cancellationToken = default);
    ValueTask<int> GetMemberCountAsync(string groupId, CancellationToken cancellationToken = default);
    ValueTask<int> GetGroupCountForUserAsync(string userId, CancellationToken cancellationToken = default);
}
```

**Public API (exact) — options:**

```csharp
public sealed class GroupOptions
{
    public const string ConfigurationSectionName = "LiveSharp:Chat:Groups";

    /// <summary>Maximum members in one group. Default 256.</summary>
    public int MaxMembersPerGroup { get; set; } = 256;

    /// <summary>Maximum groups one user may belong to. Default 128.</summary>
    public int MaxGroupsPerUser { get; set; } = 128;

    /// <summary>Maximum display-name length. Default 128.</summary>
    public int MaxGroupNameLength { get; set; } = 128;

    /// <summary>When true, plain members may add other members. Default false.</summary>
    public bool AllowMemberInvites { get; set; }

    /// <summary>When true, the sender's other connections receive group messages they sent. Default true.</summary>
    public bool EchoToSender { get; set; } = true;
}
```

**Requirements**

- **Group identity is `Id`, never `Name`.** Names are display strings: not unique, mutable, and
  user-supplied. Routing on a display name means a rename breaks every in-flight message and two
  groups called "Engineering" collide. `Id` comes from `IMessageIdGenerator` (UUIDv7), reusing the
  Phase 04 abstraction rather than adding a second ID generator.
- `GroupTransportName.For(id)` returns `"lsg:" + id`. The prefix namespaces LiveSharp's transport
  groups so a consumer using SignalR groups of their own for unrelated purposes cannot collide.
  Phase 02 pinned group names as ordinal and case-sensitive, so no normalisation happens here.
- `GetGroupCountForUserAsync` and `GetMemberCountAsync` exist as first-class store methods rather
  than `GetMembersAsync().Count`, because Phase 14's SQL implementation must be able to answer them
  with `COUNT(*)` instead of materialising 256 rows on every add.
- `TryUpdateRoleAsync` returns `false` for an absent member rather than throwing — role changes race
  with removals routinely.
- `InMemoryGroupStore` maintains both `groupId -> members` and `userId -> groupIds`, with the same
  `ImmutableHashSet` + `AddOrUpdate` technique and the same empty-key removal discipline as Phase 02.
  Two leak tests required.
- `DeleteGroupAsync` must remove the group **and** every membership row in both indexes. A deleted
  group leaving orphaned memberships means `GetGroupIdsForUserAsync` returns dangling IDs forever.
- Write `GroupStoreContractTests` as an abstract suite with a factory method, exactly as
  `MessageStoreContractTests` in Phase 04. Phase 14's EF Core store inherits it.

**Tests — `GroupStoreContractTests` (abstract, run against every implementation)**

| Test | Asserts |
|---|---|
| `Add_then_find_group_returns_it` | |
| `Find_unknown_group_returns_null` | |
| `Delete_returns_true_then_false` | |
| `Delete_removes_every_membership` | members and reverse index both empty |
| `Add_member_then_find_member_returns_the_role` | |
| `Add_member_twice_does_not_duplicate` | count is 1 |
| `Remove_member_returns_true_then_false` | |
| `Try_update_role_changes_the_role` | |
| `Try_update_role_returns_false_for_an_absent_member` | |
| `Get_members_returns_every_member` | |
| `Get_group_ids_for_user_returns_every_group` | |
| `Get_group_ids_for_an_unknown_user_returns_empty` | empty, not null |
| `Member_count_matches_get_members_count` | |
| `Group_count_for_user_matches_get_group_ids_count` | |
| `Returned_collections_are_snapshots` | later mutation does not change them |
| `All_methods_honour_a_pre_cancelled_token` | |
| `Concurrent_member_adds_all_land` | `ConcurrencyHarness` |

**Tests — `InMemoryGroupStoreTests`**

| Test | Asserts |
|---|---|
| `Empty_group_key_is_removed_after_the_last_member_leaves` | leak test 1 |
| `Empty_user_key_is_removed_after_the_last_group_membership_ends` | leak test 2 |
| `Deleting_a_group_clears_both_indexes_completely` | |
| `Group_transport_name_carries_the_prefix` | |
| `Group_options_defaults_are_valid` | |
| `Group_options_reject_non_positive_limits` | table-driven over each limit |
| `Group_options_reject_a_max_members_value_above_the_fan_out_sanity_limit` | actionable message pointing at Phase 15 for large groups |

---

### P05.T2 — `IGroupService`: lifecycle and membership

**Deliverables**

```
src/LiveSharp.Chat/Groups/IGroupService.cs
src/LiveSharp.Chat/Groups/GroupService.cs
src/LiveSharp.Chat/Groups/GroupAccess.cs
src/LiveSharp.Chat/Diagnostics/GroupLog.cs
tests/LiveSharp.Chat.Tests/Groups/GroupServiceTests.cs
tests/LiveSharp.Chat.Tests/Groups/GroupServiceConcurrencyTests.cs
```

**Public API (exact):**

```csharp
namespace LiveSharp.Chat;

public interface IGroupService
{
    Task<OperationResult<RealtimeGroup>> CreateAsync(string ownerId, CreateGroupRequest request, CancellationToken cancellationToken = default);
    Task<OperationResult> DeleteAsync(string actingUserId, string groupId, CancellationToken cancellationToken = default);

    Task<OperationResult<GroupMember>> AddMemberAsync(string actingUserId, string groupId, string userId, GroupRole role = GroupRole.Member, CancellationToken cancellationToken = default);
    Task<OperationResult> RemoveMemberAsync(string actingUserId, string groupId, string userId, CancellationToken cancellationToken = default);
    Task<OperationResult> ChangeRoleAsync(string actingUserId, string groupId, string userId, GroupRole role, CancellationToken cancellationToken = default);

    Task<OperationResult<IReadOnlyList<GroupMember>>> GetMembersAsync(string actingUserId, string groupId, CancellationToken cancellationToken = default);
    Task<OperationResult<IReadOnlyList<RealtimeGroup>>> GetUserGroupsAsync(string userId, CancellationToken cancellationToken = default);
}
```

**Role rules — implement exactly, and encode each as a named test**

| Action | Permitted to |
|---|---|
| Create group | any authenticated user (creator becomes `Owner`) |
| Delete group | `Owner` only |
| Add member | `Owner`, `Admin`; also `Member` when `AllowMemberInvites` |
| Remove member | `Owner`, `Admin`; **any user may remove themselves** (leaving) |
| Remove an `Admin` | `Owner` only |
| Remove the `Owner` | nobody — returns `Conflict` |
| Change role | `Owner` only |
| Promote to `Owner` | `Owner` only, and it **demotes the previous owner to `Admin`** in the same operation |
| Read members | members only |

**Requirements**

- **Ownership transfer is atomic.** `ChangeRoleAsync(x, Owner)` must promote `x` and demote the
  current owner in one logical step. A group with two owners or zero owners is unrecoverable through
  the public API. If the store cannot do this transactionally (the in-memory one can, EF can), the
  service must serialise per group. Test the two-owners and zero-owners invariants directly.
- **Leaving is self-removal, not a separate operation.** `RemoveMemberAsync(actor, group, actor)` is
  the leave path. One code path, one authorization rule, one set of side effects. Document it.
- Every mutation performs its side effects in this order: **store first, then transport, then
  broadcast the event.** A member who is added to the transport group but not the store loses
  membership on reconnect; the reverse merely delays delivery by one reconnect. Order matters and
  needs a call-order test.
- `AddMemberAsync` must enforce both `MaxMembersPerGroup` and the added user's `MaxGroupsPerUser`,
  returning `Conflict` with a message naming which limit was hit and its value.
- `AddMemberAsync` must add the new member's **live connections** to the transport group immediately
  (via `IConnectionRegistry.GetUserConnectionsAsync` + `IRealtimeTransport.AddToGroupAsync`), not
  just record the row. Otherwise a user added mid-session receives nothing until they reconnect.
- `RemoveMemberAsync` must remove all of that user's connections from the transport group.
- `DeleteAsync` must remove every member's connections from the transport group before deleting rows,
  then broadcast `group.deleted` to the members it just captured — capture the member list **before**
  deleting or the notification goes to nobody. This ordering trap deserves a code comment.
- `GetMembersAsync` returns `Forbidden` for a non-member. Even without Phase 09, membership is the
  minimum viable boundary and shipping without it would leak group rosters.
- Unknown group → `NotFound` for every operation. Do **not** distinguish "does not exist" from "you
  are not a member" in the error message for read operations — that is a group-existence oracle.
  Document the deliberate ambiguity.

**Tests — `GroupServiceTests`** (with `InMemoryTransport`, `InMemoryGroupStore`, `FakeTimeProvider`)

| Test | Asserts |
|---|---|
| `Create_makes_the_creator_the_owner` | |
| `Create_stamps_created_at_from_the_time_provider` | |
| `Create_generates_a_sortable_group_id` | |
| `Create_adds_the_owner_to_the_transport_group` | |
| `Create_rejects_an_empty_or_over_long_name` | table-driven |
| `Create_rejects_a_user_already_at_the_group_limit` | `Conflict`, message names the limit |
| `Delete_by_the_owner_succeeds` | |
| `Delete_by_an_admin_is_forbidden` | |
| `Delete_by_a_member_is_forbidden` | |
| `Delete_removes_every_member_from_the_transport_group` | |
| `Delete_broadcasts_group_deleted_to_the_former_members` | **the capture-before-delete test** |
| `Delete_of_an_unknown_group_returns_not_found` | |
| `Add_member_by_the_owner_succeeds` | |
| `Add_member_by_an_admin_succeeds` | |
| `Add_member_by_a_member_is_forbidden_by_default` | |
| `Add_member_by_a_member_succeeds_when_member_invites_are_allowed` | |
| `Add_member_adds_that_users_live_connections_to_the_transport_group` | **required** |
| `Add_member_at_the_group_member_limit_returns_conflict` | |
| `Add_member_when_the_invitee_is_at_their_group_limit_returns_conflict` | distinct message |
| `Add_member_twice_is_idempotent_and_does_not_rebroadcast` | |
| `Remove_member_by_the_owner_succeeds` | |
| `Remove_self_succeeds_for_a_plain_member` | the leave path |
| `Remove_another_member_by_a_plain_member_is_forbidden` | |
| `Remove_an_admin_by_another_admin_is_forbidden` | |
| `Remove_the_owner_returns_conflict` | last-owner protection |
| `Remove_member_removes_that_users_connections_from_the_transport_group` | |
| `Change_role_by_the_owner_succeeds` | |
| `Change_role_by_an_admin_is_forbidden` | |
| `Promote_to_owner_demotes_the_previous_owner_to_admin` | |
| `A_group_never_has_two_owners` | invariant, after a promotion |
| `A_group_never_has_zero_owners` | invariant, after every mutation path |
| `Get_members_by_a_member_succeeds` | |
| `Get_members_by_a_non_member_is_forbidden` | |
| `Get_members_of_an_unknown_group_returns_not_found_with_no_existence_oracle` | message identical to the forbidden case |
| `Mutations_write_the_store_before_the_transport` | call order |
| `All_methods_honour_a_pre_cancelled_token` | |

**Tests — `GroupServiceConcurrencyTests`**

| Test | Asserts |
|---|---|
| `Concurrent_adds_never_exceed_the_member_limit` | 32 workers, limit 10 → exactly 10 members |
| `Concurrent_promotions_to_owner_leave_exactly_one_owner` | |
| `Concurrent_add_and_remove_of_one_user_leaves_a_consistent_store_and_transport` | |
| `Concurrent_delete_and_add_member_does_not_resurrect_the_group` | |
| `Concurrent_leaves_by_every_member_leave_only_the_owner` | |

> `Concurrent_adds_never_exceed_the_member_limit` will fail on read-then-write. Use a striped
> `SemaphoreSlim` keyed by **group ID** — the same technique Phase 02 used per user ID, never a global
> lock. Document the choice next to the code.

---

### P05.T3 — Group messaging

**Deliverables**

```
src/LiveSharp.Chat/Groups/Protocol/CreateGroupRequest.cs
src/LiveSharp.Chat/Groups/Protocol/SendGroupMessageRequest.cs
src/LiveSharp.Chat/Groups/Protocol/GroupMessageReceivedPayload.cs
src/LiveSharp.Chat/Groups/Protocol/GroupMemberChangedPayload.cs
src/LiveSharp.Chat/Groups/Handlers/*.cs                 (six handlers)
src/LiveSharp.Chat/DependencyInjection/LiveSharpGroupsBuilderExtensions.cs
tests/LiveSharp.Chat.Tests/Groups/GroupMessagingTests.cs
tests/LiveSharp.Chat.Tests/Groups/Handlers/*.cs
```

Extend `RealtimeOperations` / `RealtimeEvents` and register every DTO in
`LiveSharpJsonSerializerContext`:

```csharp
public static class Group
{
    public const string Create           = "group.create";
    public const string Delete           = "group.delete";
    public const string MemberAdd         = "group.member.add";
    public const string MemberRemove      = "group.member.remove";
    public const string MemberRoleChange  = "group.member.role.change";
    public const string SendMessage       = "chat.group.message.send";
}

public static class Group   // events
{
    public const string Created        = "group.created";
    public const string Deleted        = "group.deleted";
    public const string MemberAdded    = "group.member.added";
    public const string MemberRemoved  = "group.member.removed";
    public const string RoleChanged    = "group.member.role.changed";
    public const string MessageReceived = "chat.group.message.received";
}
```

**`ChatMessage` change** — group messages need a nullable recipient:

`RecipientId` becomes `public string? RecipientId { get; init; }` and a new
`public string? GroupId { get; init; }` is added. Exactly one of the two must be set; enforce it in a
private validation method and test both violation cases. This is a **breaking change to a public
record** shipped in `0.3.0`: record it in `PublicAPI.Unshipped.txt`, add a `### Changed` changelog
entry under a `BREAKING` marker, and note it in the migration section of
`docs/chat/direct-messaging.md`. Pre-1.0 breaking changes are permitted (Spec §26) but must never be
silent.

**Requirements**

- `IChatService` gains `SendToGroupAsync(string senderId, SendGroupMessageRequest request, RealtimeOperationContext? context = null, CancellationToken ct = default)`.
- Fan-out uses `IRealtimeTransport.SendToGroupExceptAsync` with the sender's **originating
  connection** excluded when `context` is non-null — the same rule as Phase 04's direct echo, so a
  sender never sees their own message twice. When `EchoToSender` is false, exclude **all** of the
  sender's connections.
- Sender must be a member → otherwise `Forbidden`. The handler takes the sender from
  `context.Connection.UserId`; `SendGroupMessageRequest` must have **no** `SenderId` member. Assert
  its absence by reflection, exactly as Phase 04 does.
- Validation reuses `IMessageValidator`. Extend `SendMessageRequest`-style validation to the group
  request rather than writing a second validator — one content-limit implementation, one place to fix.
- `ConversationId` for a group message is `"grp:" + groupId`. Add `ConversationId.ForGroup(groupId)`
  next to `ForDirect`, with the same prefix-collision test discipline.
- Store the message via `IMessageStore` before fan-out, same ordering rule as Phase 04.
- Member-change events (`group.member.added` / `removed` / `role.changed`) go to the **whole group**
  including the affected user, and additionally to the removed user's connections *before* they are
  removed from the transport group — otherwise the person being removed never learns they were removed.
  This ordering is easy to get backwards; give it a named test.

**Tests — `GroupMessagingTests`**

| Test | Asserts |
|---|---|
| `Group_message_reaches_every_member_connection` | 3 members × 2 connections → 6 deliveries |
| `Group_message_does_not_reach_the_originating_connection` | |
| `Group_message_reaches_the_senders_other_connections` | echo on |
| `Group_message_reaches_no_sender_connection_when_echo_is_disabled` | |
| `Group_message_does_not_reach_a_non_member` | isolation |
| `Group_message_from_a_non_member_is_forbidden` | |
| `Group_message_to_an_unknown_group_returns_not_found` | |
| `Group_message_sets_the_group_conversation_id` | |
| `Group_message_leaves_recipient_id_null` | |
| `Chat_message_with_both_recipient_and_group_is_rejected` | invariant |
| `Chat_message_with_neither_recipient_nor_group_is_rejected` | invariant |
| `Group_message_is_stored_before_fan_out` | call order |
| `Group_message_validation_reuses_the_direct_message_limits` | oversized content |
| `Removed_member_receives_the_removal_event_before_losing_transport_membership` | **required** |
| `Member_added_event_reaches_the_whole_group_including_the_new_member` | |
| `Fan_out_to_a_thousand_member_group_delivers_a_thousand_times` | scale sanity, one transport call |
| `Send_group_message_request_has_no_sender_id_member` | **security test**, reflection |
| `Concurrent_group_messages_all_deliver_exactly_once_to_every_member` | `ConcurrencyHarness` |

---

### P05.T4 — Group rehydration on reconnect

> ## The trap
> SignalR group membership is **per connection**. When a connection drops and the client reconnects,
> it gets a **new connection ID** and belongs to **no transport groups**. Everything in `P05.T2` and
> `P05.T3` will pass its tests and still fail in production the first time a user's WiFi blinks: they
> stay in the group according to `IGroupStore`, receive no group messages, and see no error.
>
> This task is not optional and cannot be folded into another task.

**Deliverables**

```
src/LiveSharp.Chat/Groups/GroupRehydrationHandler.cs
tests/LiveSharp.Chat.Tests/Groups/GroupRehydrationHandlerTests.cs
tests/LiveSharp.IntegrationTests/Chat/GroupReconnectIntegrationTests.cs
```

**Implementation requirements**

`GroupRehydrationHandler : IConnectionLifecycleHandler`, registered by `AddGroups()`:

- `OnConnectedAsync`: `IGroupStore.GetGroupIdsForUserAsync(connection.UserId)`, then
  `IRealtimeTransport.AddToGroupAsync(connection.ConnectionId, GroupTransportName.For(id))` for each.
- `OnDisconnectedAsync`: **do nothing.** Phase 02 already calls
  `IGroupRegistry.RemoveConnectionFromAllAsync` during disconnect, and SignalR removes the connection
  from its own groups automatically when the connection ends. Adding cleanup here would be redundant
  work on the hot disconnect path. Put that reasoning in a comment — otherwise someone will "fix" the
  empty method.
- Cap the work: if the user belongs to more than `MaxGroupsPerUser` groups (possible if the limit was
  lowered after the fact), rehydrate up to the cap, log a `Warning` naming the user's group count, and
  continue. A connect handler that does unbounded work is a connection-storm amplifier.
- Failures must be isolated per group: one failing `AddToGroupAsync` logs a `Warning` and the rest
  still run. Phase 02 guarantees a throwing handler does not fail the connection, but a handler that
  gives up after the first failure leaves the user in a partially-rehydrated state, which is worse
  than either extreme.
- Log a single `Debug` summary (`userId` hashed or omitted, group count, elapsed).
- This runs on **every** connect, including the first. That is correct and simpler than distinguishing
  reconnects, and it makes `Add_member_adds_live_connections` in `P05.T2` a belt-and-braces
  optimisation rather than the only path.

**Tests — unit**

| Test | Asserts |
|---|---|
| `On_connected_adds_the_connection_to_every_group_the_user_belongs_to` | 3 groups → 3 transport calls |
| `On_connected_for_a_user_in_no_groups_does_nothing` | |
| `On_connected_uses_the_prefixed_transport_group_name` | |
| `On_connected_stops_at_the_configured_group_cap_and_warns` | cap + 1 |
| `A_failing_add_to_group_does_not_prevent_the_remaining_groups` | |
| `A_failing_add_to_group_is_logged_at_warning_with_the_group_id` | |
| `On_disconnected_does_nothing` | zero transport calls — pins the deliberate no-op |
| `Handler_does_not_log_the_user_id_above_debug` | |

**Tests — integration (`GroupReconnectIntegrationTests`, both transports)**

| Test | Asserts |
|---|---|
| `A_group_member_receives_messages_after_an_automatic_reconnect` | **the regression test**: drop the transport, wait for `Connected`, send from another member, assert arrival |
| `A_reconnected_member_is_in_the_signalr_group_under_its_new_connection_id` | |
| `The_old_connection_id_is_no_longer_in_the_group_registry_after_reconnect` | |
| `A_member_added_while_disconnected_receives_messages_after_reconnecting` | store-driven rehydration, not event-driven |
| `A_member_removed_while_disconnected_receives_nothing_after_reconnecting` | the inverse — proves the store is the source of truth |
| `Rehydration_happens_before_the_client_can_send_its_first_operation` | ordering sanity |

---

### P05.T5 — `AddGroups()` and DI

**Deliverables**

```
src/LiveSharp.Chat/DependencyInjection/LiveSharpGroupsBuilderExtensions.cs
tests/LiveSharp.Chat.Tests/Groups/RegistrationTests.cs
```

**Public API (exact):**

```csharp
public static class LiveSharpGroupsBuilderExtensions
{
    public static ILiveSharpBuilder AddGroups(this ILiveSharpBuilder builder);
    public static ILiveSharpBuilder AddGroups(this ILiveSharpBuilder builder, Action<GroupOptions> configure);
}
```

**Decision: `AddGroups()` is separate from `AddChat()` and requires it.**

Rationale: groups add a store, six handlers, a lifecycle handler, and a rehydration cost on every
connect. A consumer building a support-chat widget with only 1:1 conversations should not pay for any
of it, and Spec §7 is explicit that features must not force unnecessary dependencies. The cost is one
extra line in `Program.cs`, which is acceptable; the alternative — a `ChatOptions.EnableGroups` flag —
registers the services anyway and only pretends to be opt-out.

`AddGroups()` must throw `LiveSharpConfigurationException` naming `AddChat()` if `IChatService` is
not registered. Ordering-independent: check at build time via a validating hosted service or an
`IValidateOptions`, not at the moment `AddGroups()` executes, so `AddGroups().AddChat()` also works.

**Tests**

| Test | Asserts |
|---|---|
| `AddGroups_registers_the_group_service_and_store` | lifetimes asserted |
| `AddGroups_registers_the_rehydration_lifecycle_handler` | |
| `AddGroups_registers_all_six_operation_handlers` | `IOperationRouter.RegisteredOperations` |
| `AddGroups_is_idempotent` | |
| `AddGroups_without_AddChat_fails_at_startup_naming_AddChat` | |
| `AddGroups_before_AddChat_still_works` | order independence |
| `AddGroups_does_not_replace_a_custom_group_store` | `TryAdd` semantics |
| `AddChat_alone_registers_no_group_operations` | proves the opt-in is real |

---

### P05.T6 — Blazor Server example and client support

**Deliverables**

```
examples/BlazorServer/LiveSharp.Examples.BlazorServer.csproj
examples/BlazorServer/Program.cs
examples/BlazorServer/Components/…
examples/BlazorServer/README.md
clients/dotnet/LiveSharp.Client/Chat/LiveSharpGroupClientExtensions.cs
clients/js/packages/client/src/groups.ts
clients/js/packages/client/src/__tests__/groups.test.ts
```

**Blazor Server decision: use the LiveSharp server-side services directly, not `ILiveSharpClient`.**

A Blazor Server component already runs on the server inside a SignalR circuit. Opening a *second*
SignalR connection from the server back to its own hub to send a message is pure overhead and doubles
the connection count. So the Blazor Server example injects `IChatService` and `IGroupService`
directly, and subscribes to inbound messages by registering a component-scoped
`IConnectionLifecycleHandler`… which does not work either, because the component is not a connection.

Resolve it properly and document the resolution: the Blazor Server example registers a small
singleton in-process fan-out (`IChatNotificationHub` in the example, not in the library) that
components subscribe to, and LiveSharp delivers to it through a normal `IRealtimeTransport` because
the Blazor circuit *is* a LiveSharp connection — the example's `Program.cs` calls `MapLiveSharp()` and
the Blazor component uses `ILiveSharpClient` pointed at the local hub **only** for receiving.

If that proves awkward in practice, the acceptable fallback is: use `ILiveSharpClient` for both
directions with a `HttpMessageHandler` that loops back in-process, and document the trade-off
honestly. **Whichever way it lands, `examples/BlazorServer/README.md` must contain a section
"Blazor Server vs Blazor WebAssembly" explaining the difference**, because this is the single most
confusing part of Blazor + real-time integration and Phase 06 adds the WASM example that does it the
other way.

**Example requirements**

- Demonstrates: login as a demo user, create a group, add members, group chat, direct chat, leave a
  group, delete a group. Roles visible in the UI.
- Same demo-auth red banner and "never do this" warning as the MVC example.
- README: what it demonstrates, how to run, the Blazor Server vs WASM section, and a **"What this
  example does NOT do"** list with phase links.

**Client extensions** — thin typed wrappers on both clients: `CreateGroupAsync`, `DeleteGroupAsync`,
`AddMemberAsync`, `RemoveMemberAsync`, `ChangeRoleAsync`, `SendGroupMessageAsync`,
`OnGroupMessageReceived`, `OnGroupMemberChanged`. Add the constants to `protocol.ts`; the Phase 03
contract test enforces parity.

**Tests**

| Test | Asserts |
|---|---|
| `group constants match artifacts/protocol-names.json` | contract test |
| `send group message resolves with the response` | |
| `on group message received delivers and unsubscribes` | |
| `Client_group_helpers_surface_the_error_code_on_failure` | .NET, `LiveSharpClientException` |

Plus the `examples` CI job must build `examples/LiveSharp.Examples.slnx` including the new project.

---

## Dependency justification

No new packages. Groups reuse `IMessageIdGenerator`, `IMessageValidator`, `IMessageStore`,
`IGroupRegistry`, `IRealtimeTransport`, and `IConnectionLifecycleHandler`.

Blazor Server needs no package beyond the ASP.NET Core shared framework.

---

## Documentation deltas

- `docs/chat/groups.md` — **new**. Quick start, the full role permission matrix as a table, the
  member-change event fan-out rules, `GroupOptions`, the `Id`-vs-`Name` distinction and why, group
  message conversation IDs, and a **Limitations** section: no authorization layer (Phase 09), no
  receipts (Phase 08), no history (Phase 14), no discovery, single-node only (Phase 15), fan-out is
  O(members) per message.
- `docs/architecture/group-rehydration.md` — **new**. Why transport groups are per-connection, what
  breaks without rehydration, and the store-as-source-of-truth model. Short and blunt; a maintainer
  who deletes `GroupRehydrationHandler` as dead code must be able to find out why it exists.
- `docs/chat/direct-messaging.md` — add the `ChatMessage.RecipientId` nullability migration note.
- `docs/configuration/README.md` — `GroupOptions` table.
- `docs/configuration/service-lifetimes.md` — group service, store, rehydration handler.
- `docs/troubleshooting/README.md` — "group messages stop arriving after a network blip"
  (rehydration), "a removed member still receives messages" (transport vs store), "cannot delete a
  group" (owner-only), "group roster is empty" (non-member `Forbidden` looks like empty in a client
  that ignores errors).
- `README.md` — groups → `Available`.

## Example deltas

- `examples/BlazorServer` — new.
- `examples/AspNetCoreMvc` — add group chat to the existing UI.
- `examples/README.md` — index the new example.

## CHANGELOG entry

```markdown
### Added
- Group and channel messaging in `LiveSharp.Chat`: create, delete, membership, and three roles.
- `chat.group.message.send` operation and `chat.group.message.received` event.
- `IGroupStore` with a bounded in-memory implementation and a shared contract test suite.
- Automatic group rehydration on reconnect, so transport group membership survives connection loss.
- Blazor Server example application demonstrating direct and group chat.
- Group helpers for the .NET and TypeScript clients.

### Changed
- **BREAKING** `ChatMessage.RecipientId` is now nullable and a `GroupId` property was added. Exactly
  one of the two is set. Direct-message consumers are unaffected unless they relied on
  `RecipientId` being non-null.

### Security
- Group rosters are readable by members only; unknown-group and non-member responses are
  indistinguishable to avoid a group-existence oracle.
- Sender identity for group messages comes from the authenticated connection and cannot be supplied
  by the payload.
```

---

## Exit criteria

- [ ] `A_group_member_receives_messages_after_an_automatic_reconnect` passes on both transports.
- [ ] `A_group_never_has_two_owners` and `A_group_never_has_zero_owners` pass after every mutation path.
- [ ] `Concurrent_adds_never_exceed_the_member_limit` passes without a global lock.
- [ ] `Removed_member_receives_the_removal_event_before_losing_transport_membership` passes.
- [ ] `Delete_broadcasts_group_deleted_to_the_former_members` passes.
- [ ] `Send_group_message_request_has_no_sender_id_member` passes.
- [ ] `AddChat_alone_registers_no_group_operations` passes — the opt-in boundary is real.
- [ ] `GroupStoreContractTests` runs against `InMemoryGroupStore`.
- [ ] The `ChatMessage` breaking change is in `PublicAPI.Unshipped.txt`, the changelog, and the docs.
- [ ] `docs/architecture/group-rehydration.md` explains why the handler exists.
- [ ] The Blazor Server example runs, and its README has the Blazor Server vs WASM section.
- [ ] Group constants in `protocol.ts`; contract test green.
- [ ] `scripts/verify.ps1` green; `examples` CI job green; tag `v0.4.0`.

## Verify

```powershell
pwsh scripts/verify.ps1
dotnet test tests/LiveSharp.IntegrationTests -c Release --filter "FullyQualifiedName~GroupReconnect"
dotnet build examples/LiveSharp.Examples.slnx
dotnet run --project examples/BlazorServer
git tag v0.4.0
```

## Next

`docs/plan/phase-06-presence.md`, task `P06.T1`.
