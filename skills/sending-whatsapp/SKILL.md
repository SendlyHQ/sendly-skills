---
name: sending-whatsapp
description: Sends WhatsApp messages via the Sendly API — connect a number (or add one to a connected account by code), check senders and 24-hour windows, send free-form or template messages, manage Meta-reviewed templates, and set conversation starters and WhatsApp calling. Applies when messaging customers on WhatsApp, building WhatsApp notifications or OTP delivery, or deciding between free-form and template sends.
---

# WhatsApp Messaging with Sendly

WhatsApp is a first-class Sendly channel: connect a number you own, then send through the same Messages endpoint as SMS with `channel: "whatsapp"`.

> **Not yet generally available.** WhatsApp is gated behind the `whatsapp_channel` rollout flag. The flag is per person: the user who owns the API key, not the workspace. While it is off, `/api/v1/whatsapp/*` endpoints return `not_found` (404) and WhatsApp sends via `/api/v1/messages` return `403 whatsapp_not_enabled`. Ask Sendly to enable it before relying on these endpoints.

## Scopes, keys and roles

Sends go through `POST /v1/messages` with channel `"whatsapp"` and need `sms:send`, not `whatsapp:write`. Reads (`GET signup/{id}`, templates, window, senders, `senders/{phone}/profile`) need `whatsapp:read` and accept test keys. Signup, template create/edit/delete and profile edits need `whatsapp:write` and a live key (otherwise `403 whatsapp_requires_live_key`). Sends need a live key too: delivery is never sandbox-simulated on this channel. In a team workspace, connecting and profile edits need an owner or admin (`settings:write`), and template writes need an owner, admin or member (`templates:write`). A missing role returns `403 insufficient_permissions`.

The sender settings added later follow the same rules as profile edits: verifying or resending a code for a number added by code, removing or uploading a profile photo, changing conversational components and switching WhatsApp calling need `whatsapp:write`, a live key and, in a team workspace, an owner or admin (`settings:write`). Reading conversational components needs `whatsapp:read` and accepts test keys.

## The two rules that shape everything

1. **Connecting the first number needs a human.** The signup returns a `connectUrl` that a person must open in a browser and log in with **Facebook** to link their WhatsApp Business Account. You (the agent) cannot complete this step programmatically — hand the URL to the account owner, then poll until the status is `active`. In a team workspace, starting a signup needs an owner or admin. Once an account is connected, more numbers can be added to it [by code](#add-a-number-to-a-connected-account-by-code), with no Facebook step.
2. **The 24-hour window decides what you can send.** Free-form text and media only deliver while a customer-service window is open (it opens, and resets to 24 hours, each time the recipient messages your number). Outside a window, only a Meta-**approved template** delivers.

## Quick start

```typescript
import Sendly from "@sendly/node";

const sendly = new Sendly(process.env.SENDLY_API_KEY!);

const { senders } = await sendly.whatsapp.senders.list();
const from = senders.find((s) => s.status === "active")?.phoneNumber;
if (!from) throw new Error("No active WhatsApp sender: connect a number first");

const { open } = await sendly.whatsapp.window({ from, to: "+14155550142" });

const message = open
  ? await sendly.messages.send({
      channel: "whatsapp",
      to: "+14155550142",
      from,
      text: "Your table is ready!",
    })
  : await sendly.messages.send({
      channel: "whatsapp",
      to: "+14155550142",
      from,
      template: {
        name: "order_shipped",
        language: "en_US",
        variables: { "1": "Sam", "2": "#4821" },
      },
    });
```

## REST API

**Base URL:** `https://sendly.live/api/v1`

### Connect a number (a human must finish this)

```bash
curl -X POST https://sendly.live/api/v1/whatsapp/signup \
  -H "Authorization: Bearer $SENDLY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"phoneNumber": "+15125550190"}'
```

**Required:** `phoneNumber` — an active number in your workspace (bought, provisioned, or fully ported into Sendly; hosted-SMS numbers are refused).

**Who can call it:** a live key; in a team workspace, an owner or admin (`settings:write`); any other role gets `403 insufficient_permissions`. The API key needs the `whatsapp:write` scope (reads need `whatsapp:read`; sends need `sms:send`). A key created before these scopes existed has neither, so create a new one.

**Response:**
```json
{
  "id": "0b64c1de-…",
  "connectUrl": "https://sendly.live/whatsapp/connect?token=…",
  "status": "initiated"
}
```

Relay `connectUrl` to the account owner: they open it in a browser and sign in with Facebook to link their WhatsApp Business Account. A one-time **$19 setup fee** is charged when the signup starts (no monthly WhatsApp fee). If the connection fails, the $19 fee is refunded automatically. There is no refund once a number has connected: a later disconnect gets nothing back. Calling again for a number with an in-flight signup returns the same signup without charging again.

Two refusals come before any charge or need a wait:

- `503 whatsapp_unavailable` — WhatsApp connections are temporarily unavailable. Nothing was charged. The body carries `retryAfter: 3600` and the response a `Retry-After: 3600` header; try again after it. Only signup returns this code; no send path does.
- `429 whatsapp_signup_limit_reached` — 5 charged attempts failed in the last 24 hours. Don't retry until the next day; find out why the earlier ones failed first.

Poll the signup until it is `active`:

```bash
curl https://sendly.live/api/v1/whatsapp/signup/0b64c1de-… \
  -H "Authorization: Bearer $SENDLY_API_KEY"
```

Statuses: `initiated` (waiting on the human) → `registering` → `active`, or `failed` (a number [added by code](#add-a-number-to-a-connected-account-by-code) is `verifying` instead). After the Facebook step the signup stays `registering` while WhatsApp activates the number. Activation usually takes a few minutes but can take hours. If it hasn't finished about 6 hours after the session began, the session fails with `registration_timeout` and the fee is refunded. Keep polling every minute or so. A `failed` signup has one code in `failureReasons`:

| Reason | Meaning |
|---|---|
| `signup_abandoned` | The signup stalled before WhatsApp registration began, usually because nobody finished the Facebook step within the link's hour. Start again. |
| `phone_number_mismatch` | The number verified with Meta isn't the one the signup started with. Start again and enter the same number. |
| `waba_mismatch` | The WhatsApp Business Account chosen in the Facebook step doesn't hold the verified number. |
| `waba_already_connected` | Another workspace holds that WhatsApp Business Account or is connecting it. |
| `registration_timeout` | Activation hadn't finished about 6 hours after the session began. |
| `meta_exchange_failed`, `registration_failed` | The connection couldn't be completed. Try again. |
| `setup_fee_payment_failed` | The $19 fee couldn't be charged. |
| `verification_start_failed` | Adding a number by code: WhatsApp wouldn't start verifying the number. |
| `verification_failed` | Adding a number by code: 5 wrong codes. |
| `verification_expired` | Adding a number by code: the code wasn't entered in time. |

If the connection fails, the $19 fee is refunded automatically. The API never sends a status of `expired`.

### Add a number to a connected account (by code)

Once a WhatsApp Business Account is connected in the workspace, another number can join it without the Facebook step. Pass the account's `businessAccountId` (from `GET /whatsapp/senders`, on an active sender):

```bash
curl -X POST https://sendly.live/api/v1/whatsapp/signup \
  -H "Authorization: Bearer $SENDLY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"phoneNumber": "+15125550191", "businessAccountId": "104996582519384", "verificationMethod": "sms"}'
```

`verificationMethod` is `sms` (a text message, the default) or `voice` (a call to the number). `displayName` (optional, at most 512 characters) is the name WhatsApp shows; it defaults to the account's existing display name, else its business name (`400 display_name_required` if there is neither). The number checks, the one-time **$19 fee** (refunded automatically if adding the number fails) and the `402`/`429` refusals are the same as for a Facebook connection.

**Response** (`201`; no `connectUrl`):
```json
{
  "id": "6f1d2c7e-3a4b-4c5d-9e8f-0a1b2c3d4e5f",
  "status": "verifying",
  "phoneNumber": "+15125550191",
  "businessAccountId": "104996582519384",
  "failureReasons": null,
  "verificationMethod": "sms",
  "verificationAttemptsRemaining": 5,
  "updatedAt": "2026-10-01T14:02:55.311Z"
}
```

Calling again for a number that is already `verifying` returns that session (`200`) without a second charge or a second code. Poll `GET /whatsapp/signup/{id}`: while `verifying` it adds `verificationCode`, the 6-digit code once WhatsApp's text has arrived on the number in this workspace (else `null`). Once a code has been submitted, it only shows a code that arrived after the last submission or resend, so it never offers a code WhatsApp already turned down. Before the first submission a resend doesn't hide the earlier code: `verificationCode` keeps showing it until the new code arrives. A code sent by voice call never shows there: someone has to answer the call and read it out. Submit the code:

```bash
curl -X POST https://sendly.live/api/v1/whatsapp/signup/6f1d2c7e-3a4b-4c5d-9e8f-0a1b2c3d4e5f/verify \
  -H "Authorization: Bearer $SENDLY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"code": "482913"}'
```

`200` with `status: "active"` once WhatsApp accepts it (the `whatsapp_account.connected` webhook fires). A wrong code is `422 whatsapp_verification_code_invalid` with `attemptsRemaining`; the 5th wrong code fails the session (`409 whatsapp_verification_failed`, fee refunded). `502 whatsapp_verification_unavailable` doesn't count the attempt: submit the same code again shortly. `502 whatsapp_activation_pending` means WhatsApp accepted the code but the number couldn't be connected yet: don't submit again, check the signup later. Never retry a code submission or an add-by-code start automatically: each submission uses one of the 5 attempts, and a start that failed with `502 whatsapp_verification_start_failed` has already been refunded, so starting again is a new $19 session.

Ask for a new code with `POST /whatsapp/signup/{id}/resend` and an optional `{"verificationMethod": "voice"}` (leaving it out means `sms`). Resends are 30 seconds apart, counted from the last code request, resend or submission: sooner is `429 whatsapp_verification_resend_too_soon` with `retryAfter` (seconds) and a `Retry-After` header. After a resend, wait until `verificationCode` changes before submitting it. A session left untouched for about an hour fails with `verification_expired`, and a code is only accepted within 3 hours of the start (`409 signup_not_active` after that).

### List connected senders

Check this before sending — it answers "which numbers can I send WhatsApp from?"

```bash
curl https://sendly.live/api/v1/whatsapp/senders \
  -H "Authorization: Bearer $SENDLY_API_KEY"
```

**Response:**
```json
{
  "senders": [
    {
      "phoneNumber": "+15125550190",
      "displayName": "Acme Coffee",
      "status": "active",
      "qualityRating": null,
      "businessAccountId": "104996582519384",
      "businessName": "Acme Coffee LLC",
      "callingEnabled": false,
      "outboundCallingAllowed": false,
      "createdAt": "2026-07-30T09:12:00Z"
    }
  ]
}
```

`status` is `pending` (connection in progress), `active` (sendable), or `suspended`. `displayName` is the name recipients see (Meta-reviewed, `null` until set); `qualityRating` is Meta's quality signal (`null` until reported). `businessAccountId` is the WhatsApp Business Account's id (`null` while `pending`; use it to add another number by code) and `businessName` its business name (`null` while pending or when none is stored). `callingEnabled` says whether WhatsApp calling is on; `outboundCallingAllowed` is `false` for +1, +20, +84 and +234 numbers, where WhatsApp doesn't allow business-initiated calls. An **empty list means no number is WhatsApp-connected yet** — run the signup flow above.

### Conversation starters (ice breakers and commands)

`GET /whatsapp/senders/{phoneNumber}/conversational_components` returns `{ phoneNumber, iceBreakers, commands }`. Ice breakers (up to 4, each 1 to 80 characters) are tappable suggestions shown when someone opens a chat with the business for the first time; commands (up to 30, `command` letters, digits or underscores up to 32 characters, `description` up to 256) show when the customer types `/`.

```bash
curl -X PATCH https://sendly.live/api/v1/whatsapp/senders/%2B15125550190/conversational_components \
  -H "Authorization: Bearer $SENDLY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"iceBreakers": ["Book a table", "Opening hours"], "commands": [{"command": "menu", "description": "See the menu"}]}'
```

Each list you send replaces the stored list, `[]` clears it, and a list you leave out stays as it is. A rule broken is `400 invalid_request` with a message naming it; `502 whatsapp_conversational_components_update_failed` (or `_fetch_failed` on the GET) is transient.

### WhatsApp calling

`PATCH /whatsapp/senders/{phoneNumber}/calling` with `{"enabled": true}` turns it on; the response is `{ phoneNumber, callingEnabled, outboundCallingAllowed }`. Once on, a WhatsApp user calling the number rings exactly like a phone call (the dashboard or the number's AI agent answers, per its voice mode), at the normal inbound rate. Turning it on needs voice switched on for the number (`409 voice_not_enabled`). `422 whatsapp_calling_unavailable` means Meta didn't allow it: it needs the account at the 2,000-recipients-a-day messaging limit and an approved display name. `502 whatsapp_calling_update_failed` is transient. There is no API for placing WhatsApp calls. Calls carry `channel` (`phone`, `whatsapp` or `browser`).

### Profile photo

`POST /whatsapp/senders/{phoneNumber}/profile/photo` uploads one as `multipart/form-data` in the field `file`: JPEG or PNG by its bytes (`400 whatsapp_profile_photo_invalid` otherwise, `400 file_required` without one), at most 5 MB (`413 whatsapp_profile_photo_too_large`), ideally square and at least 192 pixels wide (640 recommended). `DELETE` on the same path removes it. Both return the profile; `502 whatsapp_profile_update_failed` means WhatsApp refused or couldn't be reached, so fix the image or retry.

### Check a 24-hour window

```bash
curl "https://sendly.live/api/v1/whatsapp/window?from=%2B15125550190&to=%2B15551234567" \
  -H "Authorization: Bearer $SENDLY_API_KEY"
```

The response is exactly `{ open, expiresAt }`: `{ "open": true, "expiresAt": "2026-07-31T09:12:00Z" }` while the window is open. After it closes the response is `{ "open": false, "expiresAt": "<the past expiry>" }`: send a template. `{ "open": false, "expiresAt": null }` means Sendly has no window on record; a free-form send may still go through if WhatsApp reports an open window (for example right after the number connects), and otherwise fails with `whatsapp_window_closed`.

### Send a message

Same endpoint as SMS. Provide **exactly one** of `text`, `mediaUrls`, or `template`:

```bash
curl -X POST https://sendly.live/api/v1/messages \
  -H "Authorization: Bearer $SENDLY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "channel": "whatsapp",
    "to": "+14155550142",
    "from": "+15125550190",
    "template": {
      "name": "order_shipped",
      "language": "en_US",
      "variables": { "1": "Sam", "2": "#4821" }
    }
  }'
```

- **Free-form text** (`text`, max 4096 bytes) — window-bound.
- **Media** (`mediaUrls`, exactly one publicly reachable HTTPS URL) — window-bound; an optional `text` becomes the caption (max 1024 bytes, not allowed on audio), and the response returns it as `text`.
- **Template** (`template`) — works regardless of the window; the template must be `APPROVED`. Priced by category and destination country.

**Pricing:** free-form text or media inside the 24-hour window: 1 credit each for the first 1,000 per sending number per calendar month (UTC), then the destination's utility template price; countries without a listed price use the default utility price of 12 credits. Templates are priced by category and destination country; countries without a listed price use 33 (marketing), 12 (utility) and 12 (authentication) credits. A failed send gives its slot back.

`from` is required — there is no default-sender fallback on this channel. `to` must be in E.164.

## Templates

Templates are Meta-reviewed message formats (review typically takes 24–48h) and the only way to start a conversation or reach someone whose window has closed.

Three categories, which drive review rules and per-message pricing. The API accepts any case and
returns them in upper case (`AUTHENTICATION`, `UTILITY`, `MARKETING`, the form the Node SDK's types use):

- `AUTHENTICATION` — OTP/verification codes (no URLs or media allowed; needs an `otp` copy-code button).
- `UTILITY` — transactional updates tied to an existing customer interaction. Costs the same as or less than authentication.
- `MARKETING` — promotions. Priced highest, and **Meta currently pauses marketing template delivery to US (+1) recipients** — plan US promotional traffic on SMS instead.

`category` is required. There is no default: leaving it out returns `400 template_category_invalid`. An edit (`PATCH`) can't change the category. Meta may reclassify a template it deems miscategorized; the category on the record is authoritative.

```bash
curl -X POST https://sendly.live/api/v1/whatsapp/templates \
  -H "Authorization: Bearer $SENDLY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "sender": "+15125550190",
    "name": "order_shipped",
    "language": "en_US",
    "category": "UTILITY",
    "body": "Hi {{1}}, your order {{2}} has shipped!",
    "examples": { "1": "Sam", "2": "#4821" }
  }'
```

Every `{{n}}` placeholder in the body needs an example value (`400 template_example_required`). A `header` is fixed text: a header containing `{{n}}` is refused with `400 template_header_variable_unsupported`, because sends fill only body and button variables, so put the variable in the body. Other pre-flight 400s: `template_authentication_otp_button_required` and `template_authentication_no_links` (a link in the body or a URL button on an authentication template). On create, `404 whatsapp_sender_not_connected` is checked first. A marketing template without an opt-out button is accepted with a warning only. Template 400s carry a readable `message`. Creating, editing or deleting a template needs a live key with `whatsapp:write` and, in a team workspace, an owner, admin or member (`templates:write`).

List templates with `GET /api/v1/whatsapp/templates`. Each one carries `body`, `header`, `footer` and `examples` alongside its `status`, which is Meta's status in uppercase, for example `PENDING`, `APPROVED`, `REJECTED`, `PAUSED` or `DISABLED` (Meta may report others). Only `APPROVED` templates are sendable.

**Rejection recovery:** a deleted template's name and language are locked for 30 days (`409 template_name_locked`). If Meta rejects a template, edit it with `PATCH /api/v1/whatsapp/templates/{id}` and resubmit under the same name — do not delete and re-create it.

## Before-send checklist

1. `GET /whatsapp/senders` — is the `from` number listed with status `active`?
2. `GET /whatsapp/window?from=…&to=…` — window open? Free-form is fine (1 credit within the monthly 1,000, then the utility price).
3. Window closed? `GET /whatsapp/templates` — pick an `APPROVED` template and send with `template`.

## Node.js SDK

```typescript
import Sendly from "@sendly/node";
const sendly = new Sendly(process.env.SENDLY_API_KEY!);

const signup = await sendly.whatsapp.signup.create({ phoneNumber: "+15125550190" });
const status = await sendly.whatsapp.signup.get(signup.id);

const { senders } = await sendly.whatsapp.senders.list();
const { open, expiresAt } = await sendly.whatsapp.window({ from: "+15125550190", to: "+14155550142" });

const { templates } = await sendly.whatsapp.templates.list();
await sendly.whatsapp.templates.create({
  sender: "+15125550190",
  name: "order_shipped",
  language: "en_US",
  category: "UTILITY",
  body: "Hi {{1}}, your order {{2}} has shipped!",
  examples: { "1": "Sam", "2": "#4821" },
});
await sendly.whatsapp.templates.update("c4e1a730-58bd-4f92-9a61-7d0e2b845c19", { body: "…" }); // rejected-template fix
```

## Error handling

| Error | Status | Meaning | Action |
|---|---|---|---|
| `not_found` | 404 | `whatsapp_channel` flag off (on `/whatsapp/*` endpoints) | Ask Sendly to enable WhatsApp |
| `whatsapp_not_enabled` | 403 | Flag off (on `/messages` sends) | Ask Sendly to enable WhatsApp |
| `whatsapp_requires_live_key` | 403 | Test key used | Use a live `sk_live_*` key |
| `insufficient_permissions` | 403 | The key lacks the `whatsapp:read`/`whatsapp:write` scope, or the role can't make the change (connecting and profile edits need owner or admin, template writes owner, admin or member) | Use a key with the scope, or ask someone with the role |
| `whatsapp_unavailable` | 503 | Signup only (never a send): refused before any charge while WhatsApp connections are unavailable (`retryAfter: 3600` in the body, `Retry-After: 3600` header) | Try again after `retryAfter` seconds |
| `whatsapp_signup_limit_reached` | 429 | 5 charged signups failed in 24 hours | Wait until the next day |
| `recipient_opted_out` | 403 | The recipient is an opted-out contact in your workspace | Don't send |
| `whatsapp_sender_not_connected` | 403 | `from` has no active WhatsApp connection (on a send) | Connect it via the signup flow |
| `whatsapp_sender_not_connected` | 404 | Template create for a sender that isn't connected (checked first) | Connect it via the signup flow |
| `whatsapp_window_closed` | 422 | Free-form/media outside the 24h window (`windowExpiredAt` only when an earlier window existed) | Send an approved template |
| `whatsapp_template_not_found` | 404 | No template with that name + language for the sender | Check name and language exactly |
| `whatsapp_template_not_approved` | 422 | Template exists but isn't sendable (includes `templateStatus`) | Wait for review, or fix a rejected template via update |
| `whatsapp_invalid_content` | 400 | Zero or multiple of text/media/template, >1 media URL, body/caption too long, or variable mismatch | Fix the payload |
| `whatsapp_send_failed` | 422 | WhatsApp refused the message; final, and you weren't charged. Cached under the idempotency key and replayed for 24 hours | Don't retry; check `errorCode` and `message` |
| `whatsapp_send_failed` | 502 | WhatsApp couldn't be reached: not sent, safe to send again. You weren't charged. Never cached | Retry with the same `Idempotency-Key` |
| `whatsapp_send_unconfirmed` | 409 | Outcome unknown: the message was marked failed and refunded but may still be delivered. Cached under the idempotency key | Check before sending it again, or it could arrive twice |
| `whatsapp_business_account_not_found` | 404 | Adding by code: no connected account with that `businessAccountId` and an active number in this workspace | Connect one with the Facebook flow first |
| `whatsapp_verification_start_failed` | 422 / 502 | WhatsApp refused to verify the number (422, final) or couldn't be reached (502). Session failed, fee refunded | 422: check the number and display name; 502: start again later, once |
| `whatsapp_verification_code_invalid` | 422 | Wrong code (`attemptsRemaining` in the body) | Check the code or request a new one |
| `whatsapp_verification_failed` | 409 | 5 wrong codes; session failed, fee refunded | Start again |
| `whatsapp_verification_busy` | 409 | Another code for the number is being checked | Try again in a moment |
| `whatsapp_verification_unavailable` | 502 | WhatsApp couldn't check the code; the attempt isn't counted | Submit the same code again shortly |
| `whatsapp_activation_pending` | 502 | Code accepted, but the number couldn't be connected yet | Don't resubmit; check the signup later |
| `whatsapp_verification_resend_too_soon` | 429 | Less than 30 seconds since the last code request, resend or submission (`retryAfter`) | Wait `retryAfter` seconds |
| `signup_not_active` | 409 | The signup isn't waiting for a code (failed, or more than 3 hours old) | Start again |
| `voice_not_enabled` | 409 | Turning WhatsApp calling on for a number whose voice is off | Switch voice on for the number first |
| `whatsapp_calling_unavailable` | 422 | Meta didn't allow calling on the number | Wait until the account qualifies |
| `whatsapp_sender_not_connected` | 404 | Sender settings (profile, photo, components, calling) for a number that isn't an active sender | Connect it first |
| `template_category_invalid` | 400 | Template category missing or not `UTILITY`, `AUTHENTICATION` or `MARKETING` (there is no default) | Pass a category |
| `template_authentication_otp_button_required` | 400 | Authentication template without an `otp` copy-code button | Add an `otp` button |
| `template_authentication_no_links` | 400 | A link in the body or a URL button on an authentication template | Remove the link |
| `template_header_variable_unsupported` | 400 | Template header contains `{{n}}` | Keep the header fixed; move the variable to the body |
| `insufficient_credits` | 402 | Balance too low | Top up credits |

## MCP tools

The Sendly MCP server covers the same flow: `start_whatsapp_signup` (pass `businessAccountId` to add a number by code), `get_whatsapp_signup_status`, `verify_whatsapp_signup_code`, `resend_whatsapp_signup_code`, `list_whatsapp_senders`, `get_whatsapp_sender_profile`, `update_whatsapp_sender_profile`, `delete_whatsapp_sender_profile_photo`, `get_whatsapp_conversational_components`, `update_whatsapp_conversational_components`, `set_whatsapp_calling`, the template tools, `get_whatsapp_window`, `send_whatsapp` and `send_whatsapp_template`. Uploading a profile photo is not an MCP tool.

## Full reference

- WhatsApp docs: https://sendly.live/docs/whatsapp
- WhatsApp API reference: https://sendly.live/docs/whatsapp-api
- Templates guide: https://sendly.live/docs/whatsapp/templates
