# Replace OneSignal with Firebase Cloud Messaging (FCM)

> In the context of Noor Nights push notifications for prayer reminders and Dhul Hijjah dhikr, facing OneSignal's 1,000-subscriber free-tier cap, its FCM-wrapping overhead, and its incompatibility with the Google Play TWA path, I decided to migrate directly to Firebase Cloud Messaging to achieve a cost-free, scalable, and Google Play-ready notification pipeline, accepting a one-time re-subscription requirement for existing OneSignal users.

## Context

Noor Nights is a static PWA on GitHub Pages. Push notifications (prayer times, Dhul Hijjah dhikr) are sent via a GitHub Actions cron job. The current stack uses OneSignal, which wraps FCM under the hood for Android delivery. Three constraints forced a re-evaluation:

- **Free tier cap**: OneSignal free plan caps web push at 1,000 subscribers — growth beyond that requires a paid plan (~$9/month minimum).
- **Indirection cost**: OneSignal proxies FCM; every message goes OneSignal → FCM → device. There is no benefit to this indirection for a project of this size.
- **TWA blocker**: Publishing Noor Nights on Google Play via Trusted Web Activity requires direct FCM integration. OneSignal cannot be used as the push layer in a TWA context.

## Options Considered

| Option | Pros | Cons |
|--------|------|------|
| **Stay on OneSignal** | No migration effort; dashboard analytics; easy A/B testing | $9+/month above 1k subscribers; FCM indirection; dead end for TWA |
| **Migrate to FCM directly** | Free unlimited messages; native Android push; enables TWA; no vendor lock-in | Migration effort; existing subscribers must re-subscribe; requires service account management |
| **Web Push API (no vendor)** | Zero dependencies; full VAPID control | No Android/TWA support; no topic fan-out; requires own token DB |

## Decision

Chosen: **Migrate to FCM directly**, because it removes the paid-tier ceiling, is the native push layer for TWA, and the subscriber re-subscription cost is a one-time event acceptable for a small active user base.

## Consequences

- Existing OneSignal subscribers will not automatically migrate — a one-time in-app notice is recommended at launch.
- GitHub Actions sender must use the FCM HTTP v1 API with a service account OAuth2 token (not the legacy Server Key directly).
- Firebase project must be created and maintained; service account JSON stored as a GitHub Actions secret.
- Migration is additive — OneSignal can be left enabled in parallel during the cutover window, then removed once FCM subscriber count stabilises.

## Artifacts

- Migration plan: `projects/noor-nights/fcm-migration-plan.md`
- Implementation phases: Phase 1 (Firebase setup) through Phase 5 (GitHub Actions sender) in the migration plan
