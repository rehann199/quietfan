# Engineering Challenges

Six problems that stand out in a fan-sound player meant to run for hours in the background. Outcomes describe observable design behaviour, not measured metrics; none are claimed. The app is an audio utility, so these are software problems, not health or sleep-science problems.

---

## 1. Large catalog, small install footprint

**Problem**  
Thirty-one loop-length recordings cannot ship inside the APK without bloating installs and store delivery. Users still expect tap-to-play responsiveness even on uneven mobile networks.

**Context**  
Assets live in remote object storage. The catalog is static in the app; only bytes are remote.

**Engineering approach**  
- Resolve a deterministic local path per sound ID before every load.
- On cache hit, load from disk immediately.
- On miss, attach a streaming URI to the player and start a de-duplicated download in parallel.
- Write downloads via a temp file, retry up to three times, verify length against the response header, then atomic rename.
- After startup, prefetch remaining sounds sequentially with a short delay between files.
- Expose cache sizes and deletion in settings; delete corrupt files and retry.

**Trade-offs**  
First play of an uncached sound needs connectivity. Prefetch consumes data and storage over time; the user can clear cache manually.

**Outcome**  
Steady-state playback is offline-capable per sound. A bad file on disk is replaced without blocking the session permanently.

---

## 2. One player, three controllers

**Problem**  
The home mini player, full-screen player and system media notification must stay synchronized while loads, fades and engine callbacks arrive asynchronously.

**Context**  
Android exposes playback through a media session; Flutter screens bind to the same engine.

**Engineering approach**  
- Instantiate exactly one audio player for catalog sounds.
- Centralize load, play, pause, stop and fade logic in a dedicated service.
- Mirror transport state into an audio handler for notifications and lock-screen controls.
- Track user intent in the view model separately from engine position and processing state.
- Swiping the app from recents triggers an explicit stop and session teardown.

**Trade-offs**  
Skip/previous/next are not offered on the compact notification; only play, pause and stop.

**Outcome**  
No split-brain UI where the notification shows play while the screen shows pause.

---

## 3. Interruptions, headphones and UI sounds

**Problem**  
Calls, other apps, headphone unplug and short UI clicks all interact with audio focus differently. A background loop should pause when the user would expect silence, but UI feedback should not hijack the main session.

**Context**  
Android dispatches focus loss, transient ducking and becoming-noisy events.

**Engineering approach**  
- Configure an audio session with a music use case and handle interruptions in application code.
- Pause on any interruption; resume automatically only after transient pauses when policy allows.
- Pause on wired headphone unplug and cancel pending auto-resume.
- Route tap clicks through a separate lightweight player that does not request long-lived focus.

**Trade-offs**  
Automatic resume after transient events may not match every user's preference; manual pause always wins until the user presses play again.

**Outcome**  
Predictable pause behaviour; UI clicks remain audible without stopping the fan loop unintentionally.

---

## 4. Crossfades without fade stacking

**Problem**  
Users can tap through sounds faster than a fade completes. Overlapping volume animations produce muddy levels or wrong end states.

**Context**  
Crossfade duration is fixed and small enough to feel responsive.

**Engineering approach**  
- Assign a token or generation counter to each requested sound change.
- Cancel in-flight volume ramps when a newer request arrives.
- Ramp out the old source and in the new source on the same player instance where possible.

**Trade-offs**  
Very rapid switching may audibly clip the fade for perceived snappiness.

**Outcome**  
Only the latest user selection wins; volume returns to the user's setting after the fade completes.

---

## 5. Sleep timer, loop and fade as one ruleset

**Problem**  
Loop implies infinite playback; a sleep timer implies a definite end. Fade-out adds a third axis tied to wall-clock countdown.

**Context**  
Users expect timer and loop toggles to behave oppositely; fade is optional in settings.

**Engineering approach**  
- Arm timer → force loop off; enable loop → disarm timer.
- Advance timer ticks only while audio is in the playing state.
- When fade-out is enabled, step volume down over the final segment; at zero, stop player and media session together.
- Persist default timer duration and fade preference in settings.

**Trade-offs**  
The timer is driven by in-process ticks while the foreground service keeps the process eligible to run; it is not a single exact alarm API end-to-end.

**Outcome**  
One coherent mental model instead of three independent toggles fighting each other.

---

## 6. Ads, billing and launch latency

**Problem**  
Consent dialogs, ad SDK initialisation and billing service connection have unbounded or slow outcomes. Blocking the splash on them would make the app feel broken on bad networks.

**Context**  
Free tier shows ads; premium removes them and unlocks catalog rows. Remote switches can disable placements without shipping a new binary.

**Engineering approach**  
- Initialise only local preferences before the first frame.
- Run consent, Firebase, messaging, audio, billing and ads in guarded post-frame steps with timeouts on consent.
- Treat remote ad switches as off until a successful read; apply per-format cooldowns.
- Refresh entitlements from purchase history on launch; offer restore in settings.
- Skip ads entirely when premium entitlements are active.

**Trade-offs**  
Entitlement checks are client-side; server-side verification is a natural next step. Ad absence on failure is indistinguishable from a switched-off placement in the UI.

**Outcome**  
Playback and browsing remain available when monetisation SDKs misbehave; premium state eventually converges when the store responds.

---

## What is not covered here

Server-side receipt validation, multi-device favorite sync, gapless loop guarantees, automated tests for audio edge cases and an iOS release are out of scope for this portfolio's claims. For component diagrams see [Architecture](ARCHITECTURE.md); for a continuous narrative see [Technical case study](TECHNICAL_CASE_STUDY.md).
