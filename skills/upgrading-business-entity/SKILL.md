---
name: upgrading-business-entity
description: Upgrades a Sendly workspace to a new business entity (e.g. sole-prop → LLC, EIN change, rebrand) with zero send disruption. The current toll-free number keeps sending throughout the 1–2 week carrier review; a newly provisioned number takes over atomically on approval. Applies when the user mentions forming an LLC, getting a new EIN, changing legal entity, rebranding, or restructuring their business.
---

# Upgrading a Sendly Workspace to a New Business Entity

## When to use this skill

Invoke this skill whenever the user signals that the legal entity behind their messaging has changed (or is about to change). Common triggers:

- "I just formed an LLC" / "I incorporated" / "we just got our EIN"
- "We're rebranding from X to Y"
- "I'm switching from sole-prop to an LLC"
- "Our parent company changed" / "we're now under a new entity"
- "The carrier rejected my verification because of EIN/SSN mismatch and I want to redo it under the proper LLC"

The skill exists because the carrier network treats the business name + BRN (EIN/SSN/etc.) as the identity of the sender. Once a workspace is verified under entity A, you cannot edit those fields in place — you must provision a fresh toll-free number under entity B and have it re-verified, then swap the workspace's active number when approval lands.

## The zero-disruption guarantee

This is the most important thing to communicate to the user:

> "Your current toll-free number keeps sending normally throughout the 1–2 week review. When the new number is approved, we swap them atomically — no downtime, no gap, no resends needed."

Behind the scenes Sendly:

1. Provisions a fresh toll-free number + messaging profile under the new entity.
2. Submits the new entity for carrier review (1–2 weeks).
3. Keeps the original (old-entity) number sending the whole time.
4. On `verified`, atomically points the workspace at the new number.
5. Asks the user what to do with the old number (`moved` to another workspace, or `released` to the carrier).

## Timeline

- **Preflight + submission**: seconds.
- **Carrier review**: typically 7–10 business days, can be up to ~2 weeks.
- **Swap**: instant on approval.
- **Old-number disposition**: anytime after approval (the workspace will surface a banner until the user picks `moved` or `released`).

## The 7 MCP tools

| Tool | When to use |
|---|---|
| `preflight_business_upgrade` | Always call first. Dry-runs the new entity details against carrier rules and returns issues + auto-fix suggestions (e.g. "sole-prop + EIN mismatch — change entity to PRIVATE_PROFIT"). Read-only. |
| `get_business_upgrade_best_prefill` | Optional. Pulls best-of (most-recent non-empty) values across the user's other verified workspaces for fields like `useCase`, `sampleMessages`, `optInWorkflow`. Useful when the current workspace's record is sparse. |
| `start_business_upgrade` | Actually submits the upgrade. Provisions new TFN + messaging profile, files carrier verification, stores EIN doc. Idempotent guard server-side (one pending upgrade per workspace). |
| `get_business_upgrade_status` | Poll on subsequent turns. Returns the pending row with `status` ∈ {`pending`, `processing`, `action_required`, `rejected`, `verified`} plus the target TFN and any rejection reason. Returns `null` if no upgrade is in flight. |
| `resubmit_business_upgrade` | Use when status is `rejected` or `action_required`. Same field shape as `start_business_upgrade`; omitted fields keep their previously-submitted values. Pair with a fresh `preflight_business_upgrade` first. |
| `cancel_business_upgrade` | Use when the user changes their mind. Releases the reserved TFN, deletes the new profile, removes the stored EIN doc, clears the pending row. Idempotent. The current (old-entity) number is untouched. |
| `set_business_upgrade_old_number_disposition` | After `verified`. `disposition: "moved"` (requires `targetWorkspaceId`) keeps the old number alive under another workspace owned by the same user; `disposition: "released"` returns it to the carrier pool. Idempotent. |

## The EIN document requirement (entities <6 months old)

If the new entity was formed within the last ~6 months, the EIN won't be visible in public business registries yet, and carriers will reject for "BRN cannot be verified."

**Always ask the user**: "When did you form the new entity? If it's been less than 6 months, please attach the IRS CP-575 or 147C letter as a PDF — without it the carrier may reject the submission."

The doc is passed to `start_business_upgrade` as `einDocBase64` (base64-encoded PDF, ≤5MB) plus optional `einDocFilename`. It is stored encrypted in R2 and auto-deleted after carrier approval.

For older entities (>6 months) the EIN doc is optional but never hurts.

## Required fields for `start_business_upgrade`

Beyond `workspaceId`:

- `businessName` — exact legal name as filed (e.g. "Acme Holdings LLC", not "Acme")
- `brn` — the registration number (EIN as `XX-XXXXXXX` for US LLC/Corp)
- `brnType` — one of `EIN | SSN | DUNS | CRA | VAT | LEI | OTHER`
- `brnCountry` — ISO 3166-1 alpha-2 (e.g. `US`)
- `entityType` — one of `SOLE_PROPRIETOR | PRIVATE_PROFIT | PUBLIC_PROFIT | NON_PROFIT | GOVERNMENT`
  - LLCs and standard corporations → `PRIVATE_PROFIT`
  - Solo founder with just an SSN → `SOLE_PROPRIETOR`

All other fields (address, contact, website, use case, sample messages, opt-in workflow, etc.) are optional but improve approval odds. Use `get_business_upgrade_best_prefill` to pull strong defaults.

## Common entity-type ↔ BRN combinations (let `preflight` catch mismatches)

| User says | `entityType` | `brnType` |
|---|---|---|
| "Just formed an LLC, here's the EIN" | `PRIVATE_PROFIT` | `EIN` |
| "I'm a sole proprietor with an EIN" | `PRIVATE_PROFIT` | `EIN` |
| "I'm a sole proprietor, no EIN, just my SSN" | `SOLE_PROPRIETOR` | `SSN` |
| "We're a non-profit / 501(c)(3)" | `NON_PROFIT` | `EIN` |
| Canadian corporation | `PRIVATE_PROFIT` | `CRA` |

If the user says "sole prop" but provides an EIN, `preflight` will return an auto-fix suggesting `entityType: PRIVATE_PROFIT`. Apply the suggestion and re-preflight before starting.

## Recommended flow

1. **Detect intent.** User mentions LLC / new EIN / rebrand / entity change.
2. **Gather the minimum.** Ask: new legal name, BRN (EIN), entity type, formation date.
3. **Preflight.** Call `preflight_business_upgrade`. If issues are returned, apply the auto-fixes and ask the user to confirm the corrected values.
4. **Optional: fill gaps.** If they have other verified workspaces, call `get_business_upgrade_best_prefill` and merge in `useCase`, `sampleMessages`, `optInWorkflow`.
5. **EIN doc check.** If the entity is <6 months old, ask for the CP-575 / 147C as a PDF and pass it as `einDocBase64`.
6. **Submit.** Call `start_business_upgrade`. Reassure: "Your old number keeps sending; new one is in review for ~1–2 weeks."
7. **Next turn(s).** Call `get_business_upgrade_status` when the user asks for an update.
   - `pending` / `processing` → "Still in carrier review, nothing to do."
   - `action_required` → carrier needs more info; surface the rejection reason and prep a `resubmit_business_upgrade` with corrected fields.
   - `rejected` → same as above but with a harder fix; preflight + resubmit.
   - `verified` → congratulate, then ask about the old number and call `set_business_upgrade_old_number_disposition`.
8. **Mid-flight cancel.** If the user changes their mind, call `cancel_business_upgrade`. No effect on the current number.

## Examples

See `examples/happy-path.md` for a complete user-to-agent interaction.

## Scope: who should use this

This skill is for **direct individual Sendly users** managing their own workspace (e.g. the user formed an LLC and wants their existing Sendly workspace upgraded). It is **not** the right path for enterprise/fleet operators (e.g. InsurDial managing hundreds of agent workspaces) — those use the admin endpoints, not the customer-facing MCP tools.

## Full reference

- Entity upgrade overview: https://sendly.live/docs/entity-upgrade
- Toll-free verification: https://sendly.live/docs/verify-toll-free
- BRN-rejection recovery: https://sendly.live/docs/how-to/fix-rejected-verification
