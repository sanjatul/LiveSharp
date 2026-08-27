# Phase 15 — Distributed / Scale-Out

| | |
|---|---|
| **Version at exit** | `0.14.0` (tag `v0.14.0`) |
| **Depends on** | Phase 14 |
| **Branch** | `feat/phase-15-distributed` |
| **Tasks** | `P15.T1` … `P15.T6` |
| **Read before starting** | `docs/architecture/connection-lifecycle.md`, and every "node-local until Phase 15" note in the docs |

---

## Goal

Make LiveSharp work correctly on more than one server. A new optional package,
`LiveSharp.Redis`, provides the SignalR backplane plus distributed implementations of the registries and
stores that Phases 02–11 built as node-local.

Every phase so far has honestly documented "single node only". This phase pays that debt:

| Node-local today | Consequence on two nodes | Fixed by |
|---|---|---|
| `InMemoryConnectionRegistry` | node A cannot see node B's connections; `SendToUserAsync` silently drops | `P15.T2` |
| `InMemoryGroupRegistry` | group fan-out reaches only members on the sending node | `P15.T2` + backplane |
| `InMemoryPresenceStore` | a user online on node B appears offline on node A | `P15.T3` |
| Rate limit partitions | per-node budgets multiply by node count | `P15.T5` |
| `InMemorySignalingSessionStore` | offer relayed on node A cannot reach a peer on node B | `P15.T4` |
| `InMemoryCallStore` | call state invisible cross-node; accept on B not seen by A | `P15.T4` |
| `SignalRRealtimeTransport.DisconnectAsync` | cannot abort a connection on another node | `P15.T4` |
| Typing tracker | typing invisible cross-node | **deliberately not fixed** — see non-goals |

---

## The honest framing: this phase reduces guarantees

Single-node LiveSharp is linearizable: one process, one dictionary, compare-and-swap. Multi-node
LiveSharp is not, and no amount of Redis makes it so. Presence will occasionally be stale; a call accept
and a call cancel racing across nodes will resolve by whichever reaches Redis first; a node dying leaves
connections registered until a heartbeat expires.

Write this into `docs/scaling/README.md` as a **consistency model** section before writing code, and
state which operations are strongly consistent (anything guarded by a Redis Lua script or a single
atomic command), which are eventually consistent (presence, connection counts), and which are
best-effort (typing, fan-out to a node that is mid-restart). A scaling document that promises
single-node semantics on a cluster is worse than no scaling document.

---

## Non-goals — DO NOT implement in this phase

- **No distributed typing indicators.** A typing indicator that arrives 150 ms late through a backplane is
  worse than one that does not arrive; and typing is per-conversation ephemeral state whose replication
  cost is disproportionate. Typing remains node-local, and the docs must say that in a two-node
  deployment users may not see each other's typing indicators. That is an acceptable, stated limitation.
- No SFU, media relay, or media-plane scaling. Media is peer-to-peer; scaling it is coturn's job.
- No Redis Cluster support beyond what `StackExchange.Redis` provides transparently. No hash-tag
  planning, no cross-slot script guarantees. Document that multi-key Lua scripts require keys in the same
  slot and that LiveSharp's key design accounts for it.
- No alternative backplanes (Azure SignalR Service, NATS, Kafka, RabbitMQ). One backplane, done properly,
  behind interfaces a consumer can implement. Azure SignalR is worth a note in the docs because it
  changes the connection model significantly.
- No leader election, distributed locks as a general facility, or consensus. Where mutual exclusion is
  needed, use single-command atomicity or a Lua script.
- No autoscaling, orchestration, or infrastructure-as-code. Deployment guidance is Phase 18.
- No cross-region or multi-datacentre topology.

---

## Tasks

### P15.T1 — Node identity, the backplane, and the key schema

**Deliverables**

```
src/LiveSharp.Redis/LiveSharp.Redis.csproj
src/LiveSharp.Redis/PublicAPI.{Shipped,Unshipped}.txt
src/LiveSharp.Redis/RedisKeys.cs
src/LiveSharp.Redis/Configuration/LiveSharpRedisOptions.cs
src/LiveSharp.Redis/Configuration/LiveSharpRedisOptionsValidator.cs
src/LiveSharp.Abstractions/INodeIdentity.cs
src/LiveSharp.Core/NodeIdentity.cs
src/LiveSharp.Redis/NodeHeartbeatService.cs
src/LiveSharp.Redis/DeadNodeReaperService.cs
src/LiveSharp.Redis/DependencyInjection/LiveSharpRedisBuilderExtensions.cs
tests/LiveSharp.Redis.Tests/…
```

**Public API (exact):**

```csharp
namespace LiveSharp;

/// <summary>Identifies this server instance within a cluster.</summary>
public interface INodeIdentity
{
    /// <summary>Stable for the lifetime of the process. Unique across the cluster.</summary>
    string NodeId { get; }
}
```

```csharp
namespace LiveSharp.Redis;

public sealed class LiveSharpRedisOptions
{
    public const string ConfigurationSectionName = "LiveSharp:Redis";

    /// <summary>Connection string. Prefer configuration or a secret store.</summary>
    public string? ConnectionString { get; set; }

    /// <summary>Existing multiplexer, when the application already owns one.</summary>
    public IConnectionMultiplexer? ConnectionMultiplexer { get; set; }

    /// <summary>Prefix for every LiveSharp key. Default "ls".</summary>
    public string KeyPrefix { get; set; } = "ls";

    /// <summary>How often this node writes its heartbeat. Default 10 seconds.</summary>
    public TimeSpan NodeHeartbeatInterval { get; set; } = TimeSpan.FromSeconds(10);

    /// <summary>A node with no heartbeat for this long is considered dead. Default 40 seconds.</summary>
    public TimeSpan NodeTimeout { get; set; } = TimeSpan.FromSeconds(40);

    /// <summary>TTL applied to connection and presence records. Refreshed by activity. Default 90 seconds.</summary>
    public TimeSpan RecordTtl { get; set; } = TimeSpan.FromSeconds(90);

    /// <summary>Maximum records processed per dead-node reap batch. Default 500.</summary>
    public int ReapBatchSize { get; set; } = 500;

    /// <summary>When true, LiveSharp registers the SignalR Redis backplane. Default true.</summary>
    public bool ConfigureSignalRBackplane { get; set; } = true;
}
```

**Requirements — key schema, designed once**

`RedisKeys` is the single source of truth. Every key derives from it; no string interpolation elsewhere.

```
{prefix}:node:{nodeId}                          STRING, TTL NodeTimeout      node heartbeat
{prefix}:nodes                                  SET                          known node ids
{prefix}:conn:{connectionId}                    HASH, TTL RecordTtl          connection record
{prefix}:user:{userId}:conns                    SET                          connection ids for a user
{prefix}:node:{nodeId}:conns                    SET                          connection ids on a node
{prefix}:group:{groupName}:conns                SET                          connection ids in a group
{prefix}:conn:{connectionId}:groups             SET                          groups for a connection
{prefix}:presence:{userId}                      HASH, TTL RecordTtl          presence record
{prefix}:presence:sub:{subjectId}               SET                          observer user ids
{prefix}:presence:obs:{observerId}              SET                          subject user ids
{prefix}:sig:{sessionId}                        HASH, TTL SessionIdleTimeout signalling session
{prefix}:sig:user:{userId}                      SET                          session ids for a user
{prefix}:call:{callId}                          HASH                         call state
{prefix}:call:user:{userId}:active              SET                          active call ids
{prefix}:ctrl                                   PUB/SUB channel              cross-node control messages
{prefix}:rl:{partition}:{operation}             STRING, TTL window           rate limit counter
```

- **Hash tags for multi-key atomicity.** A Lua script touching `{prefix}:conn:{id}` and
  `{prefix}:user:{userId}:conns` requires both keys in the same Redis Cluster slot. Use hash tags to force
  colocation where a script needs it: `{prefix}:{{{userId}}}:conn:{connectionId}` and
  `{prefix}:{{{userId}}}:user:conns`. This makes the schema uglier and it is not optional if Redis
  Cluster is to be supported. Decide per key group and document which groups are colocated and why.
  Where colocation is impossible (a group spanning many users), the operation must not be a single
  script — accept eventual consistency and say so.
- Every key must appear in a table in `docs/scaling/redis-keys.md` with its type, TTL, writer, and reader.
  An operator debugging a cluster needs this, and it forces the design to be deliberate.
- `NodeIdentity` default: `$"{Environment.MachineName}:{Guid.CreateVersion7():n}"` — machine name for
  human debuggability, GUID for uniqueness across restarts and containers on one host. Overridable, because
  a consumer may want the pod name. **Not** just the machine name: two pods on one node would collide, and
  a restarted pod inheriting its own stale records is the worst failure mode here.
- `RealtimeConnection.NodeId` (reserved since Phase 01) is finally populated. `LiveSharpHub` sets it from
  `INodeIdentity`.
- `NodeHeartbeatService : BackgroundService` writes `{prefix}:node:{nodeId}` with TTL `NodeTimeout` every
  `NodeHeartbeatInterval` and adds the node to `{prefix}:nodes`.
- `DeadNodeReaperService : BackgroundService`: for each member of `{prefix}:nodes` whose heartbeat key is
  gone, drain `{prefix}:node:{nodeId}:conns` in `ReapBatchSize` batches, removing each connection from the
  user, group, and connection keys, and invoking the local `IConnectionLifecycleHandler` chain with
  `DisconnectReason.TransportError` so presence clears. Then remove the node from `{prefix}:nodes`.
  - **Only one node should reap a given dead node**, or every survivor duplicates the work and fires
    duplicate presence broadcasts. Guard with `SET {prefix}:reap:{deadNodeId} {myNodeId} NX EX 60`.
  - Reaping must be resumable: a reaper that dies mid-drain leaves the lock to expire and another node
    finishes. Idempotent removals make this safe.
- Validator rules: `NodeTimeout > 3 × NodeHeartbeatInterval` (one missed heartbeat must not kill a node —
  actionable message); `RecordTtl > NodeTimeout`; connection string or multiplexer required, not both;
  `KeyPrefix` non-empty and free of `:` at the end.

**Tests** (Testcontainers Redis)

| Test | Asserts |
|---|---|
| `Node_identity_is_stable_for_the_process_and_unique_across_instances` | |
| `Node_identity_includes_the_machine_name_for_debuggability` | |
| `Heartbeat_writes_the_node_key_with_a_ttl` | |
| `Heartbeat_key_expires_when_the_service_stops` | real short TTL |
| `Node_is_added_to_the_nodes_set` | |
| `Dead_node_connections_are_reaped` | write records for a fake node, expire its heartbeat |
| `Dead_node_reaping_invokes_local_lifecycle_handlers` | presence clears |
| `Only_one_node_reaps_a_given_dead_node` | **two reapers, one winner** |
| `Reaping_is_resumable_after_the_reaper_dies_mid_batch` | |
| `Reaping_honours_the_batch_size` | |
| `Dead_node_is_removed_from_the_nodes_set_after_reaping` | |
| `Reaper_survives_a_redis_outage_and_resumes` | kill the container, restart it |
| `Node_timeout_not_greater_than_three_heartbeats_is_rejected_with_guidance` | |
| `Both_connection_string_and_multiplexer_configured_is_rejected` | |
| `Every_key_used_by_the_package_appears_in_RedisKeys` | reflection/source scan — no ad-hoc keys |

---

### P15.T2 — Distributed connection and group registries

**Deliverables**

```
src/LiveSharp.Redis/Registries/RedisConnectionRegistry.cs
src/LiveSharp.Redis/Registries/RedisGroupRegistry.cs
src/LiveSharp.Redis/Scripts/*.lua
tests/LiveSharp.Redis.Tests/Registries/…
```

**Requirements**

- Both implement the Phase 01 interfaces and inherit the Phase 02 contract tests — which must first be
  extracted into `tests/LiveSharp.StoreContracts` alongside the Phase 14 suites. Same mechanical
  refactor, same payoff: correctness by running existing tests.
- `AddAsync` and `RemoveAsync` must be **atomic across the three keys** (connection hash, user set, node
  set). Use a Lua script with hash-tag-colocated keys. A partial write leaves a connection in the user
  index that does not exist, which is the exact failure the Phase 02 index-ordering rule was designed to
  avoid — and on Redis the ordering trick alone is not enough, because there is no single-threaded reader.
- `RemoveAsync` must return `true` exactly once for a given connection even under concurrent removal from
  two nodes. The Lua script's `DEL` return value gives this for free; a check-then-delete does not.
- `TouchAsync` refreshes the connection hash TTL and the record's `LastSeenAt`. This is the hottest
  distributed operation in the system — one Redis round trip per inbound operation. Mitigate:
  - only refresh when more than `RecordTtl / 3` has elapsed since the last local refresh, tracked in a
    node-local dictionary. This turns per-message writes into per-30-second writes. Document it, and
    document the consequence: `LastSeenAt` may be up to `RecordTtl / 3` stale.
- `GetUserConnectionsAsync` reads the user set then pipelines hash reads. Stale set members whose hash has
  expired are filtered out **and** removed opportunistically (a fire-and-forget `SREM`). Self-healing
  beats a background sweep for this case.
- `GetConnectionCountAsync` returns a **cluster-wide** count. `SCARD` across per-node sets is O(nodes)
  round trips; instead maintain a single `{prefix}:conn:count` counter incremented and decremented inside
  the add/remove scripts. Counters drift when a node dies; the dead-node reaper decrements as it drains,
  and a periodic reconciliation is out of scope — document that the count is approximate and must not be
  used for billing or admission decisions. If an exact count is needed, `SCARD` the node sets explicitly
  via a separate diagnostic API in Phase 17.
- `RedisGroupRegistry`: group sets span users, so hash-tag colocation is impossible. Accept two commands
  and eventual consistency between the forward and reverse index; reconcile opportunistically on read,
  exactly as with connections. Document it in the consistency model.
- **Never use `KEYS`.** Use `SCAN` with a cursor for any enumeration, bounded per call. A `KEYS` in a
  library is a production outage waiting for a large database. Add a test that scans the source for
  `KEYS` usage.
- Every Lua script lives in a `.lua` file, embedded as a resource, loaded with `SCRIPT LOAD` and invoked by
  SHA with a fallback to `EVAL` on `NOSCRIPT`. Inline script strings are unreviewable and cannot be
  tested independently.

**Tests**

| Test | Asserts |
|---|---|
| `Connection_registry_contract_suite_passes_against_redis` | inherited Phase 02 suite |
| `Group_registry_contract_suite_passes_against_redis` | inherited |
| `Add_is_atomic_across_all_three_keys` | interrupt via a script error, assert none written |
| `Concurrent_removes_from_two_clients_return_true_exactly_once` | |
| `Touch_does_not_write_to_redis_within_the_coalescing_window` | **the hot-path test**: assert command count |
| `Touch_writes_after_the_coalescing_window` | |
| `Get_user_connections_filters_expired_records` | |
| `Get_user_connections_opportunistically_removes_expired_set_members` | self-healing |
| `Connection_count_is_decremented_by_the_dead_node_reaper` | |
| `No_source_file_issues_a_KEYS_command` | source scan |
| `Every_lua_script_is_an_embedded_resource_and_loads_by_sha` | |
| `Scripts_recover_from_a_NOSCRIPT_error` | flush the script cache mid-test |
| `Registries_survive_a_redis_restart_and_resume` | |
| `A_redis_timeout_surfaces_as_LiveSharpTransportException_not_a_redis_exception` | abstraction integrity |

---

### P15.T3 — Distributed presence

**Deliverables**

```
src/LiveSharp.Redis/Presence/RedisPresenceStore.cs
src/LiveSharp.Redis/Presence/RedisPresenceSubscriptionStore.cs
src/LiveSharp.Redis/Presence/RedisPresenceBroadcaster.cs
tests/LiveSharp.Redis.Tests/Presence/…
```

**Requirements**

- `RedisPresenceStore` implements `IPresenceStore` including `TrySetAsync`'s optimistic-concurrency
  contract. Map `expectedUpdatedAt` onto a Lua compare-and-swap on the hash's `updatedAt` field. Redis
  gives real atomicity here, so distributed presence is actually *stronger* than the eventual-consistency
  caveat elsewhere — say so, it is a genuine win.
- `GetStaleAsync` cannot `SCAN` the whole keyspace on every sweep. Maintain a sorted set
  `{prefix}:presence:lastseen` scored by Unix seconds, and query with `ZRANGEBYSCORE ... LIMIT`. The
  sorted set must be updated in the same script as the presence hash, or the sweep works from stale data.
- **Presence aggregation across nodes.** Phase 06's `PresenceService.GetAsync` reconciles against the
  *local* `IConnectionRegistry`. With `RedisConnectionRegistry` that reconciliation becomes cluster-wide
  automatically — which is exactly why Phase 06 put the reconciliation behind the interface rather than
  reading a local dictionary. Verify it works with no changes to `PresenceService`, and if it does not,
  fix `PresenceService` rather than special-casing Redis.
- **Broadcast fan-out across nodes** is the interesting problem. `IRealtimeTransport.SendToUsersAsync`
  goes through the SignalR backplane, which handles cross-node delivery. So `PresenceBroadcaster` needs
  **no** changes — the backplane does the work. Confirm this with a two-node integration test rather than
  assuming it. If the backplane's `Clients.Users(...)` fan-out proves inefficient at scale, note it for
  Phase 16 measurement; do not optimise on suspicion.
- `RedisPresenceSubscriptionStore`: two sets per relationship, no colocation possible, eventual
  consistency between them, opportunistic reconciliation on read. Enforce `MaxSubscriptionsPerUser` with
  a Lua script that checks `SCARD` and adds atomically — a check-then-add across nodes will exceed the
  limit.
- The Phase 06 change coalescer stays **node-local**. Coalescing across nodes would need a distributed
  timer and would add latency to the common case. Consequence: with N nodes, a flapping user can produce
  up to N broadcasts per window instead of one. Document it; it is acceptable and the alternative is
  worse.

**Tests**

| Test | Asserts |
|---|---|
| `Presence_store_contract_suite_passes_against_redis` | inherited Phase 06 suite |
| `Try_set_compare_and_swap_is_atomic_across_clients` | 32 concurrent writers, one winner per token |
| `Get_stale_uses_the_sorted_set_and_honours_the_limit` | |
| `Last_seen_sorted_set_is_updated_in_the_same_script_as_the_hash` | interrupt test |
| `Presence_service_reconciles_against_the_distributed_connection_count_unchanged` | **no `PresenceService` change needed** |
| `Subscription_limit_is_enforced_atomically_across_clients` | |
| `Subscription_indexes_reconcile_opportunistically_on_read` | |
| `Presence_ttl_expiry_reports_offline` | |
| `Presence_survives_a_redis_restart` | records persist or expire cleanly, never corrupt |

---

### P15.T4 — Distributed signalling, calls, and cross-node control

**Deliverables**

```
src/LiveSharp.Redis/Signaling/RedisSignalingSessionStore.cs
src/LiveSharp.Redis/Calls/RedisCallStore.cs
src/LiveSharp.Redis/Control/IControlChannel.cs
src/LiveSharp.Redis/Control/RedisControlChannel.cs
src/LiveSharp.Redis/Control/ControlMessage.cs
src/LiveSharp.AspNetCore/Transport/SignalRRealtimeTransport.cs   (modified)
tests/LiveSharp.Redis.Tests/…
```

**Requirements — the control channel**

Phase 03 left one operation unimplementable across nodes: `DisconnectAsync` for a connection on another
node. `IHubContext` cannot abort a remote connection, and the SignalR backplane does not expose abort.

```csharp
namespace LiveSharp.Redis;

/// <summary>Cross-node control messages that the SignalR backplane cannot express.</summary>
public interface IControlChannel
{
    ValueTask PublishAsync(ControlMessage message, CancellationToken cancellationToken = default);
    ValueTask SubscribeAsync(CancellationToken cancellationToken = default);
}

public sealed record ControlMessage(
    string Type,
    string TargetNodeId,
    string OriginNodeId,
    IReadOnlyDictionary<string, string> Data);
```

- Redis pub/sub on `{prefix}:ctrl`. Each node subscribes and ignores messages whose `TargetNodeId` is
  neither its own nor the broadcast sentinel `"*"`.
- Message types needed now: `disconnect` (abort a connection), `ring` (start ringing a connection on
  another node — see below), `stop-ring`. Keep the set minimal; every message type is a new distributed
  interaction to reason about.
- `SignalRRealtimeTransport.DisconnectAsync` becomes: look up the connection's `NodeId`; if it is ours,
  abort locally as today; otherwise publish a `disconnect` control message. If the connection is unknown,
  log at `Debug` and succeed — the Phase 03 best-effort contract is unchanged, which is why Phase 03 wrote
  it that way.
- Pub/sub is **at-most-once**. A control message lost during a Redis failover is lost. That is acceptable
  for `disconnect` (the connection's TTL and the token-expiry monitor are the backstops) and for `ring`
  (the ring timeout is the backstop). Both backstops already exist because earlier phases built them.
  State this explicitly — it is the clearest example of why those timeouts were not optional.

**Requirements — calls**

- `RedisCallStore` implements `ICallStore` with the `TryUpdateAsync` compare-and-swap on `updatedAt` via
  Lua, and `GetStuckAsync` via a sorted set keyed by state and update time.
- **The accept race becomes distributed.** Phase 11's `CallAcceptCoordinator` used a node-local striped
  semaphore. On two nodes, Bob's phone (node A) and laptop (node B) can both accept. The semaphore is
  useless across nodes; the correctness must come from `TryUpdateAsync`'s compare-and-swap, which it
  already does — the semaphore was an optimisation to avoid retry churn, not the correctness mechanism.
  **Verify this claim with a distributed test.** If Phase 11's implementation relied on the semaphore for
  correctness rather than the CAS, fix Phase 11's implementation now and add the test there too.
- **Multi-device ringing across nodes.** `InviteAsync` on node A must ring Bob's connections on node B.
  `IRealtimeTransport.SendToUserAsync` through the backplane handles the `call.incoming` message, so
  ringing works with no changes. The `ring` control message type is therefore **not needed** — remove it
  from the control channel. Verify before removing. (Keeping an unused message type is worse than
  discovering it is unnecessary.)
- Call sweeps run on **every** node, which means two nodes could both time out the same call. The
  `TryUpdateAsync` CAS makes the second one a no-op, so it is correct but wasteful. Guard with the same
  `SET NX EX` pattern as the dead-node reaper, keyed per sweep type, so one node sweeps per interval.

**Requirements — signalling**

- `RedisSignalingSessionStore` with hash + TTL, and `{prefix}:sig:user:{userId}` for the per-user cap
  enforced by Lua.
- Offer/answer/candidate relay uses `IRealtimeTransport.SendToUserAsync`, so the backplane handles
  cross-node delivery. No changes to `SignalingService`. Verify with a two-node test that a full
  offer/answer/candidate exchange works with the peers on different nodes — this is the single most
  important integration test in the phase, because a call between two users on different nodes is the
  normal case in any real deployment.
- Candidate buffering (Phase 10) stays node-local. A candidate buffered on node A for a peer who connects
  to node B is lost. Backstop: the peer's `RTCPeerConnection` will still succeed via later candidates in
  most cases, and the connecting timeout catches genuine failure. Document the limitation; fixing it means
  putting SDP-adjacent data in Redis, which is a privacy and size decision that needs its own ADR.

**Tests**

| Test | Asserts |
|---|---|
| `Control_channel_delivers_to_the_targeted_node_only` | |
| `Control_channel_broadcast_reaches_every_node` | |
| `Control_channel_ignores_messages_for_other_nodes` | |
| `Disconnect_of_a_local_connection_does_not_publish_a_control_message` | no unnecessary chatter |
| `Disconnect_of_a_remote_connection_publishes_a_control_message` | |
| `Remote_disconnect_control_message_aborts_the_connection_on_the_owning_node` | two-node |
| `Disconnect_of_an_unknown_connection_still_succeeds` | Phase 03 contract preserved |
| `Call_store_contract_suite_passes_against_redis` | inherited Phase 11 suite |
| `Concurrent_accepts_from_two_nodes_produce_exactly_one_winner` | **the distributed accept race** |
| `Call_state_compare_and_swap_prevents_cross_node_regression` | terminal states hold |
| `Only_one_node_runs_each_call_sweep_per_interval` | |
| `Two_nodes_timing_out_the_same_call_result_in_one_transition` | CAS correctness |
| `Signaling_store_contract_suite_passes_against_redis` | inherited Phase 10 suite |
| `Per_user_session_cap_is_enforced_atomically_across_nodes` | |
| `Full_offer_answer_candidate_exchange_works_across_two_nodes` | **the critical test** |
| `A_call_between_users_on_different_nodes_completes_the_full_lifecycle` | **the critical test** |
| `Buffered_candidates_are_lost_when_the_peer_connects_to_another_node` | pins the documented limitation |

---

### P15.T5 — Distributed rate limiting and the two-node test harness

**Deliverables**

```
src/LiveSharp.Redis/RateLimiting/RedisRateLimiter.cs
src/LiveSharp.Redis/Scripts/rate-limit.lua
tests/LiveSharp.IntegrationTests/Distributed/TwoNodeTestApplication.cs
tests/LiveSharp.IntegrationTests/Distributed/*.cs
```

**Requirements — rate limiting**

Phase 09's rate limiter is per-node, so N nodes permit N× the configured rate. Fix it with a Redis
sliding-window counter:

- Lua script implementing a fixed-window or sliding-window counter: `INCR` + `EXPIRE` on first increment,
  compare against the limit, return the remaining permits and the reset time. Fixed window is one command
  and has a 2× burst at boundaries; sliding window via a sorted set is exact but costs O(log n) and more
  memory. **Choose fixed window** with a documented 2× boundary burst, because rate limiting here is
  abuse prevention, not billing, and a Redis round trip per operation is already the dominant cost.
  Record the trade-off in the docs.
- **One Redis round trip per rate-limited operation** is a real latency addition on the hot path. Mitigate
  with a hybrid: check the node-local limiter first (Phase 09's), and only consult Redis when the local
  check passes. A local rejection needs no network call, and the local limit is set to
  `configuredLimit / expectedNodes` as a fast-path filter — no, that under-limits when nodes are uneven.
  Simpler and correct: local limiter set to the **full** limit as a cheap pre-filter (it can only be more
  permissive than the truth in aggregate, and it rejects the obvious floods for free), then Redis for the
  authoritative decision. Document both layers.
- `LiveSharpRedisOptions.DistributedRateLimiting` (default `true` when Redis is registered) so a consumer
  can opt out and accept per-node limits in exchange for latency.
- Fail-open or fail-closed on a Redis outage? **Fail open**, log at `Warning`, and fall back to the local
  limiter. A Redis blip must not make the application unusable; the local limiter still provides
  per-node protection. Make it configurable (`FailClosedOnRateLimiterError`, default `false`) because a
  security-sensitive deployment may prefer the opposite, and document which they are choosing.

**Requirements — the two-node harness**

This is the deliverable that makes the whole phase verifiable, and it must be built properly:

- `TwoNodeTestApplication`: two independent `WebApplicationFactory` instances with **different**
  `INodeIdentity` values, sharing one Testcontainers Redis and one Testcontainers PostgreSQL (from
  Phase 14). Exposes `NodeA`, `NodeB`, a client factory that can target either, and `IAsyncDisposable`
  teardown that asserts no leaked connections in Redis.
- A `KillNodeAsync(node)` helper that disposes a factory without graceful shutdown, so dead-node reaping
  can be tested for real rather than by expiring a key by hand.
- Deterministic waiting: `WaitForClusterStateAsync` polling Redis, never `Task.Delay`.
- Every distributed test runs against **both** transports, because backplane behaviour differs subtly
  between WebSockets and long polling.

**Distributed integration tests** — the acceptance criteria for the entire phase:

| Test | Asserts |
|---|---|
| `A_message_from_a_user_on_node_a_reaches_a_user_on_node_b` | the fundamental case |
| `A_users_two_devices_on_different_nodes_both_receive_their_messages` | |
| `A_group_message_reaches_members_on_both_nodes` | |
| `A_member_added_on_node_a_receives_group_messages_sent_from_node_b` | |
| `Group_rehydration_on_reconnect_to_a_different_node_restores_membership` | **Phase 05 across nodes** |
| `Presence_of_a_user_on_node_b_is_visible_to_a_subscriber_on_node_a` | |
| `Presence_goes_offline_when_the_users_only_node_is_killed` | dead-node reaping, end to end |
| `A_signalling_exchange_between_peers_on_different_nodes_completes` | |
| `A_call_between_users_on_different_nodes_completes_the_full_lifecycle` | |
| `Accepting_a_call_on_a_device_connected_to_another_node_stops_ringing_everywhere` | |
| `A_remote_disconnect_terminates_the_connection_on_the_owning_node` | |
| `Rate_limits_are_enforced_across_both_nodes_not_doubled` | **the distributed limit test** |
| `Rate_limiting_falls_back_to_local_limits_when_redis_is_unavailable` | fail-open |
| `Killing_a_node_mid_conversation_does_not_lose_messages_sent_after_failover` | client reconnects to the survivor |
| `Killing_a_node_reaps_its_connections_within_the_node_timeout` | |
| `A_redis_outage_and_recovery_leaves_a_consistent_cluster_state` | **the partition test** |
| `Message_history_written_on_node_a_is_readable_on_node_b` | Phase 14 + Phase 15 together |
| `Typing_indicators_do_not_cross_nodes` | **pins the documented non-goal** |

> `A_redis_outage_and_recovery_leaves_a_consistent_cluster_state` is the test that will find real bugs.
> Stop the Redis container mid-traffic, restart it, and assert that connection, group, and presence state
> converge to reality within one `NodeTimeout` — with no phantom connections, no users stuck online, and
> no orphaned group memberships. Expect this to fail the first several times.

---

### P15.T6 — Documentation and deployment guidance

**Deliverables**

```
docs/scaling/README.md
docs/scaling/consistency-model.md
docs/scaling/redis-keys.md
docs/scaling/sticky-sessions.md
docs/scaling/operations.md
examples/README.md                                              (scale-out note)
```

**Requirements**

`docs/scaling/consistency-model.md` — written **first**, as stated at the top of this file. A table of
every distributed operation with its guarantee: strongly consistent (Lua-guarded: connection add/remove,
presence CAS, call state CAS, session cap, subscription cap), eventually consistent (group indexes,
presence subscription indexes, connection count), best-effort (control-channel messages, cross-node
fan-out during a node restart), and node-local (typing, candidate buffering, presence coalescing).

`docs/scaling/sticky-sessions.md` — the question every operator asks. The honest answer:
- SignalR **requires** sticky sessions when using long polling or server-sent events, because the
  transport makes multiple HTTP requests that must reach the same server.
- WebSockets do not require stickiness after the handshake, but the `/negotiate` request and the connect
  request must reach the same server, so stickiness is still required unless `SkipNegotiation` is used
  with WebSockets only.
- Therefore: **configure sticky sessions**. Provide concrete configuration for nginx (`ip_hash` or
  `sticky cookie`), Azure Application Gateway, AWS ALB (target group stickiness), and Kubernetes Ingress
  (`nginx.ingress.kubernetes.io/affinity: cookie`). Include the exact annotations; an operator should be
  able to copy them.
- Explain what breaks without stickiness: intermittent connection failures on negotiate, which look like
  random network errors and are the single most common SignalR scaling support issue.

`docs/scaling/operations.md` — Redis sizing and memory estimation per connection (measure it in Phase 16
and put real numbers here; until then, state that numbers are pending rather than guessing), Redis
persistence recommendation (AOF vs RDB — LiveSharp's data is mostly ephemeral, so persistence is optional
and `maxmemory-policy noeviction` is **required** because evicting a connection key silently breaks
routing), monitoring (which keys to watch, what a growing `{prefix}:nodes` set means), failover behaviour,
rolling-deployment guidance (drain connections, `NodeTimeout` implications), and Redis Cluster notes with
the hash-tag explanation.

`docs/scaling/README.md` — the entry point: when you need scale-out, the two-line
`AddLiveSharp().AddRedis(...)` registration, what changes and what does not, a link to each of the above,
and an honest **Limitations** section: no typing across nodes, candidate buffering is node-local,
connection counts are approximate, rate limiting has a 2× window-boundary burst, no alternative
backplanes, Azure SignalR Service is not supported (with a note on why it differs).

---

## Dependency justification

| Package | Why | Alternative? | Licence | Notes |
|---|---|---|---|---|
| `Microsoft.AspNetCore.SignalR.StackExchangeRedis` | The SignalR backplane. Cross-node message delivery for the transport is Microsoft's problem, not ours. | Writing a backplane (absurd), Azure SignalR (a different connection model, not a drop-in). | MIT | Only in `LiveSharp.Redis`. |
| `StackExchange.Redis` | Redis client for registries, stores, control channel, and rate limiting. Transitively required by the backplane anyway, so this adds nothing. | No serious alternative in .NET. | MIT | Only in `LiveSharp.Redis`. |
| `Testcontainers.Redis` | Already added in Phase 14's justification set. | — | MIT | Test-only. |

Explicitly rejected: `RedLock.net` (LiveSharp needs no general distributed lock — `SET NX EX` covers the
two places mutual exclusion is wanted, and a real lock service would imply guarantees we do not need),
`MassTransit`/`NServiceBus` (a message bus for a pub/sub channel with three message types is
disproportionate), any Redis abstraction layer over `StackExchange.Redis`.

---

## Documentation deltas

- `docs/scaling/README.md`, `consistency-model.md`, `redis-keys.md`, `sticky-sessions.md`,
  `operations.md` — all **new**, as specified.
- `docs/security/rate-limiting.md` — replace the "per-node only" limitation with the distributed
  implementation, the window-boundary burst, and the fail-open policy.
- `docs/presence/README.md` — remove the "single node" limitation; document that presence CAS is strongly
  consistent and coalescing is node-local.
- `docs/chat/typing.md` — **strengthen** the limitation: typing is node-local by design, and in a
  multi-node deployment users may not see each other typing.
- `docs/webrtc/signaling.md` — candidate buffering is node-local; sessions and relay are cluster-wide.
- `docs/calls/README.md` — calls work across nodes; the accept race is resolved by compare-and-swap.
- `docs/architecture/connection-lifecycle.md` — node identity, the dead-node reaper, and what happens to
  a connection when its node dies.
- `docs/configuration/README.md` — `LiveSharpRedisOptions`.
- `docs/troubleshooting/README.md` — "intermittent connection failures behind a load balancer" (sticky
  sessions — link prominently), "users appear online after a server crash" (`NodeTimeout`), "messages not
  delivered across servers" (backplane not registered), "rate limits are too permissive" (distributed
  limiting disabled), "Redis memory grows" (`noeviction` and TTLs), "typing indicators do not work with
  two servers" (by design), "`NOSCRIPT` errors" (script cache after failover).
- `README.md` — scale-out → `Available` with the Redis requirement and the limitations summary.

## Example deltas

No new example applications — a two-node example is a deployment topology, not a code sample, and running
one requires Docker Compose that would obscure the library usage. Instead: `examples/README.md` gains a
"Running behind a load balancer" section with a Docker Compose file showing two app instances, Redis,
and nginx with sticky sessions. That is the artifact an operator actually wants.

## CHANGELOG entry

```markdown
### Added
- `LiveSharp.Redis`: multi-server support via the SignalR Redis backplane plus distributed
  implementations of the connection registry, group registry, presence store, signalling session store,
  and call store.
- Node identity, node heartbeats, and dead-node reaping so connections belonging to a crashed server are
  cleaned up and presence converges.
- A cross-node control channel enabling operations the SignalR backplane cannot express, such as
  disconnecting a connection owned by another server.
- Distributed rate limiting, replacing per-node limits.
- A two-node integration test harness with real Redis and PostgreSQL containers, covering messaging,
  groups, presence, signalling, calling, failover, and Redis outage recovery.
- Scaling documentation including a consistency model, the complete Redis key schema, sticky-session
  configuration for common load balancers, and operational guidance.

### Notes
- Typing indicators remain node-local by design and are not visible across servers.
- WebRTC candidate buffering remains node-local; a candidate buffered for a peer that connects to another
  server is dropped.
- Cluster-wide connection counts are approximate.
- Sticky sessions are required. See `docs/scaling/sticky-sessions.md`.
```

---

## Exit criteria

- [ ] `docs/scaling/consistency-model.md` exists and classifies every distributed operation.
- [ ] Every Phase 02, 06, 10, 11 store/registry contract suite passes against Redis.
- [ ] `A_message_from_a_user_on_node_a_reaches_a_user_on_node_b` passes on both transports.
- [ ] `A_call_between_users_on_different_nodes_completes_the_full_lifecycle` passes.
- [ ] `Full_offer_answer_candidate_exchange_works_across_two_nodes` passes.
- [ ] `Concurrent_accepts_from_two_nodes_produce_exactly_one_winner` passes — and if Phase 11 relied on
      the local semaphore for correctness, Phase 11 is fixed and its test added there too.
- [ ] `Group_rehydration_on_reconnect_to_a_different_node_restores_membership` passes.
- [ ] `Killing_a_node_reaps_its_connections_within_the_node_timeout` passes.
- [ ] `Only_one_node_reaps_a_given_dead_node` passes.
- [ ] `A_redis_outage_and_recovery_leaves_a_consistent_cluster_state` passes.
- [ ] `Rate_limits_are_enforced_across_both_nodes_not_doubled` passes.
- [ ] `Touch_does_not_write_to_redis_within_the_coalescing_window` passes — the hot path is not one round
      trip per message.
- [ ] `No_source_file_issues_a_KEYS_command` passes.
- [ ] `Every_key_used_by_the_package_appears_in_RedisKeys` passes, and `docs/scaling/redis-keys.md`
      matches.
- [ ] `Typing_indicators_do_not_cross_nodes` passes — the non-goal is pinned, not accidentally fixed.
- [ ] `A_redis_timeout_surfaces_as_LiveSharpTransportException` passes — Redis does not leak through the
      abstraction.
- [ ] Every unused control-message type has been removed after verification.
- [ ] `docs/scaling/sticky-sessions.md` contains copy-pasteable configuration for nginx, Azure, AWS, and
      Kubernetes Ingress.
- [ ] `examples/README.md` has a working Docker Compose two-node topology.
- [ ] `scripts/verify.ps1` green including the container suites in CI; tag `v0.14.0`.

## Verify

```powershell
pwsh scripts/verify.ps1
dotnet test tests/LiveSharp.Redis.Tests -c Release
dotnet test tests/LiveSharp.IntegrationTests -c Release --filter "FullyQualifiedName~Distributed"
dotnet test tests/LiveSharp.ArchitectureTests -c Release
git tag v0.14.0
```

## Next

`docs/plan/phase-16-performance.md`, task `P16.T1`.
