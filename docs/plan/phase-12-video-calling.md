# Phase 12 — Video Calling

| | |
|---|---|
| **Version at exit** | `0.11.0` (tag `v0.11.0`) |
| **Depends on** | Phase 11 |
| **Branch** | `feat/phase-12-video-calling` |
| **Tasks** | `P12.T1` … `P12.T5` |
| **Read before starting** | `docs/calls/lifecycle.md`, `docs/calls/audio.md`, `docs/webrtc/perfect-negotiation.md` |

---

## Goal

Add video to the calling feature: media negotiation on invite, camera on/off mid-call via track
renegotiation, quality hints, and the first browser end-to-end test suite.

Most of the work is **client-side**, because the server still does not touch media. The server's job
grows by exactly one concept: it must know which media kinds a call negotiated and enforce that the
requested set is permitted. Everything else is `RTCPeerConnection` track management in the browser.

Resist the urge to make this phase bigger than that.

---

## Non-goals — DO NOT implement in this phase

- No screen sharing. Phase 13, even though it is technically the same renegotiation machinery.
- No simulcast, SVC, or layer selection. Those require an SFU.
- No bandwidth estimation, adaptive bitrate logic, or network-quality scoring beyond surfacing what
  `getStats()` already reports.
- No video recording, snapshots, or virtual backgrounds.
- No group video. Still 1:1.
- No server-side transcoding or codec preference enforcement — Phase 10's no-SDP-parsing rule holds.
- No picture-in-picture, layout management, or UI component library. The examples build UI; the
  library does not ship components.

---

## Tasks

### P12.T1 — Media negotiation on the server

**Deliverables**

```
src/LiveSharp.Calls/CallMediaState.cs
src/LiveSharp.Calls/Call.cs                                (modified)
src/LiveSharp.Calls/CallParticipant.cs                     (modified)
src/LiveSharp.Calls/CallService.cs                         (modified)
src/LiveSharp.Calls/Configuration/CallOptions.cs           (modified)
src/LiveSharp.Calls/Protocol/SetMediaRequest.cs
src/LiveSharp.Calls/Handlers/SetMediaHandler.cs
tests/LiveSharp.Calls.Tests/Media/CallMediaNegotiationTests.cs
```

**Public API (exact):**

```csharp
namespace LiveSharp.Calls;

public interface ICallService
{
    // ... existing members ...

    /// <summary>Updates the media kinds the calling participant is currently sending.</summary>
    Task<OperationResult<Call>> SetMediaAsync(string actingUserId, string callId, CallMediaKind media, CancellationToken cancellationToken = default);
}

public sealed record SetMediaRequest
{
    public required string CallId { get; init; }
    public required CallMediaKind Media { get; init; }
}
```

**Changes to existing types**

- `CallParticipant.Media` (already present since Phase 11) becomes meaningful: it is the set the
  participant is **actually sending**, updated by `SetMediaAsync`. `Call.RequestedMedia` remains what the
  caller asked for at invite time. Two fields, two meanings, both needed — document the distinction
  clearly, because "why does the call say video but the participant say audio" is otherwise a support
  question.
- `CallOptions.AllowedMedia` default changes from `Audio` to `Audio | Video`. This is the honest flip:
  `0.10.0` rejected video because it did not work; `0.11.0` accepts it because it does. Record it as a
  `### Changed` changelog entry — it is behaviour-visible even though it is not source-breaking.
- Add `CallOptions.MaxMediaChangesPerMinute` (default 20). Each media change triggers a full WebRTC
  renegotiation on both peers; a client toggling video in a loop is a CPU and signalling amplifier.
  Enforced by the Phase 09 rate limiter on `call.media.set`, plus this as a per-call backstop.

**Requirements**

- `SetMediaAsync` validates: the call is `Accepted`/`Connecting`/`Connected` (media changes on a ringing
  call are meaningless); the acting user is a participant; the requested set is a subset of
  `AllowedMedia`; the set includes `Audio` **or** is empty. A video-only call is legal (a baby monitor,
  a doorbell) so do not require audio — but an empty set with the call still connected is also legal
  (both muted, both cameras off), so do not require anything at all. Simplify: validate only the
  `AllowedMedia` subset rule. Drop the tempting invariant; it has no basis.
- `ScreenShare` in the requested set → rejected in this phase with a message naming the allowed kinds,
  exactly as video was in `0.10.0`. Same honesty mechanism.
- Emits `call.state.changed` to both participants. **Not** a new event type — media state is call
  state, and adding `call.media.changed` would mean clients handle two events to maintain one model.
- Same optimistic-concurrency `TryUpdateAsync` retry loop as every other state change.
- `Call.RequestedMedia` is **not** mutated by `SetMediaAsync`. It is a historical record of the invite.

> ## The server does not verify that the media matches the SDP
> A participant can claim `Audio | Video` while sending only audio, or vice versa. The server has no
> way to know — it does not parse SDP (Phase 10) and does not see media.
>
> `CallParticipant.Media` is therefore **a declaration, not a guarantee**. It exists so the peer's UI
> can show "camera off" before the track disappears, and so a client can render intent during
> renegotiation. Applications must drive actual rendering from `RTCPeerConnection` track events, not
> from this field. State this in the XML docs and in `docs/calls/video.md`, or someone will build a UI
> on it and file a bug when it disagrees with reality.

**Tests**

| Test | Asserts |
|---|---|
| `Set_media_updates_only_the_calling_participant` | |
| `Set_media_does_not_change_requested_media` | |
| `Set_media_notifies_both_participants_via_state_changed` | |
| `Set_media_emits_no_separate_media_event` | pins the single-event decision |
| `Set_media_on_a_ringing_call_is_rejected` | |
| `Set_media_on_a_terminal_call_is_rejected` | |
| `Set_media_by_a_non_participant_is_forbidden` | |
| `Set_media_requesting_screen_share_is_rejected_naming_allowed_kinds` | |
| `Set_media_to_an_empty_set_is_permitted` | both cameras off, both muted |
| `Set_media_to_video_only_is_permitted` | no audio requirement |
| `Invite_requesting_video_is_now_accepted` | the `AllowedMedia` default flip |
| `Invite_requesting_video_is_rejected_when_allowed_media_excludes_it` | still configurable |
| `Media_changes_beyond_the_per_call_limit_are_rate_limited` | |
| `Concurrent_media_changes_from_both_participants_both_land` | `ConcurrencyHarness` |
| `Set_media_honours_a_pre_cancelled_token` | |

---

### P12.T2 — Browser video call helper

**Deliverables**

```
clients/js/packages/client/src/video-call.ts
clients/js/packages/client/src/media-devices.ts
clients/js/packages/client/src/__tests__/video-call.test.ts
clients/js/packages/client/src/__tests__/media-devices.test.ts
clients/dotnet/LiveSharp.Client/Calls/LiveSharpCallClientExtensions.cs    (modified)
```

**Public API (exact):**

```ts
export interface VideoCallOptions {
  audio?: boolean | MediaTrackConstraints;
  video?: boolean | MediaTrackConstraints;
  /** Applied to the outgoing video sender. */
  quality?: VideoQualityHint;
}

export interface VideoQualityHint {
  maxBitrateKbps?: number;
  maxFramerate?: number;
  /** Passed to RTCRtpSender.setParameters as scaleResolutionDownBy. */
  scaleResolutionDownBy?: number;
  degradationPreference?: 'maintain-framerate' | 'maintain-resolution' | 'balanced';
}

export interface MediaCallHandle {
  readonly callId: string;
  readonly peerConnection: RTCPeerConnection;
  readonly localStream: MediaStream;
  readonly remoteStream: MediaStream;

  setMuted(muted: boolean): Promise<void>;
  setCameraEnabled(enabled: boolean): Promise<void>;
  setQuality(hint: VideoQualityHint): Promise<void>;
  switchCamera(deviceId: string): Promise<void>;
  hangUp(): Promise<void>;

  onRemoteTrack(handler: (event: { kind: 'audio' | 'video'; stream: MediaStream }) => void): Unsubscribe;
}

export function startMediaCall(client: LiveSharpClient, call: CallInfo, options?: VideoCallOptions): Promise<MediaCallHandle>;

/** Enumerates devices. Returns an empty list when permission has not yet been granted. */
export function listMediaDevices(): Promise<{ audioInputs: MediaDeviceInfo[]; videoInputs: MediaDeviceInfo[]; audioOutputs: MediaDeviceInfo[] }>;
```

`startAudioCall` from Phase 11 becomes a thin wrapper over `startMediaCall(client, call, { audio: true, video: false })`, kept for
compatibility and clarity. Do not delete it — it is the 80% case and its name documents intent.
`AudioCallHandle` becomes an alias of `MediaCallHandle`.

**Requirements — camera on/off is renegotiation, and that is the whole difficulty**

Three approaches exist. Choose and document:

| Approach | Behaviour | Verdict |
|---|---|---|
| `track.enabled = false` | Track stays, sends black frames. No renegotiation. Camera light stays **on**. | Rejected: the camera indicator staying lit when the user turned the camera off is an unacceptable privacy signal. |
| `sender.replaceTrack(null)` | Track slot stays, nothing sent. No renegotiation. Camera can be stopped. | **Chosen.** |
| `removeTrack` + `addTrack` | Full renegotiation each toggle. | Rejected for toggling: two SDP exchanges per tap, and glare risk. Used only when going from never-had-video to having video. |

So: `setCameraEnabled(false)` calls `sender.replaceTrack(null)` and stops the local video track
(`track.stop()`), which turns the camera indicator off. `setCameraEnabled(true)` re-acquires via
`getUserMedia` and `replaceTrack(newTrack)`. Renegotiation happens only if no video transceiver exists
yet — and to avoid that, `startMediaCall` **always adds a video transceiver** (`direction: 'sendrecv'`)
even for an audio-only call, so a later camera-on never needs an SDP exchange. Cost: a slightly larger
initial SDP. Benefit: camera toggling never renegotiates, which removes an entire class of glare bug.
Document the trade-off; it is the single most important implementation decision in this phase.

**Other requirements**

- `getUserMedia` timing rule from Phase 11 still applies: only after acceptance.
- Permission error mapping extended for video: `NotAllowedError`, `NotFoundError`, `NotReadableError`,
  `OverconstrainedError` (a constraint the device cannot satisfy — very common with explicit resolution
  requests, and the error message must name the failing constraint).
- **Graceful degradation:** if audio succeeds and video fails, the call must continue as audio-only with
  a reported error, not fail entirely. Acquire tracks independently, never in one `getUserMedia` call
  with both constraints — a single failing constraint otherwise kills the whole call. This is a
  non-obvious requirement and needs its own test.
- `setQuality` uses `RTCRtpSender.getParameters()` / `setParameters()` with
  `encodings[0].maxBitrate`, `maxFramerate`, `scaleResolutionDownBy`, and `degradationPreference`.
  Must handle `encodings` being empty (some browsers) by creating an entry. Must not throw if the
  browser rejects a parameter — log and continue.
- `switchCamera(deviceId)` acquires the new device, `replaceTrack`s it, and stops the old track. Must
  restore the previous track if acquisition fails, so a failed switch does not leave the user with no
  video.
- `hangUp()` stops **every** track — audio and video — and the Phase 11 microphone-indicator test gains
  a camera-indicator sibling.
- `onRemoteTrack` exists because `ontrack` fires per track and applications need to attach video and
  audio to different elements. Deliver the track kind explicitly rather than making the application
  inspect it.
- No DOM at module scope; actionable errors on use outside a browser; Phase 03 SSR test extended.

**Tests** (mocked `RTCPeerConnection`, `getUserMedia`, and `MediaStream`)

| Test | Asserts |
|---|---|
| `start media call adds a video transceiver even for audio only` | **the no-renegotiation decision** |
| `start media call acquires audio and video independently` | two `getUserMedia` calls |
| `video acquisition failure degrades to audio only and reports the error` | **required** |
| `audio acquisition failure with video success continues as video only` | symmetric |
| `both acquisition failures hang up the call with a reason` | |
| `set camera enabled false replaces the track with null` | |
| `set camera enabled false stops the local video track` | **the camera-indicator test** |
| `set camera enabled false does not trigger renegotiation` | no offer created |
| `set camera enabled true reacquires and replaces the track` | |
| `set camera enabled true does not trigger renegotiation` | the transceiver already exists |
| `set camera enabled reports the media set to the server` | `call.media.set` |
| `set quality applies max bitrate framerate and degradation preference` | |
| `set quality creates an encoding entry when none exists` | browser variance |
| `set quality does not throw when the browser rejects a parameter` | |
| `switch camera replaces the track and stops the old one` | |
| `switch camera restores the previous track when acquisition fails` | |
| `over constrained error names the failing constraint` | |
| `on remote track reports audio and video separately with the kind` | |
| `hang up stops every audio and video track` | |
| `hang up unsubscribes and is idempotent` | leak test |
| `video call module imports cleanly in node with no DOM globals` | SSR safety |
| `list media devices returns empty lists without permission` | no throw |

---

### P12.T3 — Playwright end-to-end infrastructure

> ## The first real browser test suite in the project
> Everything before this phase was tested with mocks and in-memory transports. Video calling cannot be
> verified that way — the only proof that two browsers can establish a media connection is two browsers
> establishing a media connection.
>
> Build the harness properly once. Phases 13, 17, and 18 all use it.

**Deliverables**

```
tests/e2e/package.json
tests/e2e/playwright.config.ts
tests/e2e/fixtures/app.ts
tests/e2e/fixtures/two-peers.ts
tests/e2e/fixtures/webrtc-stats.ts
tests/e2e/specs/audio-call.spec.ts
tests/e2e/specs/video-call.spec.ts
tests/e2e/README.md
scripts/verify.ps1                                          (activate the E2E gate)
.github/workflows/e2e.yml
```

**Requirements**

- Playwright with Chromium **and** Firefox. Safari/WebKit is excluded initially — WebRTC in Playwright's
  WebKit build is unreliable and would produce noise, not signal. Document the exclusion and the
  intention to revisit at 1.0.
- **Fake media devices are mandatory.** Launch flags:
  `--use-fake-device-for-media-stream`, `--use-fake-ui-for-media-stream`,
  `--allow-file-access-from-files`, and for Firefox the equivalent prefs
  (`media.navigator.streams.fake = true`, `media.navigator.permission.disabled = true`). Without these,
  CI has no camera and every test hangs on a permission prompt.
- The harness starts the React example app (which by Phase 12 has audio and video calling) plus its
  ASP.NET Core host, using Playwright's `webServer` config with a real health check — not a fixed sleep.
- `two-peers.ts` fixture: two browser contexts, logged in as two different demo users, both connected,
  returning page handles plus helpers (`invite`, `acceptIncoming`, `waitForConnected`, `hangUp`).
- `webrtc-stats.ts` fixture: reads `RTCPeerConnection.getStats()` from the page and exposes assertions —
  selected candidate pair type, bytes sent/received per track kind, packets lost, and whether a video
  track is actually flowing. **Asserting on bytes received is the only real proof of media**; asserting
  on `connectionState === 'connected'` proves ICE, not media.
- **Flakiness discipline**, because a flaky E2E suite is worse than none:
  - `retries: 2` in CI, `0` locally.
  - Every wait is an explicit `expect.poll` or `waitForFunction` with a generous timeout — never
    `waitForTimeout`.
  - The E2E job is **non-blocking** (`continue-on-error: true`) until Phase 18, where it becomes
    required. Say so in the workflow with a comment referencing this file.
  - A quarantine tag (`@flaky`) that excludes a spec from the gate while keeping it running and
    reported. Any quarantined spec must have a linked issue.
- Traces, videos, and screenshots on failure, uploaded as artifacts. Debugging a failed WebRTC E2E run
  without a trace is hopeless.
- `tests/e2e` is a `pnpm` workspace member. `scripts/verify.ps1` gains a real E2E step behind `-SkipE2E`
  (default **skip** locally, run in CI).

**Specs**

| Spec | Asserts |
|---|---|
| `audio call connects and media flows` | `getStats()` shows inbound audio bytes increasing on both sides |
| `video call connects and video media flows` | inbound video bytes increasing on both sides |
| `camera off stops outbound video without dropping the call` | outbound video bytes stop; audio continues; `connectionState` stays `connected` |
| `camera on resumes outbound video` | bytes resume |
| `mute stops outbound audio without dropping the call` | |
| `hang up ends the call on both sides` | both UIs return to idle |
| `reject shows the caller a declined state` | |
| `ring timeout shows both sides a missed call` | short timeout configured for the test app |
| `accepting on a second tab stops ringing in the first` | three contexts for one callee |
| `selected candidate pair is reported in the ui` | proves the diagnostics readout works |
| `call survives a brief signalling disconnect` | drop the WebSocket via CDP, media continues, call not ended (the Phase 11 grace period, proven for real) |

---

### P12.T4 — Server-side and integration test updates

**Deliverables**

```
tests/LiveSharp.IntegrationTests/Calls/VideoCallIntegrationTests.cs
src/LiveSharp.Calls/Diagnostics/CallLog.cs                 (modified)
```

**Requirements**

- Integration tests for the server side of video: invite with `Audio | Video`, media changes propagated
  to the peer, `ScreenShare` rejected, `AllowedMedia` enforcement, and rate limiting on
  `call.media.set`.
- Add `call.media.set` to the Phase 09 `KnownOperationPolicies` manifest and the rate-limit table
  (suggested 30/60s, which combined with the per-call backstop is sufficient).
- `CallLog`: add media-change events at `Debug` with call ID and the media flags — **not** participant
  IDs, per the Phase 11 privacy rule.

**Tests**

| Test | Asserts |
|---|---|
| `Invite_with_audio_and_video_is_accepted` | |
| `Invite_with_screen_share_is_rejected` | |
| `Media_change_reaches_the_peer` | |
| `Media_change_reaches_the_changing_participants_other_connections` | |
| `Media_change_on_a_call_the_user_is_not_in_is_forbidden` | |
| `Allowed_media_configured_to_audio_only_rejects_video_invites` | |
| `Rapid_media_changes_are_rate_limited` | |
| `call.media.set_appears_in_the_known_policies_manifest` | Phase 09 gate |

---

### P12.T5 — Examples and documentation

**Deliverables**

```
examples/React/src/features/calls/…                        (video UI)
examples/NextJs/…                                          (video calling added)
examples/AspNetCoreMvc/wwwroot/js/call.js                  (video added)
docs/calls/video.md
docs/calls/media-devices.md
```

**Requirements**

- React example is the reference video UI: local preview (muted, mirrored), remote video, camera
  toggle, mute toggle, device pickers for microphone and camera, a quality selector (low / medium /
  high mapping to concrete `VideoQualityHint` values), and the existing diagnostics readout extended
  with resolution, framerate, and bitrate from `getStats()`.
- Next.js example gains video calling, which also exercises the SSR-safety guarantee against the new
  `video-call.ts` and `media-devices.ts` modules.
- MVC example gains video with a deliberately minimal UI — its role is to prove the library works
  without a build toolchain, not to be pretty.
- **Local preview must be muted.** An unmuted local preview causes immediate acoustic feedback. Every
  developer hits this once; the example should prevent them hitting it at all, with a comment saying
  why.
- `docs/calls/video.md`: the transceiver-always-added decision and why, the three camera-toggle
  approaches and why `replaceTrack` won, graceful degradation, quality hints and their real effects,
  device switching, the declaration-not-guarantee nature of `CallParticipant.Media`, and a
  **Limitations** section (1:1 only, no simulcast, no adaptive logic beyond browser defaults, no
  recording).
- `docs/calls/media-devices.md`: enumeration, the permission-before-labels behaviour (device labels are
  empty until permission is granted — a universally surprising browser behaviour), switching, and
  handling device unplugging via `devicechange`.
- `docs/calls/troubleshooting.md`: add "camera light stays on after turning the camera off"
  (`track.stop()`), "video is black" (constraints, or a paused element), "video connects but is very low
  quality" (degradation preference and bandwidth), "one side sees video, the other does not"
  (transceiver direction), "device picker shows blank names" (permission), "acoustic feedback"
  (unmuted local preview), "OverconstrainedError" (impossible constraints).

---

## Dependency justification

| Package | Why | Alternative? | Licence | Notes |
|---|---|---|---|---|
| `@playwright/test` | Browser end-to-end testing with real WebRTC. Nothing else can verify media flow. | Selenium (worse WebRTC support, no fake-device flags story), manual testing (not a test). | Apache-2.0 | Dev-only, in `tests/e2e`. Not shipped. |

No new .NET packages. No new runtime dependencies in any shipped package.

---

## Documentation deltas

- `docs/calls/video.md` — **new**, as specified.
- `docs/calls/media-devices.md` — **new**, as specified.
- `docs/calls/audio.md` — note that `startAudioCall` now wraps `startMediaCall`, and cross-link.
- `docs/calls/README.md` — video status, and the `AllowedMedia` default change.
- `docs/calls/troubleshooting.md` — the seven new entries above.
- `docs/testing/e2e.md` — **new**. How to run the E2E suite locally, the fake-device flags and why they
  are required, the flakiness policy, the quarantine process, and how to read a Playwright trace of a
  failed WebRTC test.
- `docs/security/rate-limiting.md` — `call.media.set`.
- `docs/security/authorization.md` — `call.media.set`.
- `docs/configuration/README.md` — `AllowedMedia` default change, `MaxMediaChangesPerMinute`.
- `README.md` — video calls → `Available`.

## Example deltas

- `examples/React` — full video UI with device pickers, quality selector, and extended diagnostics.
- `examples/NextJs` — video calling.
- `examples/AspNetCoreMvc` — minimal video.
- All READMEs — video instructions and the permission/TURN checklist.

## CHANGELOG entry

```markdown
### Added
- Video calling: media negotiation on invite, camera on/off without renegotiation, quality hints, and
  device switching.
- `startMediaCall` browser helper with independent audio and video acquisition and graceful degradation
  when one of them fails.
- `call.media.set` operation for declaring the media a participant is sending.
- Playwright end-to-end test suite covering audio and video calls in Chromium and Firefox, asserting on
  real media flow via `getStats()`.
- Media device enumeration and switching helpers.

### Changed
- `CallOptions.AllowedMedia` now defaults to `Audio | Video`. Video invites were rejected in `0.10.0`.

### Notes
- `CallParticipant.Media` is a participant's declaration of what it is sending, not a server-verified
  fact. Drive rendering from `RTCPeerConnection` track events.
- A video transceiver is added to every call, including audio-only calls, so enabling the camera later
  never requires renegotiation.
```

---

## Exit criteria

- [ ] `start media call adds a video transceiver even for audio only` passes.
- [ ] `set camera enabled false does not trigger renegotiation` passes.
- [ ] `set camera enabled false stops the local video track` passes.
- [ ] `video acquisition failure degrades to audio only and reports the error` passes.
- [ ] `switch camera restores the previous track when acquisition fails` passes.
- [ ] `hang up stops every audio and video track` passes.
- [ ] `Invite_with_screen_share_is_rejected` passes — screen sharing is honestly unavailable.
- [ ] E2E: `video call connects and video media flows` passes in **Chromium and Firefox**, asserting on
      inbound video bytes, not just `connectionState`.
- [ ] E2E: `call survives a brief signalling disconnect` passes — the Phase 11 grace period proven in a
      real browser.
- [ ] E2E: `accepting on a second tab stops ringing in the first` passes.
- [ ] The E2E job runs in CI, uploads traces on failure, and is explicitly marked non-blocking with a
      comment pointing at Phase 18.
- [ ] `docs/testing/e2e.md` documents the fake-device flags and the flakiness policy.
- [ ] No spec is quarantined without a linked issue.
- [ ] Video call works between two browsers in the React example, with visible resolution, framerate,
      bitrate, and candidate-pair type.
- [ ] `call.media.set` is in the `KnownOperationPolicies` manifest and the rate-limit table.
- [ ] `scripts/verify.ps1` green (E2E skipped locally, run in CI); tag `v0.11.0`.

## Verify

```powershell
pwsh scripts/verify.ps1
pwsh scripts/verify.ps1 -SkipPack        # then, separately, the E2E gate:
pnpm --dir tests/e2e install
pnpm --dir tests/e2e exec playwright install --with-deps chromium firefox
pnpm --dir tests/e2e test
dotnet test tests/LiveSharp.IntegrationTests -c Release --filter "FullyQualifiedName~VideoCall"
git tag v0.11.0
```

## Next

`docs/plan/phase-13-screen-sharing.md`, task `P13.T1`.
