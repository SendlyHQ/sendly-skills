---
name: sending-sms
description: Sends SMS messages via the Sendly API with the Node.js SDK or REST API. Handles single messages, batch sends, scheduling, conversations, and sandbox testing. Applies when sending text messages, notifications, alerts, or reminders via SMS.
---

# Sending SMS with Sendly

## Quick start

```typescript
import Sendly from "@sendly/node";

const sendly = new Sendly(process.env.SENDLY_API_KEY!);

const message = await sendly.messages.send({
  to: "+14155550142",
  text: "Your order has shipped!",
  messageType: "transactional",
});
```

## Authentication

All requests require a Bearer token. Store the API key in `SENDLY_API_KEY` env var.

- `sk_test_*` keys → sandbox mode (no real SMS sent, no credits charged)
- `sk_live_*` keys → production (real SMS on verified numbers)

## REST API

**Base URL:** `https://sendly.live/api/v1`

### Send a message

```bash
curl -X POST https://sendly.live/api/v1/messages \
  -H "Authorization: Bearer $SENDLY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"to": "+14155550142", "text": "Hello!", "messageType": "transactional"}'
```

**Required fields:** `to` (E.164 format), `text`

**Optional fields:** `messageType` (`transactional` or `marketing`, defaults to `marketing`), `metadata` (object, max 4KB), `from` (sender ID)

### Response shape

The response is `201 Created`. Message ids are bare UUIDs — they carry no prefix, so do not pattern
match on one.

```json
{
  "id": "0f1c9d2e-6b74-4c1a-9f0d-2b7c5e83a411",
  "to": "+14155550142",
  "from": "SENDLY",
  "text": "Hello!",
  "status": "queued",
  "segments": 1,
  "creditsUsed": 2,
  "senderType": "number_pool",
  "createdAt": "2026-03-31T10:00:00Z"
}
```

`status` is the status at the moment the row was created, so a real send reads `queued` even when
the carrier handoff succeeded — it is not the delivery outcome. Read it back with
`GET /api/v1/messages/{id}`, or subscribe to webhooks.

**A simulated send returns `simulated: true` and a stored status of `delivered`** (`failed` for the sandbox failure numbers below). That happens with
a test key, with a sandbox destination, and when a **live** key belongs to an account not yet
authorised to send to that destination — in which case `simulatedReason` and `actionUrl` are also
present. Check `simulated` before reporting success; nothing reached a handset.

### Schedule a message

```bash
SEND_AT=$(date -u -d '+1 hour' +%Y-%m-%dT%H:%M:%SZ 2>/dev/null || date -u -v+1H +%Y-%m-%dT%H:%M:%SZ)
curl -X POST https://sendly.live/api/v1/messages/schedule \
  -H "Authorization: Bearer $SENDLY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"to": "+14155550142", "text": "Reminder!", "messageType": "transactional", "scheduledAt": "'"$SEND_AT"'"}'
```

Schedule window: 5 minutes to 5 days in the future. Outside that range you get
`invalid_scheduled_time`. Scheduling requires an approved sender even on a test key, otherwise
`403 not_verified`.

### Batch send

```bash
curl -X POST https://sendly.live/api/v1/messages/batch \
  -H "Authorization: Bearer $SENDLY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"messages": [{"to": "+14155550142", "text": "Hello"}, {"to": "+14155550143", "text": "Hi"}], "messageType": "transactional"}'
```

Up to 10,000 recipients per batch. The response is `202` while processing and `201` when complete;
poll `GET /api/v1/messages/batch/{id}`. Opted-out and known-undeliverable recipients are skipped
without an error (the send response counts them in `optedOutSkipped` and `invalidSkipped`), and the batch is refused with `all_opted_out` or `all_recipients_blocked` if that
empties it — reconcile against the batch result rather than assuming every input was sent.

### Group MMS

Send one message to 2–8 US/Canada recipients as a single group thread — everyone sees the other participants and replies fan out to the whole group.

```bash
curl -X POST https://sendly.live/api/v1/messages/group \
  -H "Authorization: Bearer $SENDLY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"to": ["+14155550142", "+14155550143"], "text": "Team sync at noon?", "messageType": "transactional"}'
```

**Required:** `to` (array of 2–8 E.164 US/CA numbers), plus `text` and/or `mediaUrls`

**Optional:** `from` (sender ID), `mediaUrls` (or `media_urls`), `messageType`

**Response — sent (201):** the send payload carries only the message id, status, recipients, and (when the carrier assigns one) a `group_message_id`. Threading is a server-side concept — there is no `groupKey` or `participants` on this response.
```json
{
  "id": "0f1c9d2e-6b74-4c1a-9f0d-2b7c5e83a411",
  "status": "sent",
  "to": ["+14155550142", "+14155550143"],
  "group_message_id": "grp_abc123"
}
```

**Response — simulated (201):** a test key (`sk_test_*`) or a workspace whose sending number is not yet carrier-approved returns a simulated send instead — `status` is `delivered`, no credits are charged, and `simulated` plus a human-readable `message` are present.
```json
{
  "id": "0f1c9d2e-6b74-4c1a-9f0d-2b7c5e83a411",
  "status": "delivered",
  "to": ["+14155550142", "+14155550143"],
  "simulated": true,
  "message": "Group message simulated (test key or verification pending)."
}
```

Group MMS is gated behind the `group_mms` feature flag (and `enable_mms` when sending media) — calls return `feature_disabled` (403) until it is enabled for your account. Requires an MMS-capable, 10DLC-registered sending number. US/Canada destinations only; other destinations return `unsupported_destination` (400).

**Errors:** `invalid_request` (fewer than 2 or more than 8 recipients, or neither `text` nor `mediaUrls`), `invalid_phone`, `unsupported_destination`, `compliance_blocked` (400); `insufficient_credits` (402); `feature_disabled` (403); `undeliverable_number` (422); `send_failed` (422, carrier rejection — includes the case where the sending number's 10DLC brand/campaign is not registered).

### AI message enhancement

Rewrite a draft into a single, polished SMS segment (≤160 chars) and get a short explanation of what changed. Pass `messageType` to steer the rewrite; with no `text` it generates a suitable message for that type. At least one of `text` or `messageType` is required.

```bash
curl -X POST https://sendly.live/api/v1/ai/enhance \
  -H "Authorization: Bearer $SENDLY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"text": "hey come check out our sale this weekend", "messageType": "marketing"}'
```

**Response (200):**
```json
{
  "enhanced": "This weekend only: shop our sale and save. Reply STOP to opt out.",
  "explanation": "Tightened the copy, added a clear CTA and opt-out for marketing compliance."
}
```

Gated behind the `ai_classification` feature flag — returns `not_found` (404) when disabled. Passing neither `text` nor `messageType` returns `invalid_request` (400). If the enhancement model is briefly unavailable, the response falls back to the original `text` with an empty `explanation`.

### List messages

```bash
curl "https://sendly.live/api/v1/messages?limit=50" \
  -H "Authorization: Bearer $SENDLY_API_KEY"
```

Supports `limit` (max 100), `offset`, `status`, and `q` (full-text search). Without `q`, a test key
sees only sandbox messages.

## Node.js SDK

```bash
npm install @sendly/node
```

```typescript
import Sendly from "@sendly/node";

const sendly = new Sendly(process.env.SENDLY_API_KEY!);

const msg = await sendly.messages.send({ to: "+14155550142", text: "Hello!", messageType: "transactional" });
const scheduled = await sendly.messages.schedule({ to: "+14155550142", text: "Later!", messageType: "transactional", scheduledAt: new Date(Date.now() + 60 * 60 * 1000).toISOString() });
const batch = await sendly.messages.sendBatch({ messages: [{to: "+14155550142", text: "Hi"}], messageType: "transactional" });
const group = await sendly.messages.sendGroup({ to: ["+14155550142", "+14155550143"], text: "Team sync at noon?", messageType: "transactional" });
const improved = await sendly.messages.enhance({ text: "hey come check out our sale", messageType: "marketing" });
const list = await sendly.messages.list({ limit: 50 });
const single = await sendly.messages.get("0f1c9d2e-6b74-4c1a-9f0d-2b7c5e83a411");
```

## Message types

- **transactional**: OTP codes, order confirmations, appointment reminders, account alerts. Allowed 24/7.
- **marketing**: Promotions, sales, newsletters. Subject to the recipient country's quiet hours.

**Quiet hours are per country, not global.** Most countries use 21:00–08:00 in the recipient's local
time, but the UK and Australia are 20:00–09:00, France is 22:00–08:00 with no Sunday, and others
differ again. Do not hardcode a window — see
[`reference/compliance.md`](https://github.com/SendlyHQ/ai/blob/main/reference/compliance.md).

Omitting `messageType`, or sending `message_type`, resolves to `marketing` (a group send defaults to
`transactional` instead). Misclassifying marketing
as transactional violates TCPA, and during quiet hours a transactional message that reads as
promotional is refused with `code: "TRANSACTIONAL_MARKETING_MISMATCH"`.

## Sandbox testing

Use `sk_test_*` keys with magic phone numbers. On a single send (`POST /api/v1/messages`) these six
destinations are simulated for **any** key, including a live one. With a live key a batch send does
not simulate them, and a group send refuses them with `400 sandbox_number_in_live_mode`. A scheduled
send (`POST /api/v1/messages/schedule`) never simulates them, whatever the key:

| Number | Behavior |
|---|---|
| +15005550000 | Always succeeds |
| +15005550001 | Invalid number error |
| +15005550002 | Cannot route error |
| +15005550003 | Queue full |
| +15005550004 | Rate limit exceeded |
| +15005550006 | Carrier rejected |

## Credit costs

- US/CA: 2 credits per SMS segment ($0.02)
- International: 8, 12, 16, 24 or 48 credits per segment depending on the destination tier
- 1 credit = $0.01

## Conversations API

Messages are automatically threaded into conversations. Conversation ids are bare UUIDs. Use the
conversations API for two-way messaging:

```typescript
const convos = await sendly.conversations.list({ status: "active", limit: 20 });
const replies = await sendly.conversations.suggestReplies("7b2e4d16-90ac-4f53-8e21-4c6d0b93af75");
```

## Full reference

- API docs: https://sendly.live/docs/sms
- SDK docs: https://sendly.live/docs/sdks
- OpenAPI spec: https://sendly.live/openapi.yaml
- Sandbox docs: https://sendly.live/docs/sandbox
