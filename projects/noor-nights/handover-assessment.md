# noor-nights — Handover Assessment

**Date**: 2026-05-18
**Assessor**: emanmhmd
**Status**: handover

## Origin

- **Where it came from**: Owner's own project, brought under ApexYard management
- **Original owner**: emanmhmd (Noor-Nights org)
- **Repo location**: https://github.com/Noor-Nights/noor-nights.github.io
- **First commit date**: 2026-03-06
- **Last commit date**: 2026-05-17

## Current State

### Tech stack

- **Type**: Progressive Web App (PWA) — static, no build step
- **Language**: Vanilla HTML5 / CSS3 / JavaScript (ES6+)
- **Frontend framework**: None — raw DOM manipulation
- **Backend**: BaaS only
  - Supabase (Community Dua Wall — PostgreSQL via REST)
  - OneSignal (push notifications)
- **Hosting**: GitHub Pages (auto-deploy on push to `main`)
- **CI/CD**: GitHub Actions (deploy workflow + hourly push scheduler)
- **Test framework**: None detected
- **IaC**: None (static hosting, no infra config)

### Build status

- `npm install`: N/A — no package.json, no build step
- `npm run build`: N/A
- `npm run test`: N/A — no test suite
- `npm run lint`: N/A — no linter configured

### Test coverage

- **Estimated**: 0% — no tests exist

### Repo activity

- **Commits total**: ~30 (sole contributor)
- **Commits in last 90 days**: ~30 (created and actively developed March–May 2026)
- **Open issues**: 0
- **Open PRs**: 0
- **Top contributors**: emanmhmd (29 commits), Eman Mahmoud (1 commit)

## Quality Risks

### Security

- **Supabase anon key injected via `sed` at deploy time** — the anon key is public-safe (row-level security should protect data), but the injection pattern (`__SUPABASE_URL__` / `__SUPABASE_ANON_KEY__` replaced in-place with `sed`) could silently deploy broken JS if a secret is missing or the placeholder is renamed. No validation step in the deploy workflow.
- **OneSignal APP_ID hardcoded** in `automated_hourly_push.js` (line ~6) — this is a public identifier, not a secret, but it should live in the workflow's `env` block alongside `ONESIGNAL_REST_API_KEY` for consistency and easier rotation.
- **No Content-Security-Policy headers** — GitHub Pages doesn't allow server-side header injection; a `<meta>` CSP tag should be added to `index.html` to protect against XSS.

### Dependencies

- **No `package.json`** — no npm dependency surface to audit, which is a feature. The Node.js runtime dependency (`node 20`) comes from the GitHub Actions runner, not a lockfile.
- **OneSignal SDK loaded via CDN** — `OneSignalSDKWorker.js` at repo root is the bundled OneSignal service worker. CDN-hosted scripts have no lockfile equivalent; version drift is invisible until a breaking change ships.

### Technical debt

- **Zero test coverage** — all logic (countdown, worship tracker, tasbeeh counter, dua wall) is untested. Any regression is discovered by users.
- **Seasonal hardcoding**: Dhul Hijjah 1447 dates are hardcoded as `2026-05-18` to `2026-05-27` in `automated_hourly_push.js`. This needs updating for Dhul Hijjah 1448 (approximate: May 2027) — a calendar-based lookup would make this self-maintaining.
- **No linting or formatting** — no ESLint/Prettier config; code style is consistent by convention only.
- **Legacy service worker migration** (`sw.js`) — exists only to unregister itself and hand over to `OneSignalSDKWorker.js`. Once all active users have migrated (session after first visit post-deploy), this file is dead code. No mechanism to know when it's safe to remove.

### Operational

- **No error tracking** — no Sentry, Datadog, or similar. JS errors on the client are invisible to the developer.
- **No uptime monitoring** — GitHub Pages is generally reliable but there's no alert if the site goes down.
- **Hourly push workflow runs 21 times/day** — GitHub Actions free tier has 2,000 min/month; each run is ~30s. At 21 runs/day × 30s, this is ~10.5 min/day or ~315 min/month. Well within limits now, but worth tracking if workflows grow.

## Integration Plan

### Roles that apply

- `tech-lead` — technical decisions, PR review
- `frontend-engineer` — HTML/CSS/JS, PWA features, UI work
- `platform-engineer` — GitHub Actions workflows (deploy + push scheduler)
- `sre` — OneSignal + Supabase integrations, scheduled jobs, uptime

### Workflows that kick in

- [x] PR workflow (`.claude/rules/pr-workflow.md`) — every change goes through a PR
- [x] AgDR for technical decisions (e.g. adding a test framework, switching CDN strategy)
- [x] Code Reviewer agent (Rex) on every PR
- [ ] Security Reviewer on first pass (Supabase RLS review recommended)
- [ ] `/audit-deps` — N/A for npm, but OneSignal CDN and Supabase SDK versions should be reviewed

### Hooks to enable

- [ ] `block-git-add-all`
- [ ] `block-main-push`
- [ ] `validate-branch-name`
- [ ] `validate-pr-create`
- [ ] `pre-push-gate`
- [ ] `check-secrets`

### CI templates to copy in

- [ ] `golden-paths/pipelines/pr-title-check.yml` — enforce ticket IDs in PR titles
- [ ] `golden-paths/pipelines/security.yml` — Semgrep SAST on JS files

### Registry entry

See `apexyard.projects.yaml` — entry added during this handover.

## Next Steps

1. **Add a `<meta>` CSP tag to `index.html`** — mitigates XSS risk on a public-facing PWA with user-generated content (Dua Wall). Minimum viable policy: `default-src 'self'; connect-src 'self' https://*.supabase.co https://*.onesignal.com`.
2. **Validate the Supabase secret injection in `deploy.yml`** — add a check after the `sed` step that verifies `__SUPABASE_URL__` and `__SUPABASE_ANON_KEY__` are no longer present in `src/js/app.js` before deploying. A broken deploy currently reaches users silently.
3. **Parameterise Dhul Hijjah dates in `automated_hourly_push.js`** — extract `DH_START` and `DH_END` to workflow-level env vars (`DHUL_HIJJAH_START`, `DHUL_HIJJAH_END`), so next year's update is a one-line `ramadan_hourly_push.yml` edit, not a code change.
4. **Add Sentry (or equivalent) error tracking** — the PWA has non-trivial JS logic (tasbeeh, tracker, countdown, dua wall). Silent JS errors on user devices are currently invisible. `/decide` on observability tool first.
5. **Copy in `golden-paths/pipelines/pr-title-check.yml`** — enforce conventional commit format on PRs from day one to keep the git history navigable as the project grows.
6. **Code review the most recent PR on this repo as Rex** — calibrate review standards against the existing codebase before the first feature PR.

## Post-Handover Checklist

- [ ] Complete Step 1: add CSP meta tag to `index.html` — first security hardening before any new feature
- [ ] Complete Step 2: deploy-time secret validation — prevent silent broken deploys
- [ ] Complete Step 3: parameterise Dhul Hijjah dates — prevents a calendar bug in 1448
- [ ] Add `noor-nights` to weekly `/stakeholder-update` rollup
- [ ] Set up a minimal smoke test (load `index.html` and assert the countdown element renders) as a baseline before expanding features
- [ ] Review Supabase row-level security rules to confirm the Dua Wall can't be spammed or scraped

## Open Questions

- What is the intended release cadence — seasonal (Dhul Hijjah + Ramadan) or year-round development?
- Is there a plan to expand to other Islamic occasions (Ramadan, general daily tracker)?
- What does the Supabase schema look like for the Dua Wall — is there any rate-limiting or moderation?
- Is the `legacy sw.js` safe to remove now, or are there users still on the old registration?
