---
name: verifying-phones
description: Verifies phone numbers via SMS OTP using the Sendly Verify API. Sends codes, checks codes, handles expiry, and provides hosted verification sessions. Applies for phone verification, OTP, 2FA, or passwordless login flows.
---

# Phone Verification with Sendly

## Quick start

```typescript
import Sendly from "@sendly/node";

const sendly = new Sendly(process.env.SENDLY_API_KEY!);

const verification = await sendly.verify.send({ to: "+14155550142" });
console.log(verification.id); // ver_7c1f4b9a83d24e6fa0b512d7e93c4a18

const result = await sendly.verify.check(verification.id, { code: "123456" });
console.log(result.status); // "verified" while pending/expired/failed are the other states
```

Two scopes are involved: `verify:send` for the send, resend and check calls, `verify:read` for
status lookups.

## REST API

**Base URL:** `https://sendly.live/api/v1`

### Send OTP

```bash
curl -X POST https://sendly.live/api/v1/verify \
  -H "Authorization: Bearer $SENDLY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"to": "+14155550142", "app_name": "MyApp", "code_length": 6, "timeout_secs": 300}'
```

**Required:** `to` (E.164 format)

**Optional:** `app_name` (shown in SMS), `code_length` (4–10, default 6), `timeout_secs` (60–3600,
default 300), `template_id` (default `tpl_preset_otp`), `profile_id`

Out-of-range `code_length` and `timeout_secs` are **clamped, not rejected**, so `code_length: 99`
quietly becomes 10. Send the value you want.

**Response (`201 Created`):**
```json
{
  "id": "ver_7c1f4b9a83d24e6fa0b512d7e93c4a18",
  "status": "pending",
  "phone": "+14155550142",
  "expires_at": "2026-04-01T10:05:00Z",
  "sandbox": false
}
```

In sandbox mode (`sk_test_*` keys), the response also includes `sandbox: true` and `sandbox_code`
with the actual code. It is never present on a live key, so no code path may depend on it.

Store `id` — it is the handle for every later call. Never store the code.

### Check code

```bash
curl -X POST https://sendly.live/api/v1/verify/ver_7c1f4b9a83d24e6fa0b512d7e93c4a18/check \
  -H "Authorization: Bearer $SENDLY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"code": "123456"}'
```

Success is `200` with `status: "verified"` and `verified_at`. Only `status: "verified"` means the
number is proven — do not infer success from the HTTP code alone.

A verification's `status` is one of `pending`, `verified`, `expired`, `failed` or `invalid`.
`max_attempts_exceeded` is an **error code, not a status**.

The default budget is 3 attempts and it counts wrong guesses only: a correct code refunds the
attempt it consumed. Checking an already-verified id with the original correct code returns the same
`verified` payload; any other code returns `invalid_code`, so a replay never reads as a fresh
success.

### Resend OTP

```bash
curl -X POST https://sendly.live/api/v1/verify/ver_7c1f4b9a83d24e6fa0b512d7e93c4a18/resend \
  -H "Authorization: Bearer $SENDLY_API_KEY"
```

Resend is allowed in exactly two states: `status` is `expired`, or it is still `pending` and
delivery failed. Anything else returns `400 invalid_state`. Passing `expires_at` does not change
`status` at once: it becomes `expired` when a check or `GET /api/v1/verify/{id}` reads it, or at the
next hourly cleanup, and until then it is still `pending`, so read it before resending. A resend issues a **new** code, resets
the expiry and resets `attempts` to 0, so any code the user already has stops working. If the user
simply did not receive it and the verification is pending and delivered, start a fresh
`POST /api/v1/verify` instead.

### Hosted verification sessions

Zero-code verification UI hosted by Sendly:

```bash
curl -X POST https://sendly.live/api/v1/verify/sessions \
  -H "Authorization: Bearer $SENDLY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"success_url": "https://app.example/verified?token={TOKEN}", "brand_name": "MyApp"}'
```

Returns a `url` the user visits. After verification, redirects to `success_url` with a token. Validate the token server-side:

```bash
curl -X POST https://sendly.live/api/v1/verify/sessions/validate \
  -H "Authorization: Bearer $SENDLY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"token": "tok_from_redirect"}'
```

## Node.js SDK

```typescript
import Sendly from "@sendly/node";
const sendly = new Sendly(process.env.SENDLY_API_KEY!);

const v = await sendly.verify.send({ to: "+14155550142", appName: "MyApp" });
const check = await sendly.verify.check(v.id, { code: "123456" });
const resend = await sendly.verify.resend(v.id);
const status = await sendly.verify.get(v.id);
const session = await sendly.verify.sessions.create({ successUrl: "https://app.example/done?token={TOKEN}", brandName: "MyApp" });
const validated = await sendly.verify.sessions.validate({ token: "tok_xxx" });
```

## Error handling

| Error | HTTP | Meaning | Action |
|---|---|---|---|
| `invalid_phone_format` | 400 | `to` is not E.164 | Format as +{country}{number} |
| `invalid_code` | 400 | Wrong code, budget left | Re-prompt; the body carries `remaining_attempts` |
| `invalid_request` | 400 | `code` missing or not a string | Send `{"code": "123456"}` |
| `expired` | 410 | Past `expires_at` | Resend, or start a new verification |
| `max_attempts_exceeded` | 429 | Guess budget spent | Start a new verification, do not retry this one |
| `not_found` | 404 | No such id for this account | Check the id and that the key is for the same workspace |
| `rate_limit_exceeded` | 429 | Per-phone or per-account OTP throttle | Honour `retryAfter` in the body |
| `verification_required` | 403 | Live OTP send with no approved sender | Human go-live step |

## Rate limits specific to OTP

On top of the per-key request limit, sends are throttled per destination and per account: 5 per
phone per 10 minutes, 20 per phone per day, and 100 per account per minute. Exceeding any of them
returns `429 rate_limit_exceeded` with `retryAfter` in seconds. This applies to resends too.

## Sandbox testing

With `sk_test_*` keys, the `sandbox_code` field in the send response contains the code. No real SMS
is sent. OTP to `+15005550000` always succeeds.

## Full reference

- Verify API docs: https://sendly.live/docs/verify
- OTP tutorial: https://sendly.live/docs/tutorials/otp
- Error handling guide: https://sendly.live/docs/how-to/handle-otp-errors
