---
name: voice-calls
description: Places and inspects phone calls handled by a workspace's AI agents via the Sendly Calls API. Covers placing an outbound call to a US or Canadian number, choosing the from number, attaching per-call context and metadata, polling a call until it ends, reading the transcript, fetching the recording, hanging up, and recovering from every refusal code. Applies when an agent needs to phone a person (appointment confirmations, callbacks, reminders) rather than text them.
---

# AI Phone Calls with Sendly

A voice-enabled Sendly number can place a phone call that one of the workspace's AI agents (a receptionist configured in the dashboard under Calls, then Agents) talks through. Over the API you place the call, watch it, end it, and download the recording. Switching voice on for a number, choosing how it answers, registering the emergency address and creating agents are dashboard steps in this release.

> **Rolling out workspace by workspace.** Until voice is enabled for the workspace, every `/api/v1/calls` route returns `404 voice_not_enabled`. Ask Sendly to enable it before relying on these endpoints. Placing or ending a call always needs a **live** key (`sk_live_*`) with the `calls:write` scope; test keys get `403 live_key_required` on POST. Reads need `calls:read`.

## The four rules that shape everything

1. **An API call is always answered by an agent.** There is no human on your side of the line, so `agentId` is required (`400 agent_required` without it). The agent speaks first, using its instructions plus the `context` you pass for this call.
2. **US and Canada only, E.164 only.** `to` must look like `+15555550123`; anything else is `400 invalid_number` or `400 destination_not_supported`.
3. **Calls are prepaid per started minute.** An agent-handled outbound call costs 10 credits a minute (2 for the call, 8 for the agent; 1 credit = $0.01). You need at least one minute in the balance to start (`402 insufficient_credits`); if the balance runs out mid-call the call ends with `hangupClass: "credits_exhausted"`. Unanswered calls cost nothing.
4. **The from number needs an emergency address.** US law requires one before the number can place calls; without it you get `428 e911_required`. Register it in the dashboard under Calls, then Settings.

## Quick start

```typescript
import Sendly from "@sendly/node";

const sendly = new Sendly(process.env.SENDLY_API_KEY!);

const call = await sendly.calls.create({
  to: "+15555550123",
  agentId: "3c4d5e6f-7081-4293-a4b5-c6d7e8f90a1b",
  context: "You are calling Jordan to confirm the 3pm appointment on Tuesday.",
  metadata: { crmId: "lead_8812" },
});
console.log(call.id, call.status); // "6f1c2d3e-…" "ringing"
```

## REST API

**Base URL:** `https://sendly.live/api/v1`

### Find a number you can call from

```bash
curl https://sendly.live/api/v1/numbers \
  -H "Authorization: Bearer $SENDLY_API_KEY"
```

Each number carries `voiceEnabled` (boolean) and `voiceMode` (`none` | `ring_dashboard` | `agent`). Pick one with `voiceEnabled: true`. If the workspace has exactly one voice-enabled number you can omit `from`; with several, `from` is required (`400 from_number_required`).

### Place a call

```bash
curl -X POST https://sendly.live/api/v1/calls \
  -H "Authorization: Bearer $SENDLY_API_KEY" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: $(uuidgen)" \
  -d '{
    "to": "+15555550123",
    "agentId": "3c4d5e6f-7081-4293-a4b5-c6d7e8f90a1b",
    "from": "+15555550188",
    "context": "You are calling Jordan to confirm the 3pm appointment on Tuesday.",
    "metadata": { "crmId": "lead_8812" }
  }'
```

**Required:** `to` (E.164, US or CA), `agentId` (an enabled agent in the workspace).
**Optional:** `from` (a voice-enabled number in the workspace), `context` (string, max 2000 chars, appended to the agent's instructions for this call only, never echoed back), `metadata` (up to 20 string pairs; keys 1 to 40 chars matching `^[A-Za-z0-9_.:-]+$`, values up to 500 chars; echoed on every read and in every `call.*` webhook).

**Response (201):**
```json
{
  "id": "6f1c2d3e-4a5b-4c6d-8e9f-0a1b2c3d4e5f",
  "object": "call",
  "kind": "pstn",
  "direction": "outbound",
  "status": "ringing",
  "handledBy": "agent",
  "agentId": "3c4d5e6f-7081-4293-a4b5-c6d7e8f90a1b",
  "from": "+15555550188",
  "to": "+15555550123",
  "callerName": "Front Desk",
  "calleeName": "+15555550123",
  "startedAt": "2026-09-12T14:03:11.000Z",
  "answeredAt": null,
  "endedAt": null,
  "durationSecs": 0,
  "creditsCharged": 0,
  "billing": "metered",
  "hangupClass": null,
  "recordingStatus": null,
  "metadata": { "crmId": "lead_8812" }
}
```

The SDKs send an `Idempotency-Key` on every POST automatically; with curl, send your own so a retried request does not place a second call.

### Watch the call

```bash
curl https://sendly.live/api/v1/calls/6f1c2d3e-4a5b-4c6d-8e9f-0a1b2c3d4e5f \
  -H "Authorization: Bearer $SENDLY_API_KEY"
```

`status` moves `ringing` → `active` (when answered; `answeredAt` is set) → one terminal value: `completed`, `no_answer`, `busy`, `cancelled`, `declined`, `failed`. Once terminal, `endedAt`, `durationSecs`, `creditsCharged` are final, `billing` becomes `settled`, and `hangupClass` says why it ended (`normal`, `callee_hung_up`, `agent_agent_hangup` when the agent decided the conversation was over, `ring_timeout`, `callee_busy`, `credits_exhausted`, `max_duration` at the 60 minute ceiling, and so on; anything unrecognised reads `ended`).

Agent-handled calls also return `transcript`: an array of `{ "speaker": "caller" | "agent", "text": "…", "atMs": 12345 }` (empty until someone speaks). It is only on the retrieve route, never on the list.

Poll every few seconds, or subscribe a webhook to `call.started` (fires when answered) and `call.completed` (fires at the end, with the final `hangup_class`, `duration_secs`, `credits_charged`, `billing` and your `metadata`). Webhook objects are snake_case twins of the REST object.

### List calls

```bash
curl "https://sendly.live/api/v1/calls?status=completed&agentId=3c4d5e6f-7081-4293-a4b5-c6d7e8f90a1b&limit=20" \
  -H "Authorization: Bearer $SENDLY_API_KEY"
```

Filters: `limit` (1 to 100, default 50), `offset`, `status`, `direction`, `kind` (`pstn` | `internal`), `agentId`, `to`, `from` (E.164, exact). Newest first.

**Response:** `{ "data": [Call, …], "pagination": { "total": 132, "limit": 20, "offset": 0, "hasMore": true } }`

### Hang up

```bash
curl -X POST https://sendly.live/api/v1/calls/6f1c2d3e-4a5b-4c6d-8e9f-0a1b2c3d4e5f/hangup \
  -H "Authorization: Bearer $SENDLY_API_KEY"
```

Returns the Call: a ringing call becomes `cancelled` (`hangupClass: "caller_cancelled"`), an active one `completed` (`normal`). Hanging up an already ended call returns it unchanged (200, idempotent).

### Fetch the recording

```bash
curl https://sendly.live/api/v1/calls/6f1c2d3e-4a5b-4c6d-8e9f-0a1b2c3d4e5f/recording \
  -H "Authorization: Bearer $SENDLY_API_KEY"
```

**Response:**
```json
{
  "callId": "6f1c2d3e-4a5b-4c6d-8e9f-0a1b2c3d4e5f",
  "status": "ready",
  "url": "https://…signed…",
  "expiresAt": "2026-09-12T14:10:00.000Z",
  "contentType": "audio/ogg"
}
```

`status` is `none` (recording off, or the call never connected), `recording`, `ready` or `failed`. `url` and `expiresAt` are non-null only when `ready`; the URL is signed and valid for 5 minutes, so download immediately rather than storing the link. Recordings are Ogg/Opus; agent calls are dual-channel (caller left, agent right). Recording is switched on per workspace in the dashboard under Calls, then Settings. The `call.recording.ready` webhook fires when it can be fetched.

## Node.js SDK

```typescript
import Sendly from "@sendly/node";
const sendly = new Sendly(process.env.SENDLY_API_KEY!);

const call = await sendly.calls.create({
  to: "+15555550123",
  agentId: "3c4d5e6f-7081-4293-a4b5-c6d7e8f90a1b",
  from: "+15555550188",
  context: "Confirm Tuesday's 3pm appointment.",
  metadata: { crmId: "lead_8812" },
});

const live = await sendly.calls.get(call.id);            // status, hangupClass, transcript
const { data, pagination } = await sendly.calls.list({ status: "completed", limit: 20 });
await sendly.calls.hangup(call.id);
const rec = await sendly.calls.recording(call.id);       // rec.url is null until rec.status === "ready"
```

## Python SDK

```python
import os

from sendly import Sendly

client = Sendly(os.environ["SENDLY_API_KEY"])

call = client.calls.create(
    "+15555550123",
    "3c4d5e6f-7081-4293-a4b5-c6d7e8f90a1b",
    from_="+15555550188",
    context="Confirm Tuesday's 3pm appointment.",
    metadata={"crmId": "lead_8812"},
)

live = client.calls.get(call.id)                       # live.status, live.hangup_class, live.transcript
page = client.calls.list(status="completed", limit=20)  # page.data, page.pagination.has_more
client.calls.hangup(call.id)
rec = client.calls.recording(call.id)                   # rec.url is None until rec.status == "ready"
```

## Before-call checklist

1. `GET /numbers`: is there a number with `voiceEnabled: true`? If several, pick a `from`.
2. Does the workspace have at least 10 credits? (`GET /account` balance, or the `sendly credits` CLI command.)
3. Is the agent enabled? A disabled agent returns `409 agent_disabled`; create and switch agents on in the dashboard.
4. Is `to` a US or Canadian E.164 number, and not your own `from` number?

## Error handling

Refusals are checked in roughly this order; fix the first one you hit.

| Error | Status | Meaning | Action |
|---|---|---|---|
| `voice_not_enabled` | 404 | Voice is not enabled for this workspace | Ask Sendly to enable it |
| `live_key_required` | 403 | Test key on a POST | Use a live `sk_live_*` key |
| `insufficient_permissions` | 403 | Key lacks `calls:write` (or `calls:read`) | Issue a key with the scope |
| `forbidden` | 403 | Viewer role in a team workspace tried to write | Use a member or admin's key |
| `voice_unavailable` / `agents_unavailable` | 503 | Dialling or agents are not available right now | Retry later; tell the account owner |
| `outbound_calls_not_enabled` | 404 | Outbound calling is off for the workspace | Ask Sendly to enable it |
| `invalid_number` | 400 | `to` is not a valid number, or equals `from` | Send E.164 with the country code |
| `destination_not_supported` | 400 | Not a US or Canadian number | Only +1 US/CA destinations are supported |
| `agent_required` | 400 | No `agentId` | Pass the id of an enabled agent |
| `invalid_metadata` / `invalid_request` | 400 | Metadata shape, `context` over 2000 chars, body not an object | Fix the field the `message` names |
| `agent_not_found` | 404 | No such agent in the workspace | Check the id |
| `agent_disabled` | 409 | Agent exists but is switched off | Switch it on in the dashboard |
| `number_not_found` | 404 | `from` is not in the workspace | Pick a number from `GET /numbers` |
| `no_voice_number` | 409 | No voice-enabled number in the workspace | Enable voice on a number in the dashboard |
| `from_number_required` | 400 | Several voice-enabled numbers, no `from` | Pass `from` |
| `e911_required` | 428 | No emergency address on the `from` number | Register one in the dashboard (Calls, then Settings) |
| `insufficient_credits` | 402 | Below one minute at 10 credits (`creditsNeeded`, `currentBalance` included) | Top up credits |
| `lines_busy` | 409 | All the workspace's lines are in use | Retry with backoff |
| `daily_call_limit` | 429 | Today's calling limit reached | Try again tomorrow |
| `rate_limit_exceeded` | 429 | More than 10 creates a minute, or the key limit | Slow down; honour `X-RateLimit-*` |
| `call_not_found` | 404 | Not this workspace's call (get, hangup, recording) | Check the id |
| `voice_internal_error` | 500 | Something went wrong on the platform | Retry once; report if it persists |

## Full reference

- Voice docs: https://sendly.live/docs/voice
- Calls API reference: https://sendly.live/docs/voice/api
- Place a call how-to: https://sendly.live/docs/voice/place-call
- Recordings how-to: https://sendly.live/docs/voice/recordings
