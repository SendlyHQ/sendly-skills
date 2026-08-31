---
name: managing-numbers
description: Manages the phone numbers a Sendly workspace owns — list them, inspect one, set the default sender, cancel a scheduled release, or release a number. Applies when viewing owned numbers, choosing which number sends by default, or giving a number back to the carrier.
---

# Managing Numbers with Sendly

List the numbers your workspace owns, inspect a single number, pick the default sender, and release numbers you no longer need. To *buy* a new number, see the discovery/buy flow at the end.

## Quick start

```typescript
import Sendly from "@sendly/node";

const sendly = new Sendly(process.env.SENDLY_API_KEY!);

const { numbers } = await sendly.numbers.list();
for (const n of numbers) {
  console.log(`${n.phoneNumber} — ${n.status}`);
}
```

## Authentication

All requests require a Bearer token (`SENDLY_API_KEY`). Reads need the `numbers:read` scope; mutations (set default, cancel release, release) need `numbers:write`.

## REST API

**Base URL:** `https://sendly.live/api/v1`

### The owned-number shape

```json
{
  "id": "num_abc123",
  "phoneNumber": "+14155551234",
  "status": "active",
  "source": "purchased",
  "countryCode": "US",
  "phoneNumberType": "toll_free",
  "monthlyCostCents": 110,
  "isDefault": true,
  "requirementsSubmittedAt": null,
  "pendingCancellation": false,
  "scheduledReleaseAt": null
}
```

- `status` — `provisioning`, `requirements_required`, `active`, or `failed`. A number can only send once it is `active`.
- `requirementsSubmittedAt` — for a `requirements_required` number, `null` means "still needs documents", non-null means "documents submitted, under carrier review".
- `isDefault` — only present on the single-number responses (`GET /numbers/:id`, `PATCH /numbers/:id`); the list projection omits it.
- `pendingCancellation` / `scheduledReleaseAt` — set when a live paid number is scheduled to be released at the end of its paid period.

### List owned numbers

```bash
curl https://sendly.live/api/v1/numbers \
  -H "Authorization: Bearer $SENDLY_API_KEY"
```

**Response:** `{ "numbers": [ ... ] }` (each entry omits `isDefault`).

### Get one number

```bash
curl https://sendly.live/api/v1/numbers/num_abc123 \
  -H "Authorization: Bearer $SENDLY_API_KEY"
```

Returns the full owned-number record, **including** `isDefault`. Returns `not_found` (404) if the number isn't in your workspace.

### Update a number

Only two mutations are supported. Supply at least one:

- `isDefault: true` — make this the workspace's default sending number (the number must be `active`).
- `pendingCancellation: false` — cancel a previously scheduled release and keep the number.

```bash
curl -X PATCH https://sendly.live/api/v1/numbers/num_abc123 \
  -H "Authorization: Bearer $SENDLY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"isDefault": true}'
```

```bash
# "Keep this number" — cancel a scheduled release
curl -X PATCH https://sendly.live/api/v1/numbers/num_abc123 \
  -H "Authorization: Bearer $SENDLY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"pendingCancellation": false}'
```

Returns the full updated owned-number record (including `isDefault`).

**Errors:** `no_supported_fields` (400, body had neither supported field), `invalid_state` (400, `isDefault` requested for a non-active number), `not_found` (404).

### Release a number

Release the number (or, for a live paid purchase, schedule it to be released at the end of the paid period).

```bash
curl -X DELETE https://sendly.live/api/v1/numbers/num_abc123 \
  -H "Authorization: Bearer $SENDLY_API_KEY"
```

**Response — immediate release:**
```json
{ "success": true }
```

**Response — scheduled release** (a live paid purchase is cancelled at the end of the paid period):
```json
{ "success": true, "scheduled": true, "scheduledReleaseAt": "2026-08-01T00:00:00Z" }
```

To reverse a scheduled release before it takes effect, `PATCH` the number with `{"pendingCancellation": false}`.

**Errors:** `400` (the carrier release failed), `not_found` (404).

## Node.js SDK

```typescript
import Sendly from "@sendly/node";
const sendly = new Sendly(process.env.SENDLY_API_KEY!);

const { numbers } = await sendly.numbers.list();          // list owned numbers
const number = await sendly.numbers.get("num_abc123");    // one number (incl. isDefault)
await sendly.numbers.update("num_abc123", { isDefault: true });           // make default
await sendly.numbers.update("num_abc123", { pendingCancellation: false }); // keep (cancel release)

const result = await sendly.numbers.release("num_abc123"); // release / schedule release
if (result.scheduled) {
  console.log(`Releases at ${result.scheduledReleaseAt}`);
} else {
  console.log("Released");
}
```

## Buying a number

Buying is a separate, asynchronous flow (`POST /numbers/buy`) backed by number discovery (`GET /numbers/countries`, `GET /numbers/available`):

```typescript
const { countries } = await sendly.numbers.listCountries();
const { numbers: available } = await sendly.numbers.listAvailable({ country: "GB", type: "mobile" });
const buy = await sendly.numbers.buy({
  phoneNumber: available[0].phoneNumber,
  countryCode: available[0].country,
  phoneNumberType: available[0].numberType,
  monthlyCost: available[0].monthlyCost,
});
```

`buy()` returns `202` with a `status`: `provisioning` (poll `list()` until `active`), or `documents_required` / `payment_required` — which carry an `action` (a hosted-page URL + a short display `code` + an `actionCode`). Hand the user the URL and display `code`, wait for them to finish, then re-call `buy()` with the same body plus `actionCode` set to the action's `actionCode` (not its display `code`).

## Full reference

- Numbers docs: https://sendly.live/docs/numbers
</content>
