# Instructions: Upgrading a Sendly Workspace to a New Business Entity

You (the AI agent) help the user move their existing Sendly workspace onto a new legal entity — for example after they form an LLC, get a new EIN, or rebrand — without interrupting any messaging.

This document is the operational guide. The package-level `SKILL.md` mirrors the same content with a slightly more conversational framing; this file is the strict checklist.

## When to invoke

Trigger on any signal that the **legal entity behind the messaging is changing**:

- "I just formed an LLC", "I incorporated", "we LLC'd up"
- "We just got our EIN", "new EIN", "DBA is now under a corp"
- "Switching from sole-prop to LLC"
- "We're rebranding from X to Y"
- "Parent company changed", "we're now under entity Z"
- "The carrier rejected my verification — I want to redo it as the LLC, not me personally"

Do **not** invoke for:

- First-time verification of a brand-new workspace (use the `verifying-phones` skill or the carrier-verification UI).
- Enterprise/fleet operators provisioning many agent workspaces (those go through admin APIs, not these MCP tools).

## The zero-disruption guarantee (memorise this — say it back to the user)

> Your current toll-free number keeps sending normally the entire time. Sendly provisions a brand-new toll-free number under the new entity, submits it to the carrier (1–2 weeks), and on approval swaps the workspace to the new number atomically. No downtime, no resends, no gap.

This is the #1 reassurance users want.

## The 7 MCP tools

### 1. `preflight_business_upgrade` (always call first)

Read-only. Dry-runs the proposed new-entity details against carrier rules and returns:

- `issues` — list of problems with the proposed payload.
- `suggestions` / auto-fixes — usable corrections (e.g. "sole-prop + EIN mismatch → set `entityType: PRIVATE_PROFIT`").

**When to use**: every time the user provides (or revises) entity details, before any state-changing call.

**Inputs**: `businessName`, `brn`, `brnType`, `brnCountry`, `entityType` (required); messaging fields (`useCase`, `sampleMessages`, etc.) and address/contact fields (optional).

**Use the auto-fixes**: re-preflight after applying them, then confirm the corrected values with the user.

### 2. `get_business_upgrade_best_prefill`

Read-only. Returns the most-recent non-empty value across all of the caller's verified workspaces for messaging fields (`useCase`, `useCaseSummary`, `sampleMessages`, `optInWorkflow`, `privacyUrl`, `termsUrl`, etc.).

**When to use**: when the user has at least one other verified workspace and you want sane defaults for the messaging-content fields. Optional but recommended.

**Inputs**: none.

### 3. `start_business_upgrade` (the actual submit)

State-changing. Provisions a new toll-free number + messaging profile under the new entity, files carrier verification, and stores the EIN document (if provided) encrypted until approval.

**When to use**: after `preflight_business_upgrade` returns clean (or you've applied its auto-fixes and re-preflighted clean).

**Required inputs**: `workspaceId`, `businessName`, `brn`, `brnType`, `brnCountry`, `entityType`.

**Strongly recommended inputs**:

- `einDocBase64` — IRS CP-575 or 147C letter, base64-encoded PDF, ≤5MB. **Required-in-practice when the new entity was formed within the last ~6 months** (the EIN isn't yet in public registries, so the carrier rejects without proof).
- `einDocFilename` — optional, defaults to `ein-doc.pdf`.
- Address fields, contact fields, `useCase`, `sampleMessages`, `optInWorkflow` — all improve approval odds. Pull from `get_business_upgrade_best_prefill` when available.

**Idempotency**: server enforces one pending upgrade per workspace. A second `start_business_upgrade` call while one is in flight will return an error — use `get_business_upgrade_status` to check before retrying, or `cancel_business_upgrade` if you need to start over.

### 4. `get_business_upgrade_status` (poll on later turns)

Read-only. Returns either the pending upgrade row or `null`.

**Pending-row statuses**:

| Status | Meaning | Agent action |
|---|---|---|
| `pending` | Submitted, not yet picked up by carrier | "Still in review, nothing to do." |
| `processing` | Sitting in carrier queue | Same as `pending`. |
| `action_required` | Carrier wants more info (see `rejectionReason`) | Surface the reason, gather corrections, run `preflight` then `resubmit_business_upgrade`. |
| `rejected` | Hard rejection | Same as `action_required` — gather corrections, preflight, resubmit. |
| `verified` | Approved — workspace already swapped to new number | Congratulate, then offer the old-number disposition. |

**Inputs**: `workspaceId`.

**When to use**: any later turn where the user asks for status, or before deciding whether to start/resubmit/cancel.

### 5. `resubmit_business_upgrade`

State-changing. Same field shape as `start_business_upgrade`. Fields you omit keep their previously-submitted values.

**When to use**: status is `rejected` or `action_required`. Always run `preflight_business_upgrade` with the corrected fields first.

**Inputs**: same as `start_business_upgrade`. Can attach a fresh `einDocBase64` if the carrier asked for better documentation.

### 6. `cancel_business_upgrade`

State-changing but safe. Releases the reserved new TFN, deletes the new messaging profile, removes the stored EIN doc, and clears the pending row. **The current (old-entity) number is untouched and keeps sending.** Idempotent — safe to call even if there's nothing pending.

**When to use**: the user changes their mind, or you need to wipe the slate before a different upgrade attempt.

**Inputs**: `workspaceId`.

### 7. `set_business_upgrade_old_number_disposition`

State-changing. After `verified`, decide what to do with the now-superseded old number.

**Options**:

- `disposition: "moved"` + `targetWorkspaceId` — keep the old number active under another workspace owned by the same user (e.g. a personal/test workspace).
- `disposition: "released"` — return it to the carrier pool.

**When to use**: after status becomes `verified`. The workspace shows a banner until the user picks one. Idempotent.

## Required vs optional input fields (cheat sheet)

| Field | Required? | Notes |
|---|---|---|
| `workspaceId` | Yes | Found in dashboard URL or any object's `organizationId`. |
| `businessName` | Yes | Exact legal name as filed (e.g. "Acme Holdings LLC", not "Acme"). |
| `brn` | Yes | EIN as `XX-XXXXXXX` for US LLC/Corp. |
| `brnType` | Yes | `EIN \| SSN \| DUNS \| CRA \| VAT \| LEI \| OTHER`. |
| `brnCountry` | Yes | ISO-3166-1 alpha-2 (`US`, `CA`, ...). |
| `entityType` | Yes | LLCs and corps → `PRIVATE_PROFIT`. Solo + SSN only → `SOLE_PROPRIETOR`. |
| `einDocBase64` | Yes if <6mo old | IRS CP-575 / 147C, ≤5MB. |
| `address1` / `city` / `state` / `zip` / `addressCountry` | Recommended | Full state name, not 2-letter code. |
| `contactFirstName` / `contactLastName` / `contactEmail` / `contactPhone` | Recommended | Phone in E.164. |
| `useCase` / `useCaseSummary` / `sampleMessages` / `optInWorkflow` | Recommended | Pull from `get_business_upgrade_best_prefill`. |
| `privacyUrl` / `termsUrl` / `website` / `doingBusinessAs` / `monthlyVolume` / `additionalInformation` / `ageGatedContent` | Optional | All boost approval odds. |

## Common entity ↔ BRN combinations

| User says | `entityType` | `brnType` |
|---|---|---|
| "Just formed an LLC, here's the EIN" | `PRIVATE_PROFIT` | `EIN` |
| "Sole proprietor with an EIN" | `PRIVATE_PROFIT` | `EIN` |
| "Sole proprietor, no EIN, just my SSN" | `SOLE_PROPRIETOR` | `SSN` |
| "We're a 501(c)(3) / non-profit" | `NON_PROFIT` | `EIN` |
| Canadian incorporated company | `PRIVATE_PROFIT` | `CRA` |

If the user says "sole prop" but hands you an EIN, `preflight_business_upgrade` will return an auto-fix suggesting `PRIVATE_PROFIT` — apply it.

## The recommended end-to-end flow

1. **Detect intent.** Map the user's words to "they want to upgrade entity."
2. **Gather minimum inputs.** New legal name, BRN, entity type, formation date, country.
3. **Preflight.** `preflight_business_upgrade(...)`. Surface any issues / apply auto-fixes / re-preflight until clean.
4. **Fill the gaps (optional).** `get_business_upgrade_best_prefill()` → merge `useCase`, `sampleMessages`, `optInWorkflow`.
5. **EIN doc check.** If entity is <6 months old, ask for the CP-575 / 147C PDF and pass it as `einDocBase64`.
6. **Submit.** `start_business_upgrade(...)`. State the zero-disruption guarantee and the 1–2 week timeline.
7. **Status polling (later turns).** `get_business_upgrade_status(workspaceId)`. Branch on status.
8. **Recover from rejection.** `preflight_business_upgrade(...)` with corrections → `resubmit_business_upgrade(...)`.
9. **Approval.** Congratulate, then ask the user whether to move the old number to another workspace or release it. Call `set_business_upgrade_old_number_disposition(...)`.
10. **Cancel (if needed).** `cancel_business_upgrade(workspaceId)` — never destructive to the current number.

## Communication tips

- Lead with the zero-disruption guarantee; users are anxious about send interruption.
- State the 1–2 week timeline upfront — don't let them think it's same-day.
- For entities <6 months old, ask for the EIN letter proactively. Don't wait for a rejection.
- After approval, don't forget the old-number disposition — leave the workspace in a clean state.
- Never claim something will work; describe what the tool reports back.

## See also

- `examples/happy-path.md` — annotated walk-through of a successful upgrade.
- `SKILL.md` — the canonical skill manifest with YAML frontmatter.
