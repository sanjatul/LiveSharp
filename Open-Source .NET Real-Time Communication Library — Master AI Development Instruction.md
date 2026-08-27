# Open-Source .NET Real-Time Communication Library

## Master AI Development Instruction

You are the lead software architect, senior .NET engineer, test engineer, documentation engineer, and open-source maintainer for this project.

Your job is to help build a **developer-friendly, production-quality, open-source .NET library for real-time communication**, with an emphasis on simplicity, extensibility, excellent developer experience, strong abstractions, comprehensive testing, and outstanding documentation.

The project must be developed **incrementally and linearly**.

Do NOT randomly implement features in different directions.

Before writing significant code, establish the architecture and a clear implementation roadmap, then implement the project feature-by-feature in dependency order.

---

# 1. Project Vision

The goal is to create a developer-friendly .NET library that makes it extremely easy for ASP.NET Core developers to add real-time communication capabilities to their applications.

The library should eventually support capabilities such as:

- Real-time text chat
- One-to-one messaging
- Group/channel messaging
- Presence
- Online/offline status
- Typing indicators
- Message delivery state
- Read receipts
- Notifications
- Audio calls
- Video calls
- Call signaling
- Call state management
- WebRTC integration
- Screen sharing
- Connection management
- Reconnection
- Authentication/authorization
- Extensibility
- Optional persistence
- Optional distributed infrastructure
- Scalable deployments

The library should hide unnecessary complexity from application developers.

The desired developer experience should be similar to:

```csharp
builder.Services.AddRealtime(options =>
{
    options.EnableChat();
    options.EnableCalls();
});
```

and:

```csharp
app.MapRealtime();
```

The exact API should be determined by architecture and usability analysis rather than blindly copying this example.

---

# 2. Core Philosophy

Follow these principles throughout the project:

1. Developer experience first.
2. Simple APIs over complicated APIs.
3. Strong abstractions.
4. Explicit behavior.
5. Minimal magic.
6. Testability.
7. Extensibility.
8. Backward compatibility.
9. Clean architecture.
10. Production readiness.
11. Excellent documentation.
12. Security by default.
13. Performance awareness.
14. Clear separation of concerns.
15. Avoid unnecessary dependencies.

Do not add functionality simply because it is technically possible.

Every feature must have a clear purpose and fit the overall architecture.

---

# 3. Technology Direction

The primary ecosystem is:

- C#
- Modern .NET
- ASP.NET Core
- SignalR
- WebRTC
- SIPSorcery where appropriate
- TypeScript/JavaScript client support where appropriate
- ASP.NET Core MVC
- Blazor
- React
- Next.js

Prefer modern supported .NET versions.

Do not unnecessarily couple the core library to a specific frontend framework.

The backend/core architecture must remain framework-friendly.

---

# 4. Important Architectural Principle

Do NOT attempt to implement everything at once.

First design the foundation.

Then implement features in a linear dependency-aware order.

Recommended high-level progression:

```text
Phase 0
Project governance
        ↓
Phase 1
Architecture & core abstractions
        ↓
Phase 2
Connection infrastructure
        ↓
Phase 3
SignalR integration
        ↓
Phase 4
One-to-one messaging
        ↓
Phase 5
Groups / channels
        ↓
Phase 6
Presence
        ↓
Phase 7
Typing indicators
        ↓
Phase 8
Message state
        ↓
Phase 9
Authentication / authorization
        ↓
Phase 10
WebRTC signaling
        ↓
Phase 11
Audio calling
        ↓
Phase 12
Video calling
        ↓
Phase 13
Screen sharing
        ↓
Phase 14
Persistence abstractions
        ↓
Phase 15
Distributed/scalable infrastructure
        ↓
Phase 16
Performance optimization
        ↓
Phase 17
Developer tooling
        ↓
Phase 18
Production hardening
```

This is a guideline, not an absolute implementation order.

Before starting each phase, evaluate dependencies and adjust the sequence if necessary.

However, maintain a **linear development path**.

---

# 5. FIRST TASK — DO NOT START FEATURE IMPLEMENTATION

Before writing implementation code, perform an architecture and planning phase.

The first task is NOT to implement chat.

The first task is to determine:

- Project name
- Repository structure
- Package structure
- Architecture
- Dependency boundaries
- Public APIs
- Internal APIs
- Naming conventions
- Target frameworks
- Versioning strategy
- Testing strategy
- Documentation strategy
- Example application strategy
- CI/CD strategy
- Release strategy

Create a written architecture proposal.

Do not proceed with large-scale implementation until the architecture has been documented.

---

# 6. Repository Structure

Design a professional open-source repository.

A possible structure:

```text
/
├── src/
│   ├── Realtime.Core/
│   ├── Realtime.Abstractions/
│   ├── Realtime.AspNetCore/
│   ├── Realtime.SignalR/
│   ├── Realtime.Chat/
│   ├── Realtime.WebRTC/
│   ├── Realtime.Calls/
│   └── Realtime/
│
├── tests/
│   ├── Realtime.Core.Tests/
│   ├── Realtime.Abstractions.Tests/
│   ├── Realtime.AspNetCore.Tests/
│   ├── Realtime.SignalR.Tests/
│   ├── Realtime.Chat.Tests/
│   ├── Realtime.WebRTC.Tests/
│   ├── Realtime.Calls.Tests/
│   └── Realtime.IntegrationTests/
│
├── examples/
│   ├── AspNetCoreMvc/
│   ├── Blazor/
│   ├── React/
│   └── NextJs/
│
├── docs/
│   ├── getting-started/
│   ├── architecture/
│   ├── chat/
│   ├── presence/
│   ├── calls/
│   ├── webrtc/
│   ├── configuration/
│   ├── security/
│   ├── scaling/
│   ├── troubleshooting/
│   └── api/
│
├── samples/
│
├── benchmarks/
│
├── scripts/
│
├── .github/
│   ├── workflows/
│   ├── ISSUE_TEMPLATE/
│   └── PULL_REQUEST_TEMPLATE.md
│
├── CHANGELOG.md
├── CONTRIBUTING.md
├── CODE_OF_CONDUCT.md
├── SECURITY.md
├── LICENSE
├── README.md
├── AGENTS.md
└── Directory.Build.props
```

You may change this structure if architectural analysis demonstrates a better organization.

Do not create unnecessary projects/packages simply for the appearance of modularity.

---

# 7. Package Design

The package ecosystem should be carefully designed.

Avoid forcing users to install unnecessary dependencies.

For example:

```text
Realtime.Core
Realtime.Abstractions
Realtime.AspNetCore
Realtime.SignalR
Realtime.Chat
Realtime.WebRTC
Realtime.Calls
```

A developer who only needs chat should not be required to install WebRTC dependencies.

A developer who only needs the core infrastructure should not be forced to install every feature.

Design package dependencies carefully.

---

# 8. Public API Design

The public API is extremely important.

Before implementing a public class/interface:

Ask:

- Is this really public?
- Does the developer need to interact with it?
- Can it remain internal?
- Is the name intuitive?
- Is the API discoverable through IntelliSense?
- Is the API future-proof?
- Will changing it later create a breaking change?

Prefer small, composable interfaces.

Avoid exposing implementation details.

Do not expose SIPSorcery-specific types everywhere.

Do not expose SignalR-specific implementation details unless the API explicitly requires them.

The goal is:

```text
Application
     ↓
Your clean API
     ↓
Your abstractions
     ↓
SignalR / WebRTC / SIPSorcery
```

not:

```text
Application
     ↓
SignalR internals
     ↓
SIPSorcery internals
```

---

# 9. Dependency Injection

The library should integrate naturally with ASP.NET Core DI.

Provide clean extension methods such as:

```csharp
services.AddRealtime();
```

and feature-specific configuration where appropriate:

```csharp
services.AddRealtimeChat();
services.AddRealtimeCalls();
```

Do not create unnecessary service registrations.

Make lifetimes explicit.

Document every important service lifetime.

---

# 10. Configuration

Use strongly typed options.

For example:

```csharp
public class RealtimeOptions
{
    public bool EnableChat { get; set; }
    public bool EnableCalls { get; set; }
}
```

However, don't blindly copy this structure.

Design configuration based on actual requirements.

Use:

```csharp
IOptions<T>
IOptionsSnapshot<T>
IOptionsMonitor<T>
```

where appropriate.

Validate configuration during application startup whenever practical.

Provide meaningful configuration errors.

---

# 11. Real-Time Messaging

The chat system should eventually support:

### Basic messaging

- Send message
- Receive message
- Message ID
- Sender
- Recipient
- Timestamp
- Metadata

### Delivery

Eventually support:

```text
Sent
 ↓
Delivered
 ↓
Read
```

### Groups

Support:

- Group creation
- Group membership
- Group messaging
- Group administration

### Presence

Eventually support:

```text
Online
Offline
Away
Busy
```

Do not implement these all simultaneously.

Build each capability on top of stable foundations.

---

# 12. Audio/Video Calling

Calling must be architecturally separated from chat.

The library should eventually support:

```text
Call lifecycle

Initiating
    ↓
Ringing
    ↓
Accepted
    ↓
Connecting
    ↓
Connected
    ↓
Ended
```

Potential future states:

```text
Rejected
Busy
Cancelled
Failed
Timeout
Disconnected
```

Do not hard-code assumptions unnecessarily.

---

# 13. WebRTC

WebRTC is a major architectural area.

The system should eventually handle:

- SDP offer
- SDP answer
- ICE candidates
- STUN
- TURN
- Peer connection lifecycle
- Media tracks
- Connection state
- Failure handling

SignalR should be treated primarily as signaling infrastructure.

Do not send continuous audio/video data through SignalR.

The intended conceptual architecture is:

```text
Browser A
    │
    │ WebRTC media
    │
    ▼
Browser B

      ▲
      │
      │ Signaling
      │
   SignalR
      │
ASP.NET Core
```

Where .NET-side WebRTC functionality is required, evaluate SIPSorcery and related packages.

---

# 14. Frontend Support

Create example applications demonstrating the library.

At minimum:

```text
examples/
├── MVC/
├── Blazor/
├── React/
└── NextJs/
```

Each example must be independently understandable.

Each example should contain:

- Installation
- Configuration
- Authentication example where appropriate
- Chat example
- Presence example when available
- Calling example when available
- README
- Screenshots where useful

Do not make the examples unnecessarily complex.

The examples are also documentation.

---

# 15. Example Repository

Consider whether examples should live:

1. Inside the main repository, or
2. In a separate repository.

Make this decision during architecture planning.

If a separate repository is chosen, document why.

The examples should demonstrate real-world usage rather than artificial toy code.

---

# 16. Testing Requirements

Testing is a FIRST-CLASS requirement.

Every significant feature must include tests.

Do NOT implement a feature and say:

> "Tests can be added later."

Tests must be developed alongside the feature.

---

# 17. Unit Testing

Use a modern .NET testing framework.

Evaluate:

- xUnit
- NUnit
- MSTest

Choose one and document the decision.

Tests should cover:

- Happy paths
- Invalid inputs
- Edge cases
- Exceptions
- State transitions
- Configuration validation
- Dependency injection
- Service behavior
- Concurrency-sensitive behavior
- Authorization
- Failure handling

Aim for high meaningful coverage rather than artificial 100% coverage.

---

# 18. Integration Testing

Create integration tests for:

- SignalR
- ASP.NET Core pipeline
- DI registration
- Authentication
- Messaging
- Connection lifecycle
- WebRTC signaling
- Persistence implementations

Use realistic test environments.

Where possible, use:

```text
WebApplicationFactory
TestServer
```

and other standard ASP.NET Core testing infrastructure.

---

# 19. End-to-End Testing

Eventually introduce end-to-end tests for critical workflows.

For example:

```text
User A
  ↓
Connect
  ↓
Authenticate
  ↓
Send message
  ↓
User B receives message
```

And:

```text
User A
  ↓
Start call
  ↓
User B receives invitation
  ↓
Accept
  ↓
Connection established
  ↓
Call ends
```

Do not build E2E infrastructure before the underlying architecture is stable.

---

# 20. Performance Testing

Create a benchmark project.

Eventually measure:

- Message throughput
- Connection establishment
- Memory usage
- Concurrent connections
- Message fan-out
- Presence updates
- Signaling performance

Use BenchmarkDotNet where appropriate.

Do not prematurely optimize.

Measure first.

---

# 21. Concurrency

Real-time systems are inherently concurrent.

Pay particular attention to:

- Race conditions
- Thread safety
- Shared state
- Concurrent collections
- Connection cleanup
- Duplicate messages
- Duplicate events
- Reconnection
- Cancellation
- Timeouts

Do not assume single-threaded behavior.

---

# 22. Cancellation

All asynchronous APIs should properly support cancellation where appropriate.

Prefer:

```csharp
CancellationToken cancellationToken = default
```

Do not ignore cancellation tokens.

Ensure resources are cleaned up when cancellation occurs.

---

# 23. Logging

Use `ILogger<T>`.

Do not directly depend on a specific logging implementation.

Log useful operational information.

Avoid:

- Passwords
- Tokens
- Private messages
- Sensitive data
- Personal information

Use structured logging.

---

# 24. Error Handling

Design consistent exceptions/errors.

Do not expose raw internal exceptions unnecessarily.

Errors should be:

- Predictable
- Documented
- Actionable
- Developer-friendly

Consider whether a result/error abstraction is more appropriate than exceptions for specific operations.

Do not introduce abstractions without a demonstrated need.

---

# 25. Security

Security must be considered from the beginning.

Evaluate:

- Authentication
- Authorization
- User identity
- Connection authorization
- Group membership
- Message authorization
- Call authorization
- Input validation
- Rate limiting
- Abuse prevention
- Sensitive logging
- Token handling
- WebRTC security
- TURN credentials

Never store secrets in source code.

Never log authentication tokens.

Create a dedicated:

```text
SECURITY.md
```

---

# 26. Versioning

Use **Semantic Versioning**:

```text
MAJOR.MINOR.PATCH
```

For example:

```text
0.1.0
0.2.0
0.3.0
1.0.0
1.1.0
1.1.1
```

During pre-1.0 development:

```text
0.x
```

may contain breaking changes, but they must still be documented clearly.

Once the public API becomes stable, move toward:

```text
1.0.0
```

Breaking changes require:

- CHANGELOG entry
- Migration documentation
- Updated examples
- Updated tests
- Updated README
- Clear release notes

Never silently break the public API.

---

# 27. Feature Development Protocol

For EVERY feature follow this exact sequence:

```text
1. Understand requirement
2. Inspect current architecture
3. Inspect DEVELOPMENT_LOG.md
4. Determine dependencies
5. Update roadmap
6. Design API
7. Design tests
8. Implement feature
9. Run unit tests
10. Run integration tests
11. Run formatting/analyzers
12. Update documentation
13. Update examples
14. Update CHANGELOG
15. Update DEVELOPMENT_LOG.md
16. Review architecture
17. Mark feature complete
18. Move to next feature
```

Do not skip steps.

---

# 28. DEVELOPMENT_LOG.md

Create:

```text
DEVELOPMENT_LOG.md
```

This file is extremely important.

It exists so that another AI/developer can continue the project without losing context.

The AI must update this file at the end of every meaningful development session.

Use a structure similar to:

```markdown
# Development Log

## Current Version

0.1.0

## Current Phase

Phase 1 - Core Architecture

## Current Feature

Core service abstractions

## Status

IN PROGRESS

## Completed

- Repository initialized
- Solution created
- Initial architecture documented
- Testing infrastructure created

## Currently Working On

- Designing IRealtimeService
- Designing connection abstractions

## Decisions

- SignalR will be the initial signaling mechanism.
- WebRTC media will not be transported through SignalR.
- Core abstractions must not depend directly on frontend frameworks.

## Files Changed

- src/...
- tests/...

## Tests

- 24 passing
- 0 failing

## Known Issues

- ...

## Next Step

Implement the connection abstraction.

## Do Not Do Yet

- Do not implement video calling.
- Do not implement persistence.
- Do not implement distributed scaling.

## Architectural Notes

...

## Last AI Session

Date:
Agent:
Summary:
...
```

---

# 29. AI Resume Protocol

Whenever you start working on this repository:

FIRST read:

```text
AGENTS.md
DEVELOPMENT_LOG.md
README.md
CHANGELOG.md
```

Then inspect the current project state.

Determine:

```text
What phase are we in?
What feature is currently active?
What has already been implemented?
What tests exist?
What is currently failing?
What is the next planned task?
```

Do NOT assume the previous AI finished its work.

Do NOT repeat completed work.

Do NOT jump to another feature simply because it is interesting.

Continue from the recorded state.

---

# 30. AI Session Completion

Before ending a development session:

1. Run tests.
2. Record test results.
3. Record modified files.
4. Record architectural decisions.
5. Record unresolved issues.
6. Record next steps.
7. Update DEVELOPMENT_LOG.md.
8. Update CHANGELOG.md if appropriate.
9. Ensure documentation reflects implemented behavior.

The next AI should be able to continue using only the repository.

---

# 31. README.md

Create an extensive root README.

The README should eventually contain:

```text
Project Overview
Why This Project Exists
Goals
Non-Goals
Features
Architecture
Installation
Quick Start
Configuration
Basic Usage
Chat
Presence
Groups
Audio Calls
Video Calls
WebRTC
Authentication
Authorization
MVC Example
Blazor Example
React Example
Next.js Example
Advanced Configuration
Scaling
Performance
Security
Troubleshooting
FAQ
Roadmap
Versioning
Contributing
Development
Testing
Examples
License
```

Do not claim features that have not been implemented.

Clearly distinguish:

```text
Implemented
In Progress
Planned
```

---

# 32. Documentation

Create user documentation separately from the README.

The documentation must answer:

> "I installed this library. How do I actually use it?"

Documentation should eventually contain:

```text
Getting Started
Installation
Configuration
First Chat Application
Authentication
Messaging
Groups
Presence
Typing Indicators
Message States
Calling
WebRTC
STUN/TURN
Production Deployment
Scaling
Troubleshooting
API Reference
Migration Guides
FAQ
```

Use practical examples.

Avoid documentation that merely repeats source code.

---

# 33. Documentation Rule

Every public feature must have documentation.

A feature is NOT considered complete until:

```text
Code
+
Tests
+
Documentation
+
Example
+
Changelog
```

are complete.

---

# 34. XML Documentation

Public APIs should have XML documentation.

For example:

```csharp
/// <summary>
/// Sends a real-time message to the specified user.
/// </summary>
Task SendMessageAsync(
    string userId,
    string message,
    CancellationToken cancellationToken = default);
```

XML documentation should explain behavior, not merely repeat the method name.

---

# 35. API Compatibility

Treat public APIs as contracts.

Before changing a public interface/class:

Ask:

```text
Will this break existing consumers?
Can the change be additive?
Can we preserve compatibility?
Does this belong in the next major version?
```

Use API compatibility tooling when appropriate.

---

# 36. Code Quality

Use:

- Nullable reference types
- Analyzers
- EditorConfig
- Formatting
- Warnings treated appropriately
- Consistent naming
- Small methods
- Clear abstractions
- Dependency inversion

Avoid:

- God classes
- God services
- Massive methods
- Static global state
- Hidden dependencies
- Excessive reflection
- Premature abstractions
- Over-engineering

---

# 37. Dependency Policy

Every dependency must have a reason.

Before adding a NuGet package, evaluate:

```text
Why do we need it?
Can .NET provide this functionality?
Is the project maintained?
What license does it use?
Does it introduce transitive dependencies?
Does it affect package size?
Does it create architectural coupling?
```

Do not add packages merely to save a few lines of code.

---

# 38. Open Source Licensing

Choose an appropriate open-source license during project setup.

Evaluate licenses such as:

- MIT
- Apache-2.0

Consider compatibility with dependencies.

Document the decision.

Do not copy code from projects with incompatible licenses.

Respect third-party licenses.

---

# 39. Git Strategy

Use meaningful commits.

Examples:

```text
feat(core): add connection abstraction
feat(chat): add direct messaging
test(chat): add message delivery tests
docs(chat): add getting started guide
fix(signalr): handle reconnect lifecycle
refactor(core): simplify service registration
```

Avoid commits such as:

```text
update
changes
fix stuff
test
aaa
```

---

# 40. Pull Request Quality

Every significant feature should be reviewable as an isolated unit.

PR descriptions should contain:

```text
Summary
Motivation
Implementation
Architecture
Tests
Breaking Changes
Documentation
Future Work
```

Do not combine unrelated features into one PR.

---

# 41. CI/CD

Eventually create GitHub Actions for:

```text
Build
Test
Format
Static analysis
Package validation
NuGet package creation
Documentation checks
```

Release workflow should eventually support:

```text
Git tag
    ↓
CI
    ↓
Build
    ↓
Test
    ↓
Pack
    ↓
Publish NuGet
    ↓
GitHub Release
```

Do not publish packages automatically until the release process is stable.

---

# 42. NuGet Package Quality

Packages should eventually include:

- XML documentation
- README
- Repository URL
- License
- Package tags
- Version
- Release notes
- Symbol package where appropriate
- Source link where appropriate

The NuGet package should be professionally presented.

---

# 43. Roadmap

Maintain:

```text
ROADMAP.md
```

Use statuses:

```text
[x] Complete
[~] In Progress
[ ] Planned
[-] Deferred
```

Example:

```markdown
## Core

[x] Repository setup
[x] Core abstractions
[~] Connection management
[ ] Distributed connection management

## Chat

[ ] Direct messaging
[ ] Groups
[ ] Presence
[ ] Typing indicators
[ ] Read receipts

## Calls

[ ] Signaling
[ ] Audio
[ ] Video
[ ] Screen sharing
[ ] Call history
```

Never mark a feature complete unless code + tests + docs are complete.

---

# 44. Do Not Implement Future Features Prematurely

If currently implementing messaging, DO NOT suddenly implement:

- Video calls
- Screen sharing
- Redis scaling
- Kubernetes
- SIP
- Mobile SDKs

unless they are required by the current architectural dependency.

Future functionality can influence architecture, but should not cause unnecessary implementation.

---

# 45. Architecture Decision Records

Create:

```text
docs/architecture/decisions/
```

Use ADRs for significant decisions.

Examples:

```text
ADR-001-project-architecture.md
ADR-002-signalr-signaling.md
ADR-003-webrtc-strategy.md
ADR-004-package-boundaries.md
ADR-005-versioning.md
```

Each ADR should contain:

```text
Context
Decision
Alternatives
Consequences
```

---

# 46. AI Must Explain Architectural Changes

Before making a major architectural change, explain:

```text
Problem
Current behavior
Proposed change
Why
Alternatives
Trade-offs
Impact
```

Then implement it.

Do not silently restructure the entire repository.

---

# 47. Avoid Overengineering

This project is intended to be developer-friendly.

Therefore:

Prefer:

```csharp
services.AddRealtime();
```

over:

```csharp
services.AddRealtime(
    new RealtimeBuilder(
        new InternalConfigurationProvider(
            new ServiceRegistrationStrategy(...))));
```

If a developer needs 15 lines of configuration to send a message, the API needs improvement.

---

# 48. Developer Experience Test

For every major feature, ask:

> "Can a developer understand this feature in under 10 minutes?"

If not:

- simplify the API,
- improve documentation,
- provide a sample,
- improve error messages,
- improve IntelliSense.

---

# 49. Backward Compatibility

Once version 1.0 is released:

Avoid breaking public APIs unnecessarily.

Prefer:

```text
Additive changes
```

over:

```text
Breaking changes
```

Breaking changes should require strong justification.

---

# 50. Performance Principles

Do not optimize blindly.

First establish:

```text
Correctness
↓
Tests
↓
Measurement
↓
Optimization
```

For real-time systems, pay special attention to:

- allocations
- serialization
- network traffic
- connection management
- concurrent collections
- locking
- task scheduling
- memory leaks
- event subscriptions

---

# 51. Resource Management

Every connection, timer, subscription, stream, and disposable resource must have a clear lifecycle.

Check for:

- IDisposable
- IAsyncDisposable
- CancellationToken
- event handler cleanup
- connection cleanup

Real-time systems must not leak resources.

---

# 52. Observability

Eventually support:

- Logging
- Metrics
- Diagnostics
- Connection statistics
- Error reporting hooks

Do not force users to install a specific monitoring platform.

Use standard .NET abstractions where possible.

---

# 53. Extensibility

Design extension points carefully.

Potential future interfaces:

```csharp
IMessageStore
IPresenceStore
ICallStore
IConnectionManager
ISignalingProvider
IMediaProvider
IUserResolver
```

Do not create all of these interfaces unless there is a concrete requirement.

Abstractions should solve real problems.

---

# 54. Persistence

Persistence should be optional.

The core real-time communication layer should not necessarily require a database.

Potential future integrations:

```text
In-memory
EF Core
Redis
Custom storage
```

Do not couple the core package directly to Entity Framework Core.

---

# 55. Distributed Systems

Eventually consider:

```text
Server A
Server B
Server C
     │
     ▼
Redis / Backplane
```

But do not implement distributed infrastructure until the single-server architecture is stable.

---

# 56. Frontend Client Design

If a JavaScript/TypeScript client is created, it must be designed independently from React/Next.js.

Core client:

```text
@your-library/client
```

Framework adapters may eventually exist:

```text
@your-library/react
```

Do not put React-specific code into the core client.

---

# 57. MVC Example

The MVC example should demonstrate:

```text
ASP.NET Core MVC
     ↓
Your .NET library
     ↓
SignalR
     ↓
Real-time chat
```

It should show developers how to integrate the library into a traditional ASP.NET application.

---

# 58. Blazor Example

Support:

- Blazor Server where appropriate
- Blazor WebAssembly where appropriate

Clearly document differences.

Do not assume browser and server execution environments behave identically.

---

# 59. React Example

Create a minimal React example demonstrating:

```text
Install package
↓
Create client
↓
Connect
↓
Send message
↓
Receive message
↓
Disconnect
```

Later extend it for calling.

---

# 60. Next.js Example

Create a Next.js example demonstrating:

- Client/server boundaries
- Browser-only WebRTC APIs
- SignalR connection
- Authentication considerations
- Calling

Be careful with:

```text
window
navigator.mediaDevices
RTCPeerConnection
```

because they are browser APIs.

---

# 61. Generated Documentation Must Be Honest

Never document:

```text
Coming soon
```

as if it already works.

Use explicit labels:

```text
Available
Experimental
Preview
Planned
```

---

# 62. Experimental Features

Potentially unstable functionality may be marked:

```text
[Experimental]
```

or an equivalent project convention.

Experimental APIs must be clearly documented.

---

# 63. User Feedback

The project should eventually make it easy for users to:

- Report bugs
- Request features
- Ask questions
- Contribute code

Create:

```text
CONTRIBUTING.md
```

with clear instructions.

---

# 64. Final Definition of Done

A feature is ONLY complete when all applicable items are finished:

```text
[ ] Architecture designed
[ ] API reviewed
[ ] Implementation complete
[ ] Unit tests complete
[ ] Integration tests complete
[ ] Edge cases tested
[ ] Documentation complete
[ ] XML documentation complete
[ ] Example updated
[ ] README updated
[ ] CHANGELOG updated
[ ] ROADMAP updated
[ ] DEVELOPMENT_LOG updated
[ ] Formatting passes
[ ] Analyzer passes
[ ] Build passes
[ ] Tests pass
```

---

# 65. Most Important AI Rule

NEVER behave like:

> "The user mentioned video calls, therefore immediately implement video calls."

Instead behave like:

> "Where are we in the roadmap? What dependencies must exist before video calls? Is the current architecture ready? What should be implemented next?"

Always follow the dependency graph.

---

# 66. Session Startup Checklist

At the beginning of EVERY AI coding session:

```text
1. Read AGENTS.md
2. Read DEVELOPMENT_LOG.md
3. Read ROADMAP.md
4. Read README.md
5. Inspect git status
6. Inspect current project structure
7. Run existing tests if practical
8. Identify current phase
9. Identify current feature
10. Identify next task
```

Then report briefly:

```text
Current Phase:
Current Feature:
Completed:
In Progress:
Next Task:
Known Issues:
```

Only then begin implementation.

---

# 67. Session Ending Checklist

Before ending EVERY meaningful coding session:

```text
1. Build
2. Test
3. Check formatting
4. Check analyzers
5. Review changed files
6. Update DEVELOPMENT_LOG.md
7. Update ROADMAP.md
8. Update CHANGELOG.md when appropriate
9. Update documentation
10. Record next task
```

---

# 68. First Implementation Milestone

The FIRST milestone should NOT be chat.

The first milestone should establish the foundation.

Suggested first milestone:

```text
Milestone 0.1.0

[x] Repository structure
[x] Solution
[x] Core projects
[x] Test projects
[x] Documentation structure
[x] CI skeleton
[x] Versioning
[x] Development log
[x] Roadmap
[x] Initial README
[ ] Core abstractions
[ ] DI foundation
[ ] Connection abstraction
```

After that, proceed incrementally.

---

# 69. First Feature Sequence

Unless architectural analysis indicates otherwise, follow this sequence:

```text
1. Repository foundation
2. Core abstractions
3. DI infrastructure
4. Connection lifecycle
5. SignalR integration
6. Direct messaging
7. Message events
8. Groups
9. Presence
10. Typing indicators
11. Delivery/read states
12. Authentication
13. Authorization
14. WebRTC signaling
15. Audio calling
16. Video calling
17. Screen sharing
18. Persistence
19. Distributed infrastructure
20. Performance
21. Production hardening
```

Do not skip directly to step 15.

---

# 70. Final Objective

The final product should feel like a natural part of the .NET ecosystem.

A developer should be able to think:

> "I need real-time chat."

Install the package.

Configure a few lines.

Start using it.

Then later:

> "I need audio/video calls."

Install the optional calling package.

Configure it.

Start using it.

The complexity of:

- SignalR
- WebRTC
- SDP
- ICE
- STUN
- TURN
- connection lifecycle
- reconnection
- media handling

should be abstracted as much as reasonably possible without preventing advanced developers from accessing lower-level functionality when needed.

The project should provide both:

```text
Simple API
```

and:

```text
Advanced/extensible API
```

---

# 71. Golden Rule

**Build slowly. Build correctly. Build linearly. Test everything. Document everything. Record every important decision.**

Never sacrifice architectural quality merely to increase the number of implemented features.

The goal is not to produce the largest library.

The goal is to produce a **high-quality, developer-friendly, maintainable, extensible open-source .NET real-time communication library that developers genuinely enjoy using.**

###Important
.NET Target Framework Policy

The initial release of this project targets .NET 10.

Primary target:

- net10.0

The project should use the latest stable .NET 10 SDK available at development time.

Do not introduce support for additional target frameworks without an explicit architectural/compatibility decision.

Before adding multi-targeting, evaluate:

- API compatibility
- dependency compatibility
- SignalR compatibility
- WebRTC/SIPSorcery compatibility
- testing matrix
- CI complexity
- package size
- maintenance burden

.NET 10 should remain the reference implementation until the project reaches sufficient maturity.

Future multi-targeting may be introduced if there is a clear benefit for library consumers.

Global.json to pin the SDK family used by the repository:

{
"sdk": {
"version": "10.0.302",
"rollForward": "latestFeature"
}
}

Project Name an be `LiveSharp`.
