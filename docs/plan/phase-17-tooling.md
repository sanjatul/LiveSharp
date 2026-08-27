# Phase 17 — Developer Tooling

| | |
|---|---|
| **Version at exit** | `0.16.0` (tag `v0.16.0`) |
| **Depends on** | Phase 16 |
| **Branch** | `feat/phase-17-tooling` |
| **Tasks** | `P17.T1` … `P17.T6` |
| **Read before starting** | Spec §48 (the ten-minute test), and every `docs/troubleshooting/README.md` entry accumulated so far |

---

## Goal

Make the library easy to *start* with and easy to *diagnose*. The feature surface is complete; this phase
is about the distance between "I found this on NuGet" and "I have working real-time chat", and the distance
between "it is broken" and "I know why".

Two measurable targets:

1. **Under five minutes** from `dotnet new install` to a running chat application on a clean machine.
2. **Under ten minutes** for a developer to understand any single feature from the docs (Spec §48).

The troubleshooting entries written over fifteen phases are the specification for this phase: each one is a
question the library made a developer ask, and most of them can be pre-empted by a better error message, a
startup validation, an analyzer, or a diagnostics page.

---

## Non-goals — DO NOT implement in this phase

- No new library features. Templates, analyzers, diagnostics, and docs only.
- No UI component library. `@livesharp/react` provides hooks, not components. A chat UI kit is a different
  product with a different maintenance burden and a much shorter shelf life.
- No admin dashboard as a product. The diagnostics page is a debugging aid, opt-in, authorized, and
  deliberately plain.
- No Vue, Svelte, Angular, or Solid adapters. React first because it is where the demand is; others can
  follow the same pattern in a later minor version, and a hastily-written adapter for a framework nobody
  on the project uses is worse than none.
- No mobile SDKs (MAUI, Xamarin, native iOS/Android). `LiveSharp.Client` already works in MAUI; a
  dedicated SDK is out of pre-1.0 scope.
- No CLI beyond `dotnet new` templates. A `livesharp` CLI has nothing to do.
- No AI-assisted anything.

---

## Tasks

### P17.T1 — `dotnet new` templates

**Deliverables**

```
src/LiveSharp.Templates/LiveSharp.Templates.csproj
src/LiveSharp.Templates/templates/livesharp-chat/…                  (ASP.NET Core + minimal JS)
src/LiveSharp.Templates/templates/livesharp-blazor/…                (Blazor Server)
src/LiveSharp.Templates/templates/livesharp-react/…                 (ASP.NET Core host + Vite React)
src/LiveSharp.Templates/README.md
tests/LiveSharp.Templates.Tests/TemplateSmokeTests.cs
docs/getting-started/templates.md
```

**Requirements**

- A template package (`PackageType=Template`, no assembly, `IncludeContentInPack`), installable with
  `dotnet new install LiveSharp.Templates`.
- Three templates, no more. Each must produce an application that **runs on first `dotnet run`** with no
  further configuration and no database.
- Template options, consistent across all three:

| Option | Values | Default |
|---|---|---|
| `--auth` | `cookie`, `jwt`, `none` | `cookie` |
| `--persistence` | `none`, `sqlite` | `none` |
| `--features` | `chat`, `chat-presence`, `chat-calls`, `all` | `chat-presence` |
| `--scale-out` | `none`, `redis` | `none` |

- Conditional content via template `#if` symbols. Every combination must compile — that is 3 × 3 × 2 × 4 ×
  2 = 144 combinations, which is too many to test exhaustively. Test a **matrix of 12 representative
  combinations** covering each option value at least twice, and document that the full cross-product is not
  verified. Honest partial coverage beats a claim of exhaustiveness.
- Generated code must be **exemplary**, because it becomes the starting point for real applications:
  - Real authentication (not the examples' hardcoded dropdown) when `--auth` is not `none`.
  - A stub `IRealtimeAuthorizationService` with a `TODO` and a link to
    `docs/security/authorization.md` — **not** the permissive default, so a generated app does not inherit
    Phase 09's startup error.
  - Rate limits configured explicitly with a comment pointing at the tuning guidance.
  - `EnableDetailedErrors` false, TLS assumed, `docs/security/checklist.md` linked in a `README.md`.
  - `--auth none` generates `AllowAnonymous = true` **with** a prominent comment and a `#warning` directive
    so the compiler itself reminds the developer. This is the right use of `#warning`.
- Generated `README.md` per template: what was generated, how to run, what to do next, and a "before
  production" checklist derived from `docs/security/checklist.md`.
- `dotnet new livesharp-chat` output must be small enough to read. If a template produces 30 files, it is
  teaching a project structure rather than a library.

**Tests**

| Test | Asserts |
|---|---|
| `Every_template_is_discoverable_after_install` | `dotnet new list` |
| `Twelve_representative_option_combinations_all_build` | matrix, real `dotnet new` + `dotnet build` |
| `Generated_chat_template_runs_and_serves_the_hub` | start the process, hit `/livesharp/negotiate` |
| `Generated_app_with_auth_none_emits_a_compiler_warning` | **required** |
| `Generated_app_registers_a_stub_authorization_service_not_the_permissive_default` | no Phase 09 startup error |
| `Generated_app_has_detailed_errors_disabled` | |
| `Generated_app_with_sqlite_includes_migration_instructions` | |
| `Generated_readme_links_the_security_checklist` | |
| `Template_package_contains_no_compiled_assembly` | packaging sanity |

---

### P17.T2 — Startup validation and better error messages

**Deliverables**

```
src/LiveSharp.Core/Diagnostics/LiveSharpStartupValidator.cs
src/LiveSharp.Core/Diagnostics/StartupDiagnostic.cs
src/LiveSharp.AspNetCore/Diagnostics/AspNetCoreStartupChecks.cs
tests/LiveSharp.Core.Tests/Diagnostics/StartupValidationTests.cs
docs/troubleshooting/startup-diagnostics.md
```

This is the highest-value task in the phase. Every entry in `docs/troubleshooting/README.md` that could
have been a startup error should become one.

**Public API (exact):**

```csharp
namespace LiveSharp.Diagnostics;

public enum StartupDiagnosticSeverity { Information = 0, Warning = 1, Error = 2 }

public sealed record StartupDiagnostic
{
    public required string Id { get; init; }              // e.g. "LS1001"
    public required StartupDiagnosticSeverity Severity { get; init; }
    public required string Message { get; init; }
    public required string Remedy { get; init; }
    public string? HelpLink { get; init; }
}

/// <summary>Contributes startup checks. Run once during host start.</summary>
public interface IStartupCheck
{
    IEnumerable<StartupDiagnostic> Check(IServiceProvider services);
}
```

**Requirements**

- A hosted service runs every `IStartupCheck` at startup, logs each diagnostic with its ID, message,
  remedy, and help link, and **fails the host** only on `Error`. Warnings are logged loudly and the
  application continues.
- Diagnostic IDs are stable and documented in `docs/troubleshooting/startup-diagnostics.md` with one
  section per ID. A developer should be able to search `LS1007` and find the answer. This is the pattern
  ASP.NET Core and Roslyn use and it works.
- Checks derived from the accumulated troubleshooting entries, at minimum:

| ID | Severity | Condition |
|---|---|---|
| `LS1001` | Error | `MapLiveSharp()` called without `AddSignalR()` (already exists — give it an ID) |
| `LS1002` | Error | `AddChat()` without `AddSignalR()` |
| `LS1003` | Error | `AddGroups()` / `AddTyping()` / `AddMessageState()` without `AddChat()` |
| `LS1004` | Error | `AddCalls()` without `AddSignaling()` |
| `LS1005` | Error | `AddGroupPresence()` without both `AddPresence()` and `AddGroups()` |
| `LS1006` | Error | A conflicting `IUserIdProvider` is registered (was a Phase 03 warning — promote it; it silently breaks user-targeted delivery, which is not a warning-level outcome) |
| `LS1007` | Error | `AllowAnonymous = false` and no authentication scheme is registered |
| `LS1008` | Error | TURN URLs configured without a secret or acknowledged static credentials |
| `LS1009` | Error | `PresenceTtl <= HeartbeatInterval` (was options validation — keep it there, surface it here too with a remedy) |
| `LS1010` | Warning | `AllowAllAuthorizationService` in use (was a Phase 09 startup error; keep the error log **and** give it an ID so it is searchable) |
| `LS1011` | Warning | `AllowAnonymous = true` outside Development |
| `LS1012` | Warning | `EnableDetailedErrors = true` outside Development |
| `LS1013` | Warning | `Diagnostics.AllowSensitivePayloadLogging = true` outside Development |
| `LS1014` | Warning | `AddRedis()` registered but the SignalR backplane disabled |
| `LS1015` | Warning | Redis registered on a single instance with no backplane benefit — no, this is unknowable. **Drop it.** |
| `LS1016` | Warning | History operations registered but no `IMessageHistoryStore` implementation |
| `LS1017` | Warning | `AddMessageState()` without a store, so receipts are lost on restart |
| `LS1018` | Warning | `MessageRetention` configured — restate the destructive implication (Phase 14 already logs this; give it an ID) |
| `LS1019` | Information | Summary of registered features, stores, and node identity |
| `LS1020` | Warning | Reachable `turn:` URL with no `turns:` alternative |

- `LS1019` is the one developers will thank you for: a single startup log line listing what LiveSharp
  actually enabled. "Is chat registered?" should never require a debugger.
- **Consolidate, do not duplicate.** Several of these already exist as `IValidateOptions` failures or
  scattered startup logs from Phases 03–15. Route them all through this mechanism so there is one format,
  one ID scheme, and one documentation page. That refactor is most of the work in this task.
- Every `Error` message must state the remedy in code: not "AddSignalR was not called" but
  `"LS1001: MapLiveSharp() requires the SignalR transport. Add: builder.Services.AddLiveSharp().AddSignalR();  See https://…/LS1001"`.

**Tests**

| Test | Asserts |
|---|---|
| `Every_diagnostic_id_is_unique` | reflection |
| `Every_diagnostic_id_has_a_documentation_section` | scan `startup-diagnostics.md` |
| `Every_diagnostic_has_a_non_empty_remedy` | |
| `An_error_diagnostic_fails_host_start` | |
| `A_warning_diagnostic_does_not_fail_host_start` | |
| `Each_of_the_error_conditions_produces_its_diagnostic` | one test per ID, table-driven |
| `Each_of_the_warning_conditions_produces_its_diagnostic` | one test per ID |
| `A_correctly_configured_application_produces_only_the_summary_diagnostic` | **no noise on a good setup** |
| `The_summary_diagnostic_lists_every_registered_feature` | |
| `Conflicting_user_id_provider_is_now_an_error_not_a_warning` | the deliberate promotion |
| `Existing_options_validation_failures_are_surfaced_with_ids` | the consolidation |

---

### P17.T3 — Roslyn analyzers

**Deliverables**

```
src/LiveSharp.Analyzers/LiveSharp.Analyzers.csproj
src/LiveSharp.Analyzers/*.cs
src/LiveSharp.Analyzers/AnalyzerReleases.{Shipped,Unshipped}.md
src/LiveSharp.CodeFixes/…
tests/LiveSharp.Analyzers.Tests/…
docs/troubleshooting/analyzers.md
```

**Requirements**

- Shipped **inside** `LiveSharp.Core`'s package as an analyzer asset (`analyzers/dotnet/cs`), not as a
  separate package a developer must remember to install. An analyzer nobody installs prevents nothing.
- Analyzer count discipline: ship **five or six**, each preventing a mistake that actually happened during
  development or appears in the troubleshooting docs. A library that emits twenty diagnostics gets
  `NoWarn`'d wholesale.

| ID | Severity | Detects | Code fix |
|---|---|---|---|
| `LSA001` | Warning | Argument transposition risk: `SendDirectAsync`/`CanSendToUserAsync` called with two positional `string` arguments where the variable names suggest they are swapped (e.g. `Send(recipient, sender)` where parameters are `senderId, recipientId`). Heuristic on identifier names. | Add argument names |
| `LSA002` | Warning | `IDisposable` returned by `ILiveSharpClient.On<T>()` discarded — the subscription leak from Phase 03 | Assign to a variable / add to a `CompositeDisposable` |
| `LSA003` | Warning | `IRealtimeAuthorizationService` never implemented in a project that references `LiveSharp.Chat` — the Phase 09 permissive default, caught at compile time rather than startup | none (informational) |
| `LSA004` | Warning | `DateTime.UtcNow` / `DateTimeOffset.UtcNow` in a type that has a `TimeProvider` available via constructor injection — the ADR-008 rule, extended to consumer code where LiveSharp abstractions are in play | Replace with `timeProvider.GetUtcNow()` |
| `LSA005` | Error | An `IOperationHandler` implementation whose `Operation` name does not match the naming convention, or that is not registered via `AddOperationHandler` | none |
| `LSA006` | Warning | A `[LoggerMessage]` template in consumer code interpolating a LiveSharp `ChatMessage.Content`, `SessionDescription.Sdp`, or `IceCandidate.Candidate` | none |

- `LSA001` is a heuristic and heuristics produce false positives. Default severity `Warning`, documented as
  suppressible, and the detection must require *both* a positional call *and* name evidence — never fire on
  named arguments. If it cannot be made precise enough in a session, **ship the other five and drop it**,
  recording why. Better to ship five reliable analyzers than six where one cries wolf.
- `AnalyzerReleases.Shipped.md` / `Unshipped.md` are required by the analyzer release-tracking analyzer;
  maintain them.
- Every analyzer needs a documentation section in `docs/troubleshooting/analyzers.md` with the rationale,
  an example of the violation, the fix, and how to suppress it. An analyzer without documentation is an
  obstacle.
- Test with `Microsoft.CodeAnalysis.CSharp.Analyzer.Testing` (or the `Testing` verifier packages) — real
  compilation-based tests, positive and negative, plus code-fix verification.

**Tests**

| Test | Asserts |
|---|---|
| `Each_analyzer_reports_on_a_violating_snippet` | one per ID |
| `Each_analyzer_reports_nothing_on_a_compliant_snippet` | one per ID |
| `LSA001_does_not_fire_on_named_arguments` | **false-positive guard** |
| `LSA001_does_not_fire_when_names_give_no_evidence` | |
| `LSA002_does_not_fire_when_the_subscription_is_assigned` | |
| `LSA004_does_not_fire_where_no_time_provider_is_available` | |
| `Each_code_fix_produces_compiling_code` | |
| `Every_analyzer_id_has_a_documentation_section` | scan |
| `Analyzer_release_tracking_files_are_up_to_date` | the tracking analyzer itself |
| `Analyzers_ship_in_the_core_package_analyzers_folder` | packaging test on the `.nupkg` |

---

### P17.T4 — Diagnostics endpoint

**Deliverables**

```
src/LiveSharp.AspNetCore/Diagnostics/LiveSharpDiagnosticsEndpoint.cs
src/LiveSharp.AspNetCore/Diagnostics/DiagnosticsSnapshot.cs
src/LiveSharp.AspNetCore/Diagnostics/DiagnosticsOptions.cs
tests/LiveSharp.AspNetCore.Tests/Diagnostics/DiagnosticsEndpointTests.cs
docs/troubleshooting/diagnostics-endpoint.md
```

**Public API (exact):**

```csharp
namespace LiveSharp;

public static class LiveSharpDiagnosticsEndpointExtensions
{
    /// <summary>Maps a diagnostics endpoint. Requires authorization and is not mapped by default.</summary>
    public static IEndpointConventionBuilder MapLiveSharpDiagnostics(this IEndpointRouteBuilder endpoints, string path = "/livesharp/_diagnostics");
}
```

> ## Opt-in, authorized, and boring
> A diagnostics endpoint that lists connections, users, groups, and calls is a complete map of who is
> talking to whom. It must never be mapped by default, must require authorization, must have no write
> operations, and must redact user identifiers unless explicitly configured otherwise.

**Requirements**

- **Not mapped by default.** `MapLiveSharpDiagnostics()` is an explicit call. It applies
  `RequireAuthorization(DiagnosticsOptions.Policy)` and **throws at startup** if no policy is configured
  and the environment is not Development — a diagnostics endpoint reachable by any authenticated user in
  production is a data breach, and defaulting to "any authenticated user" would guarantee someone ships
  it.
- `DiagnosticsOptions`: `Policy` (required outside Development), `IncludeUserIdentifiers` (default
  **false** — counts and aggregates only), `IncludeConnectionIdentifiers` (default false).
- Returns JSON by default and a plain HTML page when the `Accept` header prefers it. The HTML page is
  server-rendered, dependency-free, and under 200 lines — no framework, no build step, no CDN reference.
  A diagnostics page that cannot render offline in a locked-down network is useless.
- Snapshot content:

| Section | Content |
|---|---|
| Build | LiveSharp version, protocol version, target framework, node identity |
| Configuration | every option's effective value, with secrets redacted (`TurnSharedSecret`, Redis connection string) |
| Registration | which features, stores, filters, and handlers are registered — the `LS1019` summary, in detail |
| Startup diagnostics | the `IStartupCheck` results from `P17.T2` |
| Connections | total, per-node, per-transport counts. Identifiers only when enabled |
| Groups | count, size distribution (p50/p95/max). Names only when identifiers are enabled |
| Presence | status distribution |
| Calls | active count, state distribution |
| Signalling | active session count, state distribution |
| Cluster | known nodes and their last heartbeat, when Redis is registered |
| Health | Redis reachable, database reachable, backplane connected |

- Secret redaction must be **allow-list based on type**, not a deny-list of property names. A deny-list
  misses the next secret someone adds. Redact anything whose option property is annotated
  `[SensitiveConfiguration]` (a new attribute) and apply it to every secret-bearing option retroactively.
  Add a test that every option property whose name contains `secret`, `password`, `key`, `token`, or
  `connectionstring` carries the attribute — belt and braces.
- The endpoint must be **cheap**. It must not enumerate 100 000 connections to produce a count, must use
  the Phase 16 metrics and the registries' count methods, and must have a bounded response size. A
  diagnostics endpoint that can be used to DoS the server is a liability; rate-limit it.
- Also provide `IHealthCheck` implementations (`livesharp`, `livesharp-redis`, `livesharp-database`) for
  `Microsoft.Extensions.Diagnostics.HealthChecks` — in the shared framework, no new dependency.

**Tests**

| Test | Asserts |
|---|---|
| `Diagnostics_endpoint_is_not_mapped_by_default` | 404 |
| `Mapping_without_a_policy_outside_development_throws_at_startup` | **required** |
| `Mapping_without_a_policy_in_development_is_allowed_with_a_warning` | |
| `Unauthorized_request_is_rejected` | |
| `Snapshot_omits_user_identifiers_by_default` | **security test** |
| `Snapshot_includes_user_identifiers_when_explicitly_enabled` | |
| `Snapshot_redacts_the_turn_shared_secret` | **security test** |
| `Snapshot_redacts_the_redis_connection_string` | **security test** |
| `Every_sensitive_option_property_carries_the_sensitive_attribute` | the belt-and-braces scan |
| `Snapshot_reports_every_registered_feature` | |
| `Snapshot_includes_startup_diagnostics` | |
| `Html_response_contains_no_external_resource_references` | offline-capable |
| `Snapshot_does_not_enumerate_connections_to_produce_counts` | command/query counting |
| `Snapshot_response_size_is_bounded` | 10 000 connections → small response |
| `Diagnostics_endpoint_is_rate_limited` | |
| `Health_checks_report_unhealthy_when_redis_is_down` | Testcontainers |

---

### P17.T5 — `@livesharp/react` and TypeScript generation

**Deliverables**

```
clients/js/packages/react/package.json                              (@livesharp/react)
clients/js/packages/react/src/*.tsx
clients/js/packages/react/src/__tests__/*.test.tsx
clients/js/packages/react/README.md
tools/LiveSharp.ProtocolGen/…                                       (constants + types generator)
clients/js/packages/client/src/protocol.generated.ts
```

**Public API (exact):**

```tsx
export function LiveSharpProvider(props: {
  client: LiveSharpClient;
  children: React.ReactNode;
}): JSX.Element;

export function useLiveSharp(): LiveSharpClient;
export function useConnectionState(): ConnectionState;

export function useLiveSharpEvent<TPayload>(event: string, handler: (payload: TPayload) => void): void;

export function useMessages(conversationId: string, options?: { pageSize?: number }): {
  messages: ChatMessage[];
  isLoading: boolean;
  error: Error | undefined;
  hasMore: boolean;
  loadMore: () => Promise<void>;
  send: (content: string) => Promise<void>;
};

export function usePresence(userIds: string[]): Record<string, UserPresence>;
export function useTyping(conversationId: string): { typingUsers: string[]; onKeystroke: () => void };
export function useCall(): {
  incoming: CallInfo | undefined;
  active: CallInfo | undefined;
  invite: (calleeId: string) => Promise<void>;
  accept: () => Promise<void>;
  reject: () => Promise<void>;
  hangUp: () => Promise<void>;
  localStream: MediaStream | undefined;
  remoteStream: MediaStream | undefined;
};
```

**Requirements**

- The hooks are extracted from the **hand-written hooks already in `examples/React`** (Phases 06–13). That
  ordering was deliberate: the package's shape is now informed by three phases of real use rather than
  guessed. Note in the README which example each hook came from.
- `react` as a **peer dependency** (`>=18`), never a direct dependency. Two React copies in one bundle is
  a well-known, hard-to-diagnose breakage.
- Strict-mode safe: every hook must survive React 18/19 double-invocation of effects. The single most
  common bug in a real-time React hook is opening two connections in strict mode. Test with
  `<StrictMode>` explicitly.
- Every subscription cleaned up in the effect's return. `useLiveSharpEvent` must use a ref for the handler
  so a re-rendering component does not resubscribe on every render — the second most common bug, and the
  one that produces exponential handler growth.
- `useMessages` implements the Phase 14 cursor pagination and **deduplicates** messages arriving both from
  the live event and from a history page. Dedupe on message ID. Without this, scrolling up shows
  duplicates, which is the third most common bug.
- `useTyping`'s `onKeystroke` wraps the Phase 07 `createTypingSignaller`, so client-side throttling is
  automatic.
- `useCall` deliberately exposes `MediaStream`s and not `<video>` elements. Attaching a stream to an
  element is a two-line `useEffect` the application owns; a `<Video>` component would be the start of a UI
  kit.
- SSR safe: importing the package in a Next.js server component must not crash. `useCall` touches
  `RTCPeerConnection` only inside effects, which never run server-side. Test it.
- Version locked to the .NET packages, same as `@livesharp/client` (ADR-010).

**Requirements — protocol generation**

The `protocol.ts` contract test has been maintained by hand across thirteen phases. Replace it:

- `tools/LiveSharp.ProtocolGen`: a small console tool that reflects over `RealtimeOperations`,
  `RealtimeEvents`, `RealtimeErrorCode`, and every protocol DTO, and emits
  `clients/js/packages/client/src/protocol.generated.ts` containing the constants and TypeScript
  interfaces for every DTO.
- Run as a build step in `scripts/verify.ps1`, with the generated file **committed** (so the npm package
  builds without .NET installed) and CI asserting it is up to date — the same pattern as
  `PublicAPI.Unshipped.txt`.
- This replaces the hand-maintained constants and the `artifacts/protocol-names.json` contract test with
  actual generation. Keep a much simpler test: the generated file is current.
- Type generation must handle nullability, enums (as string unions **and** the numeric values, since the
  wire carries numbers), `IReadOnlyDictionary<string,string>` → `Record<string,string>`, and
  `DateTimeOffset` → `string`. Emit a header comment saying the file is generated and must not be edited.

**Tests**

| Test | Asserts |
|---|---|
| `provider supplies the client to consumers` | |
| `useLiveSharp throws a clear error outside a provider` | |
| `useConnectionState reflects state changes` | |
| `useLiveSharpEvent does not resubscribe on re-render` | **the handler-ref test** |
| `useLiveSharpEvent unsubscribes on unmount` | |
| `every hook survives strict mode double invocation` | **required**, one test per hook |
| `useMessages deduplicates a live message that is also in a history page` | **required** |
| `useMessages loadMore appends older messages` | |
| `useMessages send optimistically appends and reconciles` | |
| `usePresence subscribes and unsubscribes as the id list changes` | |
| `useTyping throttles keystrokes` | |
| `useCall exposes streams and cleans them up on hang up` | |
| `package imports cleanly in a node environment` | SSR safety |
| `react is a peer dependency not a dependency` | package.json assertion |
| `generated protocol file is up to date` | CI gate |
| `generated protocol file contains every operation and event constant` | |
| `generated types cover every protocol dto` | |

---

### P17.T6 — Documentation site and the five-minute test

**Deliverables**

```
docfx.json
docs/toc.yml
docs/index.md
docs/api/                                                           (generated)
.github/workflows/docs.yml
.config/dotnet-tools.json                                           (docfx added)
docs/getting-started/five-minute-chat.md
docs/faq.md
```

**Requirements — the site**

- DocFX, chosen over Docusaurus/Starlight because it generates .NET API reference from the XML docs
  already required by Phase 01's architecture test. Maintaining a second stack to render Markdown that
  DocFX renders adequately is not worth the polish. Record the decision and its downside honestly: the
  DocFX default theme is plainer than a modern docs site, and the TypeScript API is not auto-generated
  (link to the hand-written client READMEs instead, or add TypeDoc later).
- `docs/toc.yml` organises everything written across seventeen phases into a coherent navigation:
  Getting Started → Guides (chat, presence, typing, receipts, groups, calls, WebRTC) → Configuration →
  Security → Scaling → Performance → Observability → Troubleshooting → Migration → API Reference →
  Architecture (including every ADR) → Contributing.
- **Every existing Markdown page must appear in the ToC.** Add a test that scans `docs/**/*.md` and fails
  on any file not referenced — seventeen phases of documentation will otherwise contain orphans nobody can
  find.
- `.github/workflows/docs.yml`: build the site on push to `main`, publish to GitHub Pages. Also run the
  ToC-coverage check and a **link checker** on every PR that touches `docs/**`. Broken internal links
  across a hundred pages are inevitable without automation.
- **Code samples must compile.** Phase 03 required snippets to be extracted from compiling tests; audit
  every snippet added since and enforce it with a test project (`docs/samples/` with region-extracted
  snippets, or DocFX's own snippet-include syntax pointing at real `.cs` files). A docs site full of
  samples that no longer compile is the most common way documentation rots.

**Requirements — the five-minute test**

`docs/getting-started/five-minute-chat.md` is the front door and it must be verifiably fast:

1. `dotnet new install LiveSharp.Templates`
2. `dotnet new livesharp-chat -o MyChat`
3. `cd MyChat && dotnet run`
4. Open two browsers, chat.

Then a written, timed walkthrough of what the template generated and why. **Actually time it on a clean
machine** (a fresh container) and record the measured time in the document. If it is over five minutes,
fix the templates, not the document.

Add a CI job that runs steps 1–3 in a clean container and asserts the app responds — the five-minute claim
becomes a test rather than a promise.

**Requirements — FAQ**

`docs/faq.md` written from real questions the design raises, with honest answers:

- Why not just use SignalR directly? (You can; LiveSharp adds messaging semantics, presence, calling, and
  the store abstractions. If you only need one hub method, use SignalR.)
- Does the server relay media? (No.)
- Do I need TURN? (Yes, for a meaningful fraction of users on restrictive networks.)
- Can I do group calls? (Not before 1.0; it needs an SFU.)
- Do I need a database? (No.)
- Do I need Redis? (Only for more than one server.)
- Is chat content encrypted end to end? (No. TLS in transit; the server sees content.)
- Why is the package called `LiveSharp.Signaling` and not `LiveSharp.WebRTC`?
- Why is `Delivered` client-acknowledged?
- Why are typing indicators not distributed?
- Can I use Azure SignalR Service? (Not supported; explain the connection-model difference.)
- What is the .NET version policy? (`net10.0` only pre-1.0; ADR-001.)
- Is it production ready? (Answer honestly for `0.16.0`: the API is not frozen until 1.0.)

---

## Dependency justification

| Package | Why | Alternative? | Licence | Notes |
|---|---|---|---|---|
| `docfx` (dotnet tool) | Generates API reference from the XML docs the project already requires, and renders the Markdown guides. | Docusaurus/Starlight render Markdown better but cannot generate .NET API reference, so both stacks would be needed. | MIT | Local tool in `.config/dotnet-tools.json`. Not a package dependency. |
| `Microsoft.CodeAnalysis.CSharp.Workspaces` | Analyzers and code fixes. | None. | MIT | `LiveSharp.Analyzers` only, `PrivateAssets=all`, shipped as an analyzer asset — never a runtime dependency of consumer projects. |
| `Microsoft.CodeAnalysis.CSharp.Analyzer.Testing` / `.CodeFix.Testing` | Compilation-based analyzer tests. | Hand-rolled `CSharpCompilation` harness (more code, same result). | MIT | Test-only. |
| `@testing-library/react`, `@testing-library/jest-dom`, `jsdom` | Testing hooks. | Testing React hooks without a testing library is not viable. | MIT | Dev-only in `clients/js`. |
| `react` (peer) | `@livesharp/react`. | — | MIT | **Peer**, never direct. |

Explicitly rejected: any UI component library, Storybook (no components to document), TypeDoc for now
(the client READMEs are adequate and adding a second doc generator has its own upkeep — revisit at 1.0),
`swagger`/OpenAPI generation for the hub (the protocol is not HTTP; AsyncAPI would be the right format and
is not worth the tooling before there is demand).

---

## Documentation deltas

- `docs/getting-started/five-minute-chat.md`, `docs/getting-started/templates.md` — **new**.
- `docs/faq.md` — **new**.
- `docs/troubleshooting/startup-diagnostics.md` — **new**, one section per diagnostic ID.
- `docs/troubleshooting/analyzers.md` — **new**, one section per analyzer ID.
- `docs/troubleshooting/diagnostics-endpoint.md` — **new**, including the security warnings.
- `docs/toc.yml`, `docs/index.md`, `docfx.json` — **new**; the site.
- `docs/troubleshooting/README.md` — restructure as an index pointing at the diagnostic IDs, since most
  entries now have a `LS####` code.
- `clients/js/packages/react/README.md` — **new**, the npm README.
- Every guide page — audit against the ten-minute test (Spec §48) and fix the ones that fail. Record which
  pages were revised.
- `README.md` — replace the quick start with the template flow, link the docs site, add the FAQ link.

## Example deltas

- `examples/React` — refactored to consume `@livesharp/react` instead of its hand-written hooks, which
  proves the package is actually sufficient. Keep a short comment noting the hooks moved into the package.
- `examples/NextJs` — same.
- `examples/README.md` — note that examples now use the published hooks package.

## CHANGELOG entry

```markdown
### Added
- `LiveSharp.Templates`: `dotnet new livesharp-chat`, `livesharp-blazor`, and `livesharp-react` templates
  with authentication, persistence, feature, and scale-out options.
- Startup diagnostics with stable `LS####` identifiers, actionable remedies, and documentation per
  identifier; misconfiguration now fails or warns at startup instead of at runtime.
- Roslyn analyzers shipped inside `LiveSharp.Core`, catching discarded subscriptions, missing
  authorization implementations, and time-provider violations, with code fixes.
- Opt-in, authorization-required diagnostics endpoint and health checks.
- `@livesharp/react`: hooks for connection state, messages with pagination, presence, typing, and calling,
  extracted from the example applications.
- Generated TypeScript protocol constants and types, replacing hand-maintained mirrors.
- DocFX documentation site published to GitHub Pages, with a link checker and a table-of-contents coverage
  check.
- FAQ answering the questions the architecture raises.

### Changed
- A conflicting `IUserIdProvider` registration is now a startup error rather than a warning, because it
  silently breaks user-targeted delivery.
- Existing options-validation failures and scattered startup warnings are consolidated behind the
  `LS####` diagnostic scheme.
```

---

## Exit criteria

- [ ] `dotnet new install LiveSharp.Templates && dotnet new livesharp-chat && dotnet run` produces a
      working chat application on a clean machine, and the measured time is recorded in
      `docs/getting-started/five-minute-chat.md`.
- [ ] A CI job performs that install-and-run in a clean container and asserts the app responds.
- [ ] `Twelve_representative_option_combinations_all_build` passes.
- [ ] `Generated_app_with_auth_none_emits_a_compiler_warning` passes.
- [ ] `Generated_app_registers_a_stub_authorization_service_not_the_permissive_default` passes.
- [ ] `A_correctly_configured_application_produces_only_the_summary_diagnostic` passes — no noise on a
      good setup.
- [ ] `Every_diagnostic_id_has_a_documentation_section` passes.
- [ ] Every previously-scattered startup warning or options failure now has an `LS####` ID.
- [ ] Five or six analyzers ship inside `LiveSharp.Core`'s `analyzers/` folder, proven by a package test.
- [ ] `LSA001_does_not_fire_on_named_arguments` passes, or `LSA001` was dropped with the reason recorded.
- [ ] `Every_analyzer_id_has_a_documentation_section` passes.
- [ ] `Diagnostics_endpoint_is_not_mapped_by_default` and
      `Mapping_without_a_policy_outside_development_throws_at_startup` pass.
- [ ] `Snapshot_omits_user_identifiers_by_default` and both secret-redaction tests pass.
- [ ] `Html_response_contains_no_external_resource_references` passes.
- [ ] `every hook survives strict mode double invocation` passes for every hook.
- [ ] `useLiveSharpEvent does not resubscribe on re-render` passes.
- [ ] `useMessages deduplicates a live message that is also in a history page` passes.
- [ ] `react is a peer dependency not a dependency` passes.
- [ ] `generated protocol file is up to date` runs in CI; the hand-maintained `protocol.ts` constants and
      `artifacts/protocol-names.json` contract test are removed.
- [ ] The docs site builds and publishes; the link checker and ToC-coverage check pass.
- [ ] Every `docs/**/*.md` file appears in `docs/toc.yml`.
- [ ] Every documentation code sample compiles, enforced by a test.
- [ ] `examples/React` and `examples/NextJs` consume `@livesharp/react`.
- [ ] `docs/faq.md` answers "is it production ready" honestly for `0.16.0`.
- [ ] `scripts/verify.ps1` green; tag `v0.16.0`.

## Verify

```powershell
pwsh scripts/verify.ps1
dotnet test tests/LiveSharp.Templates.Tests -c Release
dotnet test tests/LiveSharp.Analyzers.Tests -c Release
dotnet tool restore; dotnet docfx docfx.json --serve
pnpm --dir clients/js -r test
# clean-machine check, ideally in a container:
dotnet new install ./artifacts/packages/LiveSharp.Templates.0.16.0.nupkg
dotnet new livesharp-chat -o $env:TEMP/LiveSharpFiveMinute; dotnet run --project $env:TEMP/LiveSharpFiveMinute
git tag v0.16.0
```

## Next

`docs/plan/phase-18-hardening.md`, task `P18.T1`.

> Phase 18 is the last phase. It freezes the public API, runs the resilience and chaos suites, completes the
> security review, and cuts `1.0.0`. Read the whole file before starting: several of its tasks are
> irreversible.
