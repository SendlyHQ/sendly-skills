---
name: sending-whatsapp
description: Sends WhatsApp messages via the Sendly API — connect a number, check senders and 24-hour windows, send free-form or template messages, and manage Meta-reviewed templates. Applies when messaging customers on WhatsApp, building WhatsApp notifications or OTP delivery, or deciding between free-form and template sends.
---

# WhatsApp Messaging with Sendly

WhatsApp is a first-class Sendly channel: connect a number you own, then send through the same Messages endpoint as SMS with `channel: "whatsapp"`.

> **Not yet generally available.** WhatsApp is gated behind the `whatsapp_channel` rollout flag. Until it is enabled for your account, `/api/v1/whatsapp/*` endpoints return `not_found` (404) and WhatsApp sends via `/api/v1/messages` return `403 whatsapp_not_enabled`. Ask Sendly to enable it before relying on these endpoints. WhatsApp always requires a **live** API key — test keys get `403 whatsapp_requires_live_key` (delivery is never sandbox-simulated on this channel).

## The two rules that shape everything

1. **Connecting a number needs a human.** The signup returns a `connectUrl` that a person must open in a browser and log in with **Facebook** to link their WhatsApp Business Account. You (the agent) cannot complete this step programmatically — hand the URL to the account owner, then poll until the status is `active`.
2. **The 24-hour window decides what you can send.** Free-form text and media only deliver while a customer-service window is open (it opens, and resets to 24 hours, each time the recipient messages your number). Outside a window, only a Meta-**approved template** delivers.

## Quick start

```typescript
import Sendly from "@sendly/node";

const sendly = new Sendly(process.env.SENDLY_API_KEY!);

const { senders } = await sendly.whatsapp.senders.list();
const from = senders.find((s) => s.status === "active")?.phoneNumber;

const { open } = await sendly.whatsapp.window({ from, to: "+15551234567" });

const message = open
  ? await sendly.messages.send({
      channel: "whatsapp",
      to: "+15551234567",
      from,
      text: "Your table is ready!",
    })
  : await sendly.messages.send({
      channel: "whatsapp",
      to: "+15551234567",
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

**Required:** `phoneNumber` — an active number in your workspace (purchased or ported into Sendly).

**Response:**
```json
{
  "id": "0b64c1de-…",
  "connectUrl": "https://sendly.live/whatsapp/connect/…",
  "status": "initiated"
}
```

Relay `connectUrl` to the account owner: they open it in a browser and sign in with Facebook to link their WhatsApp Business Account. A one-time **$19 setup fee** is charged (no monthly WhatsApp fee). Calling again for a number with an in-flight signup returns the same signup without charging again.

Poll the signup until it is `active`:

```bash
curl https://sendly.live/api/v1/whatsapp/signup/0b64c1de-… \
  -H "Authorization: Bearer $SENDLY_API_KEY"
```

Statuses: `initiated` (waiting on the human) → `registering` → `active`; `failed` (see `failureReasons`) or `expired` (connect URL lapsed — start again).

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
      "createdAt": "2026-07-30T09:12:00Z"
    }
  ]
}
```

`status` is `pending` (connection in progress), `active` (sendable), or `suspended`. `displayName` is the name recipients see (Meta-reviewed, `null` until set); `qualityRating` is Meta's quality signal (`null` until reported). An **empty list means no number is WhatsApp-connected yet** — run the signup flow above.

### Check a 24-hour window

```bash
curl "https://sendly.live/api/v1/whatsapp/window?from=%2B15125550190&to=%2B15551234567" \
  -H "Authorization: Bearer $SENDLY_API_KEY"
```

**Response:** `{ "open": true, "expiresAt": "2026-07-31T09:12:00Z" }` — or `{ "open": false, "expiresAt": null }`, in which case send a template.

### Send a message

Same endpoint as SMS. Provide **exactly one** of `text`, `mediaUrls`, or `template`:

```bash
curl -X POST https://sendly.live/api/v1/messages \
  -H "Authorization: Bearer $SENDLY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "channel": "whatsapp",
    "to": "+15551234567",
    "from": "+15125550190",
    "template": {
      "name": "order_shipped",
      "language": "en_US",
      "variables": { "1": "Sam", "2": "#4821" }
    }
  }'
```

- **Free-form text** (`text`, max 4096 bytes) — window-bound. 1 credit.
- **Media** (`mediaUrls`, exactly one publicly reachable HTTPS URL) — window-bound; an optional `text` becomes the caption (max 1024 bytes). 1 credit.
- **Template** (`template`) — works regardless of the window; the template must be `APPROVED`. Priced by category and destination country.

`from` is required — there is no default-sender fallback on this channel. `to` must be in E.164.

## Templates

Templates are Meta-reviewed message formats (review typically takes 24–48h) and the only way to start a conversation or reach someone whose window has closed.

Three categories, which drive review rules and per-message pricing:

- `authentication` — OTP/verification codes (no URLs or media allowed). Cheapest.
- `utility` — transactional updates tied to an existing customer interaction.
- `marketing` — promotions. Priced highest, and **Meta currently pauses marketing template delivery to US (+1) recipients** — plan US promotional traffic on SMS instead.

Meta may reclassify a template it deems miscategorized; the category on the record is authoritative.

```bash
curl -X POST https://sendly.live/api/v1/whatsapp/templates \
  -H "Authorization: Bearer $SENDLY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "sender": "+15125550190",
    "name": "order_shipped",
    "language": "en_US",
    "category": "utility",
    "body": "Hi {{1}}, your order {{2}} has shipped!",
    "examples": { "1": "Sam", "2": "#4821" }
  }'
```

Every `{{n}}` placeholder in the body needs an example value. List templates and their review status with `GET /api/v1/whatsapp/templates`; only `APPROVED` templates are sendable.

**Rejection recovery:** template names are locked for ~30 days once submitted (even after deletion). If Meta rejects a template, edit it with `PATCH /api/v1/whatsapp/templates/{id}` and resubmit under the same name — do not delete and re-create it.

## Before-send checklist

1. `GET /whatsapp/senders` — is the `from` number listed with status `active`?
2. `GET /whatsapp/window?from=…&to=…` — window open? Free-form is fine (1 credit).
3. Window closed? `GET /whatsapp/templates` — pick an `APPROVED` template and send with `template`.

## Node.js SDK

```typescript
import Sendly from "@sendly/node";
const sendly = new Sendly(process.env.SENDLY_API_KEY!);

const signup = await sendly.whatsapp.signup.create({ phoneNumber: "+15125550190" });
const status = await sendly.whatsapp.signup.get(signup.id);

const { senders } = await sendly.whatsapp.senders.list();
const { open, expiresAt } = await sendly.whatsapp.window({ from: "+15125550190", to: "+15551234567" });

const { templates } = await sendly.whatsapp.templates.list();
await sendly.whatsapp.templates.create({
  sender: "+15125550190",
  name: "order_shipped",
  language: "en_US",
  category: "UTILITY",
  body: "Hi {{1}}, your order {{2}} has shipped!",
  examples: { "1": "Sam", "2": "#4821" },
});
await sendly.whatsapp.templates.update("wat_xxx", { body: "…" }); // rejected-template fix
```

## Error handling

| Error | Status | Meaning | Action |
|---|---|---|---|
| `not_found` | 404 | `whatsapp_channel` flag off (on `/whatsapp/*` endpoints) | Ask Sendly to enable WhatsApp |
| `whatsapp_not_enabled` | 403 | Flag off (on `/messages` sends) | Ask Sendly to enable WhatsApp |
| `whatsapp_requires_live_key` | 403 | Test key used | Use a live `sk_live_*` key |
| `whatsapp_sender_not_connected` | 403 | `from` has no active WhatsApp connection | Connect it via the signup flow |
| `whatsapp_window_closed` | 422 | Free-form/media outside the 24h window (includes `windowExpiredAt`) | Send an approved template |
| `whatsapp_template_not_found` | 404 | No template with that name + language for the sender | Check name and language exactly |
| `whatsapp_template_not_approved` | 422 | Template exists but isn't sendable (includes `templateStatus`) | Wait for review, or fix a rejected template via update |
| `whatsapp_invalid_content` | 400 | Zero or multiple of text/media/template, >1 media URL, body/caption too long, or variable mismatch | Fix the payload |
| `insufficient_credits` | 402 | Balance too low | Top up credits |

## Full reference

- WhatsApp docs: https://sendly.live/docs/whatsapp
- WhatsApp API reference: https://sendly.live/docs/whatsapp-api
- Templates guide: https://sendly.live/docs/whatsapp/templates
