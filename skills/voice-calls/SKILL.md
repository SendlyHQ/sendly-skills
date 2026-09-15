---
name: voice-calls
description: Places and inspects phone calls handled by a workspace's AI agents via the Sendly Calls API, and sets up what those calls need via the Voice API. Covers creating an agent, registering a number's emergency address, switching voice on for a number, placing an outbound call to a US or Canadian number, choosing the from number, attaching per-call context and metadata, polling a call until it ends, reading the transcript, fetching the recording, hanging up, and recovering from every refusal code. Applies when an agent needs to phone a person (appointment confirmations, callbacks, reminders) rather than text them.
---

# AI Phone Calls with Sendly

A voice-enabled Sendly number can place a phone call that one of the workspace's AI agents talks through. Over the API you place the call, watch it, end it, and download the recording. You can also set up everything a call depends on over the API: create the agent, register the number's emergency address, and switch voice on for the number and choose who answers it. The dashboard does the same under Calls → Agents and Calls → Voice settings.

> **Rolling out workspace by workspace.** Until voice is enabled for the workspace, every `/api/v1/calls` and `/api/v1/voice` route returns `404 voice_not_enabled`. Ask Sendly to enable it before relying on these endpoints. Every write (placing or ending a call, changing a number or an agent) needs a **live** key (`sk_live_*`) with the `calls:write` scope; test keys get `403 live_key_required`. Reads need `calls:read`.

## The four rules that shape everything

1. **An API call is always answered by an agent.** There is no human on your side of the line, so `agentId` is required (`400 agent_required` without it). The agent speaks first, using its instructions plus the `context` you pass for this call.
2. **US and Canada only, E.164 only.** `to` must look like `+15125550123`; anything else is `400 invalid_number` or `400 destination_not_supported`. 911 and other short service codes cannot be dialled (`400 invalid_number`).
3. **Calls are prepaid per started minute.** An agent-handled outbound call costs 10 credits a minute (2 for the call, 8 for the agent; 1 credit = $0.01). You need at least one minute in the balance to start (`402 insufficient_credits`); if the balance runs out mid-call the call ends with `hangupClass: "credits_exhausted"`. Unanswered calls cost nothing.
4. **The from number needs an emergency address.** US law requires one before the number can place calls; without it you get `428 e911_required`. Register it with `POST /api/v1/voice/numbers/:number/emergency-address` (see below) or in the dashboard under Calls → Voice settings.

## Quick start

```typescript
import Sendly from "@sendly/node";

const sendly = new Sendly(process.env.SENDLY_API_KEY!);

const call = await sendly.calls.create({
  to: "+15125550123",
  agentId: "3c4d5e6f-7081-4293-a4b5-c6d7e8f90a1b",
  context: "You are calling Jordan to confirm the 3pm appointment on Tuesday.",
  metadata: { crmId: "lead_8812" },
});
console.log(call.id, call.status);
```

## REST API

**Base URL:** `https://sendly.live/api/v1`

### Set up a number and an agent

Skip this when `GET /voice/agents` already shows an enabled agent and `GET /voice/numbers` shows a number with `voiceEnabled: true` and an `emergencyAddress`. Each step below changes how real phone calls are handled, so confirm it with the user first. On a team workspace these writes need a key whose role is owner or admin (`403 forbidden` otherwise).

**1. Create the agent.** Pick a voice id from `GET /voice/voices` (`ashley`, `edward`, `olivia`, `jacqueline`, `diego`; an unknown id gets the default voice).

```bash
curl -X POST https://sendly.live/api/v1/voice/agents \
  -H "Authorization: Bearer $SENDLY_API_KEY" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: $(uuidgen)" \
  -d '{
    "name": "Front desk",
    "voice": "ashley",
    "greeting": "Thanks for calling Northside Dental, how can I help?",
    "instructions": "Confirm, move or cancel appointments. Take a message for anything else."
  }'
```

Returns `201` with the agent: `id`, `object: "voice_agent"`, `name`, `enabled`, `voice`, `voiceLabel`, `language`, `greeting`, `instructions`, `tools`, `canSendSms`, `callsHandled`, `avgDurationSecs`, `createdAt`, `updatedAt`. Limits: `name` 1 to 80 characters, `greeting` up to 500, `instructions` up to 4000, `language` defaults to `en-US`, 20 agents per workspace (`409 agent_limit`). The greeting is spoken only when the agent answers an inbound call; on a call it places, it opens from its instructions plus the call's `context`. `tools.sendSms` (default true) lets it text the person on the call. `tools.transferTo` is stored, but call transfer is not available yet: the agent takes a message instead. Change an agent with `PATCH /voice/agents/:id` and any subset of the same fields (`{"enabled": false}` switches it off). Switching an agent off and `DELETE /voice/agents/:id` both return `409 agent_in_use` with `numbers` while a number still answers with it, and change nothing; point those numbers elsewhere first.

**2. Register the from number's emergency address.** Use the street address where the number is actually used; ask the user for it and never invent one.

```bash
curl -X POST https://sendly.live/api/v1/voice/numbers/%2B15125550188/emergency-address \
  -H "Authorization: Bearer $SENDLY_API_KEY" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: $(uuidgen)" \
  -d '{
    "street": "500 Example Ave",
    "unit": "Suite 2",
    "city": "Austin",
    "state": "TX",
    "zip": "78701"
  }'
```

`:number` is the number's id or its E.164 number with the `+` encoded as `%2B`. `country` is `US` (default) or `CA`; `state` is the two-letter code. Returns the number with `emergencyAddress.status` set to `"provisioning"`, which already allows calls and normally becomes `"active"` within about ten minutes, or to `"active"` straight away when the registration completes at once. The first registration adds $1.50 a month to the number; registering again replaces the address without adding the charge again. `422 invalid_address` means the address could not be verified: show `suggested` to the user and register it only once they confirm.

**3. Switch voice on for the number and choose who answers calls to it.**

```bash
curl -X PATCH https://sendly.live/api/v1/voice/numbers/%2B15125550188 \
  -H "Authorization: Bearer $SENDLY_API_KEY" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: $(uuidgen)" \
  -d '{ "voiceMode": "agent", "agentId": "3c4d5e6f-7081-4293-a4b5-c6d7e8f90a1b" }'
```

Returns the number with `voiceEnabled: true` and `voiceMode: "agent"`: from now on real callers to the number reach the agent. Without `voiceEnabled`, `"voiceMode": "ring_dashboard"` or `"agent"` on their own switch voice on, and `"voiceMode": "none"` on its own switches it off. `{"voiceMode": "ring_dashboard"}` rings the team in the dashboard instead (the number can still place agent calls), and `{"voiceEnabled": false}` switches voice off. Agent mode needs an enabled agent in the workspace (`400 agent_required`, `404 agent_not_found`, `409 agent_disabled`). Switching voice on can fail with `502 voice_attach_failed` (retry shortly) or `503 voice_unavailable`.

### Find a number you can call from

```bash
curl https://sendly.live/api/v1/voice/numbers \
  -H "Authorization: Bearer $SENDLY_API_KEY"
```

Returns `{ "data": [VoiceNumber, …] }`, the default number first. Each number carries `voiceEnabled`, `voiceMode` (`none` | `ring_dashboard` | `agent`), `agentId`, `emergencyAddress` (null until one is registered, otherwise `{ "status", "address" }`) and `ratePerMinute` in credits (`{ "inbound", "outbound", "agent" }`). Pick one with `voiceEnabled: true` and an emergency address whose status is `provisioning` or `active`. If the workspace has exactly one voice-enabled number you can omit `from`; with several, `from` is required (`400 from_number_required`).

### Place a call

```bash
curl -X POST https://sendly.live/api/v1/calls \
  -H "Authorization: Bearer $SENDLY_API_KEY" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: $(uuidgen)" \
  -d '{
    "to": "+15125550123",
    "agentId": "3c4d5e6f-7081-4293-a4b5-c6d7e8f90a1b",
    "from": "+15125550188",
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
  "from": "+15125550188",
  "to": "+15125550123",
  "callerName": "Front desk",
  "calleeName": "+15125550123",
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

`status` moves `ringing` → `active` (when answered; `answeredAt` is set) → one terminal value: `completed`, `no_answer`, `busy`, `cancelled`, `declined`, `failed`. Once terminal, `endedAt`, `durationSecs`, `creditsCharged` are final, `billing` becomes `settled`, and `hangupClass` says why it ended (`normal`, `callee_hung_up` or `agent_caller_left` when the other party hung up, `agent_agent_hangup` when the agent decided the conversation was over, `ring_timeout`, `callee_busy`, `credits_exhausted`, `max_duration` at the 60 minute ceiling, and so on; anything unrecognised reads `ended`).

Agent-handled calls also return `transcript`: an array of `{ "speaker": "caller" | "agent", "text": "…", "atMs": 12345 }` (empty until someone speaks). `agent` is the agent and `caller` is the other party, on calls you place as well as calls you receive. It is only on the retrieve route, never on the list.

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

`status` is `none` (recording off, or the call never connected), `recording`, `ready` or `failed`. `url` and `expiresAt` are non-null only when `ready`; the URL is signed and valid for 5 minutes, so download immediately rather than storing the link. Recordings are Ogg/Opus. Agent-handled calls are stereo: the agent on the left channel and the other party on the right (agent recordings made before 06:53 UTC on 15 September 2026 carry the same mix on both channels). A call between two people is one mix on both channels. Recording is switched on per workspace in the dashboard under Calls → Voice settings. The `call.recording.ready` webhook fires when it can be fetched.

## Node.js SDK

```typescript
import Sendly from "@sendly/node";
const sendly = new Sendly(process.env.SENDLY_API_KEY!);

const call = await sendly.calls.create({
  to: "+15125550123",
  agentId: "3c4d5e6f-7081-4293-a4b5-c6d7e8f90a1b",
  from: "+15125550188",
  context: "Confirm Tuesday's 3pm appointment.",
  metadata: { crmId: "lead_8812" },
});

const live = await sendly.calls.get(call.id);            // status, hangupClass, transcript
const { data, pagination } = await sendly.calls.list({ status: "completed", limit: 20 });
await sendly.calls.hangup(call.id);
const rec = await sendly.calls.recording(call.id);       // rec.url is null until rec.status === "ready"
```

Setting up a number and an agent:

```typescript
const agent = await sendly.voice.agents.create({
  name: "Front desk",
  voice: "ashley",
  instructions: "Confirm, move or cancel appointments. Take a message for anything else.",
});

await sendly.voice.numbers.registerEmergencyAddress("+15125550188", {
  street: "500 Example Ave",
  city: "Austin",
  state: "TX",
  zip: "78701",
});

await sendly.voice.numbers.update("+15125550188", { voiceMode: "agent", agentId: agent.id });

const { data: numbers } = await sendly.voice.numbers.list();
const { data: voices } = await sendly.voice.voices.list();
```

## Python SDK

```python
import os

from sendly import Sendly

client = Sendly(os.environ["SENDLY_API_KEY"])

call = client.calls.create(
    "+15125550123",
    "3c4d5e6f-7081-4293-a4b5-c6d7e8f90a1b",
    from_="+15125550188",
    context="Confirm Tuesday's 3pm appointment.",
    metadata={"crmId": "lead_8812"},
)

live = client.calls.get(call.id)                       # live.status, live.hangup_class, live.transcript
page = client.calls.list(status="completed", limit=20)  # page.data, page.pagination.has_more
client.calls.hangup(call.id)
rec = client.calls.recording(call.id)                   # rec.url is None until rec.status == "ready"
```

Setting up a number and an agent:

```python
agent = client.voice.agents.create(
    "Front desk",
    voice="ashley",
    instructions="Confirm, move or cancel appointments. Take a message for anything else.",
)

client.voice.numbers.register_emergency_address(
    "+15125550188",
    street="500 Example Ave",
    city="Austin",
    state="TX",
    zip="78701",
)

client.voice.numbers.update("+15125550188", voice_mode="agent", agent_id=agent.id)
```

## Before-call checklist

1. `GET /voice/numbers`: is there a number with `voiceEnabled: true` and an `emergencyAddress` (status `provisioning` or `active`)? If several are voice-enabled, pick a `from`. If none, set one up first (see Set up a number and an agent).
2. Does the workspace have at least 10 credits? (`GET /account` balance, or the `sendly credits` CLI command.)
3. Is the agent enabled? `GET /voice/agents/:id` shows `enabled`; a disabled agent returns `409 agent_disabled`, and `PATCH /voice/agents/:id` with `{"enabled": true}` switches it on.
4. Is `to` a US or Canadian E.164 number, and not your own `from` number?

## Error handling

### Placing and managing calls

Refusals are checked in roughly this order; fix the first one you hit.

| Error | Status | Meaning | Action |
|---|---|---|---|
| `voice_not_enabled` | 404 | Voice is not enabled for this workspace | Ask Sendly to enable it |
| `live_key_required` | 403 | Test key on a write | Use a live `sk_live_*` key |
| `insufficient_permissions` | 403 | Key lacks `calls:write` (or `calls:read`) | Issue a key with the scope |
| `forbidden` | 403 | Viewer role in a team workspace tried to place or end a call | Use a member, admin or owner key |
| `voice_unavailable` / `agents_unavailable` | 503 | Dialling or agents are not available right now | Retry later; tell the account owner |
| `outbound_calls_not_enabled` | 404 | Outbound calling is off for the workspace | Ask Sendly to enable it |
| `invalid_number` | 400 | `to` is not a valid number (911 and short service codes included), or equals `from` | Send E.164 with the country code |
| `destination_not_supported` | 400 | Not a US or Canadian number | Only +1 US/CA destinations are supported |
| `agent_required` | 400 | No `agentId` | Pass the id of an enabled agent |
| `invalid_metadata` / `invalid_request` | 400 | Metadata shape, `context` over 2000 chars, body not an object | Fix the field the `message` names |
| `agent_not_found` | 404 | No such agent in the workspace | Check the id with `GET /voice/agents` |
| `agent_disabled` | 409 | Agent exists but is switched off | `PATCH /voice/agents/:id` with `{"enabled": true}` |
| `number_not_found` | 404 | `from` is not a voice-enabled number in the workspace | Pick a number from `GET /voice/numbers` |
| `no_voice_number` | 409 | No voice-enabled number in the workspace | Switch voice on with `PATCH /voice/numbers/:number` |
| `from_number_required` | 400 | Several voice-enabled numbers, no `from` | Pass `from` |
| `e911_required` | 428 | No emergency address on the `from` number | `POST /voice/numbers/:number/emergency-address` |
| `insufficient_credits` | 402 | Below one minute at 10 credits (`creditsNeeded`, `currentBalance` included) | Top up credits |
| `lines_busy` | 409 | All the workspace's lines are in use | Retry with backoff |
| `daily_call_limit` | 429 | Today's calling limit reached | Try again tomorrow |
| `rate_limit_exceeded` | 429 | More than 10 creates a minute, or the key limit | Slow down; honour `X-RateLimit-*` |
| `call_not_found` | 404 | Not this workspace's call (get, hangup, recording) | Check the id |
| `voice_internal_error` | 500 | Something went wrong on the platform | Retry once; report if it persists |

### Setting up numbers and agents

| Error | Status | Meaning | Action |
|---|---|---|---|
| `forbidden` | 403 | The key's role in a team workspace is not owner or admin | Use an owner or admin key |
| `invalid_request` | 400 | A field has the wrong type or breaks a limit; the `message` names it | Fix that field |
| `invalid_voice_mode` | 400 | `voiceMode` is not `none`, `ring_dashboard` or `agent` | Use one of the three |
| `agent_required` | 400 | Agent mode with no `agentId` on the request or the number | Pass the id of an enabled agent |
| `invalid_address` | 400 | A required address field is missing or malformed, or the country is not US or CA | Send `street`, `city`, a two-letter `state` and a valid `zip` |
| `e911_not_applicable` | 400 | The number is outside the US and Canada | Only US and Canadian numbers take an emergency address |
| `number_not_found` | 404 | `:number` is not an active number in the workspace | Pick one from `GET /voice/numbers` |
| `agent_not_found` | 404 | No such agent in the workspace | Check the id with `GET /voice/agents` |
| `agent_disabled` | 409 | Pointing a number at a switched-off agent | Switch the agent on first |
| `agent_limit` | 409 | The workspace already has 20 agents | Reuse or delete one |
| `agent_in_use` | 409 | Deleting or switching off (`{"enabled": false}`) an agent that answers numbers; `numbers` lists them | Point each number elsewhere with `PATCH /voice/numbers/:number`, then try again |
| `invalid_address` | 422 | The address could not be verified; `suggested` holds a corrected one | Confirm `suggested` with the user, then register it |
| `voice_attach_failed` | 502 | Voice could not be switched on for the number | Retry shortly |
| `carrier_refused` | 502 | The emergency address registration was refused | Retry shortly; contact support if the message says the number could not be found for emergency registration |
| `voice_unavailable` | 503 | Voice cannot be switched on for numbers right now | Retry later |

## Full reference

- Voice docs: https://sendly.live/docs/voice
- Voice API reference: https://sendly.live/docs/voice/api
- Enable voice on a number: https://sendly.live/docs/voice/enable-number
- AI receptionist how-to: https://sendly.live/docs/voice/receptionist
- Emergency address how-to: https://sendly.live/docs/voice/emergency-address
- Place a call how-to: https://sendly.live/docs/voice/place-call
- Recordings how-to: https://sendly.live/docs/voice/recordings
