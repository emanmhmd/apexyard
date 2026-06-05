# FCM Migration Plan — Noor Nights Push Notifications + Google Play

**Status:** Planning  
**Created:** 2026-05-18  
**Goal:** Replace OneSignal with Firebase Cloud Messaging (FCM) for reliable push notifications on web and Android, then publish the app on Google Play via Trusted Web Activity (TWA).

### Technical Decisions (AgDRs)

| Decision | AgDR |
|----------|------|
| Replace OneSignal with FCM | [AgDR-0001](agdrs/AgDR-0001-fcm-over-onesignal.md) |
| Cloudflare Worker as FCM topic-subscription proxy | [AgDR-0002](agdrs/AgDR-0002-cloudflare-worker-fcm-proxy.md) |
| FCM Topics as the fan-out model (vs. token database) | [AgDR-0003](agdrs/AgDR-0003-fcm-topics-fan-out.md) |
| Trusted Web Activity (TWA) for Google Play | [AgDR-0004](agdrs/AgDR-0004-twa-for-google-play.md) |
| `google-auth-library` as OAuth2 client in GitHub Actions | [AgDR-0005](agdrs/AgDR-0005-google-auth-library-oauth2.md) |

---

## Background

Noor Nights is a static PWA hosted on GitHub Pages. It currently uses OneSignal for push notifications (prayer reminders, Dhul Hijjah dhikr) sent via a GitHub Actions cron job. Several issues have surfaced:

- OneSignal wraps FCM for Android — we pay the indirection cost without benefit
- OneSignal free tier caps at 1,000 web push subscribers
- For Google Play publishing via TWA, FCM is the native push layer — OneSignal is a dead end

This plan migrates to FCM directly and sets up the TWA pipeline for Google Play.

---

## Architecture (Target State)

```
[User Browser / TWA App]
    │
    ├─ Firebase JS SDK
    │       ├─ initializeApp(config)
    │       ├─ getToken(messaging, { vapidKey, serviceWorkerRegistration })
    │       │         → FCM registration token
    │       └─ POST token → Cloudflare Worker
    │
    └─ Service Worker (sw.js)
            ├─ Firebase Messaging (background push handler)
            └─ PWA Cache (install / activate / fetch — unchanged)

[Cloudflare Worker — subscribe.noor-nights.workers.dev]
    │  POST { token }
    └─ Calls FCM IID API → subscribes token to topic "daily-reminders"
       (server key stored as CF secret — never exposed to client)

[GitHub Actions CRON — every hour, 00:00–20:00 UTC]
    │
    └─ automated_hourly_push.js
            ├─ Gets OAuth2 token from service account JSON
            └─ FCM HTTP v1 API → sends to /topics/daily-reminders
               (prayer notifications + Dhul Hijjah dhikr — content unchanged)

[Google Play — TWA]
    └─ Android shell (Chrome) loading https://noor-nights.github.io
            ├─ Same service worker
            ├─ Same FCM push
            └─ Verified via /.well-known/assetlinks.json
```

**Why FCM Topics?** Subscribing devices to a topic (`daily-reminders`) means the GitHub Actions sender broadcasts to one endpoint — no database of tokens needed, scales to any number of users for free.

---

## Phase 1 — Firebase Project Setup (Manual — You Do This)

> No code changes. This is prerequisite setup in the Firebase Console.

### Steps

1. Go to [console.firebase.google.com](https://console.firebase.google.com) → **Create project** → name it `noor-nights`
2. Inside the project → **Add app** → choose **Web** (`</>`) → register app as `noor-nights-web`
3. Copy the Firebase config object shown (you'll need it in Phase 3):
   ```js
   const firebaseConfig = {
     apiKey: "...",
     authDomain: "...",
     projectId: "...",
     storageBucket: "...",
     messagingSenderId: "...",
     appId: "..."
   };
   ```
4. Go to **Project Settings → Cloud Messaging** tab:
   - Under **Web Push certificates** → click **Generate key pair** → copy the VAPID public key
   - Copy the **Server key** (shown under Legacy APIs — needed for topic subscription)
5. Go to **Project Settings → Service Accounts** → click **Generate new private key** → download the JSON file
6. Add secrets to the GitHub repo (`Noor-Nights/noor-nights.github.io` → Settings → Secrets → Actions):

| Secret name | Value |
|-------------|-------|
| `FIREBASE_PROJECT_ID` | Your Firebase project ID (e.g. `noor-nights-abc12`) |
| `FIREBASE_SERVER_KEY` | The server key from step 4 |
| `FIREBASE_SERVICE_ACCOUNT_JSON` | Full contents of the downloaded JSON file |

### Done when

- [ ] Firebase project created
- [ ] Web app registered, config object copied
- [ ] VAPID key generated and copied
- [ ] Server key copied
- [ ] Service account JSON downloaded
- [ ] All 3 GitHub secrets added

---

## Phase 2 — Cloudflare Worker: Topic Subscription Proxy

**Why needed:** The FCM IID API that subscribes a device token to a topic requires the Firebase Server Key. This key must never be exposed in client-side JS. A Cloudflare Worker acts as a free, serverless proxy (~30 lines of JS).

**Free tier:** 100,000 requests/day — more than sufficient.

### New file: `cloudflare-worker/subscribe.js`

```js
export default {
  async fetch(request, env) {
    if (request.method === 'OPTIONS') {
      return new Response(null, {
        headers: {
          'Access-Control-Allow-Origin': 'https://noor-nights.github.io',
          'Access-Control-Allow-Methods': 'POST',
          'Access-Control-Allow-Headers': 'Content-Type',
        },
      });
    }

    if (request.method !== 'POST') {
      return new Response('Method not allowed', { status: 405 });
    }

    const { token } = await request.json();
    if (!token) return new Response('Missing token', { status: 400 });

    const res = await fetch(
      `https://iid.googleapis.com/iid/v1/${token}/rel/topics/daily-reminders`,
      {
        method: 'POST',
        headers: { Authorization: `key=${env.FIREBASE_SERVER_KEY}` },
      }
    );

    return new Response(res.ok ? 'subscribed' : 'failed', {
      status: res.ok ? 200 : 502,
      headers: { 'Access-Control-Allow-Origin': 'https://noor-nights.github.io' },
    });
  },
};
```

### Cloudflare setup steps

1. Create a free Cloudflare account at [cloudflare.com](https://cloudflare.com)
2. Go to **Workers & Pages** → **Create Worker** → name it `noor-nights-subscribe`
3. Paste the worker code above
4. Go to **Settings → Variables** → add secret: `FIREBASE_SERVER_KEY` = your server key from Phase 1
5. Note the worker URL: `https://noor-nights-subscribe.<your-cf-account>.workers.dev`

### Done when

- [ ] Cloudflare account created
- [ ] Worker deployed and accessible
- [ ] `FIREBASE_SERVER_KEY` secret set in CF
- [ ] Worker URL noted for Phase 3

---

## Phase 3 — Replace OneSignal with Firebase Messaging in Web App

**Files changed:** `index.html`, `src/js/app.js`

### 3a. `index.html` — swap SDK scripts

Remove:
```html
<script src="https://cdn.onesignal.com/sdks/web/v16/OneSignalSDK.page.js" defer></script>
<script>window.OneSignalDeferred = window.OneSignalDeferred || [];</script>
```

Add (before closing `</body>`):
```html
<script src="https://www.gstatic.com/firebasejs/10.12.0/firebase-app-compat.js"></script>
<script src="https://www.gstatic.com/firebasejs/10.12.0/firebase-messaging-compat.js"></script>
```

### 3b. `app.js` — replace `requestNotifications()`

Remove all OneSignal references (`OneSignalDeferred`, `OneSignal.User.PushSubscription`, `optIn()`, `optedIn`).

Replace with:

```js
// Firebase config — public values, safe to embed
const FIREBASE_CONFIG = {
  apiKey: "...",
  authDomain: "...",
  projectId: "...",
  storageBucket: "...",
  messagingSenderId: "...",
  appId: "..."
};
const FIREBASE_VAPID_KEY = "..."; // from Phase 1 step 4
const CF_SUBSCRIBE_URL = "https://noor-nights-subscribe.<account>.workers.dev";

let _messaging = null;

function _getMessaging() {
  if (!_messaging) {
    firebase.initializeApp(FIREBASE_CONFIG);
    _messaging = firebase.messaging();
  }
  return _messaging;
}

async function requestNotifications() {
  const btn = document.getElementById('notify-btn');
  if (!btn) return;

  const permission = await Notification.requestPermission();
  if (permission !== 'granted') {
    _updateNotifyBtnState(btn, false);
    return;
  }

  try {
    const reg = await navigator.serviceWorker.ready;
    const messaging = _getMessaging();
    const token = await firebase.messaging.getToken(messaging, {
      vapidKey: FIREBASE_VAPID_KEY,
      serviceWorkerRegistration: reg,
    });

    // Subscribe to FCM topic via CF Worker
    await fetch(CF_SUBSCRIBE_URL, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ token }),
    });

    localStorage.setItem('noor-push-opted-in', '1');
    _updateNotifyBtnState(btn, true);
    _sendSuccessNotification();
  } catch (err) {
    console.warn('FCM subscription failed:', err);
    _updateNotifyBtnState(btn, false);
  }
}
```

Button state init on page load (localStorage restore) — no change needed, same logic as current.

### Done when

- [ ] OneSignal scripts removed from `index.html`
- [ ] Firebase scripts added
- [ ] `requestNotifications()` uses Firebase SDK
- [ ] Token is POSTed to CF Worker
- [ ] Button state persists across reloads
- [ ] Success notification fires after subscribe

---

## Phase 4 — Service Worker Refactor

**File renamed:** `OneSignalSDKWorker.js` → `sw.js`

> Also update the SW registration in `app.js` from `OneSignalSDKWorker.js` to `sw.js`.

### `sw.js` — replace OneSignal import with Firebase Messaging

Remove:
```js
importScripts("https://cdn.onesignal.com/sdks/web/v16/OneSignalSDK.sw.js");
```

Add:
```js
importScripts('https://www.gstatic.com/firebasejs/10.12.0/firebase-app-compat.js');
importScripts('https://www.gstatic.com/firebasejs/10.12.0/firebase-messaging-compat.js');

firebase.initializeApp({
  apiKey: "...",
  projectId: "...",
  messagingSenderId: "...",
  appId: "..."
});

const messaging = firebase.messaging();

messaging.onBackgroundMessage((payload) => {
  const { title, body, icon } = payload.notification;
  self.registration.showNotification(title, {
    body,
    icon: icon || 'https://noor-nights.github.io/assets/icons/icon-512.png',
    badge: 'https://noor-nights.github.io/assets/icons/icon-96-mono.png',
    silent: false,
    vibrate: [200, 100, 200],
    data: { url: 'https://noor-nights.github.io' },
  });
});
```

PWA caching block (CACHE_NAME, ASSETS, install/activate/fetch handlers) — kept exactly as-is, only bump `CACHE_NAME` to `noor-nights-v24`.

### Done when

- [ ] SW file renamed to `sw.js`
- [ ] SW registration in `app.js` updated
- [ ] Firebase Messaging initialized in SW
- [ ] `onBackgroundMessage` handler shows notification with absolute icon URLs
- [ ] PWA caching logic preserved
- [ ] Cache version bumped

---

## Phase 5 — Update GitHub Actions Sender

**File changed:** `automated_hourly_push.js`, `.github/workflows/ramadan_hourly_push.yml`

### 5a. `automated_hourly_push.js` — replace `sendPush()`

The prayer times logic, duas arrays, and all message content are **unchanged**. Only the `sendPush()` function and environment variable names change.

Remove:
```js
const APP_ID = process.env.ONESIGNAL_APP_ID;
const REST_API_KEY = process.env.ONESIGNAL_REST_API_KEY;

async function sendPush(heading, body_text, collapseId) {
  // OneSignal REST API call...
}
```

Add:
```js
const PROJECT_ID = process.env.FIREBASE_PROJECT_ID;
const SERVICE_ACCOUNT = JSON.parse(process.env.FIREBASE_SERVICE_ACCOUNT_JSON);

// Get OAuth2 token from service account
async function getAccessToken() {
  const { GoogleAuth } = require('google-auth-library');
  const auth = new GoogleAuth({
    credentials: SERVICE_ACCOUNT,
    scopes: ['https://www.googleapis.com/auth/firebase.messaging'],
  });
  const client = await auth.getClient();
  const token = await client.getAccessToken();
  return token.token;
}

async function sendPush(heading, body_text, collapseId) {
  const accessToken = await getAccessToken();
  const payload = {
    message: {
      topic: 'daily-reminders',
      notification: {
        title: heading,
        body: body_text,
      },
      android: {
        notification: {
          icon: 'icon_512',
          color: '#c9a84c',
          sound: 'default',
          channel_id: 'prayer_reminders',
          tag: collapseId,
        },
      },
      webpush: {
        notification: {
          icon: 'https://noor-nights.github.io/assets/icons/icon-512.png',
          badge: 'https://noor-nights.github.io/assets/icons/icon-96-mono.png',
          tag: collapseId,
          silent: false,
          vibrate: [200, 100, 200],
        },
        fcm_options: { link: 'https://noor-nights.github.io' },
      },
    },
  };

  const res = await fetch(
    `https://fcm.googleapis.com/v1/projects/${PROJECT_ID}/messages:send`,
    {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        Authorization: `Bearer ${accessToken}`,
      },
      body: JSON.stringify(payload),
    }
  );
  const data = await res.json();
  if (data.name) {
    console.log(`✅ Delivered! Message: ${data.name}`);
  } else {
    console.error('❌ Delivery failed:', JSON.stringify(data));
  }
}
```

Add `google-auth-library` dependency:
```json
{ "dependencies": { "google-auth-library": "^9.0.0" } }
```

### 5b. `.github/workflows/ramadan_hourly_push.yml` — swap secrets

Remove:
```yaml
env:
  ONESIGNAL_REST_API_KEY: ${{ secrets.ONESIGNAL_REST_API_KEY }}
  ONESIGNAL_APP_ID: ${{ secrets.ONESIGNAL_APP_ID }}
```

Add:
```yaml
env:
  FIREBASE_PROJECT_ID: ${{ secrets.FIREBASE_PROJECT_ID }}
  FIREBASE_SERVICE_ACCOUNT_JSON: ${{ secrets.FIREBASE_SERVICE_ACCOUNT_JSON }}
```

Add npm install step before the node run step:
```yaml
- name: 📦 Install dependencies
  run: npm install
```

### Done when

- [ ] `sendPush()` uses FCM HTTP v1 API
- [ ] OAuth2 token fetched from service account
- [ ] Sends to topic `daily-reminders`
- [ ] `google-auth-library` added to package.json
- [ ] Workflow updated with Firebase secrets
- [ ] Manual workflow dispatch test passes

---

## Phase 6 — Google Play via TWA

A Trusted Web Activity (TWA) is a Chrome-based Android wrapper that loads `https://noor-nights.github.io` as a full-screen app. It appears as a native app on Google Play but reuses the same service worker, same FCM push, and same codebase.

### 6a. Digital Asset Links (`assetlinks.json`)

This file proves you own the domain and authorises the TWA to run without the browser toolbar.

Create: `.well-known/assetlinks.json` in the repo root (served at `https://noor-nights.github.io/.well-known/assetlinks.json`):

```json
[{
  "relation": ["delegate_permission/common.handle_all_urls"],
  "target": {
    "namespace": "android_app",
    "package_name": "com.noor.nights",
    "sha256_cert_fingerprints": ["<YOUR_KEYSTORE_SHA256_FINGERPRINT>"]
  }
}]
```

The SHA256 fingerprint comes from your signing keystore (generated in 6b).

### 6b. Generate signing keystore (one-time)

```bash
keytool -genkey -v \
  -keystore noor-nights-release.keystore \
  -alias noor-nights \
  -keyalg RSA -keysize 2048 \
  -validity 10000
```

Get the SHA256 fingerprint:
```bash
keytool -list -v \
  -keystore noor-nights-release.keystore \
  -alias noor-nights \
  | grep "SHA256:"
```

> Keep the keystore file safe — if lost, you cannot update the Play Store app.

### 6c. Generate TWA APK via PWA Builder

1. Go to [pwabuilder.com](https://www.pwabuilder.com)
2. Enter `https://noor-nights.github.io` → click **Start**
3. Review the manifest analysis — ensure everything is green
4. Click **Package for stores** → choose **Android**
5. Fill in:
   - Package name: `com.noor.nights`
   - App version: `1`
   - Signing key: upload your `.keystore` file
6. Download the signed APK

### 6d. Google Play Console submission

1. Create account at [play.google.com/console](https://play.google.com/console) ($25 one-time registration fee)
2. Create new app → Internal testing track first
3. Upload the signed APK
4. Fill in store listing (screenshots, description, category: Lifestyle or Education)
5. Set content rating (Everyone)
6. Roll out to internal testing → install on device → verify push notifications work in TWA

### Done when

- [ ] `assetlinks.json` deployed to GitHub Pages
- [ ] Keystore generated and fingerprint captured
- [ ] APK generated via PWA Builder
- [ ] Internal testing track live on Play Console
- [ ] Push notifications verified working inside TWA
- [ ] Production release submitted

---

## Implementation Order & Dependencies

```
Phase 1 — Firebase Setup (manual, you)
    │
    ├── Phase 2 — CF Worker (independent, no code deps on 1 except the key)
    │
    └── Phase 3 + 4 — Web App + SW (parallel, need Firebase config from Phase 1)
            │
            └── Phase 5 — GitHub Actions sender (needs Phase 1 secrets)
                    │
                    └── Phase 6 — Google Play TWA (needs Phase 3–5 verified working)
```

Phases 2, 3, and 4 can be developed and tested independently. Phase 5 can be merged alongside 3+4. Phase 6 starts only after end-to-end push notifications are verified on the live site.

---

## GitHub Issues to Create

| Issue | Title | Depends on |
|-------|-------|------------|
| #13 | chore: Firebase project setup + GitHub secrets | — |
| #14 | feat: Cloudflare Worker for FCM topic subscription | #13 |
| #15 | feat: Replace OneSignal SDK with Firebase Messaging in web app | #13, #14 |
| #16 | feat: Refactor service worker — OneSignal → Firebase Messaging | #13 |
| #17 | feat: Update GitHub Actions cron sender to use FCM HTTP v1 API | #13, #15 |
| #18 | feat: TWA setup + Google Play submission | #15, #16, #17 |

---

## Risk Register

| Risk | Likelihood | Mitigation |
|------|-----------|------------|
| FCM topic subscription fails silently on mobile | Medium | Test with explicit `fetch` error logging; CF Worker returns 502 on failure |
| `google-auth-library` increases GitHub Actions cold start | Low | Library is ~2MB; npm install step adds ~10s to job |
| TWA verification fails (assetlinks.json) | Medium | Use `adb logcat` or [Digital Asset Links tester](https://developers.google.com/digital-asset-links/tools/generator) to debug |
| Keystore lost | High impact | Store keystore + password in a password manager immediately after creation |
| OneSignal subscribers don't migrate | Expected | Existing OneSignal subscribers need to re-subscribe via the new FCM button; consider a one-time email/in-app notice |
| FCM IID API (Phase 2) is a legacy/deprecated Google API | Medium | Google has not announced a shutdown date, but the replacement is the FCM HTTP v1 instance management API (in beta). Monitor announcements; migration path exists if IID is deprecated. See [AgDR-0002](agdrs/AgDR-0002-cloudflare-worker-fcm-proxy.md). |

---

## Cost Summary

| Service | Free tier | Expected usage | Cost |
|---------|-----------|---------------|------|
| Firebase FCM | Unlimited messages | ~21 sends/day to all subscribers | $0 |
| Firebase project | Always free for FCM | No Firestore/Functions needed | $0 |
| Cloudflare Workers | 100k req/day | <100/day | $0 |
| Google Play Console | $25 one-time | One-time | $25 |
| GitHub Actions | 2000 min/month free | ~21 runs/day × ~30s = ~315 min/month | $0 |

**Total ongoing cost: $0/month**
