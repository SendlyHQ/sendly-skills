# upgrading-business-entity

An agent skill for upgrading a Sendly workspace to a new business entity (sole-prop → LLC, EIN change, rebrand, restructure) without any send disruption.

## When an AI agent should invoke this skill

Invoke when the user signals that the legal entity behind their messaging has changed or is about to change. Triggers include:

- "I just formed an LLC" / "I incorporated"
- "We just got a new EIN"
- "I'm switching from sole-prop to an LLC"
- "We're rebranding from X to Y"
- "Our parent company changed"
- "I'm now operating under a different business name"
- "The carrier rejected my verification — I want to re-submit under the proper LLC"

This skill is for **direct individual Sendly users** managing their own workspace. Enterprise/fleet operators (e.g. InsurDial) provision agent workspaces via admin endpoints, not the customer-facing MCP tools taught here.

## What the agent gets

The skill teaches the agent to:

1. Recognise the entity-upgrade intent.
2. Use 7 MCP tools (`preflight_business_upgrade`, `get_business_upgrade_best_prefill`, `start_business_upgrade`, `get_business_upgrade_status`, `resubmit_business_upgrade`, `cancel_business_upgrade`, `set_business_upgrade_old_number_disposition`).
3. Communicate the zero-disruption guarantee — the current toll-free number keeps sending throughout the 1–2 week carrier review; the new number takes over atomically on approval.
4. Ask for the EIN documentation (IRS CP-575 / 147C) when the new entity was formed within the last ~6 months.
5. Handle resubmission after rejection and the old-number disposition once the new entity is verified.

## Files

- `SKILL.md` — canonical skill definition with YAML frontmatter (used by the `@sendly/skills` package loader).
- `instructions.md` — AI-facing prompt detailing each MCP tool, when to call it, and the recommended end-to-end flow.
- `examples/happy-path.md` — full transcript of a successful upgrade from intent detection through status check.

## Requires

- A Sendly API key (`SENDLY_API_KEY`) for the workspace being upgraded.
- An MCP-capable agent runtime with the Sendly MCP server installed (`@sendly/mcp`).
