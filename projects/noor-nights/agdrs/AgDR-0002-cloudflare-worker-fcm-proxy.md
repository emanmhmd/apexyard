# Cloudflare Worker as FCM Topic-Subscription Proxy

> In the context of subscribing browser FCM tokens to a server-side topic, facing the requirement to call the FCM IID API with a Firebase Server Key that must never be exposed in client-side JavaScript, I decided to use a Cloudflare Worker as a thin serverless proxy to achieve secret isolation at zero ongoing cost, accepting a Cloudflare account dependency and one additional network hop per subscription.

## Context

When a user opts into push notifications, the browser receives an FCM registration token via the Firebase JS SDK. To fan out messages to all users without storing tokens, the design uses FCM Topics — each token must be subscribed to the `daily-reminders` topic via the FCM IID API. That API call requires the Firebase Server Key as an `Authorization` header. Embedding the Server Key in client-side JS would expose it in every browser's devtools and in the public GitHub repo.

## Options Considered

| Option | Pros | Cons |
|--------|------|------|
| **Cloudflare Worker proxy** | Free (100k req/day); ~30 lines of JS; secret stored in CF env; no cold-start latency concern at this traffic volume; same origin as potential future CF infrastructure | Cloudflare account required; one more moving part; CF outage affects subscriptions (not deliveries) |
| **GitHub Pages server-side** | No extra account | GitHub Pages is static-only — no server-side execution possible |
| **Firebase Cloud Functions** | Same ecosystem; no extra account | Requires Blaze (pay-as-you-go) plan; overkill for a 30-line proxy |
| **Expose server key in client JS** | Zero infrastructure | Server key exposed publicly — anyone can send unlimited FCM messages to all subscribers; non-starter |
| **Skip topic subscription; store tokens in KV** | Server key not needed for HTTP v1 send | Requires a token database (KV store, D1, or similar); adds storage cost and maintenance; token churn management needed |

## Decision

Chosen: **Cloudflare Worker proxy**, because it keeps the Server Key off the client at no cost, requires no plan upgrade, and the ~30-line implementation is proportional to the problem — a full Cloud Functions setup would be over-engineered for a single-endpoint secret proxy.

## Consequences

- A Cloudflare account must be created and maintained.
- `FIREBASE_SERVER_KEY` is stored as a Cloudflare Worker secret — rotation requires a CF dashboard action.
- Worker URL must be hardcoded in `app.js` as `CF_SUBSCRIBE_URL`; a domain name alias (e.g. `subscribe.noor-nights.workers.dev`) is recommended for stability.
- CF Worker failure (outage or misconfiguration) means new subscribers cannot join the topic until fixed — existing subscribers continue receiving notifications unaffected.
- The FCM IID API used for topic subscription is a legacy Google API (see Risk Register in migration plan — flagged for future migration to FCM HTTP v1 management API when available).

## Artifacts

- Implementation: Phase 2 of `projects/noor-nights/fcm-migration-plan.md`
- Worker source: `cloudflare-worker/subscribe.js` (to be created in `Noor-Nights/noor-nights.github.io`)
