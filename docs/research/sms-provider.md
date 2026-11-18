# SMS provider and consent rules for Notifications

Research for [#12](https://github.com/pklingman/cohort-005-project/issues/12), part of the map [#5 Notifications spec](https://github.com/pklingman/cohort-005-project/issues/5). Researched 2026-11-18. Prices are list prices on that date. Vendors change them without notice.

## Recommendation

**Use Twilio Programmable Messaging, sent through a Messaging Service, and add Twilio Verify for phone-number verification.**

- It reaches the most countries of the providers checked: it publishes per-country SMS guidelines for 222 countries and territories ([Twilio SMS guidelines index](https://www.twilio.com/en-us/guidelines/sms)).
- It handles STOP/START/HELP for you and blocks sends to opted-out numbers with error 21610 ([Advanced Opt-Out](https://www.twilio.com/docs/messaging/tutorials/advanced-opt-out)). That covers most of the opt-out obligations below without any code.
- Verify includes protection against SMS pumping fraud ("Fraud Guard") in its base fee ([Verify pricing](https://www.twilio.com/en-us/verify/pricing)).
- Node integration is one typed package, `twilio`, supporting Node 20/22/24 ([twilio-node](https://github.com/twilio/twilio-node)).

Twilio costs more per US message than Telnyx or Plivo. At Cadence's likely volume (transactional notices only), that difference is cents per month. It does not outweigh the built-in opt-out handling, the fraud protection and the documentation. If US volume ever reaches millions of messages a month, revisit Telnyx.

## Provider comparison

US list prices per outbound SMS segment, long code:

| Provider | Base | Carrier pass-through | Number/month | Source |
|---|---|---|---|---|
| Twilio | $0.0083 | $0.0035 (AT&T) to $0.005 (Verizon) | $1.15 | [Twilio US pricing](https://www.twilio.com/en-us/sms/pricing/us) |
| Telnyx | $0.004 | $0.0035 to $0.005 | $1.00 | [Telnyx pricing](https://telnyx.com/pricing/messaging) |
| Plivo | $0.0077 | $0.0035 to $0.005 | $0.50 | [Plivo US pricing](https://www.plivo.com/sms/pricing/us/) |
| AWS End User Messaging | rates published as a CSV (not retrieved) | | $1 (10DLC), $2 (toll-free) | [AWS pricing](https://aws.amazon.com/end-user-messaging/pricing/) |
| Vonage | official page returned 403; third-party figures conflict, so they are not cited | | | |

All-in, one Twilio US SMS costs about $0.012–$0.013. Twilio also charges $0.001 for each message that ends in "Failed" status.

International Twilio prices per segment show the real cost driver: US messages are cheap and international ones are not.

| Destination | Twilio price/segment | Source |
|---|---|---|
| UK | $0.056 | [Twilio GB pricing](https://www.twilio.com/en-us/sms/pricing/gb) |
| Germany | $0.112 | [Twilio DE pricing](https://www.twilio.com/en-us/sms/pricing/de) |
| India | $0.0832 | [Twilio IN pricing](https://www.twilio.com/en-us/sms/pricing/in) |

**Number verification (Twilio)**

- **Lookup** ([pricing](https://www.twilio.com/en-us/user-authentication-identity/pricing/lookup)):
  - Formatting and validation: free.
  - Line Type Intelligence: $0.008 per request. Use it to reject landlines. In Germany a send to a landline fails with error 21614 ([DE guidelines](https://www.twilio.com/en-us/guidelines/de/sms)). In the UK Twilio does not check for landlines ([GB guidelines](https://www.twilio.com/en-us/guidelines/gb/sms)).
  - SMS Pumping Risk Score: free in North America, $0.025 elsewhere.
- **Verify** ([pricing](https://www.twilio.com/en-us/verify/pricing)):
  - $0.05 per successful verification, plus the channel cost (SMS: $0.0083 in the US).
  - Fraud Guard is included in that fee.
  - Twilio itself recommends Verify for one-time-passcode traffic instead of sending codes through 10DLC ([A2P 10DLC](https://www.twilio.com/docs/messaging/compliance/a2p-10dlc)).

**Node integration effort**

- **Twilio**: install `twilio`, then send with `client.messages.create({ to, from | messagingServiceSid, body })`. It ships TypeScript types ([twilio-node](https://github.com/twilio/twilio-node)).
  - Inbound STOP events arrive on a webhook with an `OptOutType` field ([Advanced Opt-Out](https://www.twilio.com/docs/messaging/tutorials/advanced-opt-out)). Cadence needs one React Router resource route to receive delivery-status and opt-out webhooks.
  - Estimate: a day or two of work, plus registration lead time (below).
- **Other providers**: Telnyx, Plivo and Vonage have comparable Node SDKs.
- **AWS**: needs IAM setup and origination identities configured per country in the AWS console. That is more operational overhead for a small app.

## Sending SMS to US numbers: carrier registration (applies to every provider)

- **A2P 10DLC** ([Twilio A2P 10DLC](https://www.twilio.com/docs/messaging/compliance/a2p-10dlc)):
  - Any app-to-person SMS to US recipients from a standard 10-digit number needs a registered Brand and Campaign.
  - Standard and Low-Volume Standard brands need a tax ID (US EIN or Canadian Business Number).
  - Sole Proprietor brands do not need a tax ID, but are capped at 1 campaign and about 3,000 messages a day across US carriers.
  - Unregistered traffic pays higher carrier fees and is filtered more.
- **10DLC fees on Twilio**:
  - Twilio's help article returned 403. These figures come from search results that quote it:
    - Brand: $4.50 (sole proprietor or low-volume), or $46 one-time for a standard brand, which includes secondary vetting.
    - Campaign vetting: $15 one-time.
    - Monthly campaign fee: about $1.50–$10, by use case.
  - AWS publishes comparable fees ([AWS pricing](https://aws.amazon.com/end-user-messaging/pricing/)):
    - Registration: $4.50.
    - Vetting: $41.50.
    - Campaign: $10/month, or $2/month for low volume.
- **Toll-free alternative**: a US toll-free number cannot send SMS to the US or Canada until it passes toll-free verification ([Twilio toll-free onboarding](https://www.twilio.com/docs/messaging/compliance/toll-free/console-onboarding)). It avoids 10DLC campaign fees. Either route works for low-volume transactional notices. Choose one before launch, because approval takes time.
- **Registration content**: the campaign registration asks for the use case, sample messages and the opt-in flow. **The spec's consent screen and message templates are inputs to this registration.**

## Consent and opt-out obligations

### US: TCPA and the carrier rules

- **Consent level**:
  - Carriers and CTIA define three message types: conversational, informational and promotional.
  - For informational messages (alerts, reminders, welcome texts), the consumer "needs to agree to receive texts for a specific informational purpose when they give … their mobile number". That is express consent, not the written consent that promotional messages need ([CTIA Messaging Principles, May 2023, §5.1](https://api.ctia.org/wp-content/uploads/2023/05/230523-CTIA-Messaging-Principles-and-Best-Practices-FINAL.pdf)).
  - Cadence's transactional-only Notifications fall in the informational category.
- **Call-to-action** (CTIA §5.1.1). The opt-in screen must state:
  1. what the program is;
  2. the sending number or sender;
  3. who the sender is (Cadence);
  4. that message and data rates may apply;
  5. how to opt out, plus links to help and the privacy policy.

  Opt-in details must not be hidden in the terms and conditions.
- **Proof of consent** (CTIA §5.1.2). Record:
  - when consent was given (timestamp);
  - how it was collected (for example, a web form);
  - what the person saw when they agreed.
- **Opt-out** (CTIA §5.1.3). Honor opt-outs at any time and through several channels. Send one final confirmation, then nothing more. Messages should say how to opt out. Variations in capitalization and punctuation must still count as an opt-out.
- **Keep the records** (CTIA §5.1.5): retain opt-in and opt-out records so that nothing is sent to an opted-out number.
- **TCPA revocation rule**:
  - The FCC's Report and Order [FCC 26-67](https://docs.fcc.gov/public/attachments/FCC-26-67A1.pdf) was adopted 2026-09-30. It takes effect 30 days after it is published in the Federal Register, and the exact date was not confirmed.
  - It lets a sender name an exclusive revocation method, such as reply-text keywords. Each text must disclose that method, for example by including a standardized word such as "STOP". Without an exclusive method, the sender must honor revocation "by any reasonable means".
  - A revocation of informational texts can be limited to that category. A buyer who opts out of one Notification type can still receive the others.
  - Revocations must be honored within **10 business days**. The FCC is asking for comment on shortening that to 7 (FCC 26-67 FNPRM; summary from [DLA Piper](https://www.dlapiper.com/en-am/insights/publications/2026/10/fcc-tcpa-rule) and the [FCC press release](https://www.fcc.gov/document/fcc-votes-give-consumers-more-choice-calls-they-receive-0)).
  - The order replaces the 2024 "revoke all" rule, whose start had been pushed back to 2027-01-31 ([DA 26-12](https://docs.fcc.gov/public/attachments/DA-26-12A1.pdf)).

### UK: PECR and the ICO

- PECR's consent rule for texts applies to **marketing** ([ICO, electronic mail marketing](https://ico.org.uk/for-organisations/direct-marketing-and-privacy-and-electronic-communications/guide-to-pecr/electronic-and-telephone-marketing/electronic-mail-marketing/)).
- The ICO treats non-promotional administrative or customer-service messages as "service messages", which fall outside those rules. If any promotional element is included, the whole message becomes marketing ([ICO, identify direct marketing](https://ico.org.uk/for-organisations/direct-marketing-and-privacy-and-electronic-communications/direct-marketing-guidance/identify-direct-marketing/)).
- The **Data Use and Access Act 2025** amended PECR's marketing definitions. Whether the ICO's service-message guidance changed afterwards was not checked; confirm it before launch.
- Collecting and using the phone number still needs a UK GDPR lawful basis. Opt-in consent is the simplest and matches the US design.
- Sending details ([GB guidelines](https://www.twilio.com/en-us/guidelines/gb/sms)):
  - An alphanumeric sender ID (for example "Cadence") works without registration, unless the name is protected.
  - International long codes are not supported, so UK traffic should use the alphanumeric ID or a UK number.
  - Support STOP/HELP in the local language, and send only during daytime hours unless the message is urgent.

### EU: ePrivacy and GDPR

- ePrivacy Directive Art. 13 requires prior consent only for unsolicited communications "for the purposes of direct marketing" ([Directive 2002/58/EC](https://eur-lex.europa.eu/legal-content/EN/TXT/HTML/?uri=CELEX:32002L0058)). Transactional notices are outside it, though national laws may add rules.
- A GDPR lawful basis is still needed. Use opt-in consent, as in the UK.
- Germany supports alphanumeric sender IDs without pre-registration ([DE guidelines](https://www.twilio.com/en-us/guidelines/de/sms)). Rules vary by country, so check Twilio's guideline page for each country Cadence enables.

### Canada: CASL (for reference)

Messages that solely confirm a transaction or report factual account or subscription information are exempt from CASL's consent requirement ([CASL s.6(6)](https://laws-lois.justice.gc.ca/eng/acts/E-1.6/page-1.html)). Carriers still expect opt-in.

## What the spec must honor

1. **Explicit SMS opt-in, separate from sign-up and from email.** Keep it per user, and record the timestamp, how it was collected and the disclosure text shown. US carriers require this, and it is the simplest compliant design for the UK and EU.
2. **Consent screen wording**: Cadence's name, what kinds of texts are sent, "Msg & data rates may apply", "Reply STOP to opt out, HELP for help", and a link to the privacy policy.
3. **Verify the number before the first Notification** with a one-time passcode (Twilio Verify). This confirms the number belongs to the user and that it is a mobile line.
4. **STOP handling**: Twilio's default handling sends the opt-out confirmation and blocks further sends (error 21610). Cadence must also:
   - receive the opt-out webhook;
   - mark SMS as off in the user's preferences;
   - treat error 21610 as "opted out", not as a failed delivery.

   STOP, a settings toggle and a support request must each turn SMS off within 10 business days. In practice that should be immediate.
5. **Per-category preferences are allowed**: under FCC 26-67, an opt-out of one informational category can stay limited to that category. Note that a reply of STOP to Twilio blocks the whole number for that sender.
6. **No promotional content in any SMS Notification**: one promotional line turns the message into marketing under the UK/EU rules and the CTIA rules.
7. **Respect quiet hours** in the recipient's local time for non-urgent texts (Twilio's UK and DE guidelines).
8. **Before launch**: register a 10DLC brand and campaign, or verify a toll-free number, for US traffic. Build the per-country sender setup from Twilio's guidelines.

## Effects on other decisions

- **Phone numbers do not exist yet.** `users` has no phone column, and `country` is recorded on `purchases`, where it is nullable (`app/db/schema.ts`), not on the user. The spec needs:
  - a verified phone number in E.164 format on the user;
  - consent fields (opted-in timestamp, collection method, disclosure version, opted-out timestamp).

  Infer the dialing country from the number itself, not from purchase country.
- **Which countries are practical**:
  - The US is cheap per message (about $0.013) but has the most registration work.
  - The UK and most of the EU cost $0.05–$0.12 per message.
  - Recommendation: launch SMS for a short allowlist of countries (for example US, CA, UK) and keep email as the fallback elsewhere.
  - Countries with sender pre-registration regimes, such as India's DLT, need extra setup, which was not checked here. Check before enabling them.
- **Reserve SMS for a few high-value events.** At $0.05–$0.11 per international message, SMS fits time-sensitive notices better than routine ones like comments. Digests should not go over SMS.
- **Failed delivery**: the spec's failed-delivery design needs to handle three Twilio outcomes differently:
  - error 21610 (opted out);
  - error 21614 (landline or invalid number);
  - the $0.001 charge on messages that end in "Failed".

## Open gaps

- Twilio's 10DLC fee figures are second-hand, because the help article returned 403.
- The effective date of FCC 26-67 depends on its Federal Register publication, which was not confirmed.
- Official Vonage and AWS per-message rates were not retrieved.
- The effect of the Data Use and Access Act 2025 on the ICO's service-message guidance was not checked.
