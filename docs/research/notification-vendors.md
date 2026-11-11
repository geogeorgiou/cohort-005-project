# Hosted notification vendors for Cadence

Research for [#8](https://github.com/geogeorgiou/cohort-005-project/issues/8) (map [#7](https://github.com/geogeorgiou/cohort-005-project/issues/7)). Researched 2026-11-11 against vendor docs and pricing pages. Pricing and docs change often, so check the linked pages again before deciding.

**Cadence constraints:** React Router 8 (framework mode, SSR) + SQLite/Drizzle. Channels: in-app **Inbox** (real-time) and **web push**. Email is optional (Postmark already sits behind a `MailTransport` seam with a no-credentials `.outbox/` fallback). Hosted SaaS with a free tier. No per-user preferences.

This document doesn't pick a vendor. That's the vendor grilling ticket.

## Comparison table

| | Knock | Novu Cloud | Courier | MagicBell | SuprSend |
|---|---|---|---|---|---|
| **Free tier** | 10,000 messages/mo, unlimited channels and seats; no hard cap, sales contacts you when you go over [1] | 10,000 workflow runs/mo, all channels, up to 5 workflows, 3 team members, 24h activity retention, dev + prod envs [2] | "Developer": up to 10k sends/mo [3] | "Builder": 1,000 deliveries/mo (counted per channel delivered), unlimited users, 1 project [4] | 10k notifications/mo, unlimited channels, Inbox carries SuprSend branding; no pay-as-you-go [5] |
| **First paid tier** | Starter $250/mo, 50k messages, then $0.005/message [1] | Pro from $30/mo, 30k+ runs, $1.20 per extra 1k runs [2] | Business: $0.005/send pay-as-you-go [3] | Startup $249/mo, 50k deliveries, $0.0025 per extra [4] | Essentials $100/mo, 50k, $2 per extra 1k; removes branding [5] |
| **In-app Inbox (React)** | `@knocklabs/react`: `NotificationFeedPopover`, `NotificationFeed`, `NotificationIconButton`, plus headless hooks; CSS import [6][7] | `@novu/react` `<Inbox />` (bell, list, preferences modal), React 18+ / 19; hooks or `@novu/js` for custom UI [8] | `@trycourier/courier-react` v9 (React 18+), `CourierInbox`, `useCourier` [9] | `@magicbell/react`: `ContextProvider`, `Inbox`, `FloatingInbox`, separate CSS [10] | `@suprsend/react`: `SuprSendProvider`, drop-in `<Inbox />`; `@suprsend/react-inbox` deprecated [11] |
| **Real-time** | "real-time feed" over socket, with `auto_manage_socket_connection` and optional cross-tab BroadcastChannel [7] | Real-time unread count and sync [8]; socket at `wss://…socket.novu.co` [12] | "Messages arrive in real time" [9] | Realtime client in `magicbell-js` (needs user token) [13] | WebSocket handled by `<Inbox />` [11] |
| **Client identity** | Public key + user `id`; production needs "enhanced security": backend signs an RS256 JWT (`sub` claim, default 1h), with `onUserTokenExpiring` for refresh [14] | `applicationIdentifier` + `subscriberId`; production uses `subscriberHash` = `HMAC-SHA256(secretKey, subscriberId)` [12] | Backend makes a JWT with the API key; client calls `signIn({ userId, jwt })` [9] | User JWT made on your server from the project's API key + secret [10][15] | Backend signs an ES256 JWT (`entity_type: "subscriber"`); provider takes a token-fetch function and refreshes it [11] |
| **Web push** | **No first-party web push.** Goes through FCM (firebase-messaging-sw.js + VAPID key); the tutorial notes iOS needs the PWA installed [16][17] | No native VAPID provider. FCM (web), OneSignal, Pushpad, Pusher Beams, or **Push Webhook** (Novu POSTs to your endpoint and you send it yourself) [18][19] | No native VAPID channel documented. Via FCM, OneSignal, Pusher Beams, MagicBell, etc. [20] | **Native web push.** You supply VAPID keys; subscription endpoint + `p256dh`/`auth` saved as a channel token. `WebPushButton` (service worker must `importScripts` MagicBell's sw.js) has `requestInstallLabel` for the iOS install requirement [10][21] | **Native web push.** VAPID public key from dashboard, `serviceworker.js` at root importing the SDK, `registerPush()` on a user gesture, `notificationPermission()` [22] |
| **Email (optional)** | Yes, many providers incl. Postmark (all vendors are multi-provider) | Yes, all channels on free [2] | Yes | Yes ("all integrations") [4] | Yes |
| **Server trigger from Node** | `@knocklabs/node` / REST workflow trigger | `@novu/api` / REST `POST /v1/events/trigger` | `@trycourier/courier` / REST send | `magicbell-js/project-client` `broadcasts.createBroadcast` [13] | `@suprsend/node-sdk` / REST |
| **Batching / digests** | Batch function (window, sliding window, batch key, up to 90 days). Says recurring scheduled digests need your own system [23] | Digest step: regular or scheduled, extra aggregation key; **window capped at 1 day on Free** [24] | Batch and digest nodes in Journeys; legacy Automations digests [25] | Workflows API (runs, multi-step) but **no documented digest/batch step** [26] | Separate Batch node and Digest node (scheduled, 2–100 items, min trigger count) [27] |
| **Local dev / no credentials** | Separate dev environment with its own keys; no mock mode documented [28] | `npx novu dev` tunnels to cloud "Local" env (needs an account) [29] | None documented | None documented | Sandbox workspace; Enhanced Security is off there, so a bad token only fails in production [11] |

Notes:
- **No vendor offers a credential-free local mode.** Whichever we pick, server-side triggering should sit behind our own seam, like the Postmark `MailTransport`, with a fake that records Notifications (an `.outbox/`-style file or in-memory list) for local dev and tests. The client Inbox component can't run without a vendor connection, so locally the Inbox needs a stub, a real dev environment, or a small fallback list read from our own records.
- **SSR:** No vendor documents React Router / Remix SSR specifically. Every Inbox here is a client component that opens a socket, so expect to render it client-only (lazy-load or render after hydration). The user token can be minted in a loader. (Inferred, not stated by vendor docs.)
- **Preferences:** Every vendor ships a preferences UI (for example Novu's `<Inbox />` includes a Preferences modal [8]). Cadence doesn't need it, so turn it off or hide it.

## Web push on iOS (vendor-independent)

- Web Push works on iOS/iPadOS 16.4+ **only for web apps added to the Home Screen**. The manifest `display` must be `standalone` or `fullscreen`, and permission must be requested "in response to direct user interaction". Delivery goes through APNs (allow `*.push.apple.com`), and no Apple Developer membership is needed. It's the same standards Web Push as Safari 16.1 on macOS, and the Badging API is supported [30].
- `PushManager.subscribe()` needs a secure context. `applicationServerKey` is the VAPID P-256 public key. Chrome and Edge reject subscriptions unless `userVisibleOnly: true`, and `subscribe()` should run from a user gesture (Firefox enforces this since 72) [31]. A service worker must be active, and the subscription endpoint should be kept secret [32].
- So Cadence needs a web app manifest, a service worker at a root scope, and an explicit "Enable notifications" button. On iOS that button should first tell users to install to the Home Screen. Cadence's `public/` has no manifest or service worker today.

## Shortlist (two, no winner)

### A. SuprSend
- **For:** Native VAPID web push (no Firebase project), a drop-in React `<Inbox />` with WebSocket real-time, a modern signed-JWT client auth with built-in refresh, and both Batch and Digest nodes on a workflow builder. The free tier of 10k notifications/mo covers a course platform's volume, and there's a sandbox workspace.
- **Against:** The free Inbox shows SuprSend branding. There's no pay-as-you-go: the next step is $100/mo. Enhanced Security is off in Sandbox, so auth bugs show up late. It's the smallest vendor of the five, and parts of its docs are stale (some pages are years old, and SDK versions differ between pages).

### B. Novu Cloud
- **For:** The cheapest way to grow (Pro from $30/mo, then $1.20 per 1k runs). Polished `@novu/react` `<Inbox />` with React 19 support. Digest step (regular and scheduled). Simple HMAC subscriber hash. All channels on free. Code-first workflows (Novu Framework, `npx novu dev`) fit a git-reviewed codebase.
- **Against:** No native VAPID web push. We'd use FCM (adds a Firebase project and SDK) or the Push Webhook provider and send VAPID push ourselves (for example with `web-push`), so we'd own part of the push stack. The free tier caps digest windows at 1 day, allows 5 workflows, and keeps activity for 24h only. The HMAC is static per subscriber and doesn't expire, unlike a JWT.

### Why the others didn't make the shortlist
- **Knock:** Best-documented SDK and token model, but no first-party web push (FCM only), and the first paid tier jumps to $250/mo.
- **MagicBell:** Native web push with an iOS-install-aware button, but the free tier is only 1,000 deliveries/mo (counted per channel, so Inbox + push uses 2 per Notification), and there's no documented batch/digest step.
- **Courier:** 10k free sends, but no native web push and no clear free-tier push/Inbox detail on the pricing page. Its automations are moving from Automations to Journeys.

## Sources

1. Knock pricing: https://knock.app/pricing
2. Novu pricing: https://novu.co/pricing
3. Courier pricing: https://www.courier.com/pricing
4. MagicBell pricing: https://www.magicbell.com/pricing
5. SuprSend pricing: https://www.suprsend.com/pricing
6. Knock React overview: https://docs.knock.app/in-app-ui/react/overview
7. Knock React feed: https://docs.knock.app/in-app-ui/react/feed
8. Novu Inbox overview: https://docs.novu.co/platform/inbox/overview
9. Courier Inbox overview: https://www.courier.com/docs/platform/inbox/inbox-overview
10. MagicBell React components: https://www.magicbell.com/docs/libraries/magicbell-react
11. SuprSend React Inbox: https://docs.suprsend.com/docs/inbox-react and client auth https://docs.suprsend.com/docs/client-authentication
12. Novu Inbox production (HMAC, EU socket URL): https://docs.novu.co/platform/inbox/prepare-for-production
13. MagicBell JS/Node client: https://www.magicbell.com/docs/libraries/magicbell-js
14. Knock security and authentication: https://docs.knock.app/in-app-ui/security-and-authentication
15. MagicBell API authentication: https://www.magicbell.com/docs/api/authentication
16. Knock push overview: https://docs.knock.app/integrations/push/overview
17. Knock web push via FCM tutorial: https://docs.knock.app/tutorials/sending-web-push-notifications-with-fcm
18. Novu push providers: https://docs.novu.co/platform/integrations/push
19. Novu Push Webhook: https://docs.novu.co/platform/integrations/push/push-webhook
20. Courier push overview: https://www.courier.com/docs/external-integrations/push/intro-to-push
21. MagicBell web push channel: https://www.magicbell.com/docs/channels/web_push
22. SuprSend React web push: https://docs.suprsend.com/docs/react-webpush and https://docs.suprsend.com/docs/js-webpush
23. Knock batch function: https://docs.knock.app/designing-workflows/batch-function
24. Novu digest step: https://docs.novu.co/platform/workflow/add-and-configure-steps/configure-action-steps/digest
25. Courier digests: https://www.courier.com/docs/platform/automations/digest
26. MagicBell API v2 reference (workflows): https://www.magicbell.com/docs/api/reference
27. SuprSend digest and batch: https://docs.suprsend.com/docs/digest, https://docs.suprsend.com/docs/batch
28. Knock environments: https://docs.knock.app/concepts/environments
29. Novu local studio / `novu dev`: https://docs.novu.co/framework/studio
30. WebKit, Web Push for Web Apps on iOS and iPadOS: https://webkit.org/blog/13878/web-push-for-web-apps-on-ios-and-ipados/
31. MDN, PushManager.subscribe(): https://developer.mozilla.org/en-US/docs/Web/API/PushManager/subscribe
32. MDN, Push API: https://developer.mozilla.org/en-US/docs/Web/API/Push_API

Node SDK package names in the "Server trigger" row (other than MagicBell's) come from the vendors' public npm packages, not from pages cited above.
