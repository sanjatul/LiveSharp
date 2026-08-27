# Phase 13 — Screen Sharing

| | |
|---|---|
| **Version at exit** | `0.12.0` (tag `v0.12.0`) |
| **Depends on** | Phase 12 |
| **Branch** | `feat/phase-13-screen-sharing` |
| **Tasks** | `P13.T1` … `P13.T4` |
| **Read before starting** | `docs/calls/video.md`, `docs/calls/lifecycle.md` |

---

## Goal

Screen sharing during a call: `getDisplayMedia`, a second video track alongside the camera, per-call
share arbitration, and the events a peer needs to render it.

This is the smallest phase in the WebRTC track. Phase 12 built the renegotiation machinery, the
transceiver strategy, and the E2E harness; screen sharing reuses all of it. If this phase starts feeling
large, something is being over-designed.

---

## The one real design decision: a second track, not a swapped track

Two implementations exist:

| Approach | Behaviour | Verdict |
|---|---|---|
| Replace the camera track with the screen track | One video transceiver. No renegotiation. Camera and screen are mutually exclusive. | Rejected. |
| Add a **second** video transceiver for the screen | Camera and screen visible simultaneously. Requires one renegotiation when screen sharing first starts. | **Chosen.** |

Rationale: every product that has screen sharing shows the presenter's camera *and* their screen —
that is the entire point in a meeting context. Mutual exclusivity would make LiveSharp's screen sharing
unusable for its primary use case in order to avoid a single SDP exchange.

Cost, stated honestly: the first `startScreenShare()` on a call triggers a renegotiation, which means a
brief glare window. Mitigate by pre-adding the second video transceiver at call setup — the same trick
Phase 12 used for the camera — so screen sharing also never renegotiates. Cost of that: every call's
initial SDP carries three transceivers (audio, camera video, screen video) instead of two. That is a few
hundred bytes on a one-time exchange. Take it.

Distinguishing the two video tracks on the receiving side is done with **transceiver `mid` mapping**
communicated through the call's participant media state, not by inspecting track labels (which are
browser-specific and unreliable). See `P13.T2`.

---

## Non-goals — DO NOT implement in this phase

- No multi-participant sharing. Only one participant may share at a time per call (see `P13.T1`).
- No screen recording, annotation, remote control, or pointer sharing.
- No region/window selection UI — `getDisplayMedia` provides the browser's own picker and applications
  must not attempt to replicate it.
- No audio capture from the shared tab beyond passing `audio: true` through to `getDisplayMedia` and
  documenting that browser support is inconsistent.
- No server-side composition, layout, or mixing.
- No screen sharing outside a call. Sharing a screen with no call is a different feature (broadcast) and
  is not in scope for 1.0.

---

## Tasks

### P13.T1 — Server-side share state and arbitration

**Deliverables**

```
src/LiveSharp.Calls/CallOptions.cs                          (modified)
src/LiveSharp.Calls/CallService.cs                          (modified)
src/LiveSharp.Calls/Protocol/SetScreenShareRequest.cs
src/LiveSharp.Calls/Handlers/SetScreenShareHandler.cs
tests/LiveSharp.Calls.Tests/ScreenShare/ScreenShareArbitrationTests.cs
```

**Public API (exact):**

```csharp
namespace LiveSharp.Calls;

public interface ICallService
{
    // ... existing members ...

    /// <summary>Starts or stops screen sharing for the calling participant.</summary>
    Task<OperationResult<Call>> SetScreenShareAsync(string actingUserId, string callId, bool sharing, CancellationToken cancellationToken = default);
}

public sealed record SetScreenShareRequest
{
    public required string CallId { get; init; }
    public required bool Sharing { get; init; }
}
```

**Changes to existing types and options**

- `CallOptions.AllowedMedia` default changes from `Audio | Video` to `Audio | Video | ScreenShare`. The
  same honest flip as Phase 12: it was rejected because it did not work, now it works.
- Add `CallOptions.ScreenSharePolicy`:

```csharp
public enum ScreenSharePolicy
{
    /// <summary>Only one participant may share at a time. A new share is rejected. Default.</summary>
    SingleSharerReject = 0,

    /// <summary>Only one participant may share at a time. A new share stops the existing one.</summary>
    SingleSharerReplace = 1,

    /// <summary>Both participants may share simultaneously.</summary>
    MultipleSharers = 2,
}
```

Default `SingleSharerReject`. Rationale: two people sharing simultaneously in a 1:1 call is almost always
an accident, and doubling outbound video from both sides is a real bandwidth event. `SingleSharerReplace`
exists because "take over presenting" is a legitimate flow, and `MultipleSharers` exists for
side-by-side comparison workflows. Three options is one more than the minimum, justified because the
right answer genuinely differs by product and none of them can be the silent default.

**Requirements**

- `SetScreenShareAsync(sharing: true)` sets `ScreenShare` on the acting participant's
  `CallParticipant.Media`; `false` clears it. It is a convenience over `SetMediaAsync` because
  "start sharing" is a discrete user action, and forcing an application to read-modify-write the flags
  set invites lost-update bugs. Implement it **in terms of** the same `TryUpdateAsync` retry loop, not
  as a parallel code path.
- Arbitration is evaluated **inside** the concurrency-guarded update, using the same striped
  `SemaphoreSlim` keyed by call ID that `CallAcceptCoordinator` uses. Checking "is anyone else sharing"
  outside the guard is the same class of bug as the Phase 11 double-accept, and two participants tapping
  Share simultaneously is a realistic race.
- `SingleSharerReject` → second sharer gets `Failure(Conflict, …)` with a message naming who is sharing
  is **not** acceptable (it leaks nothing sensitive here, but keep messages resource-free by
  convention) — return a generic "another participant is already sharing" and let the client render the
  sharer from the call state it already has.
- `SingleSharerReplace` → clear `ScreenShare` from the existing sharer, set it on the new one, and emit a
  single `call.state.changed` reflecting both changes. Two events would let a client briefly render two
  sharers.
- `ScreenShare` requested while the call is not `Accepted`/`Connecting`/`Connected` → rejected, same rule
  as Phase 12 media changes.
- Emits `call.state.changed` only. No `call.screenshare.started` event — screen sharing is call state,
  and Phase 12 already established that adding parallel events forces clients to maintain one model from
  two sources.
- Rate limited via Phase 09 (`call.screenshare.set`, suggested 20/60s) and bounded by
  `MaxMediaChangesPerMinute` from Phase 12.
- Same declaration-not-guarantee caveat as Phase 12: the server cannot verify a screen track exists.

**Tests**

| Test | Asserts |
|---|---|
| `Start_sharing_sets_the_screen_share_flag_on_the_acting_participant` | |
| `Stop_sharing_clears_the_flag` | |
| `Start_sharing_preserves_the_participants_audio_and_video_flags` | **the lost-update test** |
| `Start_sharing_notifies_both_participants_via_state_changed` | |
| `Start_sharing_emits_no_separate_screenshare_event` | pins the single-event decision |
| `Second_sharer_is_rejected_under_the_default_policy` | |
| `Second_sharer_replaces_the_first_under_the_replace_policy` | |
| `Replace_policy_emits_exactly_one_state_change_reflecting_both_participants` | |
| `Both_participants_may_share_under_the_multiple_sharers_policy` | |
| `Start_sharing_on_a_ringing_call_is_rejected` | |
| `Start_sharing_on_a_terminal_call_is_rejected` | |
| `Start_sharing_by_a_non_participant_is_forbidden` | |
| `Start_sharing_when_screen_share_is_not_in_allowed_media_is_rejected` | still configurable |
| `Concurrent_share_starts_from_both_participants_resolve_to_exactly_one_sharer_under_reject` | **`ConcurrencyHarness`** |
| `Concurrent_share_starts_under_replace_resolve_to_exactly_one_sharer` | |
| `Stopping_a_share_you_do_not_own_is_a_no_op_success` | idempotency |
| `Rapid_share_toggles_are_rate_limited` | |
| `Set_screen_share_honours_a_pre_cancelled_token` | |

---

### P13.T2 — Browser screen share helper

**Deliverables**

```
clients/js/packages/client/src/screen-share.ts
clients/js/packages/client/src/video-call.ts                (modified)
clients/js/packages/client/src/__tests__/screen-share.test.ts
```

**Public API (exact):**

```ts
export interface ScreenShareOptions {
  /** Passed through to getDisplayMedia. Browser support for audio is inconsistent. */
  audio?: boolean;
  video?: MediaTrackConstraints;
  /** Applied to the screen sender. Screen content usually wants resolution over framerate. */
  quality?: VideoQualityHint;
}

export interface MediaCallHandle {
  // ... existing members ...

  readonly isScreenSharing: boolean;

  /** Prompts the user to pick a screen, starts sharing, and reports it to the server. */
  startScreenShare(options?: ScreenShareOptions): Promise<void>;

  /** Stops sharing, releases the capture, and reports it to the server. */
  stopScreenShare(): Promise<void>;

  /** Fired when the remote participant's screen track arrives or ends. */
  onRemoteScreenShare(handler: (event: { active: boolean; stream: MediaStream | null }) => void): Unsubscribe;
}
```

**Requirements**

- `startMediaCall` pre-adds a **second** video transceiver (`direction: 'sendrecv'`) reserved for screen
  content, so `startScreenShare` only calls `sender.replaceTrack(screenTrack)` and never renegotiates.
  Record the reserved transceiver's `mid` and expose it internally.
- **Track disambiguation on the receiving side.** Both incoming video tracks arrive through `ontrack`.
  Distinguish them by `transceiver.mid` matched against the ordering established at call setup: the
  first video transceiver is camera, the second is screen. This works because both peers create
  transceivers in the same order in `startMediaCall`, and the order is preserved through negotiation.

  Add an assertion: if the mid mapping cannot be resolved, fall back to
  `track.contentHint`/`getSettings().displaySurface` and log a warning — do **not** guess silently.
  Test the fallback.

  Do not attempt to use `MediaStream.id` or `track.label` for this. Labels are browser-specific
  free-text; stream IDs are not stable across the negotiation in all browsers.
- **`getDisplayMedia` must be called from a user gesture.** Browsers reject it otherwise, with an error
  that reads like a permission failure. `startScreenShare` must document this and the error mapping must
  say "screen sharing must be started from a user action (a click)" for `InvalidStateError` /
  `NotAllowedError` without a gesture. This is the number-one screen-sharing support question.
- **The user can stop sharing from the browser's own UI.** Every browser shows a "Stop sharing" bar
  outside the page. The track's `ended` event is the only notification. `startScreenShare` **must**
  subscribe to `track.onended` and run the full `stopScreenShare` path — release the sender, report to
  the server, fire the local handler. Without this, the peer keeps rendering a frozen last frame and the
  server thinks the user is still sharing. This is the single most important line of code in the phase;
  give it its own test.
- Error mapping: `NotAllowedError` (user cancelled the picker — a **normal** outcome, not an error to
  surface loudly), `NotFoundError`, `NotReadableError`, `AbortError`, `InvalidStateError` (no gesture).
  User cancellation must resolve or reject distinguishably so the UI does not show a scary message when
  someone simply changed their mind. Recommend: reject with a specific `LiveSharpError` code
  `screen_share_cancelled` that applications can ignore.
- Quality default for screen content: `degradationPreference: 'maintain-resolution'` and a lower
  `maxFramerate` (e.g. 5–15). Screen content is text; resolution matters far more than framerate, and the
  browser default optimises for the opposite. Ship the sensible default and let `quality` override it.
  Document the reasoning — it is a genuinely useful, non-obvious piece of expertise.
- `stopScreenShare` must `track.stop()` the display track. A screen capture left running keeps the
  browser's sharing indicator visible, which is even more alarming than a lit camera light. Test it,
  mirroring the Phase 11 microphone and Phase 12 camera tests.
- `hangUp()` must stop the screen track too. Extend the existing "stops every track" test.
- No DOM at module scope; actionable error outside a browser; Phase 03 SSR test extended.

**Tests**

| Test | Asserts |
|---|---|
| `start media call pre adds a second video transceiver for screen content` | **the no-renegotiation decision** |
| `start screen share replaces the reserved transceiver track` | |
| `start screen share does not trigger renegotiation` | no offer created |
| `start screen share reports sharing to the server` | `call.screenshare.set` |
| `start screen share applies the maintain resolution default` | |
| `start screen share honours an explicit quality override` | |
| `user cancelling the picker rejects with the cancelled code` | **distinguishable from a failure** |
| `missing user gesture maps to an actionable error message` | |
| `stopping from the browser ui triggers the full stop path` | **the `track.onended` test** |
| `stopping from the browser ui reports to the server` | |
| `stopping from the browser ui fires the local handler` | |
| `stop screen share stops the display track` | **the sharing-indicator test** |
| `stop screen share reports to the server` | |
| `stop screen share when not sharing is a no op` | |
| `hang up stops the screen track along with audio and video` | |
| `remote screen track is distinguished from the remote camera track by mid` | |
| `remote screen track falls back to content hints when mid mapping fails and warns` | |
| `on remote screen share fires with active false when the remote track ends` | |
| `screen share module imports cleanly in node with no DOM globals` | SSR safety |
| `start screen share throws an actionable error outside a browser` | on use, not import |

---

### P13.T3 — E2E tests and integration tests

**Deliverables**

```
tests/e2e/specs/screen-share.spec.ts
tests/e2e/fixtures/two-peers.ts                             (modified)
tests/LiveSharp.IntegrationTests/Calls/ScreenShareIntegrationTests.cs
```

**Requirements — E2E**

Screen sharing in headless Chromium needs `--auto-select-desktop-capture-source=<name>` to bypass the
picker. Use `--auto-select-desktop-capture-source=Entire screen` (Chromium) and document that Firefox
requires `media.getdisplaymedia.prompt.testing = true` plus
`media.getdisplaymedia.prompt.testing.allow = true`. If Firefox screen capture proves unreliable in the
Playwright build, run these specs on Chromium only and record the exclusion in `docs/testing/e2e.md`
alongside the existing WebKit exclusion — do not spend a session fighting it.

| Spec | Asserts |
|---|---|
| `screen share connects and screen media flows` | `getStats()` shows a **second** inbound video track with increasing bytes |
| `camera and screen are received as two distinct video tracks` | both rendered, both flowing |
| `stopping the share stops the second track without dropping the call` | camera video and audio continue |
| `the second participant cannot share under the default policy` | UI shows the rejection |
| `stopping the share from the browser ui updates both sides` | simulate via `track.stop()` in the page, since the browser bar is not automatable |

**Requirements — server integration tests**

| Test | Asserts |
|---|---|
| `Screen_share_state_reaches_the_peer` | |
| `Screen_share_state_reaches_the_sharers_other_connections` | |
| `Second_sharer_receives_a_conflict_under_the_default_policy` | |
| `Replace_policy_transfers_the_share_and_notifies_both` | |
| `Screen_share_flag_is_cleared_when_the_call_ends` | no stale state in a terminal call |
| `Screen_share_is_rejected_when_excluded_from_allowed_media` | |
| `call.screenshare.set_appears_in_the_known_policies_manifest` | Phase 09 gate |
| `Rapid_share_toggles_are_rate_limited_end_to_end` | |

---

### P13.T4 — Examples and documentation

**Deliverables**

```
examples/React/src/features/calls/…                         (screen share UI)
examples/NextJs/…                                           (screen share added)
docs/calls/screen-sharing.md
```

**Requirements — React example**

- Share / Stop sharing button, disabled with an explanatory tooltip when the peer is already sharing
  under the default policy.
- Layout that shows remote screen large with remote camera as a small overlay when both are present, and
  falls back gracefully when only one is. This is the layout every product uses and the example should
  demonstrate it rather than stacking two equal boxes.
- Local share preview, muted, with a clear "you are sharing" indicator — because a user who forgets they
  are sharing is a real privacy incident, and the example should model good behaviour.
- Diagnostics readout extended with the second video track's resolution and bitrate.
- Handle the cancelled-picker case silently (no error toast), demonstrating the distinguishable error
  code.

**Requirements — documentation**

`docs/calls/screen-sharing.md` must contain:
- The second-transceiver decision and why, including the honest cost.
- The user-gesture requirement, with the exact error a developer will see if they get it wrong.
- **The `track.onended` requirement**, prominently — an application using the raw API without the helper
  must handle it, and one that does not will ship a broken feature.
- Track disambiguation: how camera and screen are told apart, why labels are not used, and the fallback.
- The three `ScreenSharePolicy` values with a recommendation per product type.
- Quality defaults for screen content and why they differ from camera content.
- Audio capture: pass-through only, browser support is inconsistent (Chromium tab audio yes, system
  audio platform-dependent, Firefox limited) — set expectations rather than implying it works.
- **Privacy guidance**: applications should show a persistent, unmissable sharing indicator; the browser's
  own indicator is not always visible (for example when the shared surface is a different window).
- **Limitations**: 1:1 only, one sharer by default, no annotation, no remote control, no recording,
  no region selection beyond the browser picker, no screen sharing outside a call.

`docs/calls/troubleshooting.md` additions: "getDisplayMedia fails immediately" (no user gesture),
"the peer sees a frozen frame after I stop sharing" (`track.onended` not handled), "the sharing indicator
stays after stopping" (`track.stop()`), "screen share text is unreadable" (framerate over resolution —
point at the quality defaults), "I cannot share because the other person is" (policy), "no audio in the
share" (browser support).

---

## Dependency justification

No new packages, .NET or npm. Screen sharing is `getDisplayMedia` plus the transceiver machinery Phase 12
already built.

---

## Documentation deltas

- `docs/calls/screen-sharing.md` — **new**, as specified.
- `docs/calls/video.md` — note the third pre-added transceiver and cross-link.
- `docs/calls/README.md` — screen sharing status, the `AllowedMedia` default change, `ScreenSharePolicy`.
- `docs/calls/troubleshooting.md` — the six new entries.
- `docs/testing/e2e.md` — the desktop-capture launch flags and any Firefox exclusion.
- `docs/security/authorization.md`, `docs/security/rate-limiting.md` — `call.screenshare.set`.
- `docs/configuration/README.md` — `ScreenSharePolicy`, `AllowedMedia` default change.
- `README.md` — screen sharing → `Available`. **This completes the WebRTC track**; update the feature
  table so chat, presence, typing, receipts, groups, audio, video, and screen sharing are all
  `Available`, with persistence, scale-out, and tooling remaining.

## Example deltas

- `examples/React` — screen sharing with the presenter layout and sharing indicator.
- `examples/NextJs` — screen sharing.
- Both READMEs — screen sharing instructions and the gesture requirement.

## CHANGELOG entry

```markdown
### Added
- Screen sharing during a call, delivered as a second video track alongside the camera so both can be
  shown simultaneously.
- `call.screenshare.set` operation and `ScreenSharePolicy` for arbitrating who may share.
- `startScreenShare` / `stopScreenShare` browser helpers, including handling of the browser's own
  "Stop sharing" control.
- Screen-optimised quality defaults favouring resolution over framerate.
- End-to-end tests asserting that a second video track carries screen media.

### Changed
- `CallOptions.AllowedMedia` now defaults to `Audio | Video | ScreenShare`.

### Notes
- A third video transceiver is reserved at call setup so starting a screen share never requires
  renegotiation.
- Applications using the raw WebRTC API must handle the screen track's `ended` event; the browser's own
  stop control is otherwise not reflected in call state.
```

---

## Exit criteria

- [ ] `start media call pre adds a second video transceiver for screen content` passes.
- [ ] `start screen share does not trigger renegotiation` passes.
- [ ] `stopping from the browser ui triggers the full stop path` passes — the `track.onended` handling
      is proven.
- [ ] `stop screen share stops the display track` passes.
- [ ] `hang up stops the screen track along with audio and video` passes.
- [ ] `user cancelling the picker rejects with the cancelled code` passes and is distinguishable.
- [ ] `remote screen track is distinguished from the remote camera track by mid` passes, and the
      fallback path is tested.
- [ ] `Start_sharing_preserves_the_participants_audio_and_video_flags` passes.
- [ ] `Concurrent_share_starts_from_both_participants_resolve_to_exactly_one_sharer_under_reject` passes.
- [ ] E2E: `screen share connects and screen media flows` passes, asserting on a **second** inbound
      video track's byte counters.
- [ ] E2E: `camera and screen are received as two distinct video tracks` passes.
- [ ] Any browser exclusion for screen-capture E2E is recorded in `docs/testing/e2e.md`.
- [ ] `docs/calls/screen-sharing.md` documents the `track.onended` requirement prominently.
- [ ] The React example shows the presenter layout, a persistent sharing indicator, and silent handling
      of a cancelled picker.
- [ ] `call.screenshare.set` is in the `KnownOperationPolicies` manifest and the rate-limit table.
- [ ] `README.md` feature table shows the complete WebRTC track as `Available`.
- [ ] `scripts/verify.ps1` green; tag `v0.12.0`.

## Verify

```powershell
pwsh scripts/verify.ps1
dotnet test tests/LiveSharp.Calls.Tests -c Release --filter "FullyQualifiedName~ScreenShare"
dotnet test tests/LiveSharp.IntegrationTests -c Release --filter "FullyQualifiedName~ScreenShare"
pnpm --dir clients/js -r test
pnpm --dir tests/e2e test -- --grep "screen share"
git tag v0.12.0
```

## Next

`docs/plan/phase-14-persistence.md`, task `P14.T1`.

> The WebRTC track is complete. Phases 14 onward are about making everything built so far durable,
> scalable, fast, and shippable. The feature surface stops growing here.
