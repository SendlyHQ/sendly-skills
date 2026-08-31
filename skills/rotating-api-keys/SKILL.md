---
name: rotating-api-keys
description: Rotates a Sendly API key with a grace period so callers roll over with zero downtime, using the Account Keys API. Covers issuing a replacement key, the one-time raw secret, and the overlap window. Applies when rotating, refreshing, or replacing an API key on a schedule or after a suspected leak.
---

# Rotating API Keys with Sendly

Rotation issues a **new** key and keeps the **old** one working for a grace period, so you can move every caller onto the new secret before the old one stops. The new secret's raw value is returned exactly once — store it immediately.

## Quick start

```typescript
import Sendly from "@sendly/node";

const sendly = new Sendly(process.env.SENDLY_API_KEY!);

const { newKey, oldKey, message } = await sendly.account.rotateApiKey("key_abc123");
console.log(newKey.key); // "sk_live_…" — shown once, save it now
console.log(message);    // "Old key will expire in 24 hours"
```

## Authentication

Send a Bearer token (`SENDLY_API_KEY`). Any valid key can rotate a key in its own workspace — no extra scope is required. The key you rotate does not have to be the key you authenticate with.

## REST API

**Base URL:** `https://sendly.live/api/v1`

### Rotate a key

```bash
curl -X POST https://sendly.live/api/v1/account/keys/key_abc123/rotate \
  -H "Authorization: Bearer $SENDLY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"gracePeriodHours": 72}'
```

**Optional:** `gracePeriodHours` — how long the old key keeps working, 24–168 inclusive (default 24). Both keys are valid during this window.

**Response (200):**
```json
{
  "newKey": {
    "id": "key_def456",
    "name": "Production (rotated)",
    "type": "live",
    "prefix": "sk_live",
    "lastFour": "9f2a",
    "permissions": ["sms:send", "sms:read", "..."],
    "createdAt": "2026-07-09T10:00:00Z",
    "isRevoked": false,
    "key": "sk_live_9f2a…",
    "warning": "This key will only be shown once. Store it securely."
  },
  "oldKey": {
    "id": "key_abc123",
    "name": "Production",
    "type": "live",
    "prefix": "sk_live",
    "lastFour": "1c7b",
    "isRevoked": false
  },
  "message": "Old key will expire in 24 hours"
}
```

`newKey.key` is the raw secret and is present **only** in this response — it is never returned again. `newKey` inherits the old key's scopes; `oldKey` now counts down its grace period, after which it stops working.

**Errors:**

| Error | Status | Meaning |
|---|---|---|
| `invalid_request` | 400 | `gracePeriodHours` is outside 24–168 |
| `invalid_state` | 400 | The key is inactive or revoked and can't be rotated |
| `already_rotating` | 400 | This key is already rotating — wait for its grace period to end first |
| `predecessor_in_grace` | 400 | The key's predecessor is still inside its grace period |
| `not_found` | 404 | No such key in your workspace |

## Node.js SDK

```typescript
import Sendly from "@sendly/node";
const sendly = new Sendly(process.env.SENDLY_API_KEY!);

// Rotate with the default 24-hour grace period
const { newKey, oldKey, message } = await sendly.account.rotateApiKey("key_abc123");
console.log(newKey.key); // save this — shown once!

// Rotate with a 72-hour grace period (both keys work for 72h)
await sendly.account.rotateApiKey("key_abc123", { gracePeriodHours: 72 });
```

## Recommended flow

1. **Rotate.** Call rotate; capture `newKey.key` immediately (it is shown once).
2. **Roll callers over.** Deploy the new secret everywhere while both keys still work.
3. **Let the old key expire.** It stops automatically at the end of the grace period — no manual revoke needed. To cut over sooner, revoke the old key explicitly.

## Related key operations

The rest of the key lifecycle lives under the same `/api/v1/account/keys` base and maps to `sendly.account.*`: `createApiKey(name)`, `listApiKeys()`, `getApiKey(id)`, `getApiKeyUsage(id)`, `renameApiKey(id, name)`, and `revokeApiKey(id)`. Newly created keys, like rotated ones, return their raw secret only once.

## Full reference

- API keys docs: https://sendly.live/docs/api-keys
</content>
