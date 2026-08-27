# Phase 16 — Performance

| | |
|---|---|
| **Version at exit** | `0.15.0` (tag `v0.15.0`) |
| **Depends on** | Phase 15 |
| **Branch** | `feat/phase-16-performance` |
| **Tasks** | `P16.T1` … `P16.T6` |
| **Read before starting** | `EXECUTION_PLAN.md` §5.2, and Spec §50 ("Measure first") |

---

## Goal

Measure, then optimise. Establish a benchmark suite, add observability, find the actual hot paths, fix
what the numbers say is slow, and publish honest figures.

**The order is not negotiable.** `P16.T1` and `P16.T2` produce measurements. `P16.T4` and `P16.T5`
change code. A session that starts optimising before the benchmark suite exists has failed the phase
regardless of how much faster the code gets, because there is no way to know whether it got faster.

---

## Non-goals — DO NOT implement in this phase

- No optimisation without a benchmark showing the code path matters. Every change in `P16.T4`/`P16.T5`
  must cite a specific benchmark result in its commit message.
- No new features. Zero. `MessagePack` is a serialization option, not a feature.
- No architectural rewrites. If a benchmark reveals a structural problem, write an ADR draft and stop
  (`EXECUTION_PLAN.md` §8) — do not restructure the library in a performance phase.
- No unsafe code, `Span` gymnastics, or manual memory management without a benchmark showing a specific
  allocation is a problem and a test showing correctness is preserved.
- No AOT/trimming work. That is a Phase 17/18 concern and depends on the serializer context already being
  source-generated.
- No load-testing infrastructure as a product. A harness sufficient to produce numbers, not a
  reusable load-testing framework.
- No performance claims in marketing terms. Numbers with methodology, or nothing.

---

## Tasks

### P16.T1 — Benchmark project

**Deliverables**

```
benchmarks/LiveSharp.Benchmarks/LiveSharp.Benchmarks.csproj
benchmarks/Directory.Build.props
benchmarks/LiveSharp.Benchmarks/Program.cs
benchmarks/LiveSharp.Benchmarks/Serialization/*.cs
benchmarks/LiveSharp.Benchmarks/Routing/*.cs
benchmarks/LiveSharp.Benchmarks/Registries/*.cs
benchmarks/LiveSharp.Benchmarks/FanOut/*.cs
benchmarks/LiveSharp.Benchmarks/StateMachines/*.cs
benchmarks/README.md
scripts/benchmark.ps1
```

**Requirements**

- BenchmarkDotNet, `Release` only, `[MemoryDiagnoser]` on every benchmark class. Allocation is the metric
  that matters most for a real-time server — a per-message allocation becomes gen0 pressure becomes GC
  pauses becomes latency spikes.
- `benchmarks/Directory.Build.props`: `IsPackable=false`, `TreatWarningsAsErrors=false`,
  `Optimize=true`, `ServerGarbageCollector=true`, `ConcurrentGarbageCollector=true` — match the
  configuration a real server runs under, or the numbers are fiction.
- `benchmarks/**` is **not** in `LiveSharp.slnx` (it would slow every build) but has its own
  `benchmarks/LiveSharp.Benchmarks.slnx`. Add a CI job that builds it — a benchmark project that does not
  compile is discovered at the worst moment.
- Benchmarks must be **microbenchmarks of library code**, not end-to-end network tests. BenchmarkDotNet
  cannot meaningfully measure a WebSocket round trip; that is `P16.T2`'s job. Benchmark:

| Class | Measures |
|---|---|
| `EnvelopeSerializationBenchmarks` | serialize/deserialize `RealtimeEnvelope`, `OperationResponse`, `ChatMessage`, at small (100 B), typical (1 KB), and large (16 KB) payloads |
| `OperationRoutingBenchmarks` | `OperationRouter.RouteAsync` with 1, 10, and 100 registered operations; with 0 and 5 filters |
| `ConnectionRegistryBenchmarks` | `InMemoryConnectionRegistry` add / find / `GetUserConnectionsAsync` / `TouchAsync` at 1 K, 10 K, 100 K connections |
| `GroupRegistryBenchmarks` | `GetConnectionsInGroupAsync` at 10, 100, 1 000, 10 000 members |
| `FanOutBenchmarks` | `InMemoryTransport.SendToGroupAsync` at 10, 100, 1 000, 10 000 recipients |
| `PresenceAggregationBenchmarks` | `PresenceAggregator.Aggregate` — pure function, should be allocation-free |
| `StateMachineBenchmarks` | `CallStateMachine.CanTransition`, `DeliveryStateMachine.Advance` — should be allocation-free and near-free |
| `CursorBenchmarks` | cursor encode/decode from Phase 14 |
| `TurnCredentialBenchmarks` | HMAC credential generation from Phase 10 |

- Every benchmark must have a **baseline** so relative regressions are visible, and use
  `[Params]` for the size dimensions rather than duplicated methods.
- `scripts/benchmark.ps1` runs a named subset or all, writes results to `artifacts/benchmarks/`, and
  supports `-Baseline <path>` to diff against a previous run.
- `benchmarks/README.md`: how to run, why `ServerGC` is enabled, how to read the output, and the rule that
  numbers from a laptop under load are meaningless.

**Tests** — benchmarks are not tested, but:

| Test | Asserts |
|---|---|
| `Benchmark_project_compiles` | CI job |
| `Every_benchmark_class_has_a_memory_diagnoser` | reflection over the benchmark assembly, run as a unit test in `LiveSharp.ArchitectureTests` |

---

### P16.T2 — Load test harness and baseline measurements

**Deliverables**

```
tests/load/LiveSharp.LoadTests/LiveSharp.LoadTests.csproj
tests/load/LiveSharp.LoadTests/Scenarios/*.cs
tests/load/LiveSharp.LoadTests/Metrics/*.cs
tests/load/README.md
docs/performance/methodology.md
docs/performance/baseline-0.15.0.md
```

**Requirements**

- A console application that spins up N `LiveSharp.Client` instances against a real server (not
  `TestServer` — the in-memory transport removes the network, which is a large part of what is being
  measured) and reports latency percentiles and throughput.
- **Custom, not a framework.** NBomber and k6 are good tools, but the scenarios here need a real
  `HubConnection` per virtual user with typed operation calls, and driving that from a load-testing DSL is
  more work than a purpose-built harness. Justify this in the dependency table and revisit if the harness
  grows past a few hundred lines.
- Scenarios, each parameterised by connection count:

| Scenario | Measures |
|---|---|
| `ConnectionEstablishment` | connections per second, time to `connection.ready`, at 100 / 1 000 / 10 000 concurrent |
| `IdleConnections` | memory per idle connection, CPU at rest, at 10 000 connections held for 5 minutes |
| `DirectMessageThroughput` | messages/second and p50/p95/p99 end-to-end latency, 1 000 users in pairs |
| `GroupFanOut` | messages/second and latency for a 1 000-member group |
| `PresenceStorm` | broadcast cost when 1 000 users change status within 10 seconds |
| `TypingStorm` | throughput of `typing.start` under realistic keystroke rates |
| `ReconnectStorm` | recovery time when 5 000 connections reconnect simultaneously |
| `SignalingThroughput` | offer/answer/candidate exchanges per second |
| `MixedWorkload` | a realistic blend: 70% typing/presence, 25% messages, 5% signalling |

- Metrics: use `System.Diagnostics.Metrics` for collection and `HdrHistogram`-style percentile
  accumulation. Report p50, p95, p99, p99.9, max — **never** averages. An average latency on a real-time
  system hides exactly the tail that users notice.
- Report allocations and GC pause counts server-side by scraping the metrics added in `P16.T3`.
- The harness must be able to run against a **two-node Redis cluster** (Phase 15) as well as single node,
  because the Redis round trips are expected to be the dominant cost and that needs to be measured, not
  guessed.
- **Baseline before optimising.** `docs/performance/baseline-0.15.0.md` records the numbers *before*
  `P16.T4`/`P16.T5` touch anything, with full methodology: machine spec, .NET version, GC mode,
  configuration, and how to reproduce. This document is the evidence for every later claim.
- `docs/performance/methodology.md`: what is measured, what is not, why percentiles not averages, why
  numbers from different hardware are not comparable, and an explicit statement that these are library
  benchmarks and not a guarantee of application performance.

> ## Record the numbers even when they are bad
> If 10 000 idle connections consume 3 GB, that goes in the document. The point of a baseline is to be
> honest about where the library stands and to make improvement measurable. A performance document that
> only reports flattering numbers is marketing, and `EXECUTION_PLAN.md` §13 already committed to not
> doing that.

---

### P16.T3 — Observability: metrics and tracing

**Deliverables**

```
src/LiveSharp.Abstractions/Diagnostics/LiveSharpDiagnostics.cs
src/LiveSharp.Core/Diagnostics/LiveSharpMetrics.cs
src/LiveSharp.Core/Diagnostics/LiveSharpActivitySource.cs
src/LiveSharp.Core/Filters/DiagnosticsFilter.cs
docs/observability/README.md
docs/observability/metrics.md
docs/observability/tracing.md
tests/LiveSharp.Core.Tests/Diagnostics/…
```

**Public API (exact):**

```csharp
namespace LiveSharp.Diagnostics;

public static class LiveSharpDiagnostics
{
    public const string MeterName = "LiveSharp";
    public const string ActivitySourceName = "LiveSharp";
}
```

**Requirements — metrics**

Use `System.Diagnostics.Metrics` only. No OpenTelemetry package dependency, no Prometheus client, no
App Insights SDK — Spec §52 is explicit that users must not be forced onto a monitoring platform.
Consumers wire `Meter`/`ActivitySource` to whatever they use; document the OpenTelemetry registration
snippet without depending on it.

Instruments, with names following OpenTelemetry semantic-convention style:

| Instrument | Type | Unit | Tags |
|---|---|---|---|
| `livesharp.connections.active` | `ObservableGauge<int>` | `{connection}` | — |
| `livesharp.connections.total` | `Counter<long>` | `{connection}` | — |
| `livesharp.connections.duration` | `Histogram<double>` | `s` | `reason` |
| `livesharp.operations.total` | `Counter<long>` | `{operation}` | `operation`, `outcome` |
| `livesharp.operations.duration` | `Histogram<double>` | `s` | `operation`, `outcome` |
| `livesharp.messages.sent` | `Counter<long>` | `{message}` | `kind` (direct/group) |
| `livesharp.messages.fanout` | `Histogram<int>` | `{recipient}` | `kind` |
| `livesharp.presence.broadcasts` | `Counter<long>` | `{broadcast}` | `suppressed` |
| `livesharp.calls.active` | `ObservableGauge<int>` | `{call}` | — |
| `livesharp.calls.duration` | `Histogram<double>` | `s` | `end_reason` |
| `livesharp.ratelimit.rejected` | `Counter<long>` | `{operation}` | `operation` |
| `livesharp.store.duration` | `Histogram<double>` | `s` | `store`, `method`, `outcome` |
| `livesharp.redis.duration` | `Histogram<double>` | `s` | `operation`, `outcome` |

- **Cardinality discipline.** No tag may carry a user ID, connection ID, group ID, call ID, or
  conversation ID. Every one of those is unbounded and will destroy a metrics backend. `operation` is
  bounded (a fixed set of names) and is safe. Add an **architecture test** that reflects over every metric
  recording site and fails on a tag name in a deny-list (`user`, `connection`, `group`, `call`,
  `conversation`, `session`, `message`) — the same enforcement mechanism as Phase 09's logging audit, and
  just as necessary.
- Metrics must be **near-free when nobody is listening**. Check `Counter.Enabled` /
  `Histogram.Enabled` before computing tag values, and never allocate a tag array on a disabled
  instrument. Benchmark the disabled path in `P16.T1` and assert zero allocations.

**Requirements — tracing**

- One `ActivitySource`. Spans: `LiveSharp.Connect`, `LiveSharp.Operation` (with `operation` and `outcome`
  attributes), `LiveSharp.FanOut`, `LiveSharp.Store`, `LiveSharp.Redis`.
- `DiagnosticsFilter` replaces the correlation-only `CorrelationLoggingFilter` from Phase 09 as the
  outermost filter, adding the activity and the metric recording. Keep the correlation ID as the span's
  `livesharp.correlation_id` attribute so logs and traces join up.
- **Propagate trace context from the client.** `LiveSharp.Client` and `@livesharp/client` should send
  W3C `traceparent` as an operation header so a browser-initiated trace continues server-side. This
  requires adding an optional `Headers` dictionary to the invoke path — an **additive** protocol change.
  Bump `LiveSharpProtocol.Version` to `2`? No: an optional field that older servers ignore does not
  require a version bump, and bumping would break older clients for nothing. Add the field, leave the
  version at `1`, and document the compatibility reasoning. This is exactly the "can the change be
  additive" question Spec §35 asks.
- Never put message content, SDP, candidates, or tokens in span attributes. The Phase 09 logging deny-list
  test must be extended to cover activity tags.

**Tests**

| Test | Asserts |
|---|---|
| `Operation_counter_increments_with_the_operation_and_outcome_tags` | |
| `Operation_histogram_records_duration` | |
| `Connection_gauge_reflects_the_registry_count` | |
| `Fan_out_histogram_records_the_recipient_count` | |
| `Suppressed_presence_broadcasts_are_counted_separately` | the Phase 06 coalescing is now measurable |
| `Rate_limit_rejections_are_counted_per_operation` | |
| `No_metric_tag_name_matches_the_high_cardinality_deny_list` | **architecture test** |
| `No_activity_tag_carries_message_content_sdp_or_tokens` | **security test**, extends Phase 09's |
| `Disabled_instruments_allocate_nothing` | benchmark-backed assertion |
| `Activity_is_created_per_operation_with_the_correlation_id` | |
| `Client_supplied_traceparent_continues_the_trace_server_side` | integration |
| `An_operation_from_a_client_that_sends_no_headers_still_works` | **backward compatibility** |
| `Protocol_version_is_unchanged_by_the_additive_header_field` | pins the decision |

---

### P16.T4 — Serialization and allocation optimisation

**Deliverables**

```
src/LiveSharp.AspNetCore/Protocol/MessagePackSupport.cs
src/LiveSharp.AspNetCore/Configuration/LiveSharpHubOptions.cs        (modified)
src/LiveSharp.AspNetCore/Hubs/LiveSharpHub.cs                        (modified)
src/LiveSharp.Core/Routing/OperationRouter.cs                        (modified)
docs/performance/serialization.md
```

**Every change in this task requires a cited benchmark delta in its commit message.**

**Requirements — MessagePack as an option**

- Register `Microsoft.AspNetCore.SignalR.Protocols.MessagePack` when
  `LiveSharpHubOptions.EnableMessagePack` is true (default **false**). Both protocols can be enabled
  simultaneously; SignalR negotiates per connection.
- Default false because: JSON is debuggable in browser devtools, every third-party client can speak it,
  and the benchmark from `P16.T1` will show whether the difference justifies the opacity. Enable it in the
  docs as a recommendation only if the measured difference is material — and state the measured number
  rather than "faster".
- The TypeScript client gains an optional MessagePack protocol via `@microsoft/signalr-protocol-msgpack`
  as an **optional peer dependency**, so consumers who do not use it pay no bundle cost. Document the
  bundle-size delta.
- MessagePack and the source-generated `JsonSerializerContext` must both work. Verify every protocol DTO
  round-trips under MessagePack — the Phase 03 serializer-context guard test needs a MessagePack sibling.

**Requirements — allocation reduction, only where measured**

Candidate optimisations, each to be applied **only** if the benchmark shows it matters:

1. **Payload size measurement in `LiveSharpHub`.** Phase 03 measures payload size via
   `JsonElement.GetRawText()`, which allocates a string per message. Replace with writing to a pooled
   `ArrayBufferWriter<byte>` and reading `WrittenCount`, or better, use
   `JsonElement.WriteTo` on a `Utf8JsonWriter` over a pooled buffer. Expected to be the single largest
   per-message allocation in the system. Benchmark before and after.
2. **Envelope construction.** One `RealtimeEnvelope` per recipient in a fan-out means N allocations for
   one logical message. Serialize once, share the serialized payload across recipients. `IHubContext`
   already does this internally for `Clients.Group`, so verify whether LiveSharp is defeating it by
   constructing per-recipient envelopes in `SendToUsersAsync`.
3. **`FrozenDictionary` for operation lookup** — already done in Phase 01, verify the benchmark confirms
   it beats `Dictionary` at realistic operation counts. If it does not at 10 operations, say so.
4. **Filter pipeline delegate allocation.** Phase 09 composes the pipeline once, but the
   `OperationFilterDelegate` closure chain may allocate per invocation. Benchmark with 0 and 5 filters; if
   there is per-call allocation, restructure to an index-based iteration over a filter array with a
   struct enumerator state.
5. **`ImmutableHashSet` in the registries.** The `AddOrUpdate` + immutable-set pattern from Phase 02
   allocates a new set per membership change. At 10 000 connections churning, benchmark whether this
   matters versus a lock-per-user-bucket alternative. If it does not matter, **leave it alone** and record
   that it was measured and rejected — that record prevents someone re-litigating it later.
6. **Log message allocation.** `[LoggerMessage]` source generation already avoids boxing; verify no
   remaining `logger.LogDebug($"...")` interpolation exists in `src/`. Add an analyzer rule or a source
   scan test.

- For each candidate: record the before/after in `docs/performance/serialization.md` (or an
  `optimisations.md`), including the ones that were **rejected** because the measurement showed no benefit.
  A list of rejected optimisations with numbers is one of the most useful documents a performance-sensitive
  library can have.

**Tests**

| Test | Asserts |
|---|---|
| `Message_pack_protocol_round_trips_every_protocol_dto` | the MessagePack guard sibling |
| `Message_pack_and_json_clients_can_coexist_on_one_server` | integration, one of each |
| `Message_pack_disabled_by_default` | |
| `Payload_size_measurement_allocates_no_string` | benchmark-backed |
| `Fan_out_to_a_thousand_recipients_serializes_the_payload_once` | count serializer invocations |
| `Filter_pipeline_invocation_allocates_nothing_beyond_the_result` | benchmark-backed |
| `No_source_file_uses_interpolated_string_logging` | source scan |
| `All_existing_tests_still_pass` | **the real acceptance criterion for this task** |

---

### P16.T5 — Redis and database path optimisation

**Deliverables**

```
src/LiveSharp.Redis/…                                                (modified)
src/LiveSharp.EntityFrameworkCore/…                                  (modified)
docs/performance/scaling-numbers.md
```

**Requirements**

The Phase 15 baseline will almost certainly show that Redis round trips dominate multi-node latency. Fix
what the numbers indicate, in this order of expected value:

1. **Batch pipelining.** `GetUserConnectionsAsync` reads a set then N hashes. Pipeline them into one
   round trip with `StackExchange.Redis`'s batch API. Measure the improvement at 1, 3, and 10 connections
   per user.
2. **`Touch` coalescing** was already added in `P15.T2`. Verify the benchmark confirms it eliminates the
   per-message round trip, and tune the window from the measurement rather than the guess.
3. **Local read caching for presence and group membership**, with a short TTL (1–2 s) and invalidation via
   the Phase 15 control channel. This is a real consistency trade: a cached group membership means a
   just-removed member may receive one more message. Decide deliberately: **cache group membership, do not
   cache presence.** Group membership changes rarely and one extra message to a removed member is
   tolerable; presence changes constantly and stale presence is exactly the complaint users have. Make the
   cache TTL configurable with `0` disabling it, default it based on the measurement, and document the
   consistency implication in `docs/scaling/consistency-model.md`.
4. **Fan-out through the backplane.** Measure whether `Clients.Users(1000 ids)` is efficient or whether
   `Clients.Group` on a transport group is materially better for large fan-outs. If the group path wins,
   consider maintaining transport groups for large recipient sets — but only if the measurement justifies
   the added state.
5. **EF query shape.** Verify from captured SQL that `GetConversationAsync` is a single index seek and
   `GetUnreadCountAsync` uses the filtered index. Add `AsSplitQuery` only where a measured cartesian
   explosion exists. Check for accidental N+1 in `GetConversationsForUserAsync`.
6. **Connection pool sizing** guidance, measured: how many Redis connections and database connections
   10 000 LiveSharp connections actually need. Put real numbers in `docs/scaling/operations.md`, replacing
   the "pending measurement" placeholder Phase 15 left there.

**Tests**

| Test | Asserts |
|---|---|
| `Get_user_connections_issues_a_single_round_trip` | command counting |
| `Group_membership_cache_serves_repeat_reads_without_a_round_trip` | |
| `Group_membership_cache_is_invalidated_by_a_membership_change_on_another_node` | control channel |
| `Group_membership_cache_disabled_by_zero_ttl` | |
| `Presence_is_never_served_from_a_local_cache` | pins the deliberate decision |
| `Get_conversation_generates_a_single_index_seek_query` | captured SQL |
| `Get_conversations_for_user_issues_no_n_plus_one_queries` | query counting |
| `Every_phase_15_distributed_test_still_passes` | **the acceptance criterion** |
| `Every_store_contract_suite_still_passes` | |

---

### P16.T6 — Leak tests and published results

**Deliverables**

```
tests/LiveSharp.LeakTests/LiveSharp.LeakTests.csproj
tests/LiveSharp.LeakTests/*.cs
docs/performance/README.md
docs/performance/results-0.15.0.md
.github/workflows/benchmark.yml
```

**Requirements — leak tests**

A real-time server runs for weeks. A 200-byte leak per connection is invisible in a test and fatal in
production. These tests are the ones that justify the phase.

Pattern: run N cycles of an operation, force `GC.Collect(); GC.WaitForPendingFinalizers(); GC.Collect();`,
and assert that a target object is collectable (via `WeakReference`) and that
`GC.GetTotalAllocatedBytes()` growth is bounded. Use `WeakReference` liveness rather than absolute memory
comparisons where possible — absolute memory assertions are flaky.

| Test | Asserts |
|---|---|
| `A_disconnected_connection_is_collectable` | `WeakReference` dead after disconnect + GC |
| `Ten_thousand_connect_disconnect_cycles_do_not_grow_the_registry` | count returns to zero |
| `Ten_thousand_connect_disconnect_cycles_bound_total_allocation` | growth under a threshold |
| `A_disposed_client_subscription_is_collectable` | the Phase 03 subscription leak |
| `A_disposed_LiveSharpClient_releases_its_hub_connection` | |
| `A_closed_peer_connection_handle_releases_its_event_handlers` | Phase 10 helper |
| `A_hung_up_call_releases_its_media_handle` | Phase 11/12 |
| `Group_membership_churn_does_not_grow_the_registries` | Phase 02/05 empty-key discipline, at scale |
| `Presence_churn_does_not_grow_the_subscription_indexes` | |
| `Typing_churn_does_not_grow_the_tracker` | Phase 07's main risk, verified at scale |
| `Rate_limiter_partitions_are_evicted_under_user_churn` | Phase 09's bounded-partition claim |
| `Receipt_batcher_pending_map_does_not_grow_unbounded` | Phase 08 |
| `Signalling_candidate_buffers_are_released_on_session_close` | Phase 10 |
| `Redis_local_caches_are_bounded` | Phase 16's own additions |
| `A_five_minute_mixed_workload_shows_no_monotonic_memory_growth` | the integration-level leak test |

`LiveSharp.LeakTests` is a **separate project** excluded from the default `dotnet test` run (they are slow
and GC-sensitive) but run in a dedicated CI job on every push to `main` and on release branches. A leak
test that is skipped is not a leak test — make sure it actually runs somewhere on a schedule.

**Requirements — published results**

- `docs/performance/README.md`: the honest summary. What LiveSharp costs per connection, what throughput
  it sustains, what the latency percentiles are, on what hardware, with what configuration, single-node
  and two-node. Link to the methodology and to the raw baseline.
- `docs/performance/results-0.15.0.md`: the after-optimisation numbers, side by side with the
  `baseline-0.15.0.md` before-numbers, with the delta for each optimisation and a note for each rejected
  one.
- **No unqualified claims.** Not "LiveSharp handles 100 000 connections" but "10 000 concurrent idle
  connections consumed X MB and Y% CPU on <spec>; message throughput was Z msg/s at p99 latency of W ms
  under the MixedWorkload scenario". Every number carries its scenario and its hardware.
- `.github/workflows/benchmark.yml`: runs the microbenchmark suite on a schedule and on demand, storing
  results as artifacts. **Do not** gate PRs on benchmark results — CI runners are too noisy for reliable
  regression detection, and a flaky performance gate trains people to ignore CI. Instead, publish the
  trend and review it deliberately. State this reasoning in the workflow comment.

---

## Dependency justification

| Package | Why | Alternative? | Licence | Notes |
|---|---|---|---|---|
| `BenchmarkDotNet` `0.15.8` | The standard .NET microbenchmark harness with statistically sound methodology and a memory diagnoser. | Hand-rolled `Stopwatch` loops produce numbers that are wrong in ways that are hard to see. | MIT | `benchmarks/` only. Not shipped. |
| `Microsoft.AspNetCore.SignalR.Protocols.MessagePack` `10.0.11` | Optional binary protocol. | None; it is the SignalR-supported binary protocol. | MIT | `LiveSharp.AspNetCore`, and **only** activated by an opt-in option. Adds a package reference to a package that previously had only a `FrameworkReference` — note this in ADR-004 as a deliberate cost, and confirm it does not appear in the dependency graph when the option is off (it will, because package references are unconditional; if that is unacceptable, move it to a `LiveSharp.AspNetCore.MessagePack` package and record the decision). **Resolve this before implementing.** |
| `@microsoft/signalr-protocol-msgpack` | TypeScript MessagePack protocol. | — | MIT | **Optional peer dependency** so it costs nothing when unused. |

Explicitly rejected: OpenTelemetry SDK packages (Spec §52 — do not force a monitoring platform; the
`Meter`/`ActivitySource` primitives are the integration point), `prometheus-net`, Application Insights
SDK, NBomber / k6 (see `P16.T2`), `HdrHistogram` NuGet (a percentile accumulator is 50 lines and the
harness is not shipped).

> The MessagePack package-reference question in the table above is a real decision, not boilerplate. A
> package reference that appears in the dependency graph regardless of whether the feature is enabled
> contradicts Spec §7. Decide it explicitly, write it in ADR-004, and if the answer is a separate package,
> do that instead.

---

## Documentation deltas

- `docs/performance/README.md`, `methodology.md`, `baseline-0.15.0.md`, `results-0.15.0.md`,
  `serialization.md`, `scaling-numbers.md` — all **new**.
- `docs/observability/README.md`, `metrics.md`, `tracing.md` — **new**. The full instrument table, the
  cardinality rules and why, OpenTelemetry wiring shown without depending on it, and the trace-context
  propagation model.
- `docs/scaling/operations.md` — replace the pending-measurement placeholders with real numbers.
- `docs/scaling/consistency-model.md` — add the group-membership cache and its implication.
- `docs/configuration/README.md` — `EnableMessagePack`, cache TTLs.
- `docs/troubleshooting/README.md` — "high memory usage" (per-connection cost, GC mode), "latency spikes"
  (GC pauses, and the metrics to check), "metrics backend overwhelmed" (cardinality — though LiveSharp's
  own tags are bounded, a consumer's custom filter may not be), "MessagePack client cannot connect"
  (protocol not enabled server-side), "removed group member still received a message" (membership cache).
- `README.md` — a Performance section linking to the results with one or two headline numbers **including
  their conditions**.
- `benchmarks/README.md`, `tests/load/README.md` — **new**.

## Example deltas

None. Examples are not performance artifacts, and adding benchmarking to them would obscure their purpose.

## CHANGELOG entry

```markdown
### Added
- Benchmark suite (`benchmarks/`) covering serialization, operation routing, registries, fan-out, and
  state machines, with allocation diagnostics.
- Load test harness (`tests/load/`) with scenarios for connection establishment, message throughput,
  group fan-out, presence and typing storms, reconnect storms, and a mixed workload, reported as latency
  percentiles.
- Metrics via `System.Diagnostics.Metrics` and tracing via `ActivitySource`, with no dependency on any
  monitoring platform.
- W3C trace-context propagation from the .NET and TypeScript clients, as an additive protocol field.
- Optional MessagePack protocol support, disabled by default.
- Leak test suite asserting that connections, subscriptions, calls, and all registries and caches are
  bounded under churn.
- Published performance baseline and results with full methodology.

### Changed
- Reduced per-message allocations on the hub payload-size and fan-out paths.
- Redis reads for user connections are pipelined into a single round trip.
- Optional short-lived local caching of group membership, configurable and disabled by setting the TTL to
  zero.

### Notes
- Presence is deliberately never served from a local cache, because stale presence is more visible to
  users than a stale group membership.
- Benchmark results are published as trends and do not gate pull requests; CI runners are too noisy for
  reliable regression gating.
```

---

## Exit criteria

- [ ] `docs/performance/baseline-0.15.0.md` was committed **before** any optimisation commit — verifiable
      from `git log`.
- [ ] Every optimisation commit cites a specific benchmark delta in its message.
- [ ] Every **rejected** optimisation is recorded with its measurement.
- [ ] `Every_benchmark_class_has_a_memory_diagnoser` passes.
- [ ] `No_metric_tag_name_matches_the_high_cardinality_deny_list` passes.
- [ ] `No_activity_tag_carries_message_content_sdp_or_tokens` passes.
- [ ] `Disabled_instruments_allocate_nothing` passes.
- [ ] `An_operation_from_a_client_that_sends_no_headers_still_works` passes — the additive protocol change
      is genuinely additive, and `LiveSharpProtocol.Version` is still `1`.
- [ ] `Payload_size_measurement_allocates_no_string` passes.
- [ ] `Fan_out_to_a_thousand_recipients_serializes_the_payload_once` passes.
- [ ] `Message_pack_and_json_clients_can_coexist_on_one_server` passes.
- [ ] The MessagePack package-reference question is resolved and recorded in ADR-004.
- [ ] `Presence_is_never_served_from_a_local_cache` passes.
- [ ] `Group_membership_cache_is_invalidated_by_a_membership_change_on_another_node` passes.
- [ ] Every leak test passes, and `LiveSharp.LeakTests` runs in a real CI job — not skipped everywhere.
- [ ] `A_five_minute_mixed_workload_shows_no_monotonic_memory_growth` passes.
- [ ] Every pre-existing test, contract suite, distributed test, and E2E spec still passes.
- [ ] `docs/performance/README.md` contains no unqualified performance claim.
- [ ] `docs/scaling/operations.md` has real measured numbers replacing the placeholders.
- [ ] `docs/observability/metrics.md` documents every instrument and the cardinality rules.
- [ ] `scripts/verify.ps1` green; tag `v0.15.0`.

## Verify

```powershell
pwsh scripts/verify.ps1
dotnet build benchmarks/LiveSharp.Benchmarks.slnx -c Release
pwsh scripts/benchmark.ps1 -Filter "*Serialization*"
dotnet test tests/LiveSharp.LeakTests -c Release
dotnet run --project tests/load/LiveSharp.LoadTests -- --scenario MixedWorkload --connections 1000
git tag v0.15.0
```

## Next

`docs/plan/phase-17-tooling.md`, task `P17.T1`.
