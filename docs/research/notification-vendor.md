# Which notification service vendor should Cadence use?

Research for [#15](https://github.com/pklingman/cohort-005-project/issues/15), part of the [Notifications spec map (#5)](https://github.com/pklingman/cohort-005-project/issues/5). It builds on [#14](https://github.com/pklingman/cohort-005-project/issues/14) (use a vendor), [#11](https://github.com/pklingman/cohort-005-project/issues/11) (Resend for email) and [#12](https://github.com/pklingman/cohort-005-project/issues/12) (Twilio for SMS). Sources checked 2026-11-18. Prices and plan gates change, so recheck them before signing up.

## Recommendation

**Use Knock.** Keep Resend and Twilio: Knock sits on top of them as an orchestration layer and does not replace them.

- **Preferences fit the spec as written.** Knock has per-category, per-workflow and per-channel-type opt-outs, an environment-wide default, and a per-workflow "Override recipient preferences" switch for mandatory Notifications such as password resets. ([preferences][knock-prefs])
- **The in-app feed is ready to use.** `@knocklabs/react` ships a bell with a badge, a popover feed, seen/read/archived states and socket-based real-time updates. Feed items are kept indefinitely. ([React feed][knock-feed], [data retention][knock-retention])
- **Batching is on every plan**, including free. ([batch function][knock-batch], [pricing][knock-pricing])
- **It plugs into both chosen providers** and reads their delivery webhooks: Resend `email.bounced`/`email.complained` and Twilio status callbacks via a Messaging Service SID. ([Resend][knock-resend], [Twilio][knock-twilio])
- **A Node server without a queue can use it.** The trigger API runs workflows asynchronously, accepts an `Idempotency-Key`, and identifies recipients inline. ([trigger][knock-trigger], [idempotency][knock-idem], [identify][knock-identify])

**Costs to accept:**

- **No EU data residency yet.** Knock says it expects to offer it "by the end of 2026". Until then, data is held outside the EU under GDPR transfer mechanisms. ([plans][knock-plans])
- **A steep step up from free.** The free plan covers 10,000 messages a month. After that, Starter costs $250 a month for 50,000, plus $0.005 per message. Every channel delivery counts separately, so in-app plus email is two messages. ([pricing][knock-pricing])

**Fallback: Novu Cloud.** Choose it if EU data residency becomes a hard requirement before Knock ships it, or if cost dominates. Novu offers US and EU regions on every plan, starts at $30 a month, and bills once per workflow run rather than per channel. It also has an open-source self-hosted edition as a lock-in escape hatch. Its limits:

- The Pro plan caps workflows at 20.
- The Pro plan keeps the activity log for 7 days and caps digest windows at 7 days.
- Outbound webhooks need the $250 Team plan.

([Novu pricing][novu-pricing], [webhooks][novu-webhooks], [digest][novu-digest])

### Do the Resend and Twilio decisions still stand?

**Yes, both stand.** Knock, Novu, Courier and SuprSend all use your own email and SMS provider accounts and integrate with Resend and Twilio. None of them bills for downstream sends. You still pay Resend and Twilio directly. ([Knock pricing][knock-pricing], [SuprSend pricing][suprsend-pricing], [Novu–Resend][novu-resend], [Courier–Resend][courier-resend])

What changes is **where the webhooks go**:

- **Resend delivery webhooks** go to Knock. Knock exposes them as message statuses and as its own signed outbound webhook. ([Knock–Resend][knock-resend], [Knock webhooks][knock-webhooks])
- **Twilio status callbacks** go to Knock. ([Knock–Twilio][knock-twilio])
- **Twilio inbound STOP/START messages** must still reach **Cadence**. Knock's SMS docs say nothing about opt-out keywords or error 21610, so Cadence must turn a STOP into a Knock preference change itself. ([Knock SMS overview][knock-sms])
- **Twilio Verify** for phone-number verification (#12) stays a direct Cadence-to-Twilio integration. No vendor here does number verification.

## Assumed volume

These figures follow the email research: transactional only, 10k–100k emails a month. In-app Notifications are assumed to run at about the same rate as email, and SMS stays small because it is opt-in.

| Scenario | In-app | Email | SMS | Channel messages | Workflow runs / sends (one per recipient per trigger) |
|---|---|---|---|---|---|
| Small | 6k | 4k | 0.2k | ~10k | ~6k |
| Mid | 30k | 20k | 1k | ~51k | ~30k |
| Large | 50k | 100k | 2k | ~152k | ~100k |

## Cost comparison (list prices, USD per month, vendor only; Resend and Twilio are billed on top)

| Vendor | What is billed | Small | Mid | Large | Source |
|---|---|---|---|---|---|
| **Knock** | Per message: "a single message delivered to a single user on a single channel". Triggers, preference-suppressed and failed messages are not billed. | $0 (Free, 10k) | ~$255 (Starter $250 incl. 50k + $0.005 each) | ~$760 | [pricing][knock-pricing] |
| **Novu Cloud** | Per workflow run: "one execution of a workflow for one subscriber", whatever the number of channels | $0 (Free, 10k runs, **max 5 workflows**) | $30 (Pro, 30k runs, max 20 workflows) or $250 (Team) if more than 20 workflows | ~$114 (Pro: $30 + 70k × $1.20/1k) or $250 (Team, 250k incl.) | [pricing][novu-pricing], [trigger docs][novu-trigger] |
| **Courier** | Per send: "one template sent to one recipient". Multi-channel counts as one. | $0 (Developer, 10k) | ~$150 (Business, $0.005 from the first send) | ~$500 | [pricing][courier-pricing] |
| **SuprSend** | Per notification per channel. A digest counts as one. | **$250** (preferences and digests need Business) | $250 (Business, 50k) | ~$760 ($250 + $5/1k) | [pricing][suprsend-pricing] |
| **MagicBell** | Per delivery, summed across channels | $249 (Free covers only 1k) | $249 (Startup, 50k) | ~$504 ($249 + $0.0025 each) | [pricing][magicbell-pricing] |
| **Novu self-hosted** | Infrastructure only | — | — | — | API, worker and WS services, plus MongoDB, Redis and S3. Recommended setup: 3 VMs per service. ([self-hosting][novu-selfhost]) |

Notes:

- **Knock** also offers Enterprise "monthly notified-user pricing", billed per user notified rather than per message. That could help at the Large tier. ([pricing][knock-pricing])
- **Knock's free plan** has no hard cap: Knock says it will contact you to set up billing. It keeps logs for 30 days, has a 99.9% SLA, and includes preferences, batching and conditional sending. ([pricing][knock-pricing])
- **Novu workflow cap:** the 20-workflow cap on Pro matters. The event catalogue (#6, #7, #8) across four personas may well pass 20, which pushes Novu to $250 as well. ([pricing][novu-pricing])
- **Self-hosted Novu is not free for Cadence.** It needs MongoDB and Redis, which Cadence does not run today, and it lacks Email Activity Tracking and Inbox snooze. That conflicts with #14's reason for choosing a vendor. ([activity tracking][novu-activity], [self-hosted vs cloud][novu-diff])

## Feature comparison

### In-app inbox / feed and React components

| | Knock | Novu | Courier | SuprSend |
|---|---|---|---|---|
| React package | `@knocklabs/react`: `KnockProvider`, `KnockFeedProvider`, `NotificationIconButton`, `NotificationFeedPopover`, `NotificationFeed`, `NotificationCell` ([feed][knock-feed]) | `@novu/react`: `<Inbox>`, `Bell`, `Notifications`, `Preferences`, plus headless hooks and `@novu/js` ([inbox][novu-inbox]) | `@trycourier/courier-react` v9, `<CourierInbox />` ([inbox][courier-inbox]) | Embeddable and headless inbox, toasts, on every plan ([pricing][suprsend-pricing]) |
| Unread count | Badge counts **unseen** by default; `badgeCountType="unread"` switches it to unread | Real-time unread count on the bell | "Unread counts and badges" | Not checked |
| States | seen, read, interacted, archived, each with an un- event ([statuses][knock-statuses]) | read, archive, snooze (snooze is cloud only) | read, opened, archived, synced across devices | Not checked |
| Real-time | Sockets, auto-disconnect for inactive tabs, cross-tab sync option | Documented as "real-time"; transport not named | "Messages arrive in real time" | Not checked |
| Auth | Feed is open to anyone with the feed ID **unless enhanced security mode is on**. Then an RS256 JWT is signed server-side with `signUserToken` from `@knocklabs/node`, default expiry 3600s. ([security][knock-security]) | HMAC `subscriberHash` | Backend-signed JWT | Not checked |
| Retention | Feed items "available indefinitely", even after the 30-day message log expires ([retention][knock-retention]) | Activity feed 24h on Free, 7 days on Pro, 90 days on Team (log retention; feed item retention not separately stated) | Not stated | Logs 30 days on Free, 90 days on paid plans |

### Preference model

| | Knock | Novu | Courier | SuprSend |
|---|---|---|---|---|
| Granularity | `categories`, `workflows`, `channel_types` (email, sms, in_app_feed…), `channels` ([prefs][knock-prefs]) | Per workflow × channel, plus a global per channel ([prefs][novu-prefs]) | Subscription topics, with optional per-topic channel choice ([prefs][courier-prefs]) | Categories × channels; **Business plan and up only** |
| Defaults | Environment default set, then tenant default, then recipient. Everything is opted in unless the default sets `false`. | Workflow defaults set in code or the dashboard | Topic default `OPTED_IN`, `OPTED_OUT` or `REQUIRED` | Not checked |
| Mandatory | Per workflow: "Override recipient preferences" | "Mark as critical" hides the workflow from preferences | `REQUIRED` topic | Not checked |
| Evaluation | Any `false` on an applicable category blocks the send. A specific channel overrides its channel type. | The most specific layer wins (global email off + workflow email on = sends) | A user's topic choice overrides the default unless it is `REQUIRED` | — |
| Hosted page / UI | API plus React; preference UI built by Cadence or from examples (not verified) | `<Preferences>` component inside the Inbox | Hosted page or embedded | Hosted preference page |

Knock and Courier both evaluate by category or topic, which matches how the spec groups Notifications. Knock is the one that also combines a category opt-out with per-channel-type opt-outs. Novu's "most specific wins" rule lets a global channel opt-out be silently overridden by a per-workflow setting, which is harder to explain in a settings screen.

### Digests and batching

- **Knock batch step, on every plan:**
  - Fixed, dynamic, relative or sliding windows. A window can be scheduled, e.g. "until Monday 9am" in the recipient's timezone.
  - Optional batch key, e.g. per Cohort.
  - Window up to 90 days. Can close early once it holds 2–1000 activities.
  - Templates get `total_activities` and `total_actors`, but **only 10 activities are rendered**; Enterprise raises that to 100.
  - Knock itself says batching is "not well-suited for powering regular daily or weekly digests". For those it recommends a scheduled workflow that queries your own data. ([batch][knock-batch])
- **Novu digest step:**
  - Regular, "when events repeat", or scheduled (hourly, daily, weekly, monthly in the subscriber's timezone).
  - Max window: 1 day on Free, 7 days on Pro, 30 days on Team. ([digest][novu-digest])
- **Courier:**
  - The batching automation supports every channel.
  - The Digest Send API is "available for business and enterprise customers" and needs Courier Support to turn it on. ([digest send][courier-digest])
- **SuprSend:** batching and digests need the Business plan. ([pricing][suprsend-pricing])

### Email bounces/complaints and SMS STOP through the vendor

- **Knock and Resend:**
  - Turn on "Incoming webhook" on the Resend channel and point a Resend webhook at Knock for `email.delivered`, `email.suppressed`, `email.bounced` and `email.complained`.
  - `email.bounced` maps to Knock status `bounced`.
  - A complaint records `message.complaint` but does not change the delivery status.
  - Knock documents no mapping for `email.suppressed`. ([Knock–Resend][knock-resend])
  - **Knock does not stop sending to a suppressed address.** Resend still drops those sends, so Cadence must mirror suppression onto the user (see the email research).
- **Knock and Twilio:**
  - Set the Messaging Service's status callback to Knock's webhook URL.
  - Twilio `failed` maps to `bounced`, and `undelivered` maps to `undelivered`.
  - STOP, opt-outs and error 21610 are **not covered** in Knock's docs. ([Knock–Twilio][knock-twilio], [SMS overview][knock-sms])
  - So Cadence keeps the Twilio inbound-message webhook, records the opt-out against its own consent record (#12), and sets `channel_types.sms = false` in Knock.
- **Novu:**
  - "Email Activity Tracking" normalises Resend events: bounced, complaint, delivered and others. It is cloud only (or Enterprise self-hosted). ([activity tracking][novu-activity])
  - Forwarding those events to Cadence needs outbound webhooks, which are on Team and Enterprise only. ([webhooks][novu-webhooks])
  - Novu's Twilio page shows only a sender phone number, not a Messaging Service SID, and says nothing about STOP. ([Novu–Twilio][novu-twilio])
- **SuprSend:** Resend's webhook points at `hub.suprsend.com/webhook/resend/`. ([SuprSend–Resend][suprsend-resend])

### Delivery logs and webhooks

- **Knock:**
  - Delivery statuses: `undelivered`, `bounced`, `delivery_attempted`, `queued`, `not_sent`, `sent`, `delivered`. ([statuses][knock-statuses])
  - Outbound webhook events include `message.sent`, `message.delivered`, `message.undelivered`, `message.bounced`, `message.complaint`, `message.read`, `message.seen`, `message.archived`, `message.link_clicked` and `workflow_recipient_run.*`. ([event types][knock-events])
  - Webhooks are HMAC-SHA256 signed (`x-knock-signature: t=…,s=…`) and retried for several hours. ([webhooks][knock-webhooks])
  - No plan restriction is stated.
  - Message logs are kept 30 days (90 on Enterprise). ([retention][knock-retention])
- **Novu:** outbound webhooks are signed with Svix but are Team/Enterprise only. Activity logs are kept 24h, 7 days or 90 days depending on plan. ([webhooks][novu-webhooks], [pricing][novu-pricing])

### Lock-in, data residency and GDPR

| | Knock | Novu | Courier | SuprSend |
|---|---|---|---|---|
| EU residency | **Not yet.** "We expect to support EU data residency by the end of 2026." ([plans][knock-plans]) | US + EU on every plan; more regions on Enterprise ([pricing][novu-pricing]) | "EU Data Residency" listed on Business ([pricing][courier-pricing]) | Custom region on Enterprise only ([pricing][suprsend-pricing]) |
| GDPR | GDPR, CCPA and SOC 2 Type 2 listed on every plan ([pricing][knock-pricing]) | Not checked | DPA review on Enterprise | SOC 2 report from Essentials |
| Self-host / exit | None; proprietary SaaS | Community self-hosted edition | None | Self-hosted / BYOC on Enterprise |

Lock-in is moderate for any of them. Workflows, templates and preference state live with the vendor. Cadence's own data (events, users, SMS consent, suppression) stays in SQLite. To keep switching cheap, Cadence should:

- trigger workflows from one internal `notify(event)` module;
- keep the event catalogue and the SMS consent record in its own database;
- treat the vendor's preference store as the place preferences are enforced, with Cadence able to rebuild it.

### Integration from a Node server without a job queue

- **Sending:** Cadence calls `POST /v1/workflows/{key}/trigger` with the `@knocklabs/node` SDK.
  - "All workflow runs are executed asynchronously", and the call returns a `workflow_run_id`. ([trigger][knock-trigger])
  - Delays, batch windows, scheduling, provider calls, retries and delivery-status tracking all run inside Knock. So **no worker or queue is needed for delivery**.
- **Recipients:** each trigger can identify recipients inline (`id`, `email`, `phone_number` in E.164, `timezone`, `locale`), so there is no separate sync job. ([identify][knock-identify])
- **What it does not remove:** the hand-off itself is still a network call from Cadence's request handler.
  - Make the call after the database write commits.
  - Send an `Idempotency-Key` such as the event id. It deduplicates for 24h, so retrying is safe. ([idempotency][knock-idem])
  - Retry a few times in-process.
  - If losing a Notification during a Knock outage ever becomes unacceptable, add an outbox table in SQLite drained by a small loop. That is much smaller than the pipeline #14 ruled out.
- **Inbound routes:** one React Router resource route for Knock's outbound webhooks (signature checked against the raw body), and one for Twilio inbound STOP.
- **Auth for the feed:** sign the user token in a loader with `signUserToken` and pass it to `KnockProvider`. ([security][knock-security])

## Why not the others

- **Novu**
  - The cheapest per workflow run, and EU-hosted today.
  - Its Pro plan is limited for Cadence: 20 workflows, 7-day logs and digests, and no outbound webhooks. Those limits push it to the same $250 tier as Knock.
  - Its preference rule (most specific layer wins) is weaker for a category-based settings screen.
  - It is the right choice if EU residency is required now.
- **Courier**
  - Per-send pricing that counts multi-channel as one send is attractive in the middle of the range.
  - But digests (Digest Send API) are Business/Enterprise only and must be enabled by support, and its React inbox docs are thinner on states.
  - It is a reasonable second fallback.
- **SuprSend**: preferences and digests, both core to this spec, need the $250 Business plan even at small volume.
- **MagicBell**: its free tier covers 1k deliveries, and its pricing page lists no digest feature or EU region.

## Findings that affect other tickets on the map

1. **In-app inbox prototype (#10):**
   - Prototype against `@knocklabs/react`.
   - The vendor's states are *seen* (displayed), *read*, *interacted* and *archived*. The badge counts *unseen* by default and can count *unread*. The map's open term "unseen" should be settled against these.
   - Enhanced security mode is required in production. Without it, anyone with the feed ID can read a user's feed.
2. **Preferences (#9):**
   - Model each Notification type as a Knock workflow in one or more **categories**. Users toggle categories × channel types (`in_app_feed`, `email`, `sms`).
   - Mandatory Notifications use "Override recipient preferences".
   - Set the environment default to `channel_types.sms = false`, because SMS needs an explicit opt-in (#12). Knock stores the switch, but Cadence must keep the consent record (timestamp, method, disclosure text).
   - `commercial_subscribed` is irrelevant, since there is no marketing.
   - Team Admin settings could later use Knock tenants, which give per-tenant default preferences.
3. **Event catalogue (#6, #7, #8):**
   - Each event becomes one workflow, so keep a stable key per event.
   - Assign each event a category (for #9), and decide which events batch and on what batch key.
   - Pass the triggering event's id as the idempotency key.
   - The number of workflows matters only if Novu is chosen (20 on Pro).
4. **SMS countries (#13):**
   - Unchanged. Twilio still sends, and the per-country rules in #12 still apply.
   - Knock needs `phone_number` in E.164 and can use the recipient's `timezone` for send windows, which helps with quiet hours.
5. **Fog items on #5:**
   - **Digests:**
     - Knock's batch step covers "30 comments become one Notification".
     - Templates show the counts plus the first or last 10 items. Showing more needs Enterprise.
     - True daily or weekly digests are better built as a scheduled workflow that queries Cadence's own data, as Knock itself advises. That needs a scheduled trigger, so it is the one place a cron could reappear.
   - **Phone numbers:** the shape is unchanged. Cadence verifies the number with Twilio Verify, stores it and the consent, and sends `phone_number` to Knock only once SMS is opted in.
   - **In-app freshness and retention:**
     - The feed is real-time over sockets, not refresh-on-load.
     - Knock keeps feed items indefinitely, so Cadence must decide its own retention and deletion policy (e.g. hide items older than N days, delete on account deletion for GDPR).
   - **Failed delivery:**
     - Knock records `bounced` and `undelivered` and sends `message.bounced`, `message.undelivered` and `message.complaint` webhooks.
     - Cadence should consume these to set an "email undeliverable" flag and fall back to in-app.
     - Knock does not stop future sends to a suppressed address on its own. Resend drops them silently.

## Open gaps

- **Knock EU residency:** only the "by end of 2026" roadmap line was found. No launch announcement was found as of 2026-11-18. Where Knock hosts data today (presumably the US) is not stated on the pages checked. Check the DPA and subprocessor list before signing.
- **Knock and SMS opt-out:** how Knock records a Twilio 21610 error (probably `bounced` or `undelivered`) is not documented.
- **Knock and Resend `email.suppressed`:** Knock documents no status mapping for it.
- **Knock outbound webhooks:** no plan restriction is stated, which is not the same as confirmed on Free.
- **Knock preference UI:** whether `@knocklabs/react` ships a ready-made preference-centre component was not verified.
- **Knock workflows as code:** whether workflows and templates can be versioned in the repo (Knock CLI or management API) was not verified, and it bears on lock-in.
- **Novu billing:** the workflow-run definition and idempotency-key billing come from Novu docs search excerpts. The pricing page's FAQ answers did not render.
- **Novu plan table:** the Inbox, preference and snooze gating per plan could not be read reliably from the pricing table.
- **Courier:** inbox message retention and per-plan preference gating are not stated.
- **Resend endpoints:** whether Resend allows several webhook endpoints (one to Knock, one to Cadence) was not confirmed. Use Knock's outbound webhooks instead.

[knock-pricing]: https://knock.app/pricing
[knock-plans]: https://docs.knock.app/manage-your-account/knock-plans
[knock-retention]: https://docs.knock.app/manage-your-account/data-retention
[knock-feed]: https://docs.knock.app/in-app-ui/react/feed
[knock-security]: https://docs.knock.app/in-app-ui/security-and-authentication
[knock-prefs]: https://docs.knock.app/preferences/overview
[knock-batch]: https://docs.knock.app/designing-workflows/batch-function
[knock-resend]: https://docs.knock.app/integrations/email/resend
[knock-twilio]: https://docs.knock.app/integrations/sms/twilio
[knock-sms]: https://docs.knock.app/integrations/sms/overview
[knock-trigger]: https://docs.knock.app/api-reference/workflows/trigger
[knock-idem]: https://docs.knock.app/api-reference/overview/idempotent-requests
[knock-identify]: https://docs.knock.app/managing-recipients/identifying-recipients
[knock-statuses]: https://docs.knock.app/send-notifications/message-statuses
[knock-webhooks]: https://docs.knock.app/developer-tools/outbound-webhooks/overview
[knock-events]: https://docs.knock.app/developer-tools/outbound-webhooks/event-types
[novu-pricing]: https://novu.co/pricing
[novu-trigger]: https://docs.novu.co/platform/concepts/trigger
[novu-inbox]: https://docs.novu.co/platform/inbox/overview
[novu-prefs]: https://docs.novu.co/platform/concepts/preferences
[novu-digest]: https://docs.novu.co/platform/workflow/add-and-configure-steps/configure-action-steps/digest
[novu-resend]: https://docs.novu.co/platform/integrations/email/resend
[novu-twilio]: https://docs.novu.co/platform/integrations/sms/twilio
[novu-activity]: https://docs.novu.co/platform/integrations/email/activity-tracking
[novu-webhooks]: https://docs.novu.co/platform/developer/webhooks
[novu-selfhost]: https://docs.novu.co/community/self-hosting-novu/overview
[novu-diff]: https://docs.novu.co/community/self-hosted-and-novu-cloud
[courier-pricing]: https://www.courier.com/pricing
[courier-inbox]: https://www.courier.com/docs/platform/inbox/inbox-overview
[courier-prefs]: https://www.courier.com/docs/platform/preferences/preferences-overview
[courier-digest]: https://www.courier.com/docs/platform/sending/digest-send
[courier-resend]: https://www.courier.com/docs/external-integrations/email/resend
[suprsend-pricing]: https://www.suprsend.com/pricing
[suprsend-resend]: https://docs.suprsend.com/docs/resend
[magicbell-pricing]: https://www.magicbell.com/pricing
