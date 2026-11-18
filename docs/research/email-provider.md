# Which email provider should Cadence use?

Research for [#11](https://github.com/pklingman/cohort-005-project/issues/11), part of the [Notifications spec map (#5)](https://github.com/pklingman/cohort-005-project/issues/5). Sources checked 2026-11-18. Prices change, so recheck them before signing up.

## Recommendation

**Use Resend.**

- It costs $0 while Cadence is in development (3,000 emails a month) and $20 a month for up to 50,000 emails.
- It signs its webhooks, and it suppresses hard bounces and spam complaints automatically.
- Its first-party TypeScript SDK accepts React Email components and idempotency keys, which suits a React Router + Node app.

**Fallbacks:**

- **Postmark**, if deliverability turns out to matter more than cost. Postmark keeps transactional mail on its own infrastructure, separate from broadcast mail, but costs more at volume and does not sign its webhooks.
- **Amazon SES**, if volume grows into the hundreds of thousands a month. It costs about a tenth as much, but it starts in a sandbox and needs more setup work.

## Assumed volume

Cadence sends transactional Notifications only, no marketing (see #5). A small-to-mid course platform is assumed to send **10k–100k emails a month**. The cost table uses 10k, 50k and 100k.

## Cost comparison (list prices, USD per month)

| Provider | Free / dev tier | ~10k/mo | ~50k/mo | ~100k/mo | Source |
|---|---|---|---|---|---|
| **Resend** | 3,000/mo, 100/day cap, no overage | $20 (Pro, 50k included) | $20 | $35 (Pro 100k) | [resend.com/pricing][resend-pricing] |
| **Postmark** | 100/mo, no expiry | $15 (Basic, 10k) | ~$68.50 (Pro: $16.50 + 40k × $1.30/1k) | ~$133.50 (Pro: $16.50 + 90k × $1.30/1k) | [postmarkapp.com/pricing][pm-pricing] |
| **Amazon SES** | Only the general AWS new-account credits (up to $200, 6-month plan) | ~$1 | ~$5 | ~$10 (à la carte $0.10/1k) + $0.12/GB attachments | [aws.amazon.com/ses/pricing][ses-pricing] |
| **Mailgun** | 100/day, 1-day log retention | $15 (Basic, 10k) | $35 (Foundation, 50k) | $90 (Scale, 100k) | [mailgun.com/pricing][mg-pricing] |
| **SendGrid** | *Unverified:* 60-day trial, about 100/day | *Unverified:* about $19.95 (Essentials) | about $19.95 | *Unverified:* about $89.95 (Pro) | Official page would not load (redirect loop); figures come from third-party write-ups |

Notes:
- The Postmark figures for 50k and 100k use the *starting* 10k bundle plus overage. Postmark sells larger bundles that may cost less, but the page only says "starting at 10,000". ([pricing][pm-pricing])
- Resend keeps logs for 30 days on every self-serve plan. On paid plans you can turn on pay-as-you-go overage at $0.90/1k. ([pricing][resend-pricing])
- On Resend a dedicated IP is a $30/mo add-on, available on Scale only and "intended for >3,000 emails/day". Pro sends from shared IPs. ([pricing][resend-pricing])
- On Mailgun the Basic plan keeps logs for only 1 day. ([pricing][mg-pricing])

## Bounce and complaint handling

| | Resend | Postmark | Amazon SES |
|---|---|---|---|
| Bounce / complaint events | `email.bounced` (permanent), `email.delivery_delayed` (temporary), `email.complained`, `email.failed`, `email.suppressed` ([event types][resend-events]) | Separate Bounce and Spam Complaint webhooks, plus Delivery and Subscription Change ([webhooks][pm-webhooks]) | Event publishing to SNS, EventBridge, CloudWatch or Firehose through configuration sets ([enforcement FAQ][ses-enforcement]) |
| Automatic suppression | Yes. A hard bounce or complaint adds the address to the suppression list, which emits `suppression.added`. Later sends to that address emit `email.suppressed`. ([event types][resend-events]) | Yes. Suppressions are kept per Message Stream, with reasons HardBounce, SpamComplaint and ManualSuppression. **SpamComplaint suppressions cannot be removed.** ([suppressions API][pm-suppressions]) | Yes, by default on accounts created after Nov 2019. **Only hard bounces** are added (not soft bounces), and they also go onto AWS's global list. ([suppression list][ses-suppression]) |
| Webhook authenticity | Signed with Svix headers (`svix-id`, `svix-timestamp`, `svix-signature`). Verify with `resend.webhooks.verify` against the **raw body**. ([verify][resend-verify]) | **Not signed:** "Postmark does not currently support HMAC webhook signature verification". The docs suggest Basic Auth in the URL plus an IP allowlist. ([webhooks][pm-webhooks]) | SNS messages are signed. The endpoint must confirm the SNS subscription. |
| Retries | Svix retries. Deduplicate on `svix-id`. ([event types][resend-events]) | Retries after 1, 5, 10, 10, 10 and 15 minutes on 5xx, 408 and 429. Other 4xx responses are dropped. An event type is disabled if it fails for 48h. ([webhooks][pm-webhooks]) | SNS delivery policy |

**SES enforcement thresholds** (the published figures; the other providers enforce similar policies on shared IPs):
- Bounce rate: the account is placed under review at **≥5%** and may be paused at **≥10%**.
- Complaint rate: under review at **≥0.1%**, may be paused at **≥0.5%**.
- SES computes these rates over a "representative volume", not a fixed window. ([enforcement FAQ][ses-enforcement])

## Deliverability

None of these vendors publishes inbox-placement data that can be compared across providers, so this section relies on structural facts:
- **Postmark** runs transactional and broadcast mail on separate infrastructure and IP pools: they "do not mix in Postmark, including IP ranges". This is its main deliverability argument. ([message streams][pm-streams])
- **Resend** Pro sends from shared IPs. A dedicated IP is offered only at Scale, for senders above 3,000 a day. ([pricing][resend-pricing])
- **SES** reputation depends on your own account metrics (see the thresholds above). Gmail **does not send complaint data to SES**, so complaints from Gmail users never reach the suppression list. ([suppression list][ses-suppression])
- At Cadence's volume, authenticating the sending domain matters more than which provider you pick. Gmail requires SPF/DKIM from every sender, and DMARC plus one-click unsubscribe from bulk senders (>5,000/day). ([Gmail sender guidelines][gmail])

## Integration effort with React Router + Node + SQLite/Drizzle

- **Resend: lowest effort.**
  - Install `npm i resend`. Sending is one call.
  - The `react` prop takes a React Email component, so templates can live in the same React codebase.
  - `idempotencyKey` (24h TTL, e.g. `notification/<id>`) prevents duplicate sends when a job is retried. ([Node guide][resend-node])
  - Add one React Router resource route for webhooks and verify each request with the SDK.
- **Postmark: low effort.**
  - Use the official `postmark` npm client.
  - Each webhook type has its own endpoint. Basic Auth replaces signature checking.
- **SES: highest effort.**
  - Requires an AWS account, IAM, `@aws-sdk/client-sesv2`, a configuration set, an SNS topic and an HTTPS subscription-confirmation handshake.
  - New accounts start in a **sandbox**: only verified recipients, 200 messages a day, 1 message a second. Getting production access requires a review, with an initial response "within 96 hours", and you must confirm that a bounce and complaint process exists. ([production access][ses-sandbox])

Whichever provider Cadence picks, it needs:
- a public webhook route that reads the raw request body
- a table recording each email's provider message id against its Notification (SES warns that some providers redact the recipient address in complaint notices, so you need your own mapping) ([enforcement FAQ][ses-enforcement])
- an "email undeliverable" flag on the user, set when the address is suppressed

## Findings that affect other tickets on the map

1. **Failed delivery** ("Not yet specified" on #5): every provider suppresses an address after a hard bounce or complaint, and later sends are silently dropped (Resend emits `email.suppressed`). The spec should:
   - mirror suppression into Cadence, for example as an `emailUndeliverable` state on the user
   - fall back to in-app delivery
   - prompt the user to fix their address

   On Postmark a spam complaint is **permanent**, so the user would have to change addresses. ([suppressions API][pm-suppressions])
2. **Unsubscribe and preferences:**
   - Gmail's one-click unsubscribe requirement applies to "marketing messages and subscribed messages" from senders of more than 5,000 a day. Transactional mail is not named, so pure transactional notices appear exempt. ([Gmail][gmail])
   - Optional notification types, such as digests and comment notices, are closer to "subscribed" mail. Giving them a `List-Unsubscribe` header that links to Cadence's preference settings is cheap insurance.
   - Mandatory notices, such as password resets and purchase receipts, should not offer an unsubscribe.
3. **Complaints are undercounted.** Gmail sends no complaint feedback loop to SES, and probably sends none to the other providers either. A low complaint rate therefore does not prove recipients are happy with the email, so the preference controls need to be easy to find. ([suppression list][ses-suppression])
4. **Log retention:** Resend keeps logs for 30 days and Mailgun Basic for 1 day. Cadence should keep its own record of email delivery status rather than rely on the provider dashboard. ([Resend pricing][resend-pricing], [Mailgun pricing][mg-pricing])
5. **Sender domain:** whichever provider Cadence uses, it needs a sending domain with SPF, DKIM and DMARC records in DNS. ([Gmail][gmail])

[resend-pricing]: https://resend.com/pricing
[resend-events]: https://resend.com/docs/dashboard/webhooks/event-types
[resend-verify]: https://resend.com/docs/dashboard/webhooks/verify-webhooks-requests
[resend-node]: https://resend.com/docs/send-with-nodejs
[pm-pricing]: https://postmarkapp.com/pricing
[pm-webhooks]: https://postmarkapp.com/developer/webhooks/webhooks-overview
[pm-suppressions]: https://postmarkapp.com/developer/api/suppressions-api
[pm-streams]: https://postmarkapp.com/message-streams
[ses-pricing]: https://aws.amazon.com/ses/pricing/
[ses-suppression]: https://docs.aws.amazon.com/ses/latest/dg/sending-email-suppression-list.html
[ses-enforcement]: https://docs.aws.amazon.com/ses/latest/dg/faqs-enforcement.html
[ses-sandbox]: https://docs.aws.amazon.com/ses/latest/dg/request-production-access.html
[mg-pricing]: https://www.mailgun.com/pricing/
[gmail]: https://support.google.com/a/answer/81126
