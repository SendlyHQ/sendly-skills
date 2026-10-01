---
name: sms-best-practices
description: Provides SMS compliance, formatting, and delivery best practices for the Sendly API. Covers TCPA compliance, quiet hours, opt-out handling, E.164 phone formatting, message segmentation, and carrier verification. Applies when building SMS features that must be compliant and deliverable.
---

# SMS Best Practices

## Phone number formatting

Always use E.164 format: `+{country_code}{number}` with no spaces, dashes, or parentheses.

| Input | E.164 | Valid? |
|---|---|---|
| (415) 555-0142 | +14155550142 | After formatting |
| 415-555-0142 | +14155550142 | After formatting |
| +14155550142 | +14155550142 | Yes |
| 4155550142 | +14155550142 | Need country code |

A malformed number is rejected before anything else happens, and the code depends on the endpoint:
`POST /api/v1/messages` returns `400 invalid_request`, `POST /api/v1/verify` returns
`400 invalid_phone_format`. Normalise at the edge of your system, not at the call site.

## Message types and compliance

**transactional**: OTP codes, order confirmations, appointment reminders, account alerts, shipping updates. Allowed 24/7. The recipient has an existing relationship with the sender.

**marketing**: Promotions, sales, newsletters, product announcements. Subject to:
- Quiet hours in the **recipient's own country**, evaluated in their local time
- Prior express written consent required (TCPA)
- Must include opt-out instructions

`messageType` is camelCase and takes exactly `marketing` or `transactional`. **Omitting it, or
sending `message_type`, resolves to `marketing`**, which subjects the message to quiet hours (group
MMS is the exception and defaults to `transactional`). Set it
explicitly on every send.

Misclassifying marketing as transactional violates TCPA. It is also checked: during quiet hours a
transactional message whose body reads as promotional is refused with
`code: "TRANSACTIONAL_MARKETING_MISMATCH"` (HTTP 400), listing the `matchedKeywords`. On a single
send that response carries a `code` but **no** `error` field; batch, scheduled and group sends return
it with `error: "compliance_blocked"`, so match on `code`.

## Quiet hours

**There is no single global quiet-hours window.** Each country has its own, defined by its own
regulator, and Sendly evaluates it against the recipient's local time — for US and Canada numbers
from the area code's timezone, and for 6 US states a window stricter than the federal one.

Most countries use 21:00–08:00 local (the US federal TCPA window), but a number of them do not: the
UK and Australia are 20:00–09:00, France is 22:00–08:00 with no Sunday and no public holidays, and
South Africa is 20:00–08:00 with no Sunday and a 13:00 Saturday cutoff. An unrecognised country
falls back to a deliberately conservative 20:00–09:00 with no Sunday rather than a permissive
default.

Do not hardcode a window. The authoritative per-country table is generated from source in
[`reference/compliance.md`](https://github.com/SendlyHQ/ai/blob/main/reference/compliance.md).

A blocked marketing send returns `400 compliance_blocked` with `code: "QUIET_HOURS_VIOLATION"`, and
the body carries `recipientTimezone`, `recipientLocalTime`, `quietHoursStart`, `quietHoursEnd` and
`nextAllowedTime`. The correct recovery is to schedule for `nextAllowedTime` with
`POST /api/v1/messages/schedule`, not to retry.

Transactional messages skip the quiet-hours check entirely, at any hour, in every country.

## Opt-out handling

Sendly handles opt-outs automatically, and you cannot send through one. When a recipient replies
STOP, STOPALL, UNSUBSCRIBE, CANCEL, END, or QUIT:

- The contact is marked opted out on your account
- Future sends to that number are refused with `400 contact_opted_out`
- A `message.opt_out` webhook event fires

START, YES and UNSTOP opt them back in and fire `message.opt_in`.

Note the exact names: the event is `message.opt_out` (there is no `opt_out.created` event) and the
error is `contact_opted_out` (there is no `opted_out` error). Matching on the wrong name means your
handler silently falls through to a default branch.

Do not retry an opt-out and do not work around it by creating a duplicate contact. To record an
opt-out you collected in your own UI, `POST /api/v1/contacts/opt-out` with `{"phone_number": "..."}`
(needs `contacts:write`).

## Message segmentation

SMS messages are limited to 160 characters (GSM-7 encoding) or 70 characters (UCS-2 for emoji/Unicode). Longer messages are split into segments:

| Encoding | Single segment | Multi-segment per part |
|---|---|---|
| GSM-7 (ASCII) | 160 chars | 153 chars |
| UCS-2 (emoji/Unicode) | 70 chars | 67 chars |

Each segment costs credits separately. Keep messages concise.

## SHAFT content filtering

Sendly screens every message body against six restricted categories, known as SHAFT. The category
names the API returns are exactly `sex`, `hate`, `alcohol`, `firearms`, `tobacco` and `cannabis`:

- **S**ex/adult content
- **H**ate speech
- **A**lcohol
- **F**irearms
- **T**obacco — and cannabis, which the API reports as its own `cannabis` category

Screening runs on the text of **every** send, including test keys and sandbox destinations, so a
blocked word fails identically in a test suite and in production.

A blocked message returns `400 compliance_blocked` with `code: "SHAFT_CONTENT_DETECTED"`, naming the
`category` and the exact `matchedTerms`. Rewrite around them rather than obfuscating the wording:
carriers run their own screening downstream.

## Credit costs

- US/CA: 2 credits ($0.02) per message segment
- International: 8, 12, 16, 24 or 48 credits per segment depending on the destination tier
- 1 credit = $0.01

Check balance: `GET /api/v1/credits`

## Carrier verification

To send SMS in the US/CA, your toll-free number must be verified by carriers. This requires:
- Business name and address
- Website URL (use hosted business pages if you don't have one)
- Use case description
- Sample messages

Verification takes a few business days. You submit it in the dashboard; an enterprise account can
also submit it for a workspace with `POST /api/v1/enterprise/workspaces/{workspaceId}/verification/submit`.
Use sandbox mode (`sk_test_*` keys) while waiting.

The alternative route for US destinations is a local number registered under 10DLC: register a
brand and a campaign, then assign the number to the campaign. Until a number is assigned to an
active campaign it is owned but unregistered, and the send gate keeps it closed.

## Common errors

| Error | HTTP | Cause | Fix |
|---|---|---|---|
| `invalid_request` | 400 | `to` is not E.164 on a send | Format as +{country}{number} |
| `invalid_phone_format` | 400 | `to` is not E.164 on `POST /verify` | Format as +{country}{number} |
| `insufficient_credits` | 402 | Balance too low | Purchase credits at sendly.live |
| `compliance_blocked` + `code: "QUIET_HOURS_VIOLATION"` | 400 | Marketing inside the recipient country's window | Schedule for `nextAllowedTime`, or correct `messageType` |
| `compliance_blocked` + `code: "SHAFT_CONTENT_DETECTED"` | 400 | Restricted content matched | Revise content; read `category` and `matchedTerms` |
| `contact_opted_out` | 400 | Recipient unsubscribed | Do not retry — respect the opt-out |
| `undeliverable_number` | 422 | Number previously hard-bounced (landline, invalid, non-SMS) | Stop sending; `POST /contacts/{id}/mark-valid` if the flag is wrong |
| `not_verified` | 403 | No approved sender on the account | Complete verification at sendly.live |

## Full reference

- Compliance docs: https://sendly.live/docs/concepts/compliance
- Quiet hours: https://sendly.live/docs/quiet-hours
- Country requirements: https://sendly.live/docs/country-requirements
- Opt-out handling: https://sendly.live/docs/how-to/handle-opt-outs
