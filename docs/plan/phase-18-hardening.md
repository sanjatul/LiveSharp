# Phase 18 — Production Hardening & 1.0.0

| | |
|---|---|
| **Version at exit** | `1.0.0` (tag `v1.0.0`) |
| **Depends on** | Phase 17 |
| **Branch** | `feat/phase-18-hardening` → `release/1.0` |
| **Tasks** | `P18.T1` … `P18.T8` |
| **Read before starting** | this entire file, `EXECUTION_PLAN.md` §2 and §4, every ADR, and `docs/security/checklist.md` |

> ## Several tasks in this phase are irreversible
> `P18.T1` freezes the public API. `P18.T8` publishes to NuGet and npm, which cannot be undone — a
> published package version can be unlisted but never replaced. Read the whole phase, complete the tasks in
> order, and do not begin `P18.T8` until every other exit criterion is met.

---

## Goal

Turn `0.16.0` into something worth calling `1.0.0`: a frozen public API, proven resilience under failure,
a completed security review, honest deployment guidance, and an automated release.

Nothing in this phase adds features. Everything in it is about being able to say "this API will not break
under you" and mean it.

---

## What 1.0.0 commits you to

Say this explicitly in `docs/versioning.md` before doing the work, because it changes how the rest of the
phase should be executed:

- **The public API is a contract.** After 1.0.0, breaking it requires 2.0.0, and 2.0.0 requires a migration
  guide, updated examples, and a deprecation period. Spec §49.
- Additive changes only, in minors. Bug fixes in patches.
- The wire protocol version stays `1` until there is a reason; a protocol change requires a negotiated
  version and a compatibility window.
- Every `[Experimental]` API is explicitly excluded from the guarantee and must be labelled as such in the
  API reference.

Consequence for this phase: **`P18.T1` is the most important task**, and anything that should not be public
must become internal *before* the freeze, not after.

---

## Non-goals — DO NOT implement in this phase

- No new features. If a gap is discovered, it goes in `ROADMAP.md` for 1.1, not into 1.0.
- No new packages, no new dependencies.
- No conference calling, SFU, mobile SDKs, Azure SignalR support, or additional framework adapters. All
  post-1.0.
- No `LiveSharp.Media.SIPSorcery`. It remains unbuilt and unpromised.
- No performance work beyond fixing something the chaos suite reveals as a correctness problem.
- No infrastructure-as-code templates (Terraform, Bicep, Helm charts). Deployment *guidance* is in scope;
  maintaining deployment *artifacts* for four clouds is a separate product.

---

## Tasks

### P18.T1 — Public API review and freeze

**Deliverables**

```
src/**/PublicAPI.Shipped.txt                                        (populated from Unshipped)
src/**/PublicAPI.Unshipped.txt                                      (emptied)
src/LiveSharp.Abstractions/ExperimentalAttribute usage              (applied where needed)
docs/api-surface-review.md
Directory.Build.props                                               (EnablePackageValidation)
```

**Requirements**

Go through **every public type and member** in every shipping assembly and answer the Spec §8 questions:

```
Is this really public?
Does a consumer need to interact with it?
Can it be internal?
Is the name intuitive?
Is it discoverable through IntelliSense?
Is it future-proof?
Will changing it later be a breaking change?
```

Produce `docs/api-surface-review.md` recording the decision for every public type: **keep**,
**make internal**, **rename**, **mark experimental**, or **remove**. A type with no recorded decision has
not been reviewed.

Specific things to look for, based on what the plan created:

| Suspect | Question |
|---|---|
| `InMemoryTransport`, `InMemoryMessageStore`, and every other `InMemory*` | These are documented as development-only. Should they be public at all, or should they move to a `LiveSharp.Testing` package? **Decide.** Keeping a bounded, non-durable store public invites production use; a testing package makes intent unambiguous. Recommend the package split — it is additive and clarifies the contract. |
| `OperationRegistration`, `OperationFilterPipeline`, `CallStateMachine`, `DeliveryStateMachine`, `PresenceAggregator` | Internal machinery that is currently public because it was convenient to test. Make internal + `InternalsVisibleTo`, unless a consumer genuinely needs them. The state machines are arguably useful publicly for building a UI; decide and record. |
| `RedisKeys` | Documented for operators, but public makes the key schema a contract. Recommend internal, with the schema documented in Markdown only. |
| `LiveSharpDbContext` | Must stay public and inheritable (Phase 14). Confirm the `[DesignedForInheritance]` marker and that its virtual surface is deliberate. |
| Every `record` with `required` members | Adding a `required` member later is breaking. Review each for missing fields that will be wanted in 1.x. |
| Every enum | Adding a value is non-breaking; reordering or renaming is not. Confirm all numeric values are pinned and tested. |
| Every interface | Adding a member is breaking for implementers. Consider whether any should have default interface implementations to allow future growth — and note that DIMs have their own versioning subtleties. Decide per interface. |
| `IRealtimeTransport`, `IConnectionRegistry`, and the store interfaces | These are the extension points third parties will implement. Are they complete enough to survive 1.x without additions? This is the highest-risk review item. |

**Then freeze:**

1. Move every entry from `PublicAPI.Unshipped.txt` to `PublicAPI.Shipped.txt` in each project.
2. Set `EnablePackageValidation=true` in `src/Directory.Build.props` with
   `PackageValidationBaselineVersion=0.16.0`, so any breaking change against the last release fails the
   build.
3. Add an `[Experimental("LS####")]` diagnostic (or the project's own equivalent) to anything not covered
   by the guarantee, and list them in `docs/versioning.md`.

**Tests**

| Test | Asserts |
|---|---|
| `Every_public_type_appears_in_the_api_surface_review` | reflection cross-referenced against the Markdown |
| `PublicAPI_Unshipped_is_empty_for_every_shipping_project` | the freeze happened |
| `Package_validation_is_enabled_with_a_baseline` | |
| `Every_experimental_api_is_listed_in_the_versioning_doc` | |
| `No_public_type_lacks_xml_documentation` | Phase 01's test, re-verified at the freeze |
| `Every_public_enum_has_pinned_numeric_values_with_a_test` | |
| `No_shipping_assembly_exposes_a_type_from_a_dependency_it_does_not_own` | e.g. no `IHubContext`, `DbContext` (except `LiveSharpDbContext`), `IConnectionMultiplexer`, or SIPSorcery type in a public signature where it was not intended |

---

### P18.T2 — Resilience and chaos testing

**Deliverables**

```
tests/LiveSharp.ChaosTests/LiveSharp.ChaosTests.csproj
tests/LiveSharp.ChaosTests/*.cs
docs/operations/resilience.md
```

**Requirements**

Everything so far has been tested against cooperative failure. Chaos testing is about uncooperative
failure, and it is where the remaining real bugs are.

Each scenario runs a realistic mixed workload (reuse the Phase 16 load harness scenarios at modest scale)
while injecting a fault, then asserts the system **converges** — not that nothing went wrong, but that
after the fault clears, state is consistent and traffic flows.

| Scenario | Injection | Convergence assertion |
|---|---|---|
| `RedisOutage` | stop the Redis container for 30 s, restart | within one `NodeTimeout`: no phantom connections, presence matches reality, groups intact, rate limiting resumes |
| `RedisSlowdown` | Toxiproxy latency injection (500 ms) | operations complete or fail cleanly; no deadlock; no unbounded queue growth |
| `DatabaseOutage` | stop PostgreSQL for 30 s | message sends fail with `InternalError`, live delivery continues if the store is optional, history returns errors; recovery restores writes |
| `NodeKill` | `SIGKILL` one of two nodes mid-traffic | clients reconnect to the survivor; connections reaped; presence converges; active calls end as `Disconnected` after the grace period |
| `RollingRestart` | drain and restart node A, then node B | no message loss for clients that reconnect; no duplicate deliveries |
| `ReconnectStorm` | drop 5 000 connections simultaneously | server does not exhaust threads or Redis connections; all reconnect within a bounded time; group rehydration completes for all |
| `BackpressureFlood` | one client invoking as fast as possible | rate limiter engages; other clients' p99 latency stays within a stated bound; server memory stays bounded |
| `SlowClient` | a client that stops reading its socket | SignalR's own buffer limits engage; the connection is dropped; the server does not buffer unboundedly. **Verify what SignalR actually does here and document it** — this is a real DoS vector and the answer is a configuration setting most people never set |
| `ClockSkew` | node B's clock 60 s ahead | TURN credentials still validate (or fail with a clear symptom); presence TTL and call timeouts behave predictably. Document the requirement for NTP |
| `PartialNetworkPartition` | node A can reach Redis, node B cannot | node B degrades to local-only behaviour with warnings rather than failing all requests; recovery converges |
| `MalformedClient` | a client sending garbage frames, oversized payloads, unknown operations, and rapid connect/disconnect | no exception escapes to crash the host; no unbounded resource growth; the connection is rejected or closed |

- **Graceful shutdown / connection draining** is a deliverable of this task, not just a test: on
  `ApplicationStopping`, stop accepting new connections, send `connection.closing` to all connections with
  a reason so clients reconnect elsewhere immediately rather than waiting for a timeout, wait a bounded
  drain period, then abort the remainder. Add `LiveSharpOptions.Connections.DrainTimeout` (default 15 s).
  Without this, every deployment causes a thundering-herd reconnect after a client-side timeout.
- Chaos tests are slow and Docker-dependent. Separate project, excluded from `dotnet test` by default, run
  in a dedicated CI job on `main` and release branches — the same policy as the Phase 16 leak tests. Not
  skipped everywhere.
- Record the findings in `docs/operations/resilience.md`: what LiveSharp does under each failure, what the
  operator should configure, and what is **not** handled. Failures found and fixed get a regression test;
  failures found and accepted get documented as limitations.

**Tests** — the scenarios above, each with an explicit convergence assertion and a bounded time limit.

---

### P18.T3 — Security review and threat model

**Deliverables**

```
docs/security/threat-model.md
docs/security/checklist.md                                          (revised)
SECURITY.md                                                         (revised)
docs/security/dependencies.md
tests/LiveSharp.SecurityTests/…                                     (consolidated)
```

**Requirements**

- `docs/security/threat-model.md`: a real threat model, not a list of features. Use a simple structure —
  assets, trust boundaries, threats, mitigations, residual risk:

| Asset | Threat | Mitigation | Residual risk |
|---|---|---|---|
| Message content | eavesdropping in transit | TLS required | server operator sees content; no E2EE |
| Message content | unauthorized read | `IRealtimeAuthorizationService`, participant checks | a permissive implementation defeats it |
| User identity | impersonation | identity from the authenticated connection only; no payload-supplied actor | depends on the host's authentication |
| TURN bandwidth | credential theft | ephemeral per-user credentials, short TTL | static-credential escape hatch exists |
| Network topology | ICE candidate disclosure | candidates relayed only to session participants; never logged | a participant learns the peer's IPs — inherent to WebRTC; document it |
| Server availability | connection flood | connection limits, rate limiting, payload limits, drain on shutdown | per-node limits pre-Redis; no IP-level protection (that is the proxy's job) |
| Server availability | slow-loris / slow reader | SignalR buffer limits | requires configuration; documented in `P18.T2` |
| Presence data | activity-pattern harvesting | default-deny subscriptions | a permissive subscription source defeats it |
| Read receipts | read-time disclosure | restricted to sender and recipients | inherent to the feature |
| Call metadata | CDR disclosure | not logged; stored only when a call store is configured | persisted CDRs are the application's responsibility |
| Signalling relay | covert messaging channel | two-gate authorization, session participation, size and rate limits | — |
| Access tokens | leakage via query string | short TTLs, redaction guidance | present in proxy logs unless configured out |

  For each row, cite the test that proves the mitigation. A mitigation with no test is a claim.
- **Consolidate the security tests.** Security assertions are currently spread across ten test projects.
  Add `tests/LiveSharp.SecurityTests` that references them and provides a single suite mapping
  one-to-one onto the threat model rows, so a reviewer can run one command. Do not duplicate the tests —
  reference or re-expose them.
- `docs/security/dependencies.md`: every shipped dependency, its licence, why it is present, and its
  transitive footprint. Generated from the projects where possible. Required for consumers doing their own
  supply-chain review, and it makes the Phase 00–17 dependency discipline auditable.
- Re-run every audit built along the way and confirm they still pass: the Phase 09 logging deny-list, the
  Phase 16 metric-cardinality deny-list, the Phase 17 secret-redaction attribute scan, and the
  `dotnet list package --vulnerable` gate.
- `SECURITY.md` revised for a 1.0 project: supported versions (1.0.x), the private reporting process via
  GitHub Security Advisories, a response-time commitment you can actually keep, and the disclosure policy.
- `docs/security/checklist.md` revised into the definitive pre-production checklist, and cross-checked so
  that every item is (a) actually configurable and (b) surfaced by a Phase 17 `LS####` diagnostic where
  possible. An item on the checklist that the library cannot help you verify is a weak item.

**Tests**

| Test | Asserts |
|---|---|
| `Every_threat_model_mitigation_cites_an_existing_test` | scan the Markdown, resolve test names by reflection |
| `Every_security_checklist_item_is_configurable` | manual review recorded, plus automated where possible |
| `The_logging_deny_list_audit_passes` | Phase 09, re-verified |
| `The_metric_cardinality_audit_passes` | Phase 16, re-verified |
| `The_secret_redaction_audit_passes` | Phase 17, re-verified |
| `No_shipped_package_has_a_known_vulnerability` | `dotnet list package --vulnerable --include-transitive` |
| `No_shipped_package_has_a_licence_outside_the_approved_set` | MIT / Apache-2.0 / BSD-3 / PostgreSQL |

---

### P18.T4 — Deployment guidance

**Deliverables**

```
docs/operations/README.md
docs/operations/deployment.md
docs/operations/kubernetes.md
docs/operations/azure.md
docs/operations/turn-deployment.md
docs/operations/monitoring.md
docs/operations/capacity-planning.md
```

**Requirements**

Guidance, not artifacts. Each page must be something an operator can act on, with real values from the
Phase 16 measurements rather than invented ones.

- `deployment.md`: the reverse-proxy requirements (WebSocket upgrade headers, timeouts long enough for
  keep-alive, buffering disabled, sticky sessions — linking Phase 15's page), TLS, health-check endpoints
  from Phase 17, the drain-on-shutdown behaviour from `P18.T2` and what `terminationGracePeriodSeconds`
  must therefore be, and the environment variables/configuration keys that matter.
- `kubernetes.md`: a complete, minimal, **commented** manifest set (Deployment, Service, Ingress with
  sticky-session annotations, HPA, PDB) as documentation — explicitly marked as a starting point, not a
  supported artifact. Cover the interaction between `terminationGracePeriodSeconds`, `DrainTimeout`, and
  `NodeTimeout`, because getting that triple wrong causes exactly the phantom-connection and
  reconnect-storm problems Phase 15 and `P18.T2` worked to avoid. Include readiness vs liveness guidance:
  readiness should fail during drain so the load balancer stops sending new connections; liveness should
  not, or Kubernetes kills the pod mid-drain.
- `azure.md`: App Service (WebSockets must be explicitly enabled — a classic gotcha), Container Apps,
  Application Gateway session affinity, and a clear statement that **Azure SignalR Service is not
  supported**, with the reason (it changes the connection model, moves connection ownership out of the
  process, and LiveSharp's registries assume the server owns its connections). Point at the roadmap.
- `turn-deployment.md`: the operator-facing counterpart to Phase 10's `stun-turn.md`. A complete coturn
  deployment: `turnserver.conf` with `static-auth-secret`, TLS certificates, firewall ports (3478,
  5349, and the relay range), bandwidth cost estimation, monitoring relay usage, and the managed
  alternatives with the trade-off stated (cost vs operational burden). Include how to verify the credential
  scheme end to end and how to confirm from `getStats()` that relay is working.
- `monitoring.md`: which Phase 16 metrics to alert on and at what thresholds, with reasoning — active
  connections trending vs a baseline, `livesharp.operations.duration` p99, `livesharp.ratelimit.rejected`
  rate, `livesharp.redis.duration` p99, node count vs expected. Include the OpenTelemetry Collector
  configuration for scraping the meter, and Grafana panel definitions as JSON in an appendix.
- `capacity-planning.md`: from the Phase 16 numbers — memory and CPU per connection, Redis memory per
  connection, database growth per message, connections per instance before degradation, and a worked
  example ("10 000 concurrent users, 5 messages/minute each, 30-day retention → N instances, M GB Redis,
  P GB database"). This is the page people actually need and almost no library provides.

No tests, but every number in these pages must be traceable to
`docs/performance/results-0.15.0.md`. Add a review note: any figure without a source is removed.

---

### P18.T5 — Documentation completeness and honesty audit

**Deliverables**

```
docs/versioning.md
docs/migration/README.md
docs/roadmap-post-1.0.md
ROADMAP.md                                                          (revised)
README.md                                                           (revised)
docs/**                                                             (audited)
```

**Requirements**

- **The honesty audit.** Spec §61 and §33 require that documentation never claim more than the code does.
  Go through every page and verify: every feature marked `Available` is fully implemented, tested,
  documented, and demonstrated in an example; every `Preview`/`Experimental` label is accurate; every
  "coming in Phase N" reference is either resolved or converted to a roadmap link (there are no more
  phases after this one, so every such reference must be resolved or moved to post-1.0 roadmap language);
  every `TODO(maintainer)` placeholder from Phase 00 is filled in.
- Add a test that scans `docs/**/*.md` for the strings `Phase 1`…`Phase 18`, `TODO`, `TBD`, `coming soon`,
  and `pending measurement`, and fails on any hit outside `docs/plan/**`. Mechanical enforcement of
  honesty is the only kind that survives.
- `docs/versioning.md`: the SemVer policy, what the 1.0 guarantee covers and excludes, the experimental-API
  list, the wire-protocol versioning policy, the .NET target-framework policy (ADR-001 and when
  multi-targeting would be reconsidered), and the deprecation process for 2.0.
- `docs/migration/README.md`: an index of the migration guides written along the way (`0.7-to-0.8`,
  the Phase 05 `ChatMessage` change), plus a `0.16-to-1.0` guide covering the `P18.T1` API changes —
  which, if the `InMemory*` types moved to `LiveSharp.Testing`, is a real breaking change and needs
  precise instructions.
- `ROADMAP.md`: every 1.0 item marked `[x]`, and a **post-1.0 section** with honest status: conference
  calling via an SFU, `LiveSharp.Media.SIPSorcery`, Azure SignalR support, additional framework adapters,
  mobile SDKs, multi-targeting, end-to-end encryption. Marked `[ ]` planned or `[-]` deferred with a
  reason. No dates.
- `README.md` final form: the feature table with every 1.0 feature `Available`, the five-minute quick start
  from Phase 17, the architecture diagram, honest non-goals (`EXECUTION_PLAN.md` §13), the performance
  headline with its conditions, links to the docs site, security policy, contributing, and versioning
  policy. Delete the pre-alpha warnings.

**Tests**

| Test | Asserts |
|---|---|
| `No_documentation_page_references_a_development_phase` | outside `docs/plan/**` |
| `No_documentation_page_contains_TODO_TBD_or_coming_soon` | outside `docs/plan/**` |
| `No_documentation_page_contains_a_pending_measurement_placeholder` | |
| `Every_feature_marked_available_has_a_docs_page_and_an_example` | cross-reference the README table |
| `Every_experimental_api_appears_in_the_versioning_doc` | |
| `Every_migration_guide_referenced_in_the_index_exists` | |
| `Every_docs_page_is_in_the_toc` | Phase 17's test, re-verified |
| `Every_docs_code_sample_compiles` | Phase 17's test, re-verified |
| `All_internal_documentation_links_resolve` | link checker |

---

### P18.T6 — Package quality and metapackage

**Deliverables**

```
src/LiveSharp/LiveSharp.csproj                                      (metapackage)
src/LiveSharp.Testing/…                                             (if P18.T1 decided the split)
assets/icon.png                                                     (real icon)
src/**/*.csproj                                                     (package metadata completed)
tests/LiveSharp.PackagingTests/…
```

**Requirements**

- The `LiveSharp` metapackage: `Core` + `AspNetCore` + `Chat`. Deliberately **not** presence, calls,
  signalling, Redis, or EF — a metapackage that pulls everything defeats the whole package-boundary
  exercise (Spec §7). Its README must list what it includes and what to add for each additional feature.
- If `P18.T1` decided to move the `InMemory*` types, create `LiveSharp.Testing` now and write the
  migration note. It should also contain the test kit helpers consumers would want (`FakeTimeProvider`
  wiring, `InMemoryTransport` assertions, a `LiveSharpTestHost`) — the helpers Phase 01 built for internal
  use are genuinely useful to consumers writing tests against LiveSharp, and shipping them is a real DX
  win that costs one package.
- Per Spec §42, every shipped package must have: XML documentation, README, repository URL, licence
  expression, tags, version, release notes (from `CHANGELOG.md`), a `.snupkg` symbol package, Source Link,
  deterministic build, and an icon. A real icon, not the Phase 00 placeholder.
- **Automated packaging tests**, because package metadata problems are only ever discovered by consumers:

| Test | Asserts |
|---|---|
| `Every_package_contains_an_xml_documentation_file` | |
| `Every_package_contains_a_readme` | |
| `Every_package_declares_the_MIT_licence_expression` | |
| `Every_package_declares_a_repository_url_and_commit` | |
| `Every_package_has_an_icon` | |
| `Every_package_has_release_notes` | |
| `Every_package_produces_a_symbol_package` | |
| `Source_link_metadata_resolves_to_a_real_commit` | |
| `The_metapackage_does_not_depend_on_calls_signaling_redis_or_ef` | **the boundary test** |
| `Analyzers_are_packaged_under_analyzers_dotnet_cs` | Phase 17 |
| `The_templates_package_contains_no_assembly` | Phase 17 |
| `No_package_declares_a_dependency_on_a_test_only_package` | |
| `Installing_the_metapackage_into_a_clean_project_compiles_a_minimal_chat_app` | end-to-end packaging proof |
| `Package_validation_reports_no_breaking_change_against_0_16_0` | the `P18.T1` freeze |

- The last test is the one that catches everything else: create a temporary project, `dotnet add package`
  from the local artifacts folder, write ten lines of chat code, compile. If that fails, the packages are
  broken regardless of what the metadata says.

---

### P18.T7 — Release automation

**Deliverables**

```
.github/workflows/release.yml                                       (guard removed)
.github/workflows/ci.yml                                            (E2E gate made blocking)
scripts/release.ps1
docs/contributing/releasing.md
```

**Requirements**

- Remove the `if: false` guard Phase 00 put on the publish job. This is the moment it was created for.
- Release flow, tag-triggered on `v*`:

```
git tag v1.0.0
      ↓
full verify (build, test, format, analyzers, pack, JS, E2E, leak, chaos)
      ↓
pack .nupkg + .snupkg, build npm tarballs
      ↓
manual approval  (GitHub environment: nuget-release)
      ↓
push to NuGet (--skip-duplicate) and npm (--provenance)
      ↓
create the GitHub release with notes extracted from CHANGELOG.md
      ↓
publish the docs site for the release
```

- **The manual approval gate stays.** Automating the final push to a public registry removes the last
  chance to catch a mistake that cannot be undone. Spec §41 says not to publish automatically until the
  process is stable; one 1.0 release is not a track record.
- npm publish with `--provenance` and NuGet with a trusted publishing / OIDC flow if available, otherwise
  an API key from a repository secret scoped to this repository only. Never a personal token with broad
  scope.
- Version consistency: `scripts/release.ps1` must verify that the MinVer version from the tag, the
  `package.json` versions in `clients/js`, and the `CHANGELOG.md` top section all agree, and fail loudly
  if not. Publishing NuGet `1.0.0` alongside npm `0.16.0` is exactly the kind of mistake that survives
  code review.
- **Make the E2E gate blocking** in `ci.yml`. Phase 12 marked it non-blocking with a comment pointing here;
  honour it. If the suite is too flaky to block, that is a signal to fix the suite, not to keep the
  exemption — and if it genuinely cannot be stabilised, record that decision explicitly rather than
  leaving a stale comment.
- Add the leak-test and chaos-test jobs to the release workflow's verify stage. They do not need to gate
  every PR, but they must gate a release.
- `docs/contributing/releasing.md`: the human runbook. Pre-release checklist, how to cut the tag, what to
  do if the publish half-fails (NuGet succeeded, npm failed), how to yank/unlist, and how to publish a
  patch.

**Tests**

| Test | Asserts |
|---|---|
| `Release_script_fails_when_the_tag_and_changelog_disagree` | |
| `Release_script_fails_when_the_npm_and_nuget_versions_disagree` | |
| `Release_workflow_publish_job_requires_a_protected_environment` | workflow file assertion |
| `CI_workflow_e2e_job_is_not_marked_continue_on_error` | the promised flip |
| `Release_workflow_includes_the_leak_and_chaos_jobs` | |

Do a **dry run** to a local feed or a test package name before the real thing. A release process that has
never been executed is not a release process.

---

### P18.T8 — Cut 1.0.0

**Deliverables**

```
CHANGELOG.md                                                        (1.0.0 section)
DEVELOPMENT_LOG.md                                                  (final state)
ROADMAP.md                                                          (1.0 complete, post-1.0 planned)
git tag v1.0.0
```

> **Do not begin this task until every exit criterion below is met and `P18.T1` through `P18.T7` are
> complete.** Publishing is irreversible.

**Requirements**

- `CHANGELOG.md` gains a `## [1.0.0]` section. It is not a list of every commit — it is the summary a
  developer evaluating the library reads. Structure: what LiveSharp is, the complete feature list, the
  supported topologies, the API stability guarantee, breaking changes from `0.16.0`, and links to the
  migration guide and the docs site.
- `DEVELOPMENT_LOG.md` final entry: the full state, the total test count, every architectural decision that
  shaped the result, known issues carried into 1.0, and the post-1.0 next steps. This file was written so
  another developer could continue without context (Spec §28); at 1.0 it should read as a coherent history
  of eighteen phases, not a stack of session notes. Tidy it.
- Run the full gate one final time, on a clean clone, on both Windows and Linux. A `git clean -xdf`
  followed by `scripts/verify.ps1` is the real test — accumulated local state has hidden broken builds
  before.
- Tag, let the workflow run, approve the publish, verify the packages appear and are installable from the
  public registries, and verify the docs site is live.
- Post-release: create the `1.0` maintenance branch, open the `1.1` milestone, and move the post-1.0
  roadmap items into issues so the project has a visible next step rather than going quiet at its most
  visible moment.

---

## Dependency justification

| Package | Why | Alternative? | Licence | Notes |
|---|---|---|---|---|
| `Toxiproxy.Net` or Testcontainers' Toxiproxy module | Latency and partition injection for the chaos suite. | Hand-rolled TCP proxy (more code, less reliable). | MIT | Test-only. If it proves awkward, container pause/unpause covers most scenarios; drop the dependency and record why. |
| `Microsoft.DotNet.ApiCompat.Tool` | Only if `EnablePackageValidation` proves insufficient for the freeze. Check first — package validation is built in and probably enough. | — | MIT | **Verify necessity before adding.** |

No new runtime dependencies. Any shipped-package dependency added in this phase is a red flag: a hardening
phase that grows the dependency graph has not hardened anything.

---

## Documentation deltas

- `docs/versioning.md`, `docs/migration/README.md`, `docs/migration/0.16-to-1.0.md` — **new**.
- `docs/operations/README.md`, `deployment.md`, `kubernetes.md`, `azure.md`, `turn-deployment.md`,
  `monitoring.md`, `capacity-planning.md`, `resilience.md` — **new**.
- `docs/security/threat-model.md`, `docs/security/dependencies.md` — **new**.
- `docs/security/checklist.md`, `SECURITY.md` — revised for 1.0.
- `docs/api-surface-review.md` — **new**; the record of the freeze.
- `docs/contributing/releasing.md` — **new**.
- `docs/roadmap-post-1.0.md`, `ROADMAP.md` — post-1.0 planning.
- `README.md` — final form.
- Every page — honesty audit.

## Example deltas

- All six examples verified against `1.0.0` packages (not project references) at least once, so the
  published packages are proven to work in a consumer-shaped build.
- Every example README's "What this example does NOT do" section reviewed: several limitations from Phases
  04–08 no longer apply.

## CHANGELOG entry

```markdown
## [1.0.0]

LiveSharp 1.0 is a real-time communication library for ASP.NET Core covering chat, presence, and
WebRTC-based audio, video, and screen sharing.

### Features
- Direct and group messaging with delivery and read receipts, typing indicators, and message history.
- Presence with multi-device aggregation, explicit status, subscriptions, and heartbeat expiry.
- One-to-one audio and video calling with multi-device ringing, a fully specified twelve-state lifecycle,
  timeouts, and a reconnection grace period.
- Screen sharing as a second video track alongside the camera.
- WebRTC signalling with ephemeral TURN credentials; media never traverses the LiveSharp server.
- Authentication and layered authorization, per-operation rate limiting, and token-expiry enforcement.
- Optional Entity Framework Core persistence, tested against SQLite and PostgreSQL.
- Optional Redis scale-out with distributed registries, presence, calls, and rate limiting.
- Metrics and tracing through standard .NET diagnostics primitives, with no monitoring-platform
  dependency.
- .NET and TypeScript clients, React hooks, and `dotnet new` templates.

### API stability
The public API is now stable. Breaking changes require a major version. APIs marked experimental are
excluded; see `docs/versioning.md`.

### Breaking changes from 0.16.0
See `docs/migration/0.16-to-1.0.md`.

### Not included
Conference calling, server-side media, end-to-end encryption, Azure SignalR Service support, and mobile
SDKs. See `docs/roadmap-post-1.0.md`.
```

---

## Exit criteria

**API freeze**
- [ ] `docs/api-surface-review.md` records a decision for every public type.
- [ ] `PublicAPI.Unshipped.txt` is empty in every shipping project.
- [ ] `EnablePackageValidation=true` with baseline `0.16.0`, and validation passes.
- [ ] Every experimental API is attributed and listed in `docs/versioning.md`.
- [ ] `No_shipping_assembly_exposes_a_type_from_a_dependency_it_does_not_own` passes.

**Resilience**
- [ ] Every chaos scenario passes with an explicit convergence assertion.
- [ ] Graceful shutdown drains connections and sends `connection.closing`; `DrainTimeout` is configurable.
- [ ] The slow-reader behaviour is verified and documented, with the required configuration stated.
- [ ] `docs/operations/resilience.md` records what is handled and what is not.
- [ ] Chaos and leak jobs run in CI on `main`.

**Security**
- [ ] `docs/security/threat-model.md` exists, and every mitigation cites a passing test.
- [ ] `tests/LiveSharp.SecurityTests` maps one-to-one onto the threat model.
- [ ] The logging, metric-cardinality, and secret-redaction audits all pass.
- [ ] `No_shipped_package_has_a_known_vulnerability` passes.
- [ ] `No_shipped_package_has_a_licence_outside_the_approved_set` passes.
- [ ] Every checklist item is configurable and, where possible, surfaced by an `LS####` diagnostic.

**Documentation**
- [ ] `No_documentation_page_references_a_development_phase` passes.
- [ ] `No_documentation_page_contains_TODO_TBD_or_coming_soon` passes.
- [ ] `Every_feature_marked_available_has_a_docs_page_and_an_example` passes.
- [ ] Every number in `docs/operations/capacity-planning.md` traces to a Phase 16 measurement.
- [ ] `All_internal_documentation_links_resolve` passes.
- [ ] Every Phase 00 `TODO(maintainer)` placeholder is filled.

**Packaging**
- [ ] Every packaging test passes, including
      `Installing_the_metapackage_into_a_clean_project_compiles_a_minimal_chat_app`.
- [ ] `The_metapackage_does_not_depend_on_calls_signaling_redis_or_ef` passes.
- [ ] A real icon ships; the Phase 00 placeholder is gone.
- [ ] All six examples build against the `1.0.0` **packages**, not project references.

**Release**
- [ ] The publish guard is removed; the protected environment gate is in place.
- [ ] `CI_workflow_e2e_job_is_not_marked_continue_on_error` passes.
- [ ] Version-consistency checks fail correctly when versions disagree.
- [ ] A dry-run publish to a local feed succeeded.
- [ ] `docs/contributing/releasing.md` documents the half-failed-publish recovery path.

**Final**
- [ ] `git clean -xdf` then `scripts/verify.ps1` green on Windows **and** Linux.
- [ ] `CHANGELOG.md` has a complete `## [1.0.0]` section.
- [ ] `DEVELOPMENT_LOG.md` reads as a coherent history and records the final test count.
- [ ] `ROADMAP.md` shows 1.0 complete and a populated post-1.0 section.
- [ ] `v1.0.0` tagged; packages live on NuGet and npm and verified installable; docs site live.
- [ ] The `1.0` maintenance branch, the `1.1` milestone, and post-1.0 issues exist.

## Verify

```powershell
git clean -xdf
pwsh scripts/verify.ps1
dotnet test tests/LiveSharp.ChaosTests -c Release
dotnet test tests/LiveSharp.LeakTests -c Release
dotnet test tests/LiveSharp.SecurityTests -c Release
dotnet test tests/LiveSharp.PackagingTests -c Release
dotnet list package --vulnerable --include-transitive
pwsh scripts/release.ps1 -DryRun -Version 1.0.0
# only when everything above is green:
git tag v1.0.0; git push origin v1.0.0
```

## Next

Nothing. The plan ends here.

Post-1.0 work is tracked in `ROADMAP.md` and `docs/roadmap-post-1.0.md` as issues and milestones, not as
phases. The linear-phase model existed to get from an empty repository to a stable 1.0 without drifting;
after 1.0 the project is driven by user demand, and `AGENTS.md` should be updated to say so.
