# Technical Case Study

QuietFan is a Flutter application for Android that plays looping fan and appliance recordings from a fixed catalog, caches them on the device, and keeps a single long-running audio session controllable from the UI, notification shade and lock screen. This document walks through the engineering end to end. It describes software behaviour only; the app is an audio utility and makes no health or sleep-outcome claims (see [Product overview](PRODUCT_OVERVIEW.md)).

Approximate sizes: **96** hand-written Dart files and about **18.6k** lines under the application source folder (including roughly **1.9k** lines of language tables), plus one widget test. Diagram labels are conceptual and do not correspond to production identifiers.

## 1. Design goals

1. **One session, many surfaces.** Home, library, favorites, full player and the system media notification must never disagree about which sound is loaded or whether playback is active.
2. **Network as bootstrap, disk as steady state.** The install size cannot hold thirty-one long recordings, yet playback on a weak link must still start promptly.
3. **Polite audio citizenship.** Phone calls, other apps, headphone unplug and user pause each get an explicit policy; UI tap sounds must not steal focus from the main loop.
4. **Launch never blocks on SDKs.** Consent, ads, billing, push and analytics can take seconds or fail; the first frame should depend only on local preferences.
5. **Monetisation degrades safely.** Ad and store failures should mean fewer ads or locked premium rows, not a frozen splash screen.

## 2. Application structure

The codebase follows a familiar Flutter layout: screens and widgets at the top, `ChangeNotifier` view models exposed through Provider, and service classes for audio, downloads, billing, ads, reminders and Firebase integration.

- **Presentation:** tab shell (home, sounds, favorites, settings), modal sheets for volume and timer, a mini player anchored above the tab bar, and a dedicated now-playing route. The in-app product name on screen is Fan Sounds.
- **View models:** separate notifiers for the player, sleep timer, favorites and settings. The player notifier tracks **user intent** (what the user last asked for) separately from **engine state** (what the audio library reports), so slow loads and late callbacks cannot invert the play button.
- **Audio stack:** a singleton player backed by `just_audio`, wrapped by `audio_service` for the foreground media session, with `audio_session` configuring focus and interruption callbacks. A second, lightweight player handles short UI clicks without taking focus.
- **Download layer:** resolves a local path per catalog entry, streams on cache miss, writes atomically with retries and length verification, and runs a deferred prefetch queue after startup.
- **Persistence:** `shared_preferences` for favorites, volume, loop flags per sound, timer defaults, entitlements and UI settings; the app documents directory for cached audio bytes.
- **Cross-cutting:** Google Play Billing through `in_app_purchase`, mobile ads with a consent form, Firestore-backed placement switches, Firebase Analytics and Crashlytics, FCM topic subscription, local notifications for the bedtime reminder, Play in-app updates and connectivity checks.

## 3. Catalog and entitlements

The sound list is static data: identifier, display name, fan type, free-or-premium flag and remote audio location. The UI filters and groups by type; previous and next on the player screen walk the filtered ordering.

Premium access combines three paths: an active subscription, ownership of a one-time product family, or a time-limited unlock stored locally after a rewarded ad completes. Entitlements are refreshed from purchase history on launch, with a restore action in settings. Checks run on the device; server-side receipt validation is a natural next step but is not described here as present.

## 4. Cache-first playback

When the user selects a sound, the download service looks for a non-empty file at the expected path. On a hit, the player loads a file URI. On a miss, it attaches a streaming URI immediately so playback can begin while a background download runs. Downloads use a temporary file, up to three attempts, comparison of written length against the response content length, then rename into the final name. Duplicate download requests for the same sound are de-duplicated.

After the UI is up, a background iterator walks remaining catalog entries one at a time with a short pause between files so the radio is not saturated. Corrupt cache entries are deleted and re-fetched; the player can fall back to streaming for that session.

A settings screen lists cached objects with human-readable sizes and supports bulk deletion.

## 5. Media session and foreground service

Long sessions require Android's foreground media pattern. An audio handler implements play, pause and stop, publishes metadata and artwork for the active sound, and exposes compact notification actions. Media button events toggle play and pause. Tapping the notification returns to the now-playing route. When the user swipes the app away from recents, the handler stops playback and disposes the session deliberately rather than leaving a silent notification.

Previous and next are intentionally omitted from the compact session controls; skipping stays on the full player where type-scoped navigation is defined.

## 6. Interruptions, focus and fades

The audio session uses a music-oriented profile with the player's built-in interruption handling disabled so application code owns the policy. Any interruption pauses playback. Transient interruptions (short ducking events) may resume automatically when the platform signals an end; permanent ones do not. Unplugging wired headphones pauses and clears any pending auto-resume token.

Sound changes run a crossfade: the outgoing source ramps down while the incoming ramps up over a bounded interval. A monotonic token cancels in-flight fades when the user picks another sound quickly, preventing stacked envelopes.

## 7. Loop mode and sleep timer

Loop is implemented with the platform player's single-item loop mode and is remembered per sound in preferences. The sleep timer is an in-app countdown driven by a periodic ticker while playback is active; it pauses when the user pauses audio.

Mutual exclusion keeps semantics clear: arming a timer forces loop off; turning loop on disarms the timer. With fade-out enabled, volume steps down over the final segment of the countdown; at zero the player stops and the media session ends. This is a playback convenience, not sleep tracking.

## 8. Reminders and notifications

The bedtime reminder schedules weekly repeating local notifications via `flutter_local_notifications` with timezone-aware firing. Permission prompts follow platform rules. The reminder does not start audio automatically; it only notifies at the configured wall time on selected weekdays.

## 9. Ads and remote switches

Before the ad SDK starts, the consent form runs with bounded waits so launch cannot hang indefinitely. Each ad placement consults a remote switch document; failed reads default to off. Cooldown timers gate interstitial-style formats. Premium entitlements bypass ad presentation. Rewarded ads gate temporary premium playback per product rules implemented in the view layer and ad callback.

No fill rate, eCPM or revenue figures are stated in this repository.

## 10. Start-up sequence

`runApp` waits only for essential local preference reads. Plugin initialisation—audio session, download hooks, billing connection, notification channels, Firebase, messaging, ads—runs in ordered, try/caught steps after the first frame. Failures log to crash reporting where configured and leave the corresponding feature inactive rather than crashing the isolate.

## 11. Localisation and theming

Seven locales ship as bundled ARB-style tables compiled into Dart. Settings language changes rebuild the widget tree with the chosen locale. Dark mode is a three-way preference: light, dark or follow system.

## 12. Tests and quality

Automated coverage is intentionally thin: one widget test asserts that the splash route renders. Audio timing, download retries, timer edge cases, billing and ad state machines have no dedicated test files in the counted suite. No test pass or failure is claimed here.

## 13. Trade-offs

- Timer accuracy depends on foreground service keep-alive and periodic ticks, not on a single wall-clock alarm at end time.
- Minimal media session actions reduce notification clutter but also limit control from the lock screen.
- Client-side entitlements match common store SDK patterns; server verification would harden premium access.
- iOS configuration exists for background audio and tracking consent, but only Android is released.

Deeper problem statements: [Engineering challenges](ENGINEERING_CHALLENGES.md). Component boundaries: [Architecture](ARCHITECTURE.md).
