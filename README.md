# @sendly/skills

Agent Skills for Sendly — give AI agents the knowledge to send SMS, verify phone numbers, and follow compliance best practices.

## Install

```bash
npx skills add SendlyHQ/sendly-skills
```

Or install individual skills:

```bash
npx skills add SendlyHQ/sendly-skills/sending-sms
npx skills add SendlyHQ/sendly-skills/verifying-phones
npx skills add SendlyHQ/sendly-skills/sms-best-practices
```

## Skills

| Skill | Description |
|---|---|
| **sending-sms** | Send SMS via the Sendly API — single messages, batch, scheduling, conversations |
| **verifying-phones** | OTP phone verification — send codes, check codes, hosted sessions |
| **sms-best-practices** | SMS compliance — TCPA, quiet hours, opt-outs, SHAFT filtering, E.164 formatting |

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

## Requirements

- A Sendly API key (dashboard → [API Keys](https://sendly.live/api-keys))
- Use `sk_test_*` keys for sandbox mode (no real SMS sent)
