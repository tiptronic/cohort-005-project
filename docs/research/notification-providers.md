# Hosted notification SaaS for the Cadence Inbox

Researched 2026-11-19 against primary sources (provider docs, pricing pages, npm registry, published SDK tarballs). npm facts come from `npm view` / `npm pack` run on that date.

## Question

Which hosted notification SaaS should back an in-app **Inbox** (notification feed) for Cadence (React Router 8.3 framework mode with SSR, React 19, Node loaders/actions, SQLite/Drizzle, a `users` table with integer/string ids, and a DevUI user switcher in local dev)? Constraints: hosted SaaS only, a free tier usable for local dev, and email only as a tiebreaker.

## Short answer

**Recommendation: Knock.**
- Its free tier is the least constraining for development: 10,000 messages/month, unlimited workflows, unlimited isolated environments, unlimited feed retention, and no credit card ([pricing](https://knock.app/pricing)).
- The `@knocklabs/react` peer range covers React 19 ([npm](https://www.npmjs.com/package/@knocklabs/react)). It ships prebuilt feed components plus headless hooks, including `usePreferences` (checked in the published `@knocklabs/react-core@0.17.1` types).
- Secure mode works with a token signed locally: an RS256 JWT from `signUserToken(userId)` in `@knocklabs/node`, so no network call is needed per token ([docs](https://docs.knock.app/in-app-ui/security-and-authentication)).
- `Idempotency-Key` on trigger is honored for 24 h ([docs](https://docs.knock.app/api-reference/overview/idempotent-requests)).
- Messages can be listed through the API ([docs](https://docs.knock.app/api-reference/messages/list)), and the CLI pulls and pushes workflows as local files ([docs](https://docs.knock.app/cli/overview)).
- Weak spots: there is no prebuilt preference-center component (you build one with the hooks or client, [docs](https://docs.knock.app/in-app-ui/react/preferences)), Knock branding shows on Free, and the first paid tier is $250/month.

**Runner-up: Novu Cloud.**
- It has the most complete drop-in Inbox. Preferences are built in (global plus per-workflow, and critical workflows are hidden, [docs](https://docs.novu.co/platform/inbox/configuration/preferences)), and the hooks expose seen, read, archived and snoozed states ([useNotifications](https://docs.novu.co/platform/sdks/react/hooks/use-notifications)).
- Subscribers are created automatically on first Inbox load ([quickstart](https://docs.novu.co/platform/quickstart/react)), which suits the DevUI user switcher.
- Secure mode is a plain HMAC-SHA256 over the subscriber id ([docs](https://docs.novu.co/platform/inbox/prepare-for-production)).
- The core is MIT open source ([repo](https://github.com/novuhq/novu)), and the first paid tier is cheaper ($30/month Pro, [pricing](https://novu.co/pricing)).
- It loses to Knock on the free tier and on idempotency:
  - The free tier is capped at **5 workflows**, **2 environments** and **24 h activity-feed retention**.
  - Trigger idempotency is weaker: `transactionId` is checked during processing, not at the API boundary, and the `Idempotency-Key` header "isn't enabled for all organizations" ([API ref](https://docs.novu.co/api-reference/events/trigger-event)).

**Close third: Courier.** It has prebuilt Inbox, Toast and Preferences components, and `Idempotency-Key` is retained 25 h by default ([docs](https://www.courier.com/docs/reference/send/message)). Business pricing is pay-as-you-go at $0.005 per send from the first send ([pricing](https://www.courier.com/pricing)). But each user JWT needs a round-trip to `POST /auth/issue-token` ([docs](https://www.courier.com/docs/platform/inbox/authentication)), and the pricing page does not say whether a card is needed for the free tier.

MagicBell (1,000 deliveries/month and 1 project on free) and SuprSend (now on a new React SDK and a new auth scheme) are viable but weaker fits. Engagespot and Customer.io are excluded (see below).

## Comparison

| Criterion | Knock | Novu Cloud | Courier | MagicBell | SuprSend |
|---|---|---|---|---|---|
| Free tier | 10k msgs/mo, unlimited workflows and envs, no card | 10k runs/mo, **5 workflows**, 2 envs, 24 h activity retention, no card | 10k sends/mo; card requirement not stated on pricing page | **1k deliveries/mo, 1 project** | 10k notifs/mo, no card, 30-day full trial then Free, branding on feed |
| First paid tier | Starter $250/mo (50k) | Pro from $30/mo | Business $0.005 per send, pay-as-you-go | Startup $249/mo (50k) | Essentials $100/mo (50k) |
| React package (React 19?) | `@knocklabs/react` 0.15.1, peer `^19` yes | `@novu/react` 3.20.0, peer `^19` yes | `@trycourier/courier-react` 9.3.5, peer `>=18` | `@magicbell/react` 1.14.0 (peers not declared) | `@suprsend/react` 1.6.0, peer `>=16.8` |
| Headless hooks | Yes (`useNotifications`, `useNotificationStore`, `usePreferences`) | Yes (`useNotifications`, `useCounts`, `usePreferences`) | `useCourier()` | None documented on the React page | `@suprsend/react-core` (`useSuprSendClient` and others) |
| Prebuilt preference UI | No (hooks or client) | Yes (`<Preferences/>`, inside Inbox) | Yes (`<CourierPreferences/>`) | Not documented on the React page | Yes (Preferences component per README) |
| Secure-mode identity | RS256 JWT, `sub`=user id, signed locally | HMAC-SHA256(subscriberId, secret key) | JWT minted by Courier API call | HS256 JWT signed with project secret | ES256 JWT (`entity_id`=distinct_id), signed locally |
| Server trigger | `client.workflows.trigger(key,{recipients,data})` | `novu.trigger({workflowId,to,payload})` (`@novu/api`) | `client.send.message({message:{to:{user_id}}})` | `client.broadcasts.createBroadcast({...recipients:[{externalId}]})` | `workflows.trigger(new WorkflowTriggerRequest(body,{idempotency_key}))` |
| Idempotency | `Idempotency-Key`, 24 h, trigger endpoint only | `transactionId` (processing time) and `Idempotency-Key` (needs support to enable) | `Idempotency-Key`, 25 h default, up to 1 yr | Not documented | `idempotency_key` field |
| States | seen, read, interacted, archived | read, seen, archived, snoozed | read, opened, archived | read, archived, deleted (no "seen" documented) | read, archived (others not verified) |
| Real-time | Hosted WebSocket (Phoenix channels) | Hosted WebSocket (socket.io / partysocket deps) | Hosted WebSocket | "Persistent connection" (`reconnecting-websocket` dep) | Hosted socket (`socket.io-client` dep) |
| Lock-in / export | `GET /v1/messages`; CLI pull/push; MIT SDKs | `GET /v1/notifications` (retention by plan); MIT core and code-first `@novu/framework` | `GET /messages`; MIT web SDKs | Not verified | MIT React SDK; export not verified |
| Email later | Yes | Yes | Yes (core strength, routing) | Yes (channels) | Yes |

## Per-provider findings

### Knock

**Free tier**
- 10,000 messages/month, "no credit card required", unlimited isolated environments, unlimited feeds per recipient, unlimited feed retention, 30-day log retention ([pricing](https://knock.app/pricing)).
- Free cannot remove Knock branding ("Remove Knock branding | No | Yes | Yes", [pricing](https://knock.app/pricing)).
- Each environment has its own API keys, and data is not shared between environments ([docs](https://docs.knock.app/concepts/environments)).

**React SDK and SSR**
- Package: `@knocklabs/react` 0.15.1. Peers are `react ^17||^18||^19`, with `next` and `@tanstack/react-router` as optional peers. MIT license ([npm](https://www.npmjs.com/package/@knocklabs/react)).
- The ESM entry `dist/esm/index.mjs` starts with `"use client"` (inspected tarball). That only matters for RSC. In React Router framework mode the components still SSR unless you gate them.
- Components: `KnockProvider`, `KnockFeedProvider`, `NotificationIconButton`, `NotificationFeedPopover`, `NotificationFeed`, `UnseenBadge` ([overview](https://docs.knock.app/in-app-ui/react/overview), tarball exports).
- Hooks: `useAuthenticatedKnockClient`, `useNotifications`, `useNotificationStore`, `usePreferences` ([GitHub](https://github.com/knocklabs/javascript/tree/main/packages/react), tarball types).
- Provider shape is `<KnockProvider apiKey user={{id}}>` with `<KnockFeedProvider feedId>`. Inactive tabs disconnect their socket after 2000 ms when `auto_manage_socket_connection` is on ([feed guide](https://docs.knock.app/in-app-ui/react/feed)).

**Identity and secure mode**
- With enhanced security, Knock rejects public-key requests that lack a signed user token. The token is an RS256 JWT with `sub` = user id and a default expiry of 3600 s, refreshed via `onUserTokenExpiring`. "Production environments should enable enhanced security mode" ([docs](https://docs.knock.app/in-app-ui/security-and-authentication)).
- Map Knock's user id to `String(users.id)`.

**Triggering from Node**
- `@knocklabs/node` 1.41.1 is generated with Stainless (README). The call is `client.workflows.trigger('key', { recipients: ['id'], data })`.
- The trigger endpoint inline-identifies recipients, actors and tenants ([API](https://docs.knock.app/api-reference/workflows/trigger)), so users need no separate sync step.
- `Idempotency-Key` is honored only on `POST /workflows/:key/trigger` and is retained for 24 h ([docs](https://docs.knock.app/api-reference/overview/idempotent-requests)). The SDK exposes it as an `idempotencyKey` request option (`internal/request-options.d.ts`).

**State and preferences**
- Feed states are seen, read, interacted and archived. The feed API returns unread, read and seen counts for badges ([feeds overview](https://docs.knock.app/in-app-ui/feeds/overview)).
- Preferences are included on Free ([pricing](https://knock.app/pricing)). The React guide builds a `PreferenceCenter` yourself with `knockClient.user.getPreferences()`/`setPreferences()` ([docs](https://docs.knock.app/in-app-ui/react/preferences)).

**Real-time:** a hosted service pushes new messages "over a websocket" ([feeds overview](https://docs.knock.app/in-app-ui/feeds/overview)). `@knocklabs/client` depends on `phoenix` (npm).

**Lock-in and export**
- `GET /v1/messages` returns a paginated list filterable by status, channel, workflow and tenant ([API](https://docs.knock.app/api-reference/messages/list)).
- The CLI pulls and pushes workflows, templates, layouts, partials and translations to a local directory ([CLI](https://docs.knock.app/cli/overview)). That gives version control, but the format is Knock-specific and not portable to other providers.
- The client SDKs are MIT. The service is closed.

**Email:** a first-class channel in the same workflows.

### Novu Cloud

**Free tier**
- $0, "No credit card required", 10,000 workflow runs/month, "Up to 5 workflows", 2 environments (Dev + Prod), activity-feed retention 24 h ([pricing](https://novu.co/pricing)).
- The branding-removal toggle is disabled on Free ([prod guide](https://docs.novu.co/platform/inbox/prepare-for-production)).

**React SDK and SSR**
- Package: `@novu/react` 3.20.0. Peers are `react ^18||^19`, ISC license. It wraps `@novu/js`, which renders with `solid-js` and pulls in `socket.io-client` and `partysocket` (npm).
- The Next.js quickstart puts `'use client'` on the Inbox file ([docs](https://docs.novu.co/platform/quickstart/nextjs)). No SSR guidance exists for other frameworks.
- `<Inbox applicationIdentifier subscriber=... subscriberHash routerPush>` creates the subscriber automatically if the id is new ([quickstart](https://docs.novu.co/platform/quickstart/react)).

**Identity and secure mode**
- `subscriberHash = createHmac('sha256', NOVU_SECRET_KEY).update(subscriberId).digest('hex')`. Once HMAC is toggled on, "the Inbox will not load" without the hash ([prod guide](https://docs.novu.co/platform/inbox/prepare-for-production)).

**Triggering from Node**
- `@novu/api`: `novu.trigger({ workflowId, to: { subscriberId }, payload })` ([SDK](https://docs.novu.co/platform/sdks/server/typescript)).
- `@novu/node` is deprecated on npm: "March 20, 2025 marks the end of support … migrate to the new @novu/api" (`npm view @novu/node deprecated`).
- `transactionId` dedupes, with retention "depends on your billing tier". It is checked during processing, so the docs recommend `Idempotency-Key`. That header returns 409 while the first request is in flight but "isn't enabled for all organizations" and requires a support request ([API ref](https://docs.novu.co/api-reference/events/trigger-event)).

**State and preferences**
- `useNotifications` filters by `seen`, `archived` and `snoozed` and has `readAll`/`seenAll`/`archiveAll`. `useCounts` returns unread and unseen counts in real time ([useNotifications](https://docs.novu.co/platform/sdks/react/hooks/use-notifications), [useCounts](https://docs.novu.co/platform/sdks/react/hooks/use-counts)).
- Preferences are global plus per-workflow and per-channel, with a prebuilt `<Preferences/>` component and a `usePreferences` hook. Critical workflows are hidden from users ([docs](https://docs.novu.co/platform/inbox/configuration/preferences)).

**Real-time:** a hosted socket at `socketUrl` (default `https://socket.novu.co`, EU `wss://eu.socket.novu.co`) ([React SDK](https://docs.novu.co/platform/sdks/react), [prod guide](https://docs.novu.co/platform/inbox/prepare-for-production)).

**Lock-in and export**
- `GET /v1/notifications` filters by channel, subscriber and transactionId. The earliest `after` date is "allowed by subscription plan" ([API](https://docs.novu.co/api-reference/notifications/list-all-events)).
- The core is MIT; the `enterprise` folder is under a commercial license ([repo](https://github.com/novuhq/novu)).
- The `@novu/framework` package (2.14.0) provides code-first workflows (npm). That is the most portable workflow definition of the five.

**Email:** supported as a channel. The 5-workflow free cap limits experimentation.

### Courier

**Free tier**
- Developer: up to 10k sends/month. Test and custom environments count the same as production. Custom environments are Enterprise-only.
- Business: $0.005 per send, charged on every send including the first 10k ([pricing](https://www.courier.com/pricing)).
- The pricing page does not list Inbox under Developer and does not say whether signup needs a card. Courier's docs overview says Developer includes Inbox and preferences ([overview](https://www.courier.com/docs/overview), via search snippet, not read in full).

**React SDK and SSR**
- Package: `@trycourier/courier-react` 9.3.5, peer `react >=18`. It is built on web components (`courier-ui-inbox`, `courier-ui-preferences`, and others; MIT, [GitHub](https://github.com/trycourier/courier-web)).
- Components "render client-side only" and need `'use client'` in Next.js.
- It provides `<CourierInbox/>`, `<CourierInboxPopupMenu/>`, `<CourierToast/>`, `<CourierPreferences/>` and `useCourier()`. Real time requires calling `inbox.listenForUpdates()` after `signIn` ([docs](https://www.courier.com/docs/sdk-libraries/courier-react-web)).

**Identity:** the backend calls `POST /auth/issue-token` with scope `user_id:<id> inbox:read:messages inbox:write:events` and `expires_in`. Omitting `expires_in` "mints a token that never expires". The client then calls `signIn({ userId, jwt })` ([docs](https://www.courier.com/docs/platform/inbox/authentication)).

**Triggering from Node:** `@trycourier/courier` 10.0.1, `client.send.message({ message: { to: { user_id }, template, data } })`. `Idempotency-Key` is retained 25 h by default, up to 1 year via `x-idempotency-expiration` ([API](https://www.courier.com/docs/reference/send/message)).

**State:** messages carry read, opened and archived timestamps, and counts come from `totalUnreadCount` ([docs](https://www.courier.com/docs/sdk-libraries/courier-react-web)).

**Lock-in and export:** `GET /messages` with cursor paging and filters for recipient, status, tags and tenant ([API](https://www.courier.com/docs/reference/messages/list)).

**Email:** Courier's core strength (multi-provider routing).

### MagicBell

**Free tier:** Builder costs $0 with 1,000 deliveries/month and 1 project. Startup is $249/month for 50k ([pricing](https://www.magicbell.com/pricing)). One project means dev and prod share a project (verify).

**React SDK**
- The current package is `@magicbell/react` 1.14.0, with subpath imports such as `@magicbell/react/floating-inbox`. `ContextProvider` takes a user JWT as `token` ([docs](https://www.magicbell.com/docs/libraries/magicbell-react)).
- No SSR guidance and no hooks are documented.
- The older `@magicbell/magicbell-react` (11.5.1) and `@magicbell/react-headless` (5.5.1) are still published with peer `react >=18.3.1` (npm), so the naming is easy to confuse.

**Identity:** an HS256 JWT signed with the project secret, carrying the claims `user_external_id`/`user_email` and `api_key` ([docs](https://www.magicbell.com/docs/api/authentication/user)).

**Trigger:** `magicbell-js/project-client`, `client.broadcasts.createBroadcast({ title, content, actionUrl, recipients: [{ externalId }] })`. The docs do not mention idempotency ([docs](https://www.magicbell.com/docs/libraries/magicbell-js), [broadcast](https://www.magicbell.com/docs/primitive/broadcast)).

**State and real-time:** read, archived and deleted, with no "seen" documented. Delivery uses a "persistent connection" ([in-app](https://www.magicbell.com/docs/channels/in_app)); `@magicbell/react` depends on `reconnecting-websocket` (npm).

### SuprSend

**Free tier:** 10k notifications/month and "No Credit Card required", with multiple environments. The in-app feed carries SuprSend branding on Free. It opens with a 30-day full-feature trial before dropping to Free. Essentials costs $100/month ([pricing](https://www.suprsend.com/pricing)).

**React SDK**
- `@suprsend/react-inbox` is deprecated ("Package no longer supported", npm). The current package is `@suprsend/react` 1.6.0 on top of `@suprsend/react-core` ([docs](https://docs.suprsend.com/docs/inbox-react)).
- `SuprSendProvider` takes `publicApiKey`, `distinctId`, `userToken` and `refreshUserToken` ([GitHub](https://github.com/suprsend/suprsend-react-sdk), MIT).

**Identity:** the scheme changed from HMAC `subscriber_id` to a JWT ([docs](https://docs.suprsend.com/docs/inbox-react)). The token is ES256 with claims `entity_type: "subscriber"`, `entity_id`, `scope.tenant_id`, `iat` and `exp`, signed with a PEM key ([docs](https://docs.suprsend.com/docs/client-authentication)).

**Trigger:** `@suprsend/node-sdk` 1.15.1, `supr_client.workflows.trigger(new WorkflowTriggerRequest(body, { idempotency_key }))` (README). The dedup window was not verified.

**Real-time:** a socket connection via `feedClient.initializeSocketConnection()` ([quick start](https://docs.suprsend.com/docs/inbox-quick-start)); `@suprsend/web-sdk` depends on `socket.io-client` (npm).

### Excluded

- **Engagespot.** The official site returned 404 for both `engagespot.co/` and `/pricing` on 2026-11-19. `@engagespot/react-component` was last published 2025-05 and `@engagespot/node` 2024-11 (npm). Pricing could only be found on aggregator sites. Treat it as unverified or stagnant.
- **Customer.io.** It has a notification inbox through `@customerio/cdp-analytics-browser` ([docs](https://docs.customer.io/integrations/data-in/connections/javascript/inbox/)), but there is no free plan, only a trial. Essentials "Starts at $100 per month" ([pricing](https://customer.io/pricing)). It is a marketing-automation suite rather than an inbox product.
- **Stream** was not evaluated. Its activity-feeds product is a general feed API, not a notification-workflow inbox.

## Open risks and hands-on checks

1. **React Router SSR.** Every SDK is client-side (Courier states it explicitly; Knock and Novu use `'use client'` and browser sockets). Render the Inbox behind a client-only boundary, such as a hydration flag or a lazy import inside the root layout. Then check for hydration warnings and for `window` access during SSR in RR 8.3.
2. **DevUI user switcher.**
   - Generate the user token in the root `loader` from the session user (Knock `signUserToken`, Novu HMAC) so it revalidates when the user switches.
   - Re-key the provider (`key={userId}`) so the socket reconnects as the new user.
   - Verify that Knock's provider tears down the old socket cleanly.
3. **Seeded users.** Knock inline-identifies on trigger and Novu auto-creates on Inbox load. Decide whether the seed script should also upsert users or subscribers into the dev environment.
4. **Knock signing import path.** The security doc imports `signUserToken` from `@knocklabs/node/lib/tokenSigner`. The 1.x README and `index.d.ts` export it from the package root. Use the root export.
5. **Novu idempotency.** Confirm whether `Idempotency-Key` is enabled on our org. Otherwise rely on `transactionId`, whose dedup window is plan-dependent and undocumented.
6. **Free-tier limits.** Novu's 5-workflow cap and 24 h activity retention may bite during development. MagicBell's single project mixes dev and prod.
7. **Unverified:**
   - whether Courier's free signup needs a card, and whether Inbox is on Developer (stated only in a search snippet of the docs overview);
   - SuprSend "seen" state and dedup window;
   - MagicBell idempotency and SSR behavior;
   - Knock's exact React 19 test coverage beyond the peer range;
   - export of full in-app feed history in bulk (all providers expose list APIs; none documents a bulk export).

## Sources

- Knock: https://knock.app/pricing · https://docs.knock.app/in-app-ui/security-and-authentication · https://docs.knock.app/in-app-ui/react/overview · https://docs.knock.app/in-app-ui/react/feed · https://docs.knock.app/in-app-ui/react/preferences · https://docs.knock.app/in-app-ui/feeds/overview · https://docs.knock.app/api-reference/workflows/trigger · https://docs.knock.app/api-reference/overview/idempotent-requests · https://docs.knock.app/api-reference/messages/list · https://docs.knock.app/cli/overview · https://docs.knock.app/concepts/environments · https://github.com/knocklabs/javascript/tree/main/packages/react · npm `@knocklabs/react@0.15.1`, `@knocklabs/react-core@0.17.1`, `@knocklabs/node@1.41.1`, `@knocklabs/client@0.23.1`
- Novu: https://novu.co/pricing · https://docs.novu.co/platform/inbox/prepare-for-production · https://docs.novu.co/platform/quickstart/react · https://docs.novu.co/platform/quickstart/nextjs · https://docs.novu.co/platform/sdks/react · https://docs.novu.co/platform/sdks/react/hooks/use-notifications · https://docs.novu.co/platform/sdks/react/hooks/use-counts · https://docs.novu.co/platform/inbox/configuration/preferences · https://docs.novu.co/platform/sdks/server/typescript · https://docs.novu.co/api-reference/events/trigger-event · https://docs.novu.co/api-reference/notifications/list-all-events · https://github.com/novuhq/novu · npm `@novu/react@3.20.0`, `@novu/js@3.20.0`, `@novu/api@3.20.0`, `@novu/node@2.6.6` (deprecated), `@novu/framework@2.14.0`
- Courier: https://www.courier.com/pricing · https://www.courier.com/docs/sdk-libraries/courier-react-web · https://www.courier.com/docs/platform/inbox/authentication · https://www.courier.com/docs/platform/inbox/inbox-overview · https://www.courier.com/docs/reference/send/message · https://www.courier.com/docs/reference/messages/list · https://www.courier.com/docs/overview · https://github.com/trycourier/courier-web · npm `@trycourier/courier-react@9.3.5`, `@trycourier/courier@10.0.1`
- MagicBell: https://www.magicbell.com/pricing · https://www.magicbell.com/docs/libraries/magicbell-react · https://www.magicbell.com/docs/api/authentication/user · https://www.magicbell.com/docs/libraries/magicbell-js · https://www.magicbell.com/docs/primitive/broadcast · https://www.magicbell.com/docs/channels/in_app · npm `@magicbell/react@1.14.0`, `@magicbell/magicbell-react@11.5.1`, `magicbell-js@4.5.1`
- SuprSend: https://www.suprsend.com/pricing · https://docs.suprsend.com/docs/inbox-react · https://docs.suprsend.com/docs/react-sdk · https://docs.suprsend.com/docs/client-authentication · https://docs.suprsend.com/docs/inbox-quick-start · https://github.com/suprsend/suprsend-react-sdk · npm `@suprsend/react@1.6.0`, `@suprsend/react-inbox@3.9.0` (deprecated), `@suprsend/node-sdk@1.15.1`, `@suprsend/web-sdk@5.5.0`
- Excluded: https://customer.io/pricing · https://docs.customer.io/integrations/data-in/connections/javascript/inbox/ · npm `@engagespot/react-component@3.0.1`, `@engagespot/node@1.7.0`
