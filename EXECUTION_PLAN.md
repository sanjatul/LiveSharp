# LiveSharp — Execution Plan

> **This file is the entry point for every AI coding session in this repository.**
>
> Derived from `Open-Source .NET Real-Time Communication Library — Master AI Development Instruction.md`
> (referred to below as **the Spec**). The Spec is the *why*. This plan is the *what, in what order, with
> what exit criteria*. Where the two conflict, the Spec wins on principles and this plan wins on
> sequencing and concrete naming — except for the five deliberate deviations recorded in
> [§4 Deviations from the Spec](#4-deviations-from-the-spec), which override the Spec by explicit
> maintainer approval.

---

## 0. How to use this plan

The work is decomposed into **19 phases** (Phase 0 … Phase 18) and, inside each phase, into
**tasks** identified as `P<phase>.T<task>` — for example `P03.T2`.

**One task is intended to be one AI session.** A task is sized so that a single session can
complete implementation + tests + docs + changelog and finish with a green
`scripts/verify.ps1`.

```
EXECUTION_PLAN.md            <- you are here: global rules, protocol, phase index
docs/plan/task-index.md      <- flat, ordered checklist of every task
docs/plan/phase-NN-*.md      <- the session brief for one phase
```

### Loading rules (context hygiene)

At the start of a session read **only**:

1. `AGENTS.md`
2. `DEVELOPMENT_LOG.md`
3. `ROADMAP.md`
4. `EXECUTION_PLAN.md` (this file — §5–§9 in particular)
5. `docs/plan/phase-NN-*.md` for **the current phase only**

**Do not read later phase files.** They exist to prevent scope creep, not to inspire it. Reading
ahead is the single most common way this project will go off the rails. If you believe a later
phase must be pulled forward, follow [§8 Escalation](#8-escalation-when-the-plan-is-wrong).

---

## 1. Project identity

| Item | Value |
|---|---|
| Product name | **LiveSharp** |
| Repository | `git@github.com-personal:sanjatul/LiveSharp.git` (`sanjatul/LiveSharp`) |
| License | MIT |
| Target framework | `net10.0` **only** |
| SDK | pinned in `global.json` to `10.0.302`, `rollForward: latestFeature` |
| Language | C# `latest`, nullable enabled, implicit usings enabled |
| NuGet prefix | `LiveSharp.*` |
| npm scope | `@livesharp/*` |
| Versioning | SemVer 2.0 via MinVer (git tags are the source of truth) |
| Test stack | xUnit v3 + Shouldly + NSubstitute + FakeTimeProvider |
| Docs site | DocFX → GitHub Pages |

---

## 2. Architecture decision record index

Phase 0 creates these files under `docs/architecture/decisions/`. They are **already decided** —
Phase 0's job is to write them down with full Context / Decision / Alternatives / Consequences,
not to re-open them.

| ADR | Title | Decision |
|---|---|---|
| `ADR-000` | Project name and scope | `LiveSharp`; a .NET real-time communication library, not a hosted service |
| `ADR-001` | Target framework policy | `net10.0` only pre-1.0; no multi-targeting without a new ADR |
| `ADR-002` | License | MIT — compatible with SignalR (MIT/Apache-2.0) and SIPSorcery (BSD-3-Clause) |
| `ADR-003` | Testing strategy | xUnit v3 + Shouldly + NSubstitute; Shouldly chosen over FluentAssertions because FluentAssertions v8+ requires a paid licence for commercial use, which is hostile to an OSS consumer base |
| `ADR-004` | Package boundaries | See [§3](#3-package-graph). No separate `LiveSharp.SignalR` package |
| `ADR-005` | Wire protocol | Single hub + operation routing. See [§6](#6-wire-protocol-normative) |
| `ADR-006` | WebRTC strategy | Server relays signalling only; media is peer-to-peer. Server-side media (SIPSorcery) is a separate, experimental, post-1.0 package |
| `ADR-007` | Error model | Exceptions for programmer/configuration errors; `OperationResult` for expected operation failures |
| `ADR-008` | Time abstraction | `TimeProvider` injected everywhere; `DateTime.UtcNow`/`DateTimeOffset.UtcNow`/`Stopwatch` banned in `src/` |
| `ADR-009` | Identity representation | `string` user IDs pre-1.0; no strongly-typed ID structs |
| `ADR-010` | Repository layout | Examples in-repo with their own solution; clients in `clients/` |
| `ADR-011` | Public API governance | `Microsoft.CodeAnalysis.PublicApiAnalyzers` + `PublicAPI.{Shipped,Unshipped}.txt` per shipping project |

Later phases add ADRs as their decisions arrive. Written at plan time, expected:

| ADR | Added by | Decision |
|---|---|---|
| `ADR-012` | `P06.T1` | `LiveSharp.Presence` is its own package, and group-derived presence subscriptions live in a `LiveSharp.Chat.Presence` bridge rather than coupling presence to chat |

Any further ADR follows the escalation process in [§8](#8-escalation-when-the-plan-is-wrong). Two
existing ADRs are amended by later phases: `ADR-004` by `P16.T4` (the MessagePack package-reference
decision) and by `P18.T1`/`P18.T6` (the `LiveSharp.Testing` split, if taken).

---

## 3. Package graph

```
LiveSharp.Abstractions                              [P01]
    contracts only. Deps: Microsoft.Extensions.{Logging,DependencyInjection,Options}.Abstractions
    MUST NOT reference ASP.NET Core, SignalR, EF Core, Redis, or SIPSorcery.
        |
LiveSharp.Core                                      [P01]
    in-memory implementations, options + validation, operation router, DI entry point.
    MUST NOT reference ASP.NET Core. Fully unit-testable with no web host.
    Also ships the Roslyn analyzers as an analyzer asset.            [P17]
        |
LiveSharp.AspNetCore                                [P03]
    FrameworkReference Microsoft.AspNetCore.App.
    LiveSharpHub, SignalRRealtimeTransport, MapLiveSharp(), user-id provider, auth glue.
        |
        +-- LiveSharp.Chat                  messaging, groups, typing, receipts, history   [P04]
        +-- LiveSharp.Presence              presence; does NOT depend on Chat              [P06]
        |       +-- LiveSharp.Chat.Presence bridge: group-derived presence subscriptions   [P06]
        +-- LiveSharp.Signaling             SDP/ICE relay, ICE config, ephemeral TURN      [P10]
        |       +-- LiveSharp.Calls         call lifecycle state machine                   [P11]
        +-- LiveSharp.EntityFrameworkCore   optional persistence provider                  [P14]
        +-- LiveSharp.Redis                 backplane + distributed registries/stores      [P15]

LiveSharp                          metapackage: Core + AspNetCore + Chat                  [P18]
LiveSharp.Templates                dotnet new templates                                   [P17]
LiveSharp.Testing                  in-memory stores + test helpers, if P18.T1 splits them [P18]

clients/dotnet/LiveSharp.Client    SignalR.Client wrapper (Blazor WASM, MAUI, console)     [P03]
clients/js/packages/client         @livesharp/client   (framework-agnostic TypeScript)     [P03]
clients/js/packages/react          @livesharp/react    (hooks adapter)                     [P17]

LiveSharp.Media.SIPSorcery         EXPERIMENTAL, post-1.0, NOT in the 1.0 scope
```

`docs/plan/task-index.md` holds the authoritative package-delivery order.

### Dependency direction rules (enforced by tests, not just convention)

1. `LiveSharp.Abstractions` depends on nothing but `Microsoft.Extensions.*.Abstractions`.
2. `LiveSharp.Core` does **not** reference `Microsoft.AspNetCore.*`.
3. Feature packages (`Chat`, `Presence`, `Signaling`, `Calls`) depend on `Abstractions` + `Core` +
   `AspNetCore`. They never depend on each other except `Calls → Signaling`. `Presence` must **not**
   reference `Chat`; the `Chat.Presence` bridge is the only package that references both.
4. No package references `LiveSharp` (the metapackage).
5. Nothing in `src/` references anything in `clients/` or `examples/`.
6. Only `LiveSharp.EntityFrameworkCore` references EF Core. Only `LiveSharp.Redis` references
   `StackExchange.Redis`. No shipped package references SIPSorcery.

Phase 1 adds `ArchitectureTests` that reflect over assembly references and fail the build on
violation. Every later phase keeps them green.

---

## 4. Deviations from the Spec

These five deviations were reviewed and approved. They are binding.

| # | Spec says | This plan does | Why |
|---|---|---|---|
| D1 | `Realtime.SignalR` as its own package | SignalR transport lives inside `LiveSharp.AspNetCore` | The SignalR **server** ships in the `Microsoft.AspNetCore.App` shared framework. Splitting it into its own NuGet package adds a package to the ecosystem without removing a single dependency from the consumer — pure ceremony. The `IRealtimeTransport` seam still exists and is exercised by an in-memory implementation in tests, so the abstraction is real, not decorative. |
| D2 | `Realtime.WebRTC` | `LiveSharp.Signaling` | The server never touches SDP semantics, codecs, or media. Naming the package `.WebRTC` would strongly imply media capability it does not have. Actual server-side WebRTC endpoints live in `LiveSharp.Media.SIPSorcery` (post-1.0). |
| D3 | Typed hub methods per feature | Single `LiveSharpHub` with operation routing | One connection per user. Multiple hubs means multiple connections, multiple reconnect lifecycles, duplicated presence bookkeeping, and worse mobile battery behaviour. Operation routing also gives one central place for authorization, validation, rate limiting, metrics, and protocol versioning. Typed developer experience is delivered by `LiveSharp.Client` / `@livesharp/client`, which is what Spec §8 actually asks for ("your clean API", not "SignalR internals"). |
| D4 | Clients under `src/` | Clients under `clients/` | They are not part of the server package graph and must not be swept up by `src/**` globs in build props, packaging, or coverage. |
| D5 | Examples location left open (Spec §15) | In-repo under `examples/`, excluded from `LiveSharp.slnx`, with their own `LiveSharp.Examples.slnx` using `ProjectReference` | In-repo keeps examples honest (they break when the library breaks). A separate solution keeps main CI fast and stops example dependencies from polluting the library restore graph. |

---

## 5. Global engineering rules (apply to every phase)

### 5.1 Definition of Done

A task is complete only when **all applicable** items pass. This mirrors Spec §64.

```
[ ] Public API designed and reviewed against Spec §8 questions
[ ] Implementation complete
[ ] XML docs on every public member (explain behaviour, not the method name)
[ ] Unit tests: happy path + invalid input + edge cases + cancellation + concurrency
[ ] Integration tests where the feature crosses a process/transport boundary
[ ] PublicAPI.Unshipped.txt updated
[ ] Documentation page written or updated under docs/
[ ] Example app updated (from Phase 4 onward)
[ ] CHANGELOG.md updated
[ ] ROADMAP.md status updated
[ ] DEVELOPMENT_LOG.md updated
[ ] scripts/verify.ps1 green
[ ] Conventional commit created
```

### 5.2 Non-negotiable code rules

- **Time.** Inject `TimeProvider`. Never call `DateTime.UtcNow`, `DateTimeOffset.UtcNow`,
  `Stopwatch.StartNew()`, `Task.Delay(x)` (use `Task.Delay(x, timeProvider, ct)`) or
  `new Timer(...)` (use `timeProvider.CreateTimer(...)`) in `src/`. Phase 0 adds banned-API
  analyzer entries; Phase 1 adds `BannedSymbols.txt`.
- **Cancellation.** Every async public method ends with
  `CancellationToken cancellationToken = default`. It must be passed down, not swallowed. Every
  such method has a test that asserts `OperationCanceledException` on a pre-cancelled token.
- **Async shape.** `ValueTask`/`ValueTask<T>` for hot-path abstractions with a synchronous
  in-memory fast path (registries, stores). `Task` for public service APIs that always do I/O.
  No `async void`. No `.Result`/`.Wait()`/`GetAwaiter().GetResult()` in `src/`.
- **Concurrency.** Assume every abstraction is called concurrently from many connections. Prefer
  `ConcurrentDictionary` and immutable snapshots over `lock`. Document thread-safety on every
  public type. Every registry/store gets a parallel-stress test.
- **Disposal.** Anything holding a timer, subscription, connection, or stream implements
  `IDisposable`/`IAsyncDisposable` and has a test proving cleanup and idempotent disposal.
- **Logging.** `ILogger<T>` only, with a per-project `Log` partial class using
  `[LoggerMessage]` source generation and a documented `EventId` range. **Never** log tokens,
  passwords, message bodies, SDP, or ICE candidates at `Information` or above. Message content may
  appear only behind `Trace` and only when
  `LiveSharpOptions.Diagnostics.AllowSensitivePayloadLogging` is explicitly `true`.
- **Nullability.** Nullable enabled everywhere. No `!` null-forgiving operator in `src/` without
  an adjacent comment justifying it.
- **Visibility.** Default to `internal` + `sealed`. Promote to `public` only when a consumer
  provably needs it. `InternalsVisibleTo` the matching test project.
- **Options.** All configuration through `IOptions<T>` with an `IValidateOptions<T>` and
  `.ValidateOnStart()`. Validation messages must name the property and state the fix.

### 5.3 Dependency policy

No NuGet or npm package may be added unless the current phase file has an entry for it under
**Dependency justification** answering all of Spec §37: why needed, can the BCL do it, is it
maintained, licence, transitive weight, coupling risk. Versions are declared centrally in
`Directory.Packages.props`. Adding a package is a reviewable decision, not a convenience.

### 5.4 Commit and branch convention

- One phase = one branch: `feat/phase-NN-<slug>` (e.g. `feat/phase-04-direct-messaging`).
- One task = one or more commits, conventional format:
  `feat(chat): add direct messaging`, `test(core): cover registry concurrency`,
  `docs(presence): add subscription model guide`, `fix(aspnetcore): rejoin groups on reconnect`,
  `chore(build): enable package validation`, `refactor(core): extract operation router`.
- No commits named `update`, `changes`, `fix stuff`, `wip`, `test`.
- One PR per phase. PR description sections: Summary, Motivation, Implementation, Architecture,
  Tests, Breaking Changes, Documentation, Future Work.

---

## 6. Wire protocol (normative)

Implemented in Phase 3. Every later phase adds operations and events to this protocol and must not
change its shape without a new ADR.

### 6.1 Hub

Single hub, single connection per client, mapped by default at `/livesharp`.

```csharp
// Client -> Server, request/response
Task<OperationResponse> InvokeAsync(string operation, JsonElement? payload);

// Client -> Server, fire-and-forget (typing indicators, heartbeats)
Task NotifyAsync(string operation, JsonElement? payload);

// Server -> Client, single delivery method
Task Receive(RealtimeEnvelope envelope);
```

### 6.2 Envelope and response shapes

Both types live in `LiveSharp.Abstractions`. Note the namespaces — they differ deliberately:
`RealtimeEnvelope` is used by every feature package and lives in the root namespace, while the
transport-protocol types are grouped under `LiveSharp.Protocol`.

```csharp
namespace LiveSharp;                     // defined in P01.T2

public sealed record RealtimeEnvelope(
    string Event,
    JsonElement Payload,
    string? CorrelationId = null);
```

```csharp
namespace LiveSharp.Protocol;            // defined in P03.T1

public sealed record OperationResponse(
    bool Success,
    string CorrelationId,
    JsonElement? Data = null,
    string? Code = null,       // machine-readable, from RealtimeErrorCode
    string? Message = null);   // human-readable, safe to surface to developers
```

### 6.3 Naming

- Operations (client → server): `<area>.<noun>.<verb>` — `chat.message.send`,
  `group.member.add`, `call.invite`, `signaling.offer`.
- Events (server → client): `<area>.<noun>.<past-tense>` — `chat.message.received`,
  `presence.changed`, `typing.started`, `call.state.changed`.
- All names are declared as `const string` in `LiveSharp.Abstractions`
  (`RealtimeOperations`, `RealtimeEvents`) so the server, the .NET client, and the generated
  TypeScript constants cannot drift.

### 6.4 Versioning and negotiation

- `LiveSharpProtocol.Version` is an integer, starting at `1`.
- The client sends its protocol version as a query string parameter `lsv` on connect.
- Unsupported version → the hub rejects the connection with a structured error before
  `OnConnectedAsync` completes any registration work.
- Adding an operation or event is a **minor** change. Changing or removing one is **major**.

### 6.5 Serialization

- `System.Text.Json` with a source-generated `JsonSerializerContext`
  (`LiveSharpJsonSerializerContext`) — no reflection-based serialization in the hot path.
- Handlers are `IOperationHandler<TRequest, TResponse>`; the router owns deserialization so
  handlers receive typed payloads and are unit-testable without JSON.
- MessagePack is an opt-in protocol added in Phase 16, never a default.

---

## 7. Session protocol

### 7.1 Session start — do this before writing any code

1. Read `AGENTS.md`, `DEVELOPMENT_LOG.md`, `ROADMAP.md`, this file, and the current
   `docs/plan/phase-NN-*.md`.
2. `git status` and `git log --oneline -10`.
3. Run `scripts/verify.ps1` (skip if no projects exist yet).
4. Print exactly this block before proceeding:

```
Current Phase:
Current Task:
Completed:
In Progress:
Next Task:
Known Issues:
```

5. If `DEVELOPMENT_LOG.md` says a task is `IN PROGRESS`, **finish that task**. Do not start a new
   one. Do not assume the previous session finished cleanly — verify against the working tree.

### 7.2 Session end — do this before reporting done

1. `scripts/verify.ps1` and record the actual test counts.
2. Update `DEVELOPMENT_LOG.md` (schema in §7.3).
3. Update `ROADMAP.md` status markers.
4. Update `CHANGELOG.md` under `## [Unreleased]`.
5. Confirm docs and examples reflect what the code actually does.
6. Commit with a conventional message.
7. State the next task ID explicitly.

### 7.3 DEVELOPMENT_LOG.md schema

Phase 0 creates this file with the following headings. Every session appends to
**Session History** and rewrites the state block at the top.

```markdown
# Development Log

## State
- Version: 0.1.0
- Phase: 01 — Core Abstractions
- Task: P01.T2 — Options and DI foundation
- Status: IN PROGRESS | COMPLETE | BLOCKED

## Completed Tasks
- P00.T1 … P00.T4

## Currently Working On
- …

## Next Task
- P01.T3 — Operation router

## Tests
- Passing: 0 / Failing: 0 / Skipped: 0

## Known Issues
- …

## Blocked On
- …

## Decisions Made This Session
- …

## Do Not Do Yet
- (copied from the current phase file's Non-goals section)

## Session History
### YYYY-MM-DD — <agent/model>
Task: P01.T2
Summary: …
Files changed: …
Tests: …
Next: …
```

### 7.4 Hard stops

Stop and report instead of proceeding if any of these is true:

1. The task requires functionality from a **later** phase. Record it under **Blocked On** and stop.
2. A public API decided in an earlier phase needs to change. Write an ADR proposal in
   `docs/architecture/decisions/` as `ADR-NNN-<slug>.draft.md` and stop for review.
3. `scripts/verify.ps1` fails for a reason you cannot fix inside the current task's scope.
4. You are about to add a dependency not listed in the current phase file.
5. You are about to mark a roadmap item `[x]` without tests **and** docs **and** changelog.

Never delete or rewrite a test to make a build pass. If a test is genuinely wrong, fix the test in
its own commit with a message explaining why the original expectation was incorrect.

---

## 8. Escalation: when the plan is wrong

This plan was written before the code existed. It will be wrong somewhere. When you find it:

1. Do **not** silently improvise a different architecture (Spec §46).
2. Write `docs/architecture/decisions/ADR-NNN-<slug>.draft.md` containing: Problem, Current
   behaviour, Proposed change, Why, Alternatives considered, Trade-offs, Impact on later phases.
3. Record it in `DEVELOPMENT_LOG.md` under **Blocked On**.
4. Stop and report. A human promotes the draft to an accepted ADR and amends the affected phase
   files.

---

## 9. Phase index

Versions are the tag cut at the **end** of the phase. `[ ]` planned, `[~]` in progress,
`[x]` complete — keep this table in sync with `ROADMAP.md`.

| # | Phase | Version | Ships | File |
|---|---|---|---|---|
| 00 | Governance & scaffolding | `0.1.0-alpha` | repo hygiene, CI, ADRs, logs | [phase-00-governance.md](docs/plan/phase-00-governance.md) |
| 01 | Core abstractions | `0.1.0` | `Abstractions`, `Core` | [phase-01-abstractions.md](docs/plan/phase-01-abstractions.md) |
| 02 | Connection infrastructure | `0.1.0` | registries, lifecycle, reaper | [phase-02-connections.md](docs/plan/phase-02-connections.md) |
| 03 | SignalR transport & clients | `0.2.0` | `AspNetCore`, `LiveSharp.Client`, `@livesharp/client` | [phase-03-signalr-transport.md](docs/plan/phase-03-signalr-transport.md) |
| 04 | Direct messaging | `0.3.0` | `Chat`, MVC example | [phase-04-direct-messaging.md](docs/plan/phase-04-direct-messaging.md) |
| 05 | Groups & channels | `0.4.0` | group service, Blazor Server example | [phase-05-groups.md](docs/plan/phase-05-groups.md) |
| 06 | Presence | `0.5.0` | `Presence`, `Chat.Presence`, Blazor WASM + React examples | [phase-06-presence.md](docs/plan/phase-06-presence.md) |
| 07 | Typing indicators | `0.6.0` | typing service, Next.js example | [phase-07-typing.md](docs/plan/phase-07-typing.md) |
| 08 | Message delivery state | `0.7.0` | receipts, idempotency | [phase-08-message-state.md](docs/plan/phase-08-message-state.md) |
| 09 | Authentication & authorization | `0.8.0` | filter pipeline, policies, rate limiting | [phase-09-auth.md](docs/plan/phase-09-auth.md) |
| 10 | WebRTC signalling | `0.9.0` | `Signaling`, ICE/TURN | [phase-10-webrtc-signaling.md](docs/plan/phase-10-webrtc-signaling.md) |
| 11 | Audio calling | `0.10.0` | `Calls`, state machine | [phase-11-audio-calling.md](docs/plan/phase-11-audio-calling.md) |
| 12 | Video calling | `0.11.0` | media negotiation, Playwright E2E | [phase-12-video-calling.md](docs/plan/phase-12-video-calling.md) |
| 13 | Screen sharing | `0.12.0` | display media | [phase-13-screen-sharing.md](docs/plan/phase-13-screen-sharing.md) |
| 14 | Persistence | `0.13.0` | `EntityFrameworkCore` | [phase-14-persistence.md](docs/plan/phase-14-persistence.md) |
| 15 | Distributed / scale-out | `0.14.0` | `Redis` | [phase-15-distributed.md](docs/plan/phase-15-distributed.md) |
| 16 | Performance | `0.15.0` | benchmarks, MessagePack, metrics | [phase-16-performance.md](docs/plan/phase-16-performance.md) |
| 17 | Developer tooling | `0.16.0` | templates, analyzers, `@livesharp/react`, docs site | [phase-17-tooling.md](docs/plan/phase-17-tooling.md) |
| 18 | Production hardening | `1.0.0` | API freeze, chaos tests, release automation | [phase-18-hardening.md](docs/plan/phase-18-hardening.md) |

**112 tasks in total.** The flat, ordered checklist is
[docs/plan/task-index.md](docs/plan/task-index.md) — work it top to bottom.

### Review gate

**Stop for human review after Phase 3.** The wire protocol and transport are the two decisions
that are expensive to change later. Do not begin Phase 4 until the Phase 3 PR is approved.

---

## 10. Verification gate

`scripts/verify.ps1` is the single Definition-of-Done command. Phase 0 creates it. Later phases
extend it. It must run clean on Windows and Linux.

```
1. dotnet build LiveSharp.slnx -c Release -warnaserror
2. dotnet test  LiveSharp.slnx -c Release
3. dotnet format LiveSharp.slnx --verify-no-changes
4. dotnet pack  LiveSharp.slnx -c Release -o artifacts/packages
5. public API check: fail if any PublicAPI.Unshipped.txt has uncommitted additions
6. (Phase 3+)  pnpm -r --dir clients/js build && pnpm -r --dir clients/js test && pnpm -r --dir clients/js lint
7. (Phase 12+) E2E suite, non-blocking until Phase 18
```

Flags: `-SkipJs`, `-SkipE2E`, `-SkipPack` for fast inner loops. CI always runs the full gate.

---

## 11. Toolchain versions

Declared centrally in `Directory.Packages.props`. Verified against nuget.org at plan time. Bump
deliberately, never opportunistically; Dependabot opens PRs but a human merges them.

| Purpose | Package | Version |
|---|---|---|
| Test framework | `xunit.v3` | `4.0.0` |
| Test runner integration | `xunit.runner.visualstudio` | latest matching v3 |
| Assertions | `Shouldly` | `4.3.0` |
| Mocking | `NSubstitute` | `6.2.0` |
| Deterministic time | `Microsoft.Extensions.TimeProvider.Testing` | `10.9.0` |
| Integration host | `Microsoft.AspNetCore.Mvc.Testing` | `10.0.11` |
| .NET SignalR client | `Microsoft.AspNetCore.SignalR.Client` | `10.0.11` |
| MessagePack protocol (P16) | `Microsoft.AspNetCore.SignalR.Protocols.MessagePack` | `10.0.11` |
| Redis backplane (P15) | `Microsoft.AspNetCore.SignalR.StackExchangeRedis` | `10.0.11` |
| Redis client (P15) | `StackExchange.Redis` | transitive from the backplane |
| Container-based tests (P14, P15) | `Testcontainers.Redis`, `Testcontainers.PostgreSql` | `4.14.0` |
| Persistence (P14) | `Microsoft.EntityFrameworkCore.Relational` | latest `10.x` |
| Test providers (P14) | `Microsoft.EntityFrameworkCore.Sqlite`, `Npgsql.EntityFrameworkCore.PostgreSQL` | latest `10.x` |
| Public API gate | `Microsoft.CodeAnalysis.PublicApiAnalyzers` | `5.6.0` |
| Banned API analyzer | `Microsoft.CodeAnalysis.BannedApiAnalyzers` | latest `5.x` |
| Analyzer authoring (P17) | `Microsoft.CodeAnalysis.CSharp.Workspaces` | latest matching the SDK |
| Versioning | `MinVer` | `7.0.0` |
| Benchmarks (P16) | `BenchmarkDotNet` | `0.15.8` |
| Browser E2E (P12) | `@playwright/test` | latest |
| Docs site (P17) | `docfx` (dotnet local tool) | latest |
| Fault injection (P18) | Toxiproxy via Testcontainers | latest, **only if needed** |
| Server-side WebRTC (post-1.0) | `SIPSorcery` | `10.0.16` |

Only the first block through `MinVer` is added in Phase 00. Everything else arrives in the phase
noted in parentheses, and only with a **Dependency justification** entry in that phase file.

**Not needed:** `Microsoft.SourceLink.GitHub` — Source Link is in the SDK since .NET 8. Enable it
with MSBuild properties only.

---

## 12. Known risks

| Risk | Where addressed | Mitigation |
|---|---|---|
| `.slnx` solution format tooling gaps | P00.T1 | Prove `build`/`test`/`pack`/`format` on `.slnx` in CI immediately. Documented one-command fallback to `.sln` if any of the four fails |
| xUnit v3 + Microsoft.Testing.Platform runner behaviour under `dotnet test` | P00.T4 | A single throwaway smoke test proves the runner in CI before any real tests are written |
| SignalR over `TestServer` — WebSockets need explicit wiring | P03.T5 | Exact `WebSocketFactory` + `HttpMessageHandlerFactory` recipe is in the phase file; both WebSocket and LongPolling transports are tested |
| SignalR groups are per-connection and lost on reconnect | P05.T4 | Explicit group-rehydration-on-reconnect task with a dedicated reconnect test |
| Browser WebRTC E2E flakiness | P12.T5 | Fake media device flags mandated; E2E lane is non-blocking in CI until Phase 18; retries + quarantine |
| `SIPSorcery` `net10.0` compatibility unknown | Post-1.0 | Isolated in one experimental package; a compatibility spike is required before any adoption |
| Scope creep into calling before the foundation is stable | Everywhere | Phase-scoped file loading, Non-goals sections, hard stops, and the Phase 3 review gate |
| Distributed correctness assumed rather than tested | P15 | Two-node integration harness with a real Redis container is a Phase 15 exit criterion |

---

## 13. Anti-goals for this project

Write these into `README.md` §Non-Goals in Phase 0 so expectations are set publicly:

- LiveSharp is **not** a media server, SFU, or MCU. Media is peer-to-peer.
- LiveSharp is **not** a hosted service or a SaaS SDK.
- LiveSharp does **not** ship a UI component library.
- LiveSharp does **not** require a database.
- LiveSharp does **not** replace SignalR — it is built on it and does not hide it from developers
  who need to reach through.
- LiveSharp does **not** implement SIP, PSTN, or telephony interop pre-1.0.
