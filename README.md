# @sendly/skills

Agent Skills for Sendly — give AI agents the knowledge to send SMS and WhatsApp messages, verify phone numbers, manage numbers and API keys, and follow compliance best practices.

## Install

```bash
npx skills add SendlyHQ/sendly-skills
```

Or install individual skills:

```bash
npx skills add SendlyHQ/sendly-skills/sending-sms
npx skills add SendlyHQ/sendly-skills/sending-whatsapp
npx skills add SendlyHQ/sendly-skills/verifying-phones
npx skills add SendlyHQ/sendly-skills/sms-best-practices
npx skills add SendlyHQ/sendly-skills/shortening-links
npx skills add SendlyHQ/sendly-skills/managing-numbers
npx skills add SendlyHQ/sendly-skills/rotating-api-keys
npx skills add SendlyHQ/sendly-skills/upgrading-business-entity
```

## Skills

| Skill | Description |
|---|---|
| **sending-sms** | Send SMS via the Sendly API — single messages, batch, group MMS, scheduling, AI message enhancement, conversations |
| **sending-whatsapp** | WhatsApp messaging — connect a number (a human finishes the Facebook step), 24-hour window vs approved templates, sender and window checks before sending (rollout-flagged) |
| **verifying-phones** | OTP phone verification — send codes, check codes, hosted sessions |
| **sms-best-practices** | SMS compliance — TCPA, quiet hours, opt-outs, SHAFT filtering, E.164 formatting |
| **shortening-links** | Branded URL shortening — mint short links, track clicks, per-link kill switch (rollout-flagged) |
| **managing-numbers** | Manage owned phone numbers — list, inspect, set the default sender, cancel a scheduled release, release |
| **rotating-api-keys** | Rotate an API key with a grace period so callers roll over with zero downtime |
| **upgrading-business-entity** | Move a workspace to a new legal entity (LLC formation, new EIN, rebrand) with zero send disruption |

## Compatibility

Works with any agent that supports SKILL.md files:

- Claude Code
- Cursor
- Codex
- VS Code Copilot
- Windsurf
- Gemini CLI
- OpenClaw
- OpenCode

## Beyond these skills

These skills wrap the most common flows, but the full Sendly API surface is much broader — messages (single, batch, group MMS, scheduled), AI message enhancement, phone verification, branded URL shortening, numbers and porting, conversations, contacts, campaigns, templates, and webhooks. Reach the whole toolset through:

- **Node.js SDK** — [`@sendly/node`](https://www.npmjs.com/package/@sendly/node)
- **Python SDK** — [`sendly`](https://pypi.org/project/sendly/)
- **MCP server** — [`@sendly/mcp`](https://www.npmjs.com/package/@sendly/mcp) for agents that speak the Model Context Protocol
- **CLI** and **REST API** — see the [docs](https://sendly.live/docs)

## Requirements

- A Sendly API key (dashboard → [API Keys](https://sendly.live/api-keys))
- Use `sk_test_*` keys for sandbox mode (no real SMS sent)
