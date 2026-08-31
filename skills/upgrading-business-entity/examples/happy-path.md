# Happy-path example: user upgrades to a freshly-formed LLC

A complete user ↔ agent interaction across three turns: intent detection, submission, and a status check on a later day.

The user has an existing Sendly workspace (`org_h7q2p9`) currently verified under their personal name + SSN as a sole proprietor. They just formed Acme Holdings LLC and got an EIN two weeks ago.

---

## Turn 1 — intent + submission

**User:**
> I just formed an LLC for my SMS work — Acme Holdings LLC, EIN 87-1234567. I want my Sendly workspace to be under the LLC now, not under me personally. Will my campaigns keep going while you do this?

**Agent (internal reasoning):**

The user is asking for an entity upgrade. They've given the legal name and EIN. They formed it recently (likely <6 months ago given they "just" did it) so I should ask for the IRS letter. First, preflight with what I have to surface any obvious issues.

**Agent calls `preflight_business_upgrade`:**

```json
{
  "businessName": "Acme Holdings LLC",
  "brn": "87-1234567",
  "brnType": "EIN",
  "brnCountry": "US",
  "entityType": "PRIVATE_PROFIT"
}
```

**Response:**

```json
{ "issues": [], "suggestions": [], "ok": true }
```

Preflight is clean. Now pull best-of defaults from the user's other workspaces (if any).

**Agent calls `get_business_upgrade_best_prefill`:**

```json
{}
```

**Response:**

```json
{
  "useCase": "Appointment reminders for an auto-detailing service",
  "sampleMessages": "Hi {{first_name}}, reminder: your detailing appointment is tomorrow at {{time}}. Reply CONFIRM or CANCEL.",
  "optInWorkflow": "Customers opt in at booking via a checkbox on acme-detailing.com/book",
  "privacyUrl": "https://acme-detailing.com/privacy"
}
```

**Agent (to user):**

> Got it — I can move your workspace onto Acme Holdings LLC. Two important things first:
>
> 1. **Your current toll-free number keeps sending the whole time.** Sendly will provision a brand-new toll-free number under Acme Holdings LLC, submit it to the carrier (1–2 weeks), and on approval swap your workspace to the new number atomically. No downtime, no resends, no gap in service.
> 2. **EIN documentation:** because Acme Holdings LLC was formed recently, the EIN isn't in public business registries yet, so carriers will reject the application unless we include your IRS letter. Can you attach your CP-575 (the confirmation letter the IRS mailed you) or a 147C as a PDF? Without it, this submission will likely be rejected.
>
> I'll pre-fill the use case, sample messages, opt-in workflow, and privacy URL from your existing verified workspace — let me know if any of those need to change for the LLC.

**User:**
> Here's the CP-575 PDF. Everything else is correct, the LLC is doing the same auto-detailing reminders.

**Agent (internal):**

The user attached the EIN doc. Base64-encode it and submit.

**Agent calls `start_business_upgrade`:**

```json
{
  "workspaceId": "org_h7q2p9",
  "businessName": "Acme Holdings LLC",
  "brn": "87-1234567",
  "brnType": "EIN",
  "brnCountry": "US",
  "entityType": "PRIVATE_PROFIT",
  "useCase": "Appointment reminders for an auto-detailing service",
  "sampleMessages": "Hi {{first_name}}, reminder: your detailing appointment is tomorrow at {{time}}. Reply CONFIRM or CANCEL.",
  "optInWorkflow": "Customers opt in at booking via a checkbox on acme-detailing.com/book",
  "privacyUrl": "https://acme-detailing.com/privacy",
  "einDocBase64": "JVBERi0xLjQKJ...<truncated>",
  "einDocFilename": "acme-cp575.pdf"
}
```

**Response:**

```json
{
  "id": "upg_4kf8x2",
  "workspaceId": "org_h7q2p9",
  "status": "pending",
  "targetTollFreeNumber": "+18334567890",
  "createdAt": "2026-05-25T14:02:11Z"
}
```

**Agent (to user):**

> Submitted. Here's where things stand:
>
> - **Current number** (`+18331112222`): still sending normally — no change.
> - **New number** (`+18334567890`): reserved under Acme Holdings LLC, now in carrier review.
> - **Expected timeline:** 7–10 business days, occasionally up to 2 weeks.
> - **EIN letter:** attached. Stored encrypted, auto-deleted after approval.
>
> I'll let you know as soon as the carrier responds. If you need to check in sooner, just ask.

---

## Turn 2 — user checks back 5 days later

**User:**
> Any update on the LLC upgrade?

**Agent calls `get_business_upgrade_status`:**

```json
{ "workspaceId": "org_h7q2p9" }
```

**Response:**

```json
{
  "id": "upg_4kf8x2",
  "status": "processing",
  "targetTollFreeNumber": "+18334567890",
  "rejectionReason": null,
  "submittedAt": "2026-05-25T14:02:14Z"
}
```

**Agent (to user):**

> Still in carrier review — status is `processing`, which means it's in the carrier's queue and they haven't reviewed it yet. No action needed from you. Your current number is still sending normally. I'd expect a response in the next 3–7 business days.

---

## Turn 3 — approval lands, agent picks up the disposition

**User (a week later):**
> Did the LLC thing go through?

**Agent calls `get_business_upgrade_status`:**

```json
{ "workspaceId": "org_h7q2p9" }
```

**Response:**

```json
{
  "id": "upg_4kf8x2",
  "status": "verified",
  "targetTollFreeNumber": "+18334567890",
  "rejectionReason": null,
  "verifiedAt": "2026-06-03T09:14:22Z",
  "oldTollFreeNumber": "+18331112222",
  "oldNumberDisposition": null
}
```

**Agent (to user):**

> It went through! Your workspace is now under Acme Holdings LLC and active on the new toll-free number `+18334567890`. The swap happened atomically — anything sent in the last few hours is already going out under the new number.
>
> One last question: what would you like to do with the old number (`+18331112222`)?
>
> - **Move it** to another workspace you own (useful if you still want it active somewhere, e.g. a personal/test workspace).
> - **Release it** back to the carrier pool (clean break, simplest).

**User:**
> Release it. I don't need that number anymore.

**Agent calls `set_business_upgrade_old_number_disposition`:**

```json
{
  "workspaceId": "org_h7q2p9",
  "disposition": "released"
}
```

**Response:**

```json
{ "ok": true, "disposition": "released" }
```

**Agent (to user):**

> Done. The old number has been released to the carrier pool. Your workspace is now fully under Acme Holdings LLC on `+18334567890`. Anything else you'd like to do?

---

## Variations not shown here

- **Rejection path.** If status had returned `rejected` or `action_required`, the agent would surface the `rejectionReason`, gather corrections, re-run `preflight_business_upgrade`, then call `resubmit_business_upgrade` with the corrected fields (and optionally a fresh `einDocBase64`).
- **Sole-prop with EIN mismatch.** If the user had said "I'm a sole proprietor with an EIN" and given `entityType: SOLE_PROPRIETOR`, the preflight call would have returned an auto-fix recommending `entityType: PRIVATE_PROFIT`. The agent would apply it and re-preflight before proceeding.
- **Cancel.** If the user had changed their mind between turns 1 and 3, the agent would call `cancel_business_upgrade({ workspaceId })`. The current number would have continued sending unaffected.
- **Move instead of release.** If the user wanted to keep `+18331112222` active on a personal/test workspace, the agent would call `set_business_upgrade_old_number_disposition({ workspaceId, disposition: "moved", targetWorkspaceId: "<that-other-org>" })`.
