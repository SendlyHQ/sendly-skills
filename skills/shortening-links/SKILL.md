---
name: shortening-links
description: Mints branded, owned-domain short links via the Sendly Links API and tracks click analytics. Covers creating short links, listing them with click counts, and disabling a link (per-link kill switch). Applies when shortening URLs for SMS to improve deliverability and measure clicks.
---

# Branded URL Shortening with Sendly

Branded, owned-domain short links improve SMS deliverability — carriers filter public shorteners — and give you click analytics per link.

> **Not yet generally available.** URL shortening is gated behind the `url_shortener` rollout flag. Until it is enabled for your account, every API-key call returns `not_found` (404) — the feature reads as absent. Ask Sendly to enable it before relying on these endpoints.

## Quick start

```typescript
import Sendly from "@sendly/node";

const sendly = new Sendly(process.env.SENDLY_API_KEY!);

const link = await sendly.links.create({
  url: "https://example.com/spring-sale?utm_source=sms",
});
console.log(link.shortUrl); // https://sendly.live/l/Ab3xY7
```

## REST API

**Base URL:** `https://sendly.live/api/v1`

### Create a short link

```bash
curl -X POST https://sendly.live/api/v1/links \
  -H "Authorization: Bearer $SENDLY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"url": "https://example.com/welcome"}'
```

**Required:** `url` (http/https only)

**Response:**
```json
{
  "code": "Ab3xY7",
  "shortUrl": "https://sendly.live/l/Ab3xY7",
  "destinationUrl": "https://example.com/welcome"
}
```

Uses your workspace's brand slug when one is configured (e.g. `https://sendly.live/l/acme/Ab3xY7`).

**Errors:** `invalid_url` (400, not an http/https URL), `workspace_required` (400, the key has no workspace), `not_found` (404, `url_shortener` flag off).

### List short links

```bash
curl "https://sendly.live/api/v1/links?limit=50" \
  -H "Authorization: Bearer $SENDLY_API_KEY"
```

Supports `limit` (default 50, max 200) and `offset` (default 0).

**Response:**
```json
{
  "links": [
    {
      "code": "Ab3xY7",
      "shortUrl": "https://sendly.live/l/Ab3xY7",
      "destinationUrl": "https://example.com/welcome",
      "brandSlug": "acme",
      "clickCount": 42,
      "disabled": false,
      "lastCountry": "US",
      "lastClickedAt": "2026-04-01T10:00:00Z",
      "createdAt": "2026-03-31T10:00:00Z",
      "spark": [0, 1, 3, 5, 8, 2, 0, 4, 6, 1, 0, 0, 2, 10]
    }
  ],
  "total": 1
}
```

`spark` is a 14-day daily click histogram (oldest first) for quick trend sparklines.

### Disable or re-enable a link

A disabled link's redirect returns 404 until you re-enable it — a per-link kill switch.

```bash
curl -X PATCH https://sendly.live/api/v1/links/Ab3xY7 \
  -H "Authorization: Bearer $SENDLY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"disabled": true}'
```

**Required:** `disabled` (boolean)

**Response:**
```json
{ "code": "Ab3xY7", "disabled": true }
```

**Errors:** `disabled_boolean_required` (400, body missing the `disabled` boolean), `link_not_found` (404, no such code in your workspace), `not_found` (404, `url_shortener` flag off).

### Public redirect

The short link itself is a public URL — no auth needed. Visiting it 302-redirects to the destination (or 404s if the code is unknown or disabled):

```
GET https://sendly.live/l/Ab3xY7
GET https://sendly.live/l/acme/Ab3xY7
```

## Node.js SDK

```typescript
import Sendly from "@sendly/node";
const sendly = new Sendly(process.env.SENDLY_API_KEY!);

const link = await sendly.links.create({ url: "https://example.com/welcome" });
const { links, total } = await sendly.links.list({ limit: 50 });
await sendly.links.disable(link.code);           // kill a link
await sendly.links.enable(link.code);            // re-enable it
await sendly.links.setDisabled(link.code, true); // set state explicitly
```

## Error handling

| Error | Status | Meaning | Action |
|---|---|---|---|
| `not_found` | 404 | `url_shortener` flag off for your account | Ask Sendly to enable it |
| `invalid_url` | 400 | Not an http/https URL | Pass a full `https://…` URL |
| `workspace_required` | 400 | The API key has no workspace | Use a workspace-scoped key |
| `link_not_found` | 404 | No such code in your workspace (PATCH) | Check the `code` |
| `disabled_boolean_required` | 400 | PATCH body missing `disabled` boolean | Send `{"disabled": true}` or `{"disabled": false}` |

## Full reference

- Links docs: https://sendly.live/docs/links
</content>
</invoke>
