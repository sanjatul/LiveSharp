# LiveSharp — Task Index

Flat, ordered checklist of every task in the plan. **112 tasks across 19 phases.**

One task ≈ one AI session. Work strictly top to bottom. Mark a task `[x]` only when
`scripts/verify.ps1` is green and `DEVELOPMENT_LOG.md` records it — see
`EXECUTION_PLAN.md` §5.1 for the full Definition of Done.

Legend: `[ ]` planned · `[~]` in progress · `[x]` complete · `[-]` deferred

> Keep this file, `ROADMAP.md`, and `EXECUTION_PLAN.md` §9 in sync. This file is the operational
> checklist; `ROADMAP.md` is the public progress view.

---

## Phase 00 — Governance & scaffolding → `0.1.0-alpha`
`docs/plan/phase-00-governance.md` · branch `feat/phase-00-governance`

- [ ] `P00.T1` Repository skeleton, build props, solution
- [ ] `P00.T2` Community and legal files
- [ ] `P00.T3` Architecture decision records
- [ ] `P00.T4` Documentation skeleton and test-runner smoke proof
- [ ] `P00.T5` Scripts and the verification gate
- [ ] `P00.T6` GitHub automation

## Phase 01 — Core abstractions → `0.1.0`
`docs/plan/phase-01-abstractions.md` · branch `feat/phase-01-abstractions`

- [ ] `P01.T1` Projects, DI wiring of the test harness, architecture tests
- [ ] `P01.T2` Identity, connection, and envelope contracts
- [ ] `P01.T3` Error model: `OperationResult`, error codes, exceptions
- [ ] `P01.T4` Transport, registry, and handler contracts
- [ ] `P01.T5` Options, validation, and `AddLiveSharp`
- [ ] `P01.T6` Operation names, event names, and the operation router

## Phase 02 — Connection infrastructure → `0.1.0` (tag)
`docs/plan/phase-02-connections.md` · branch `feat/phase-02-connections`

- [ ] `P02.T1` `InMemoryConnectionRegistry`
- [ ] `P02.T2` `InMemoryGroupRegistry`
- [ ] `P02.T3` `DefaultUserResolver`
- [ ] `P02.T4` `InMemoryTransport`
- [ ] `P02.T5` Connection lifecycle service and hooks
- [ ] `P02.T6` Stale connection reaper

## Phase 03 — SignalR transport & clients → `0.2.0`
`docs/plan/phase-03-signalr-transport.md` · branch `feat/phase-03-signalr-transport`

- [ ] `P03.T1` `LiveSharp.AspNetCore` project and protocol types
- [ ] `P03.T2` `LiveSharpHub`
- [ ] `P03.T3` `SignalRRealtimeTransport` and the user-id provider
- [ ] `P03.T4` `MapLiveSharp`, options, and SignalR wiring
- [ ] `P03.T5` Integration tests over a real SignalR connection
- [ ] `P03.T6` `LiveSharp.Client` (.NET)
- [ ] `P03.T7` `@livesharp/client` (TypeScript)

> ### REVIEW GATE
> **Stop after `P03.T7`.** The wire protocol and transport are frozen here and are expensive to
> change later. Do not start `P04.T1` until the Phase 03 PR is reviewed and approved by a human, and
> `DEVELOPMENT_LOG.md` records the approval.

## Phase 04 — Direct messaging → `0.3.0`
`docs/plan/phase-04-direct-messaging.md` · branch `feat/phase-04-direct-messaging`

- [ ] `P04.T1` `LiveSharp.Chat` package, message model, and ID generation
- [ ] `P04.T2` Validation and limits
- [ ] `P04.T3` `IMessageStore` and `NullMessageStore`
- [ ] `P04.T4` `IChatService`, the send handler, and DI
- [ ] `P04.T5` Integration tests and client support
- [ ] `P04.T6` The MVC example application

## Phase 05 — Groups & channels → `0.4.0`
`docs/plan/phase-05-groups.md` · branch `feat/phase-05-groups`

- [ ] `P05.T1` Group model, store, and options
- [ ] `P05.T2` `IGroupService`: lifecycle and membership
- [ ] `P05.T3` Group messaging
- [ ] `P05.T4` Group rehydration on reconnect ← **the phase's critical task**
- [ ] `P05.T5` `AddGroups()` and DI
- [ ] `P05.T6` Blazor Server example and client support

## Phase 06 — Presence → `0.5.0`
`docs/plan/phase-06-presence.md` · branch `feat/phase-06-presence`

- [ ] `P06.T1` Package, model, store
- [ ] `P06.T2` `IPresenceService`: aggregation and status precedence
- [ ] `P06.T3` Subscriptions and scoping ← **security boundary**
- [ ] `P06.T4` Broadcasting, coalescing, and the expiry sweep
- [ ] `P06.T5` Operations, handlers, and `AddPresence()`
- [ ] `P06.T6` Blazor WebAssembly and React examples

## Phase 07 — Typing indicators → `0.6.0`
`docs/plan/phase-07-typing.md` · branch `feat/phase-07-typing`

- [ ] `P07.T1` Model, tracker, and options
- [ ] `P07.T2` `ITypingService`, sweep service, and lifecycle integration
- [ ] `P07.T3` Operations, `ClearOnMessageSent`, and `AddTyping()`
- [ ] `P07.T4` Next.js example and client support

## Phase 08 — Message delivery state → `0.7.0`
`docs/plan/phase-08-message-state.md` · branch `feat/phase-08-message-state`

- [ ] `P08.T1` State model and state machine
- [ ] `P08.T2` Receipt store and summary aggregation
- [ ] `P08.T3` Receipt service, batching, and send-path integration
- [ ] `P08.T4` Operations, handlers, `AddMessageState()`, and clients
- [ ] `P08.T5` Example applications and documentation

## Phase 09 — Authentication & authorization → `0.8.0`
`docs/plan/phase-09-auth.md` · branch `feat/phase-09-auth`

**Contains a breaking change: anonymous connections stop being allowed by default.**
Read the whole phase file before writing any code.

- [ ] `P09.T1` Require authentication ← **breaking change**
- [ ] `P09.T2` JWT bearer over WebSockets
- [ ] `P09.T3` The operation filter pipeline
- [ ] `P09.T4` Declarative operation authorization
- [ ] `P09.T5` Resource-based authorization ← **Phase 10 depends on this**
- [ ] `P09.T6` Rate limiting
- [ ] `P09.T7` Token expiry, sensitive-logging audit, and examples

## Phase 10 — WebRTC signalling → `0.9.0`
`docs/plan/phase-10-webrtc-signaling.md` · branch `feat/phase-10-webrtc-signaling`

- [ ] `P10.T1` Package, model, and options
- [ ] `P10.T2` Session store, service, and idle sweep
- [ ] `P10.T3` Relay authorization ← **an unauthorized relay is a covert messaging channel**
- [ ] `P10.T4` ICE servers and ephemeral TURN credentials ← **highest-consequence security surface**
- [ ] `P10.T5` Operations, handlers, and `AddSignaling()`
- [ ] `P10.T6` Client helpers and the data-channel example

## Phase 11 — Audio calling → `0.10.0`
`docs/plan/phase-11-audio-calling.md` · branch `feat/phase-11-audio-calling`

- [ ] `P11.T1` Model and the call state machine ← **144-cell transition table**
- [ ] `P11.T2` Store, options, and `ICallService`
- [ ] `P11.T3` Multi-device ringing ← **exactly one device may win**
- [ ] `P11.T4` Timeouts, disconnect handling, glare, and signalling integration
- [ ] `P11.T5` Operations, handlers, and `AddCalls()`
- [ ] `P11.T6` Browser call helper and example applications

## Phase 12 — Video calling → `0.11.0`
`docs/plan/phase-12-video-calling.md` · branch `feat/phase-12-video-calling`

- [ ] `P12.T1` Media negotiation on the server
- [ ] `P12.T2` Browser video call helper
- [ ] `P12.T3` Playwright end-to-end infrastructure ← **first real browser test suite**
- [ ] `P12.T4` Server-side and integration test updates
- [ ] `P12.T5` Examples and documentation

## Phase 13 — Screen sharing → `0.12.0`
`docs/plan/phase-13-screen-sharing.md` · branch `feat/phase-13-screen-sharing`

- [ ] `P13.T1` Server-side share state and arbitration
- [ ] `P13.T2` Browser screen share helper ← **`track.onended` is mandatory**
- [ ] `P13.T3` E2E tests and integration tests
- [ ] `P13.T4` Examples and documentation

> **The feature surface stops growing after Phase 13.** Phases 14–18 make what exists durable,
> scalable, fast, and shippable.

## Phase 14 — Persistence → `0.13.0`
`docs/plan/phase-14-persistence.md` · branch `feat/phase-14-persistence`

- [ ] `P14.T1` The history query API ← **design before touching EF**
- [ ] `P14.T2` The EF Core package and schema
- [ ] `P14.T3` Store implementations
- [ ] `P14.T4` Migrations, DI, and provider testing
- [ ] `P14.T5` Retention, call history, and background maintenance
- [ ] `P14.T6` Examples, documentation, and the architecture guarantee

## Phase 15 — Distributed / scale-out → `0.14.0`
`docs/plan/phase-15-distributed.md` · branch `feat/phase-15-distributed`

- [ ] `P15.T1` Node identity, the backplane, and the key schema
- [ ] `P15.T2` Distributed connection and group registries
- [ ] `P15.T3` Distributed presence
- [ ] `P15.T4` Distributed signalling, calls, and cross-node control
- [ ] `P15.T5` Distributed rate limiting and the two-node test harness
- [ ] `P15.T6` Documentation and deployment guidance

## Phase 16 — Performance → `0.15.0`
`docs/plan/phase-16-performance.md` · branch `feat/phase-16-performance`

**Order is not negotiable.** `T1` and `T2` measure. `T4` and `T5` change code. An optimisation
commit without a cited benchmark delta has failed the phase.

- [ ] `P16.T1` Benchmark project
- [ ] `P16.T2` Load test harness and baseline measurements ← **commit the baseline before optimising**
- [ ] `P16.T3` Observability: metrics and tracing
- [ ] `P16.T4` Serialization and allocation optimisation
- [ ] `P16.T5` Redis and database path optimisation
- [ ] `P16.T6` Leak tests and published results

## Phase 17 — Developer tooling → `0.16.0`
`docs/plan/phase-17-tooling.md` · branch `feat/phase-17-tooling`

- [ ] `P17.T1` `dotnet new` templates
- [ ] `P17.T2` Startup validation and better error messages ← **highest-value task in the phase**
- [ ] `P17.T3` Roslyn analyzers
- [ ] `P17.T4` Diagnostics endpoint
- [ ] `P17.T5` `@livesharp/react` and TypeScript generation
- [ ] `P17.T6` Documentation site and the five-minute test

## Phase 18 — Production hardening → `1.0.0`
`docs/plan/phase-18-hardening.md` · branch `feat/phase-18-hardening` → `release/1.0`

**`P18.T1` and `P18.T8` are irreversible.** Complete the tasks in order.

- [ ] `P18.T1` Public API review and freeze ← **irreversible**
- [ ] `P18.T2` Resilience and chaos testing
- [ ] `P18.T3` Security review and threat model
- [ ] `P18.T4` Deployment guidance
- [ ] `P18.T5` Documentation completeness and honesty audit
- [ ] `P18.T6` Package quality and metapackage
- [ ] `P18.T7` Release automation
- [ ] `P18.T8` Cut 1.0.0 ← **irreversible; do not start until every other criterion is met**

---

## Milestone summary

| Tag | Phases | Ships |
|---|---|---|
| `v0.1.0-alpha.1` | 00 | repo hygiene, CI, ADRs, verification gate |
| `v0.1.0` | 01–02 | `Abstractions`, `Core`, connection lifecycle |
| `v0.2.0` | 03 | `AspNetCore`, `LiveSharp.Client`, `@livesharp/client` |
| `v0.3.0` | 04 | direct messaging, MVC example |
| `v0.4.0` | 05 | groups, Blazor Server example |
| `v0.5.0` | 06 | `LiveSharp.Presence`, Blazor WASM + React examples |
| `v0.6.0` | 07 | typing indicators, Next.js example |
| `v0.7.0` | 08 | delivery and read receipts |
| `v0.8.0` | 09 | authentication, authorization, rate limiting |
| `v0.9.0` | 10 | `LiveSharp.Signaling`, ICE/TURN |
| `v0.10.0` | 11 | `LiveSharp.Calls`, audio |
| `v0.11.0` | 12 | video, Playwright E2E |
| `v0.12.0` | 13 | screen sharing |
| `v0.13.0` | 14 | `LiveSharp.EntityFrameworkCore`, history |
| `v0.14.0` | 15 | `LiveSharp.Redis`, scale-out |
| `v0.15.0` | 16 | benchmarks, metrics, tracing, leak tests |
| `v0.16.0` | 17 | templates, analyzers, diagnostics, `@livesharp/react`, docs site |
| `v1.0.0` | 18 | API freeze, chaos suite, security review, release automation |

## Package delivery order

| Package | Introduced |
|---|---|
| `LiveSharp.Abstractions` | `P01.T1` |
| `LiveSharp.Core` | `P01.T1` |
| `LiveSharp.AspNetCore` | `P03.T1` |
| `LiveSharp.Client` (.NET) | `P03.T6` |
| `@livesharp/client` | `P03.T7` |
| `LiveSharp.Chat` | `P04.T1` |
| `LiveSharp.Presence` | `P06.T1` |
| `LiveSharp.Chat.Presence` | `P06.T3` |
| `LiveSharp.Signaling` | `P10.T1` |
| `LiveSharp.Calls` | `P11.T1` |
| `LiveSharp.EntityFrameworkCore` | `P14.T2` |
| `LiveSharp.Redis` | `P15.T1` |
| `LiveSharp.Templates` | `P17.T1` |
| `LiveSharp.Analyzers` (inside `Core`) | `P17.T3` |
| `@livesharp/react` | `P17.T5` |
| `LiveSharp` (metapackage) | `P18.T6` |
| `LiveSharp.Testing` | `P18.T6`, if `P18.T1` decides the split |
| `LiveSharp.Media.SIPSorcery` | **never in 1.0** — post-1.0, experimental |
