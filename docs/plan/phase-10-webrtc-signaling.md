# Phase 10 — WebRTC Signalling

| | |
|---|---|
| **Version at exit** | `0.9.0` (tag `v0.9.0`) |
| **Depends on** | Phase 09 (specifically `P09.T5`) |
| **Branch** | `feat/phase-10-webrtc-signaling` |
| **Tasks** | `P10.T1` … `P10.T6` |
| **Read before starting** | `docs/architecture/decisions/ADR-006`, `docs/security/authorization.md`, `EXECUTION_PLAN.md` §4 (deviation D2) |

---

## Goal

A new package, `LiveSharp.Signaling`, that relays SDP and ICE between two authenticated peers and
hands out ICE server configuration with ephemeral TURN credentials.

The server is a **postbox**. It does not parse SDP, does not know about codecs, does not touch media,
and has no opinion about what the peers negotiate. That constraint is what makes this package small,
stable, and immune to browser-version churn.

```
Browser A ══════════ media (SRTP, peer to peer or via TURN) ══════════ Browser B
    │                                                                      │
    └──────── signalling (SDP, ICE) ──▶ LiveSharp ◀── signalling ──────────┘
                                      ASP.NET Core
```

---

## Naming: `LiveSharp.Signaling`, not `LiveSharp.WebRTC`

Deviation **D2**, already accepted. The rationale, restated here because this is the phase where it
matters: the package contains zero WebRTC code. Calling it `.WebRTC` would advertise media capability
it does not have, and would leave no sensible name for the package that *does* handle server-side
media (`LiveSharp.Media.SIPSorcery`, post-1.0).

Downside to mitigate: someone searching NuGet for "webrtc" will not find it. Add `webrtc`, `sdp`,
`ice`, `stun`, `turn`, and `signaling` to `PackageTags`, and say "WebRTC signalling" in the package
description and the first line of the README.

---

## Non-goals — DO NOT implement in this phase

- **No call semantics.** No `Call`, no ringing, no accept/reject, no call state. A signalling session
  is a plumbing primitive; a call is a product concept built on it. Phase 11.
- **No media.** No SIPSorcery, no server-side `RTCPeerConnection`, no SFU, no MCU, no recording, no
  transcoding, no mixing, no simulcast, no bandwidth estimation.
- **No SDP parsing, munging, filtering, or validation beyond size.** See the callout in `P10.T1`.
- **No TURN server.** LiveSharp *configures* a TURN server (coturn, or a managed service). It is not
  one, and never will be.
- No group/multi-party sessions. A `SignalingSession` has exactly two participants. Mesh and SFU
  topologies are out of scope for 1.0.
- No data-channel abstraction. The client helper establishes a peer connection; what the application
  puts on it is the application's business.
- No persistence. Sessions are in-memory and node-local; Phase 15 handles cross-node.

---

## Tasks

### P10.T1 — Package, model, and options

**Deliverables**

```
src/LiveSharp.Signaling/LiveSharp.Signaling.csproj
src/LiveSharp.Signaling/PublicAPI.{Shipped,Unshipped}.txt
src/LiveSharp.Signaling/AssemblyInfo.cs
src/LiveSharp.Signaling/SessionDescription.cs
src/LiveSharp.Signaling/SdpType.cs
src/LiveSharp.Signaling/IceCandidate.cs
src/LiveSharp.Signaling/SignalingSession.cs
src/LiveSharp.Signaling/SignalingSessionState.cs
src/LiveSharp.Signaling/Configuration/SignalingOptions.cs
src/LiveSharp.Signaling/Configuration/SignalingOptionsValidator.cs
src/LiveSharp.Signaling/Diagnostics/SignalingLog.cs
tests/LiveSharp.Signaling.Tests/…
```

**Public API (exact):**

```csharp
namespace LiveSharp.Signaling;

public enum SdpType
{
    Offer    = 0,
    Answer   = 1,
    Rollback = 2,
}

/// <summary>An opaque SDP blob. LiveSharp never parses the contents.</summary>
public sealed record SessionDescription(SdpType Type, string Sdp);

/// <summary>An opaque ICE candidate. LiveSharp never parses the contents.</summary>
public sealed record IceCandidate(
    string Candidate,
    string? SdpMid = null,
    int? SdpMLineIndex = null,
    string? UsernameFragment = null);

public enum SignalingSessionState
{
    New         = 0,
    OfferSent   = 1,
    AnswerSent  = 2,
    Established = 3,
    Closed      = 4,
    Failed      = 5,
}

/// <summary>A two-party signalling relay session.</summary>
public sealed record SignalingSession
{
    public required string SessionId { get; init; }
    public required string InitiatorUserId { get; init; }
    public required string PeerUserId { get; init; }
    public required SignalingSessionState State { get; init; }
    public required DateTimeOffset CreatedAt { get; init; }
    public required DateTimeOffset UpdatedAt { get; init; }

    /// <summary>Number of ICE candidates relayed. Used to enforce the per-session cap.</summary>
    public required int CandidateCount { get; init; }

    /// <summary>Application metadata. Never interpreted by LiveSharp. Size-limited.</summary>
    public IReadOnlyDictionary<string, string> Metadata { get; init; }

    /// <summary>True when the user is one of the two participants.</summary>
    public bool IsParticipant(string userId);

    /// <summary>Returns the other participant's user identifier, or null if not a participant.</summary>
    public string? GetPeerOf(string userId);
}
```

> ## Do not parse the SDP
> It will be tempting: validate the `m=` lines, strip codecs you do not like, check that the offer
> matches the media kind the call claimed. Every one of those makes the server a codec-policy
> gatekeeper that breaks on the next Chrome release, and none of them provides a security property
> that matters — a hostile peer can put anything in an SDP the *other peer* will accept or reject on
> its own merits.
>
> Validate exactly two things: the byte length against `MaxSdpBytes`, and that the string is non-empty.
> Nothing else. Put this paragraph in the XML docs on `SessionDescription`.

**Public API (exact) — options:**

```csharp
public sealed class SignalingOptions
{
    public const string ConfigurationSectionName = "LiveSharp:Signaling";

    /// <summary>Maximum SDP size in UTF-8 bytes. Default 64 KiB.</summary>
    public int MaxSdpBytes { get; set; } = 64 * 1024;

    /// <summary>Maximum ICE candidate size in UTF-8 bytes. Default 1 KiB.</summary>
    public int MaxCandidateBytes { get; set; } = 1024;

    /// <summary>Maximum ICE candidates relayed per session. Default 200.</summary>
    public int MaxCandidatesPerSession { get; set; } = 200;

    /// <summary>Sessions with no activity for this long are closed. Default 2 minutes.</summary>
    public TimeSpan SessionIdleTimeout { get; set; } = TimeSpan.FromMinutes(2);

    /// <summary>Idle sweep interval. Default 30 seconds.</summary>
    public TimeSpan SweepInterval { get; set; } = TimeSpan.FromSeconds(30);

    /// <summary>Maximum concurrent sessions per user. Default 4.</summary>
    public int MaxSessionsPerUser { get; set; } = 4;

    /// <summary>Maximum candidates buffered before a remote description arrives. Default 50.</summary>
    public int MaxBufferedCandidates { get; set; } = 50;

    /// <summary>How long buffered candidates are retained. Default 30 seconds.</summary>
    public TimeSpan CandidateBufferTtl { get; set; } = TimeSpan.FromSeconds(30);
}
```

**Requirements**

- `SessionId` from `Guid.CreateVersion7()` (`"n"` format), reusing `IMessageIdGenerator` if it is
  accessible without depending on `LiveSharp.Chat` — it is not, so add a tiny
  `ISignalingIdGenerator` in this package rather than referencing chat. Note in a comment that the two
  generators are intentionally duplicated to keep the packages independent, and that if a third appears,
  the abstraction moves to `LiveSharp.Abstractions`.
- Session IDs being unguessable is **defence in depth, not the security boundary**. Authorization is
  the boundary (`P10.T3`). Say this in the XML docs so nobody later concludes the ID is sufficient.
- `IsParticipant` / `GetPeerOf` on the record rather than in the service: they are pure, they are
  needed in three places, and having them on the model means the authorization check reads as
  `session.IsParticipant(userId)` instead of two string comparisons someone will eventually get
  backwards.
- Validator rules: `MaxSdpBytes` in `(0, 256 KiB]` — an SDP above 256 KiB is pathological, and the
  message should say so; `MaxCandidateBytes` in `(0, 4 KiB]`; `SweepInterval < SessionIdleTimeout`;
  `CandidateBufferTtl <= SessionIdleTimeout`; all counts positive; `MaxSdpBytes` must not exceed
  `LiveSharpOptions.Limits.MaxPayloadBytes` — cross-option validation with an actionable message,
  because an SDP limit larger than the payload limit is unreachable configuration.

**Tests**

| Test | Asserts |
|---|---|
| `Sdp_type_and_session_state_numeric_values_are_pinned` | wire contract |
| `Is_participant_is_true_for_both_participants` | |
| `Is_participant_is_false_for_a_third_party` | |
| `Get_peer_of_returns_the_other_participant` | both directions |
| `Get_peer_of_returns_null_for_a_non_participant` | |
| `Session_metadata_defaults_to_empty` | |
| `Session_id_generator_produces_unique_sortable_ids` | 10 000 |
| `Options_defaults_are_valid` | |
| `Max_sdp_bytes_above_the_core_payload_limit_is_rejected_with_guidance` | cross-option |
| `Max_sdp_bytes_above_256_kibibytes_is_rejected` | |
| `Sweep_interval_not_less_than_the_idle_timeout_is_rejected` | |
| `Candidate_buffer_ttl_above_the_idle_timeout_is_rejected` | |

---

### P10.T2 — Session store, service, and idle sweep

**Deliverables**

```
src/LiveSharp.Signaling/Storage/ISignalingSessionStore.cs
src/LiveSharp.Signaling/Storage/InMemorySignalingSessionStore.cs
src/LiveSharp.Signaling/ISignalingService.cs
src/LiveSharp.Signaling/SignalingService.cs
src/LiveSharp.Signaling/CandidateBuffer.cs
src/LiveSharp.Signaling/SignalingSweepService.cs
src/LiveSharp.Signaling/SignalingLifecycleHandler.cs
tests/LiveSharp.Signaling.Tests/Storage/SignalingSessionStoreContractTests.cs
tests/LiveSharp.Signaling.Tests/…
```

**Public API (exact):**

```csharp
namespace LiveSharp.Signaling;

public interface ISignalingService
{
    Task<OperationResult<SignalingSession>> CreateSessionAsync(string initiatorUserId, string peerUserId, IReadOnlyDictionary<string, string>? metadata = null, CancellationToken cancellationToken = default);

    Task<OperationResult> SendOfferAsync(string senderUserId, string sessionId, SessionDescription description, RealtimeOperationContext? context = null, CancellationToken cancellationToken = default);
    Task<OperationResult> SendAnswerAsync(string senderUserId, string sessionId, SessionDescription description, RealtimeOperationContext? context = null, CancellationToken cancellationToken = default);
    Task<OperationResult> SendCandidateAsync(string senderUserId, string sessionId, IceCandidate candidate, RealtimeOperationContext? context = null, CancellationToken cancellationToken = default);

    Task<OperationResult> CloseSessionAsync(string actingUserId, string sessionId, SignalingCloseReason reason, CancellationToken cancellationToken = default);
    Task<OperationResult<SignalingSession>> GetSessionAsync(string actingUserId, string sessionId, CancellationToken cancellationToken = default);
}

public enum SignalingCloseReason
{
    Normal            = 0,
    PeerDisconnected  = 1,
    IdleTimeout       = 2,
    Failed            = 3,
    Superseded        = 4,
}
```

```csharp
namespace LiveSharp.Signaling.Storage;

public interface ISignalingSessionStore
{
    ValueTask AddAsync(SignalingSession session, CancellationToken cancellationToken = default);
    ValueTask<SignalingSession?> FindAsync(string sessionId, CancellationToken cancellationToken = default);
    ValueTask<bool> TryUpdateAsync(SignalingSession session, DateTimeOffset expectedUpdatedAt, CancellationToken cancellationToken = default);
    ValueTask<bool> RemoveAsync(string sessionId, CancellationToken cancellationToken = default);
    ValueTask<IReadOnlyList<SignalingSession>> GetSessionsForUserAsync(string userId, CancellationToken cancellationToken = default);
    ValueTask<int> GetSessionCountForUserAsync(string userId, CancellationToken cancellationToken = default);
    ValueTask<IReadOnlyList<SignalingSession>> GetIdleAsync(DateTimeOffset olderThan, int limit, CancellationToken cancellationToken = default);
}
```

**Requirements**

- `TryUpdateAsync` takes `expectedUpdatedAt` for optimistic concurrency, exactly like Phase 06's
  `IPresenceStore.TrySetAsync`. Two peers trickling candidates simultaneously will race on
  `CandidateCount`; last-write-wins there means the cap can be exceeded. Retry up to 3 times, then
  `Failure(Conflict, …)`.
- Session state transitions: `New → OfferSent → AnswerSent → Established`, plus `→ Closed` and
  `→ Failed` from anywhere. A renegotiation offer on an `Established` session is legal and moves it
  back to `OfferSent`. Implement as a small table like Phase 08's `DeliveryStateMachine`, and test it
  exhaustively — same reasoning, same cost, same payoff.
- `Established` is set when the first candidate is relayed **after** `AnswerSent`. The server cannot
  observe the actual ICE connection state; `Established` here means "both descriptions exchanged and
  candidates flowing". Document that it is a signalling-level state, not an RTC-level one, or someone
  will build a UI on it and wonder why it lies.
- **Candidate buffering.** Candidates routinely arrive before the peer has applied a remote
  description. The server does not know the peer's `RTCPeerConnection` state, so it cannot decide when
  it is safe to deliver. Two options:
  - relay immediately and make it the client's problem;
  - buffer server-side until the corresponding description has been relayed.

  Choose **relay immediately**, and buffer only for the narrow case where the *peer has no live
  connection yet* (a session created before the peer connected). Rationale: `RTCPeerConnection`
  already buffers early candidates correctly when the client uses the perfect-negotiation pattern, and
  duplicating that logic server-side means two buffers with different bugs. `CandidateBuffer` therefore
  exists only for the offline-peer case, bounded by `MaxBufferedCandidates` and `CandidateBufferTtl`,
  and drops with a `Debug` log when full. Document the division of responsibility clearly — this is the
  question every WebRTC implementer asks.
- `MaxCandidatesPerSession` exceeded → `Failure(RateLimited, …)`. A peer trickling unbounded candidates
  is a relay-amplification vector, and 200 is already far more than any real negotiation needs.
- `SignalingLifecycleHandler : IConnectionLifecycleHandler`: on a user's **last** disconnect, close
  every session they participate in with `PeerDisconnected` and notify the peer. Not on any disconnect —
  a user with two devices may be signalling from one and closing the other.
- `SignalingSweepService : BackgroundService` at `SweepInterval`: close idle sessions with
  `IdleTimeout`, notify both peers, prune expired candidate buffers. Same resilience contract as the
  Phase 02 reaper.
- Never log SDP or candidate contents at any level unless `Diagnostics.AllowSensitivePayloadLogging`.
  Candidates contain IP addresses — including private LAN addresses and, with TURN, the relay address.
  That is PII and network-topology disclosure. Log sizes and counts, never contents.

**Tests**

| Test | Asserts |
|---|---|
| `Create_session_stores_it_in_the_new_state` | |
| `Create_session_stamps_timestamps_from_the_time_provider` | |
| `Create_session_beyond_the_per_user_limit_returns_conflict_naming_the_limit` | |
| `Create_session_with_yourself_as_peer_is_rejected` | |
| `Offer_moves_the_session_to_offer_sent` | |
| `Answer_moves_the_session_to_answer_sent` | |
| `Candidate_after_answer_moves_the_session_to_established` | |
| `Renegotiation_offer_on_an_established_session_returns_it_to_offer_sent` | |
| `Every_state_transition_pair_has_a_defined_result` | exhaustive table |
| `Offer_on_a_closed_session_is_rejected` | |
| `Offer_exceeding_the_sdp_size_limit_is_rejected` | limit + 1 byte, measured in UTF-8 |
| `Offer_at_exactly_the_sdp_size_limit_is_accepted` | boundary |
| `Empty_sdp_is_rejected` | |
| `Sdp_containing_unusual_codec_lines_is_relayed_unchanged` | **pins the no-parsing contract** |
| `Candidate_exceeding_the_size_limit_is_rejected` | |
| `Candidate_beyond_the_per_session_cap_returns_rate_limited` | |
| `Candidate_to_an_offline_peer_is_buffered` | |
| `Buffered_candidates_are_delivered_when_the_peer_connects` | |
| `Buffered_candidates_beyond_the_buffer_cap_are_dropped_with_a_debug_log` | |
| `Buffered_candidates_expire_after_the_buffer_ttl` | `FakeTimeProvider` |
| `Close_notifies_the_other_peer_with_the_reason` | |
| `Close_of_an_already_closed_session_is_idempotent` | |
| `Close_removes_the_session_from_the_store` | |
| `Last_disconnect_closes_every_session_for_that_user_and_notifies_peers` | |
| `Non_last_disconnect_leaves_sessions_open` | multi-device |
| `Sweep_closes_idle_sessions_and_notifies_both_peers` | |
| `Sweep_does_not_close_a_session_at_exactly_the_idle_timeout` | boundary |
| `Sweep_survives_a_throwing_transport` | |
| `Sdp_content_is_never_logged_by_default` | **security test** |
| `Candidate_content_is_never_logged_by_default` | **security test** |
| `Concurrent_candidates_from_both_peers_never_exceed_the_cap` | `ConcurrencyHarness` |
| `Concurrent_state_updates_converge_without_losing_the_candidate_count` | |
| `All_methods_honour_a_pre_cancelled_token` | |

---

### P10.T3 — Relay authorization

> ## An unauthorized signalling relay is a covert messaging channel
> `signaling.offer` takes a session ID and a 64 KiB opaque string and delivers it to another user. If
> the only check is "the session exists", then any authenticated user who can guess or obtain a session
> ID can push 64 KiB of arbitrary content to a stranger, repeatedly, with no rate accounting attributable
> to a conversation. That is an unmoderated messaging system with a WebRTC-shaped hat on.
>
> Every relay operation needs **two** independent checks. Both must pass.

**Deliverables**

```
src/LiveSharp.Signaling/Security/SignalingAuthorization.cs
src/LiveSharp.Signaling/Security/ISignalingAuthorizationService.cs
tests/LiveSharp.Signaling.Tests/Security/RelayAuthorizationTests.cs
tests/LiveSharp.IntegrationTests/Signaling/SignalingSecurityTests.cs
```

**Requirements — the two gates**

1. **Participation.** `session.IsParticipant(senderUserId)` must be true. The destination is
   **derived** from the session via `GetPeerOf(senderUserId)` and is **never taken from the request
   payload.** `SendOfferRequest` and friends must have no `RecipientId`/`PeerUserId` member at all —
   assert its absence by reflection, exactly as Phases 04–08 do for `SenderId`.
2. **Application permission.** `IRealtimeAuthorizationService.CanSendToUserAsync(sender, peer)` from
   Phase 09, plus a signalling-specific `ISignalingAuthorizationService.CanEstablishSessionAsync(initiator, peer)`
   checked at `CreateSessionAsync`. The second exists because "may send a chat message" and "may open a
   media channel" are legitimately different permissions in most products — a user might be allowed to
   message a support agent but not call them.

```csharp
namespace LiveSharp.Signaling.Security;

public interface ISignalingAuthorizationService
{
    /// <summary>Whether <paramref name="initiatorUserId"/> may open a signalling session with the peer.</summary>
    ValueTask<bool> CanEstablishSessionAsync(string initiatorUserId, string peerUserId, CancellationToken cancellationToken = default);
}
```

Default implementation delegates to `IRealtimeAuthorizationService.CanSendToUserAsync` — a sensible
default that is not silently permissive, because Phase 09's `AllowAllAuthorizationService` already
logs an error at startup when nothing real is configured.

**Requirements — error responses**

- Non-participant → `Forbidden`. Unknown session → also `Forbidden`, with an **identical message**.
  Distinguishing them is a session-ID oracle that turns guessing into enumeration. Same rule Phase 05
  applied to groups.
- Authorization is checked **before** size validation and before any store write, so a forbidden request
  does the least possible work.
- Every authorization failure logs at `Information` with actor, session ID, and correlation ID — but not
  the peer's user ID (that would let a log reader reconstruct a social graph from failed attempts).

**Attack tests — these are the point of the task**

| Test | Asserts |
|---|---|
| `Offer_from_a_non_participant_is_forbidden` | |
| `Answer_from_a_non_participant_is_forbidden` | |
| `Candidate_from_a_non_participant_is_forbidden` | |
| `Close_by_a_non_participant_is_forbidden` | |
| `Get_session_by_a_non_participant_is_forbidden` | |
| `Forged_session_id_is_forbidden_not_not_found` | **the oracle test**: identical message to the non-participant case |
| `Relay_destination_cannot_be_supplied_by_the_payload` | reflection: no peer member on any request DTO |
| `Create_session_denied_by_the_signaling_authorization_service_is_forbidden` | |
| `Create_session_allowed_by_chat_but_denied_by_signaling_is_forbidden` | **the two-permission distinction** |
| `Create_session_denied_by_can_send_to_user_is_forbidden` | |
| `Replaying_an_offer_on_a_closed_session_is_rejected` | |
| `Authorization_is_checked_before_the_sdp_size_check` | a forbidden oversized offer reports forbidden |
| `Authorization_is_checked_before_any_store_write` | store untouched |
| `Authorization_failure_does_not_log_the_peer_user_id` | **security test** |
| `Session_id_enumeration_yields_no_information` | 100 random IDs, all identical responses |
| `A_user_cannot_join_an_existing_session_by_sending_an_answer_to_it` | third-party hijack |

---

### P10.T4 — ICE servers and ephemeral TURN credentials

> ## TURN credentials are money
> A TURN server relays media. Leaked long-lived credentials mean a stranger relaying arbitrary
> bandwidth through your infrastructure, billed to you, and attributable to you. This is the single
> highest-consequence security surface in the entire library.

**Deliverables**

```
src/LiveSharp.Signaling/Ice/IceServer.cs
src/LiveSharp.Signaling/Ice/IIceServerProvider.cs
src/LiveSharp.Signaling/Ice/StaticIceServerProvider.cs
src/LiveSharp.Signaling/Ice/EphemeralTurnCredentialProvider.cs
src/LiveSharp.Signaling/Configuration/IceOptions.cs
src/LiveSharp.Signaling/Configuration/IceOptionsValidator.cs
tests/LiveSharp.Signaling.Tests/Ice/…
```

**Public API (exact):**

```csharp
namespace LiveSharp.Signaling;

public sealed record IceServer(
    IReadOnlyList<string> Urls,
    string? Username = null,
    string? Credential = null);

/// <summary>Supplies the ICE server list a specific user should use.</summary>
public interface IIceServerProvider
{
    ValueTask<IReadOnlyList<IceServer>> GetIceServersAsync(string userId, CancellationToken cancellationToken = default);
}
```

```csharp
public sealed class IceOptions
{
    public const string ConfigurationSectionName = "LiveSharp:Signaling:Ice";

    /// <summary>STUN server URLs. Credentials are never required for STUN.</summary>
    public IList<string> StunUrls { get; } = new List<string>();

    /// <summary>TURN server URLs.</summary>
    public IList<string> TurnUrls { get; } = new List<string>();

    /// <summary>Shared secret matching the TURN server's static-auth-secret. Load from a secret store.</summary>
    public string? TurnSharedSecret { get; set; }

    /// <summary>Lifetime of generated TURN credentials. Default 1 hour. Maximum 24 hours.</summary>
    public TimeSpan TurnCredentialTtl { get; set; } = TimeSpan.FromHours(1);

    /// <summary>Static TURN username. Requires <see cref="IUnderstandStaticTurnCredentialsAreInsecure"/>.</summary>
    public string? StaticTurnUsername { get; set; }

    /// <summary>Static TURN password. Requires <see cref="IUnderstandStaticTurnCredentialsAreInsecure"/>.</summary>
    public string? StaticTurnPassword { get; set; }

    /// <summary>Must be explicitly set to true to permit static, long-lived, shared TURN credentials.</summary>
    public bool IUnderstandStaticTurnCredentialsAreInsecure { get; set; }
}
```

The flag name is deliberately embarrassing. Nobody sets
`IUnderstandStaticTurnCredentialsAreInsecure = true` in a code review without someone asking why.
Keep the name; do not soften it to `AllowStaticCredentials`.

**Requirements — the credential algorithm**

Implement the coturn `static-auth-secret` / TURN REST API scheme exactly:

```
expiryUnixSeconds = timeProvider.GetUtcNow().Add(TurnCredentialTtl).ToUnixTimeSeconds()
username          = $"{expiryUnixSeconds}:{userId}"
credential        = Convert.ToBase64String(HMACSHA1(key: TurnSharedSecret_utf8, data: username_utf8))
```

- HMAC-SHA1 is not a choice — it is what coturn implements. Add a comment saying so, because a
  security reviewer will flag SHA-1 and needs to find the answer next to the code. Note that it is used
  as an HMAC (not for collision resistance) and that the scheme is defined by the TURN server.
- Include a **known-answer test vector** with a fixed secret, fixed user ID, and fixed expiry, so a
  refactor cannot silently change the output. Cross-check the vector against coturn's documented
  example if one is available; otherwise generate it once, verify it against a real coturn instance
  manually, and record the provenance in a comment.
- `userId` must be sanitised: a user ID containing `:` would let a user shift the expiry field. Reject
  user IDs containing `:` at credential-generation time with a clear error, or hash the user ID into
  the username. Choose **reject with a clear error** — silently hashing breaks TURN server-side
  per-user accounting. Test it.
- Credentials are per-user and short-lived. `GetIceServersAsync` is authenticated
  (`signaling.iceservers.get` requires authentication like every other operation).
- `TurnSharedSecret` must **never** be logged, never appear in an exception message, and never be
  returned in any response. Add tests for all three.
- Validator rules: `TurnUrls` non-empty requires either `TurnSharedSecret` **or**
  (`StaticTurnUsername` + `StaticTurnPassword` + the embarrassing flag), otherwise a startup error
  explaining both options; `TurnCredentialTtl` in `(0, 24h]`; STUN URLs must not carry credentials;
  URLs must parse and use a `stun:`/`stuns:`/`turn:`/`turns:` scheme. Log a `Warning` at startup when
  `turn:` (unencrypted) is used without any `turns:` alternative.
- `StaticIceServerProvider` handles the STUN-only case with no credentials at all — the correct
  configuration for a development environment and for deployments where relay is not needed.

**Tests**

| Test | Asserts |
|---|---|
| `Static_provider_returns_configured_stun_urls_with_no_credentials` | |
| `Ephemeral_provider_matches_the_known_answer_vector` | **the regression anchor** |
| `Ephemeral_username_is_expiry_colon_user_id` | |
| `Ephemeral_expiry_is_now_plus_the_configured_ttl` | `FakeTimeProvider` |
| `Ephemeral_credential_is_base64_of_the_hmac_of_the_username` | recomputed independently in the test |
| `Two_users_receive_different_credentials` | |
| `The_same_user_receives_a_new_expiry_on_a_later_call` | |
| `A_user_id_containing_a_colon_is_rejected_with_a_clear_error` | **the injection test** |
| `Shared_secret_never_appears_in_the_returned_ice_servers` | **security test** |
| `Shared_secret_never_appears_in_logs` | **security test** |
| `Shared_secret_never_appears_in_a_validation_exception_message` | **security test** |
| `Turn_urls_without_a_secret_or_static_credentials_fail_at_startup_naming_both_options` | |
| `Static_credentials_without_the_acknowledgement_flag_fail_at_startup` | |
| `Static_credentials_with_the_acknowledgement_flag_are_permitted` | |
| `Ttl_above_twenty_four_hours_is_rejected` | |
| `Non_positive_ttl_is_rejected` | |
| `Malformed_url_is_rejected_naming_the_url` | table-driven over schemes |
| `Stun_url_carrying_credentials_is_rejected` | |
| `Unencrypted_turn_without_a_turns_alternative_logs_a_startup_warning` | |
| `Ice_servers_operation_requires_authentication` | integration |

---

### P10.T5 — Operations, handlers, and `AddSignaling()`

**Deliverables**

```
src/LiveSharp.Signaling/Protocol/*.cs
src/LiveSharp.Signaling/Handlers/CreateSessionHandler.cs
src/LiveSharp.Signaling/Handlers/SendOfferHandler.cs
src/LiveSharp.Signaling/Handlers/SendAnswerHandler.cs
src/LiveSharp.Signaling/Handlers/SendCandidateHandler.cs
src/LiveSharp.Signaling/Handlers/CloseSessionHandler.cs
src/LiveSharp.Signaling/Handlers/GetIceServersHandler.cs
src/LiveSharp.Signaling/DependencyInjection/LiveSharpSignalingBuilderExtensions.cs
tests/LiveSharp.Signaling.Tests/Handlers/…
tests/LiveSharp.IntegrationTests/Signaling/SignalingIntegrationTests.cs
```

Operations and events — add to `RealtimeOperations`/`RealtimeEvents`, register every DTO in
`LiveSharpJsonSerializerContext`, mirror in `protocol.ts`, and add every operation to the Phase 09
`KnownOperationPolicies` manifest (the manifest test will fail otherwise — that is the mechanism
working as intended):

```
signaling.session.create      InvokeAsync    signaling.offer.received
signaling.offer               InvokeAsync    signaling.answer.received
signaling.answer              InvokeAsync    signaling.candidate.received
signaling.candidate           NotifyAsync    signaling.session.closed
signaling.session.close       InvokeAsync
signaling.iceservers.get      InvokeAsync
```

**Requirements**

- `signaling.candidate` is `NotifyAsync`. Trickle ICE produces a dozen or more candidates per peer in
  the first second of negotiation; a round trip each is waste, and a lost candidate degrades to a
  slower or relayed connection rather than a failure. Consistent with the Phase 07/08 reasoning: state
  it and cross-reference.
- All other operations are `InvokeAsync` — the caller needs to know an offer was accepted for relay
  before waiting for an answer.
- **Rate limits** must be configured for these operations in this phase, not left to the Phase 09
  defaults. Suggested starting points, documented as starting points: `signaling.session.create`
  10/60s, `signaling.offer` 20/60s, `signaling.answer` 20/60s, `signaling.candidate` 300/60s,
  `signaling.iceservers.get` 20/60s. The candidate limit must be generous enough for trickle ICE and
  is additionally bounded per session by `MaxCandidatesPerSession`.
- `AddSignaling()` requires `AddLiveSharp().AddSignalR()` and nothing else. It must **not** require
  chat, groups, presence, or calls. Add a test proving a signalling-only application composes.
- `AddSignaling(Action<SignalingOptions>)` and `AddIceServers(Action<IceOptions>)` as separate
  builder methods, because ICE configuration is environment-specific and usually comes from
  `IConfiguration` while signalling options rarely change.

**Tests**

| Test | Asserts |
|---|---|
| `Every_signaling_request_dto_lacks_a_peer_or_sender_member` | **security**, reflection |
| `Create_session_handler_uses_the_connection_user_id_as_initiator` | |
| `Offer_handler_derives_the_destination_from_the_session` | |
| `Candidate_handler_maps_a_rate_limited_result_to_silent_success` | `NotifyAsync` contract |
| `Ice_servers_handler_returns_the_provider_output` | |
| `AddSignaling_registers_the_service_store_sweep_and_lifecycle_handler` | lifetimes |
| `AddSignaling_registers_all_six_operation_handlers` | |
| `AddSignaling_is_idempotent` | |
| `AddSignaling_composes_without_chat_presence_or_groups` | **the decoupling proof** |
| `Signaling_assembly_does_not_reference_chat_presence_or_calls` | architecture test |
| `Every_signaling_operation_appears_in_the_known_policies_manifest` | Phase 09 gate |

**Integration tests (both transports)** — a complete negotiation with **no media**:

| Test | Asserts |
|---|---|
| `Two_peers_complete_an_offer_answer_candidate_exchange` | full happy path |
| `Offer_reaches_only_the_peer` | a third connected user receives nothing |
| `Candidates_from_both_peers_are_relayed_in_both_directions` | |
| `Session_reaches_established_after_the_first_post_answer_candidate` | |
| `Peer_disconnect_closes_the_session_and_notifies_the_survivor` | |
| `Idle_session_is_closed_and_both_peers_are_notified` | real short timeout, documented |
| `Renegotiation_offer_is_relayed_on_an_established_session` | |
| `Ice_servers_response_contains_a_usable_short_lived_turn_credential` | shape and expiry, not connectivity |
| `Session_created_before_the_peer_connects_delivers_buffered_candidates_on_connect` | |
| `Fifty_concurrent_sessions_all_negotiate_independently` | isolation under load |

---

### P10.T6 — Client helpers and the data-channel example

**Deliverables**

```
clients/dotnet/LiveSharp.Client/Signaling/LiveSharpSignalingClientExtensions.cs
clients/js/packages/client/src/signaling.ts
clients/js/packages/client/src/webrtc.ts
clients/js/packages/client/src/__tests__/signaling.test.ts
clients/js/packages/client/src/__tests__/webrtc.test.ts
examples/React/src/features/datachannel/…
docs/webrtc/*.md
```

**Public API (exact) — TypeScript:**

```ts
export interface SignalingSessionInfo {
  sessionId: string;
  initiatorUserId: string;
  peerUserId: string;
  state: 'new' | 'offerSent' | 'answerSent' | 'established' | 'closed' | 'failed';
}

export function createSession(client: LiveSharpClient, peerUserId: string, metadata?: Record<string, string>): Promise<SignalingSessionInfo>;
export function getIceServers(client: LiveSharpClient): Promise<RTCIceServer[]>;
export function closeSession(client: LiveSharpClient, sessionId: string): Promise<void>;

export interface PeerConnectionHandle {
  readonly connection: RTCPeerConnection;
  readonly sessionId: string;
  close(): Promise<void>;
}

/**
 * Wires an RTCPeerConnection to a LiveSharp signalling session using the perfect
 * negotiation pattern. Returns the raw RTCPeerConnection so callers can do anything
 * WebRTC allows; this helper only handles signalling plumbing.
 */
export function attachPeerConnection(
  client: LiveSharpClient,
  options: {
    sessionId: string;
    polite: boolean;
    iceServers?: RTCIceServer[];
    configuration?: RTCConfiguration;
  },
): Promise<PeerConnectionHandle>;
```

**Requirements — perfect negotiation**

Implement the standard pattern (`makingOffer`, `ignoreOffer`, `isSettingRemoteAnswerPending`,
rollback on glare) faithfully. Do not invent a variant.

**Politeness assignment: the session initiator is the impolite peer.** The server does not resolve
glare — it relays and exposes `initiatorUserId`, and each client derives its own politeness as
`polite = (myUserId !== session.initiatorUserId)`. Provide a helper so applications do not compute it
by hand and get it inverted, which produces an intermittent failure that only appears when both sides
offer within the same round trip. Document the rule in `docs/webrtc/perfect-negotiation.md` with the
reasoning: the initiator is the party that decided to start, so it should win a simultaneity race.

- `attachPeerConnection` returns the **raw** `RTCPeerConnection`. This is a helper, not an abstraction.
  Advanced users must be able to add tracks, create data channels, inspect
  `getStats()`, and set arbitrary `RTCConfiguration`. Spec §70 asks for a simple API *and* an advanced
  one; this is where that promise is kept. Say so in the doc comment.
- `close()` must remove every event listener, unsubscribe from every LiveSharp event, close the
  `RTCPeerConnection`, and close the signalling session. Subscription leaks here are the classic
  WebRTC memory leak. Test it.
- **No DOM at module scope.** `webrtc.ts` references `RTCPeerConnection`, which does not exist in
  Node. Guard it: throw a clear, actionable error on first *use* in a non-browser environment, never on
  import. The Phase 03 SSR-safety test must still pass with this module present — extend it to cover
  `webrtc.ts` explicitly.

**Requirements — example**

Add a "Connect two peers" panel to the React example that establishes an `RTCDataChannel` and sends
text over it. **No `getUserMedia`, no audio, no video** — this example exists to prove signalling works
in isolation, before Phase 11 adds calling. A data channel is the minimum viable proof and needs no
device permissions, which also makes it work in CI and on machines with no camera.

Include a visible state readout: signalling session state, `RTCPeerConnection.connectionState`,
`iceConnectionState`, and the selected candidate pair type (host / srflx / relay) from `getStats()`.
That last one is what tells a developer whether TURN is actually being used, and it is the single most
requested piece of WebRTC diagnostics.

**Tests**

| Test | Asserts |
|---|---|
| `signaling constants match artifacts/protocol-names.json` | contract test |
| `create session resolves with the session info` | |
| `politeness helper marks the non-initiator as polite` | both directions |
| `attach peer connection subscribes to offer answer and candidate events` | mocked `RTCPeerConnection` |
| `attach peer connection sends candidates as they are gathered` | |
| `glare on the impolite peer ignores the incoming offer` | perfect negotiation |
| `glare on the polite peer rolls back and accepts` | |
| `close removes every event listener and unsubscribes` | **leak test** |
| `close is idempotent` | |
| `webrtc module imports cleanly in node with no DOM globals` | **SSR safety** |
| `attach peer connection throws an actionable error in a non browser environment` | on use, not import |
| `Client_signaling_helpers_surface_the_error_code_on_failure` | .NET |

---

## Dependency justification

| Package | Why | Alternative? | Licence | Notes |
|---|---|---|---|---|
| *(none)* | `LiveSharp.Signaling` uses only the BCL: `System.Security.Cryptography.HMACSHA1` for TURN credentials, `System.Text.Json` for protocol types, `TimeProvider` for expiry. | — | — | This package must stay dependency-free. It is the strongest evidence that "the server relays only" is a real architectural property and not marketing. |

Explicitly rejected: `SIPSorcery` — it is for server-side WebRTC *endpoints*, and this package is not
one. It arrives, if ever, in `LiveSharp.Media.SIPSorcery` after 1.0, and a `net10.0` compatibility
spike is required first. Also rejected: any SDP parsing library, for the reasons in `P10.T1`.

---

## Documentation deltas

- `docs/webrtc/README.md` — **new**. What LiveSharp does and does not do for WebRTC, the postbox
  diagram, the package-naming explanation, and a pointer to the three pages below.
- `docs/webrtc/signaling.md` — **new**. Session lifecycle and the state table, every operation and
  event with JSON examples, the candidate-buffering division of responsibility, size and rate limits,
  and the two-gate authorization model.
- `docs/webrtc/stun-turn.md` — **new and operationally important**. Why STUN is not enough, when TURN
  is required (symmetric NAT, restrictive firewalls — with rough percentages), a complete working
  `turnserver.conf` for coturn using `static-auth-secret`, how to verify the credential scheme end to
  end, `turns:` vs `turn:`, bandwidth cost warnings, and how to confirm relay usage from
  `getStats()`.
- `docs/webrtc/perfect-negotiation.md` — **new**. The pattern, the politeness rule and why the
  initiator is impolite, what goes wrong if politeness is inverted, and rollback.
- `docs/security/README.md` — SDP and ICE candidates contain IP addresses and are never logged; TURN
  credentials are ephemeral and per-user; the static-credential escape hatch and its flag name; the
  relay-authorization model.
- `docs/security/authorization.md` — add the six signalling operations to the policy table and document
  `ISignalingAuthorizationService`.
- `docs/security/rate-limiting.md` — add the signalling operation limits.
- `docs/configuration/README.md` — `SignalingOptions` and `IceOptions`.
- `docs/troubleshooting/README.md` — "peers never connect" (no TURN), "connection works locally but not
  across networks" (the canonical TURN symptom), "offer forbidden" (authorization), "candidates
  rejected" (per-session cap), "TURN credentials rejected by coturn" (secret mismatch, clock skew —
  mention NTP), "session closes after two minutes" (idle timeout).
- `README.md` — WebRTC signalling → `Available`; audio/video calls remain `Planned`.

## Example deltas

- `examples/React` — data-channel connect panel with the WebRTC state readout.
- `examples/React/README.md` — a section explaining that this proves signalling only, and that calling
  arrives in `0.10.0`.

## CHANGELOG entry

```markdown
### Added
- `LiveSharp.Signaling`: WebRTC signalling relay for SDP offers, answers, and ICE candidates between
  two authenticated peers.
- Two-party signalling sessions with an explicit state machine and idle expiry.
- `IIceServerProvider` with STUN-only and ephemeral-TURN-credential implementations, implementing the
  coturn `static-auth-secret` scheme.
- `ISignalingAuthorizationService` so permission to open a media channel is distinct from permission to
  send a message.
- `attachPeerConnection` TypeScript helper implementing the perfect negotiation pattern over LiveSharp
  signalling, returning the raw `RTCPeerConnection`.
- Data-channel example demonstrating peer connectivity without media.

### Security
- Relay destinations are derived from the session, never from the request payload.
- Unknown and unauthorized sessions return identical responses, so session identifiers cannot be
  enumerated.
- TURN credentials are per-user and short-lived by default; static shared credentials require an
  explicit acknowledgement flag.
- SDP and ICE candidate contents are never logged, because candidates disclose network topology.

### Notes
- LiveSharp relays signalling only. Media flows peer to peer or through your TURN server and never
  through the LiveSharp server.
```

---

## Exit criteria

- [ ] `LiveSharp.Signaling` has **zero** package dependencies — architecture test green.
- [ ] `LiveSharp.Signaling` does not reference `LiveSharp.Chat`, `LiveSharp.Presence`, or
      `LiveSharp.Calls` — architecture test green.
- [ ] `AddSignaling_composes_without_chat_presence_or_groups` passes.
- [ ] `Relay_destination_cannot_be_supplied_by_the_payload` passes.
- [ ] `Forged_session_id_is_forbidden_not_not_found` passes with an identical message.
- [ ] `Session_id_enumeration_yields_no_information` passes.
- [ ] `Create_session_allowed_by_chat_but_denied_by_signaling_is_forbidden` passes.
- [ ] `Ephemeral_provider_matches_the_known_answer_vector` passes, and the vector's provenance is
      documented in a comment.
- [ ] `A_user_id_containing_a_colon_is_rejected_with_a_clear_error` passes.
- [ ] All three shared-secret leakage tests pass (response, logs, exception message).
- [ ] `Static_credentials_without_the_acknowledgement_flag_fail_at_startup` passes.
- [ ] `Sdp_containing_unusual_codec_lines_is_relayed_unchanged` passes — no parsing crept in.
- [ ] `Sdp_content_is_never_logged_by_default` and the candidate equivalent pass.
- [ ] `Every_state_transition_pair_has_a_defined_result` is exhaustive.
- [ ] `webrtc module imports cleanly in node with no DOM globals` passes.
- [ ] `close removes every event listener and unsubscribes` passes.
- [ ] Two peers exchange data-channel text in the React example, and the state readout shows the
      selected candidate pair type.
- [ ] `docs/webrtc/stun-turn.md` contains a complete working coturn configuration.
- [ ] Every signalling operation is in the Phase 09 `KnownOperationPolicies` manifest.
- [ ] Signalling constants in `protocol.ts`; contract test green.
- [ ] `scripts/verify.ps1` green; tag `v0.9.0`.

## Verify

```powershell
pwsh scripts/verify.ps1
dotnet test tests/LiveSharp.Signaling.Tests -c Release
dotnet test tests/LiveSharp.IntegrationTests -c Release --filter "FullyQualifiedName~Signaling"
dotnet test tests/LiveSharp.ArchitectureTests -c Release
pnpm --dir clients/js -r test
git tag v0.9.0
```

## Next

`docs/plan/phase-11-audio-calling.md`, task `P11.T1`.
