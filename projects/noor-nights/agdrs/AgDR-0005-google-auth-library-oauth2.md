# google-auth-library as OAuth2 Client for FCM HTTP v1 in GitHub Actions

> In the context of authenticating the GitHub Actions cron sender against the FCM HTTP v1 API using a service account, facing the need to exchange a service account JSON key for a short-lived OAuth2 Bearer token on every invocation, I decided to use the official `google-auth-library` npm package to achieve standards-compliant service account authentication with minimal code, accepting a ~2MB npm dependency and ~10 seconds of additional GitHub Actions install time.

## Context

The FCM HTTP v1 API (the current, non-deprecated FCM send API) requires an OAuth2 Bearer token scoped to `https://www.googleapis.com/auth/firebase.messaging`. This token must be obtained from Google's token endpoint by signing a JWT with the service account private key. Two approaches exist: implement the JWT signing and token exchange manually, or use Google's official auth library.

The GitHub Actions job is a Node.js script (`automated_hourly_push.js`) that runs on a schedule. It has no persistent state between runs — it must fetch a fresh OAuth2 token on every invocation.

## Options Considered

| Option | Pros | Cons |
|--------|------|------|
| **`google-auth-library` npm package** | Official Google library; handles JWT signing, token caching within a process, and automatic refresh; well-tested; 3-line integration | Adds ~2MB to node_modules; `npm install` step adds ~10s to CI job cold start |
| **Manual JWT + token exchange** | Zero dependencies; full control | ~80 lines of boilerplate (RS256 JWT signing with `crypto` module, `POST` to token endpoint, parse response); error-prone; maintenance burden when Google rotates endpoints |
| **`firebase-admin` SDK** | Official SDK; messaging send built-in | ~15MB package; designed for server-side apps with persistent connections; significant overkill for a one-shot cron script |
| **gcloud CLI in Actions** | No Node.js dependency | Requires `google-github-actions/auth` setup action; adds complexity; not idiomatic for a Node.js script |

## Decision

Chosen: **`google-auth-library`**, because it is Google's official Node.js auth library, handles service account token lifecycle correctly out of the box, and the 3-line integration (`new GoogleAuth({ credentials, scopes }) → getClient() → getAccessToken()`) is far preferable to implementing JWT RS256 signing manually. The ~10s CI overhead is negligible against the ~30s total job duration.

## Consequences

- `package.json` in the root of `Noor-Nights/noor-nights.github.io` gains a `google-auth-library: ^9.0.0` dependency.
- The GitHub Actions workflow must include an `npm install` step before the Node.js script runs.
- `FIREBASE_SERVICE_ACCOUNT_JSON` is stored as a GitHub Actions secret and parsed at runtime with `JSON.parse(process.env.FIREBASE_SERVICE_ACCOUNT_JSON)`.
- The service account JSON must never be committed to the repository — it is a full private key and grants FCM send permissions.
- Token caching within `google-auth-library` is per-process only; since each cron invocation is a new process, a fresh token is fetched on every run. This is acceptable — the token endpoint latency is ~200ms.

## Artifacts

- Implementation: Phase 5 of `projects/noor-nights/fcm-migration-plan.md` (`automated_hourly_push.js`, `getAccessToken()` function)
- Package: [npmjs.com/package/google-auth-library](https://www.npmjs.com/package/google-auth-library)
