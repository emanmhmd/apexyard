# Trusted Web Activity (TWA) for Google Play Distribution

> In the context of publishing Noor Nights on Google Play to reach Android users who prefer a native app experience, facing the need to distribute an app with zero additional native codebase, I decided to use Trusted Web Activity (TWA) via PWA Builder to achieve a Google Play-listed app that shares the existing service worker, FCM push stack, and codebase, accepting that the app is a Chrome shell and therefore inherits Chrome's constraints on certain device APIs.

## Context

Noor Nights is a fully-featured PWA: it has a manifest, service worker, offline capability, and (post-migration) FCM push notifications. Android users can "Add to Home Screen" via Chrome, but Google Play listing increases discoverability significantly — users searching for "Islamic prayer reminders" or "dhikr app" will not find a GitHub Pages PWA through Play Store search. Two approaches exist for publishing a PWA to Google Play without writing native Android code.

## Options Considered

| Option | Pros | Cons |
|--------|------|------|
| **TWA via PWA Builder** | Reuses 100% of existing PWA codebase; FCM push works natively (same service worker); no Kotlin/Java; no Android Studio required; Digital Asset Links verification removes Chrome UI chrome; free tooling | App is a Chrome shell — no Bluetooth, no NFC, no background geolocation; Play review still required; keystore loss = cannot update app |
| **React Native / Capacitor wrapper** | Full native API access; more control over splash/navigation | Requires maintaining a separate native codebase; FCM integration is a separate setup; significantly more engineering effort |
| **Flutter Web wrapper** | Good PWA support | Same codebase duplication problem; Flutter overhead for a primarily content app |
| **Stay PWA-only (no Play Store)** | Zero effort; Android users can still add to home screen | No Play Store discoverability; no push permission prompt on install; no star ratings |

## Decision

Chosen: **TWA via PWA Builder**, because the existing PWA already meets all TWA requirements (manifest, service worker, HTTPS, icons), the FCM push stack works identically inside a TWA, and the total engineering effort is configuration + APK generation rather than a new codebase. The Chrome-shell constraint does not affect any current Noor Nights feature.

## Consequences

- `assetlinks.json` must be deployed to `https://noor-nights.github.io/.well-known/assetlinks.json` and kept in sync with the signing keystore's SHA256 fingerprint.
- The signing keystore is a permanent artefact — if lost, the app cannot be updated on Play Store. It must be backed up in a password manager immediately after generation.
- Package name `com.noor.nights` is permanent once published — changing it requires a new Play Store listing.
- TWA validation can be tested before Play submission using Chrome's `adb logcat` or the Digital Asset Links testing tool.
- A $25 one-time Google Play Console registration fee is required.
- FCM notifications work inside the TWA via the same service worker — no additional FCM setup needed beyond the Phase 3–5 migration.

## Artifacts

- Implementation: Phase 6 of `projects/noor-nights/fcm-migration-plan.md`
- Tooling: [pwabuilder.com](https://www.pwabuilder.com)
- Verification: [Digital Asset Links tester](https://developers.google.com/digital-asset-links/tools/generator)
