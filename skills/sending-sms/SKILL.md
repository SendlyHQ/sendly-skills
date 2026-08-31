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
  to: "+15551234567",
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
  -d '{"to": "+15551234567", "text": "Hello!", "messageType": "transactional"}'
```

**Required fields:** `to` (E.164 format), `text`

**Optional fields:** `messageType` (`transactional` or `marketing`, defaults to `marketing`), `metadata` (object, max 4KB), `from` (sender ID)

### Response shape

```json
{
  "id": "msg_abc123",
  "to": "+15551234567",
  "text": "Hello!",
  "status": "sent",
  "segments": 1,
  "creditsUsed": 2,
  "createdAt": "2026-03-31T10:00:00Z"
}
```

### Schedule a message

```bash
curl -X POST https://sendly.live/api/v1/messages/schedule \
  -H "Authorization: Bearer $SENDLY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"to": "+15551234567", "text": "Reminder!", "messageType": "transactional", "scheduledAt": "2026-04-01T14:00:00Z"}'
```

Schedule window: 5 minutes to 5 days in the future.

### Batch send

```bash
curl -X POST https://sendly.live/api/v1/messages/batch \
  -H "Authorization: Bearer $SENDLY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"messages": [{"to": "+15551234567", "text": "Hello"}, {"to": "+15559876543", "text": "Hi"}], "messageType": "transactional"}'
```

Up to 10,000 recipients per batch.

### Group MMS

Send one message to 2–8 US/Canada recipients as a single group thread — everyone sees the other participants and replies fan out to the whole group.

```bash
curl -X POST https://sendly.live/api/v1/messages/group \
  -H "Authorization: Bearer $SENDLY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"to": ["+14155551234", "+14155555678"], "text": "Team sync at noon?", "messageType": "transactional"}'
```

**Required:** `to` (array of 2–8 E.164 US/CA numbers), plus `text` and/or `mediaUrls`

**Optional:** `from` (sender ID), `mediaUrls` (or `media_urls`), `messageType`

**Response — sent (201):** the send payload carries only the message id, status, recipients, and (when the carrier assigns one) a `group_message_id`. Threading is a server-side concept — there is no `groupKey` or `participants` on this response.
```json
{
  "id": "msg_abc123",
  "status": "sent",
  "to": ["+14155551234", "+14155555678"],
  "group_message_id": "grp_abc123"
}
```

**Response — simulated (201):** a test key (`sk_test_*`) or a workspace whose sending number is not yet carrier-approved returns a simulated send instead — `status` is `delivered`, no credits are charged, and `simulated` plus a human-readable `message` are present.
```json
{
  "id": "msg_abc123",
  "status": "delivered",
  "to": ["+14155551234", "+14155555678"],
  "simulated": true,
  "message": "Group message simulated (test key or verification pending)."
}
```

Group MMS is gated behind the `group_mms` feature flag (and `enable_mms` when sending media) — calls return `feature_disabled` (403) until it is enabled for your account. Requires an MMS-capable, 10DLC-registered sending number. US/Canada destinations only; other destinations return `unsupported_destination` (400).

**Errors:** `invalid_request` (fewer than 2 or more than 8 recipients, or neither `text` nor `mediaUrls`), `invalid_phone`, `unsupported_destination`, `compliance_blocked` (400); `insufficient_credits` (402); `feature_disabled` (403); `undeliverable_number` (422); `send_failed` (502, carrier rejection — includes the case where the sending number's 10DLC brand/campaign is not registered).

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

Supports `limit` (max 100), `offset`, `status`, and `q` (full-text search).

## Node.js SDK

```bash
npm install @sendly/node
```

```typescript
import Sendly from "@sendly/node";

const sendly = new Sendly(process.env.SENDLY_API_KEY!);

const msg = await sendly.messages.send({ to: "+15551234567", text: "Hello!", messageType: "transactional" });
const scheduled = await sendly.messages.schedule({ to: "+15551234567", text: "Later!", messageType: "transactional", scheduledAt: "2026-04-01T14:00:00Z" });
const batch = await sendly.messages.sendBatch({ messages: [{to: "+15551234567", text: "Hi"}], messageType: "transactional" });
const group = await sendly.messages.sendGroup({ to: ["+14155551234", "+14155555678"], text: "Team sync at noon?", messageType: "transactional" });
const improved = await sendly.messages.enhance({ text: "hey come check out our sale", messageType: "marketing" });
const list = await sendly.messages.list({ limit: 50 });
const single = await sendly.messages.get("msg_abc123");
```

## Message types

- **transactional**: OTP codes, order confirmations, appointment reminders, account alerts. Allowed 24/7.
- **marketing**: Promotions, sales, newsletters. Subject to quiet hours (9pm–8am recipient local time).

Misclassifying marketing as transactional violates TCPA.

## Sandbox testing

Use `sk_test_*` keys with magic phone numbers:

| Number | Behavior |
|---|---|
| +15005550000 | Always succeeds |
| +15005550001 | Invalid number error |
| +15005550002 | Cannot route error |
| +15005550003 | Queue full |
| +15005550004 | Rate limit exceeded |
| +15005550006 | Carrier rejected |

## Credit costs

- US/CA: 2 credits per SMS ($0.02)
- International: varies by country (2–48 credits)
- 1 credit = $0.01

## Conversations API

Messages are automatically threaded into conversations. Use the conversations API for two-way messaging:

```typescript
const convos = await sendly.conversations.list({ status: "active", limit: 20 });
const replies = await sendly.conversations.suggestReplies("conv_abc123");
```

## Full reference

- API docs: https://sendly.live/docs/sms
- SDK docs: https://sendly.live/docs/sdks
- OpenAPI spec: https://sendly.live/openapi.yaml
- Sandbox docs: https://sendly.live/docs/sandbox
