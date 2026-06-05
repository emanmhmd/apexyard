# FCM Topics as the Notification Fan-Out Model

> In the context of broadcasting daily prayer reminders to all opted-in Noor Nights users from a GitHub Actions cron job, facing the need to send to every subscriber without maintaining a persistent device-token database, I decided to use FCM Topics (specifically the `daily-reminders` topic) to achieve a stateless, zero-database fan-out at no cost, accepting that topic membership cannot be queried for analytics without additional infrastructure.

## Context

Once users subscribe, the GitHub Actions cron job needs to send push notifications to all of them every hour. There are two canonical FCM patterns for broadcast:

1. **Device token list** — store each token in a database; send using the `registration_ids` batch field or loop over tokens.
2. **FCM Topics** — subscribe each token to a named topic once (at opt-in time); send a single message to `/topics/daily-reminders` to reach all subscribers.

The project has no backend database and no budget for one. GitHub Pages is static. Any database would introduce a new paid dependency (Firestore, D1, Supabase, etc.) and a token-management lifecycle (stale tokens, churn, deduplication).

## Options Considered

| Option | Pros | Cons |
|--------|------|------|
| **FCM Topics** | Zero database; single send call; scales to any subscriber count; subscription managed by Google | Cannot enumerate subscribers; no per-user segmentation; topic membership not queryable via API |
| **Token database (e.g. Cloudflare D1 or KV)** | Full subscriber list; per-user analytics; segmentation possible | Requires storage account + writes + token churn management; adds $0–$5/month and significant complexity for a single-topic broadcast |
| **FCM Multicast (registration_ids batch)** | No Google-managed state | 500-token limit per request; requires batching logic; database still needed to store tokens |

## Decision

Chosen: **FCM Topics**, because the only use case is broadcast (one message → all subscribers) and there is no current requirement for per-user targeting, segmentation, or subscriber count analytics. Topics are the idiomatic FCM pattern for exactly this use case, and the zero-database approach matches the project's static-PWA constraint.

## Consequences

- The GitHub Actions sender becomes a single `fetch` to `https://fcm.googleapis.com/v1/projects/{id}/messages:send` with `topic: "daily-reminders"` — no pagination, no batching.
- Subscriber count cannot be tracked server-side without additional infrastructure. If analytics become a future requirement, a Cloudflare KV token store would need to be added at that point.
- Topic subscription happens at opt-in via the Cloudflare Worker (Phase 2). Unsubscription is not yet implemented — users who revoke notification permission in the browser will stop receiving pushes but remain a topic member until their token rotates.
- Token rotation is handled automatically by FCM: when a token changes, the old topic membership is eventually invalidated by Google.

## Artifacts

- Migration plan: `projects/noor-nights/fcm-migration-plan.md` (Architecture section, "Why FCM Topics?")
- Sender implementation: Phase 5 of the migration plan (`automated_hourly_push.js`)
