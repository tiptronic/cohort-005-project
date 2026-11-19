# Knock delivers Notifications

Cadence needs a hosted service to deliver Notifications to each person's Notification Feed, and its free tier has to be usable in local dev. We picked Knock. Its free tier has unlimited workflows and environments, its React 19 hooks fit the app, it signs user tokens locally so the root loader needs no network call, and it honours an `Idempotency-Key` for 24 hours. Knock user ids are Cadence `users.id` values.

## Considered Options

- **Novu Cloud.** It has the most complete drop-in feed and a built-in preference center, and its core is open source. We rejected it because the free tier allows only 5 workflows, and Cadence already has 8 live events that could trigger Notifications. Its idempotency is also weaker: the `Idempotency-Key` header is not enabled for every organization.
- **Courier, MagicBell, SuprSend.** Weaker fits on the same comparison.

## Consequences

- Workflows are defined in Knock's own format, and no provider offers a bulk export of feed history, so switching later means rebuilding the workflows and losing past Notifications.
- If people get to mute Notification kinds, that screen is built from `usePreferences`, because Knock has no prebuilt preference center.
- The free tier shows Knock branding, and the first paid tier is $250/month (Novu Pro costs $30/month). We accepted this knowingly.
- Two things are still unverified: SSR and hydration under React Router, and re-keying the feed when the DevUI user switches. If hands-on setup or prototyping shows either is unworkable, this decision is reopened.
