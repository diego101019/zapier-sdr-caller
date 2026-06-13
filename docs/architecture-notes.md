# Architecture Notes - Working Configuration

## Credentials & IDs

| Key | Value |
|-----|-------|
| Zap 2 Catch Hook URL | `<your-zapier-webhook-url>` |
| Vapi Assistant ID | <your-vapi-assistant-id> |
| Vapi Phone Number ID | <your-vapi-phone-number-id> |
| HubSpot Trigger Deal Stage ID | <your-hubspot-deal-stage-id> |
| HubSpot Portal ID | <your-hubspot-portal-id> |

Register the Zap 2 URL in Vapi: Dashboard → Assistant → Server URL.

---

## Zap Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│  ZAP 1: "Trigger Vapi Call on Deal Stage Change"            │
│                                                             │
│  TRIGGER: HubSpot - Updated Deal (polling)                  │
│  FILTER:  dealstage = <your-hubspot-deal-stage-id>                            │
│  ACTION 1: HubSpot - Find Associated Contact                │
│  ACTION 2: Formatter - Format phone to E.164                │
│  ACTION 3: Webhooks - POST to Vapi /call/phone              │
└─────────────────────────────────────────────────────────────┘
                          │
                          │  Vapi dials prospect
                          │  Call ends
                          ▼
                   Vapi fires webhook
                          │
┌─────────────────────────────────────────────────────────────┐
│  ZAP 2: "Log Vapi Call Result to HubSpot + Slack"           │
│                                                             │
│  TRIGGER: Webhooks - Catch Hook                             │
│  FILTER:  message.type = end-of-call-report                 │
│  ACTION 1: Code by Zapier - extract deal ID                 │
│  ACTION 2: AI by Zapier - generate structured call note     │
│  ACTION 3: HubSpot - Create Note Engagement on Deal         │
│  FILTER:  successEvaluation = true                          │
│  ACTION 4: HubSpot - Get Deal (for deal name)               │
│  ACTION 5: Slack - Send Channel Message to AE channel       │
└─────────────────────────────────────────────────────────────┘
```

---

## Zap 1 - Detail

### Trigger: HubSpot "Updated Deal" (polling)

- Polls HubSpot every 5–15 minutes depending on Zapier plan - not instant
- Add a Filter step immediately after: only continue when `dealstage` equals `<your-hubspot-deal-stage-id>`
- **Known limitation:** For a live demo, the delay may be noticeable. To make it instant, replace this trigger with a HubSpot Workflow that POSTs to a Zapier Catch Hook - see Gotcha #1.

### Action 1: Find Associated Contact

- App: **HubSpot**, Action: **Find Associated Contact**
- Input: Deal ID from trigger
- Pulls `firstname`, `lastname`, `email`, `phone`, `company`, `hs_lead_source`

### Action 2: Format Phone Number

- App: **Formatter by Zapier**, Transform: **Numbers**, Action: **Format Phone Number**
- Input: `phone` from contact lookup
- Output format: E.164 (`+15551234567`)
- Vapi rejects non-E.164 numbers - this step is required

### Action 3: POST to Vapi `/call/phone`

- App: **Webhooks by Zapier**, Action: **Custom Request (POST)**
- Endpoint: `https://api.vapi.ai/call/phone`
- Headers:
  - `Authorization: Bearer <your_vapi_private_key>`
  - `Content-Type: application/json`
- Body (JSON):
```json
{
  "assistantId": "<your-vapi-assistant-id>",
  "phoneNumberId": "<your-vapi-phone-number-id>",
  "customer": {
    "number": "{{formatted_phone}}",
    "name": "{{firstname}} {{lastname}}",
    "email": "{{email}}"
  },
  "assistantOverrides": {
    "variableValues": {
      "company": "{{company}}",
      "leadSource": "{{hs_lead_source}}"
    }
  },
  "metadata": {
    "hubspot_deal_id": "{{deal_id}}",
    "hubspot_contact_id": "{{contact_id}}"
  }
}
```

- `customer.email` feeds directly into `{{customer.email}}` in the Vapi system prompt
- `assistantOverrides.variableValues` feeds `{{company}}` and `{{leadSource}}` into the system prompt
- `metadata` passes CRM IDs through the call so Zap 2 can write back to the right records

---

## Zap 2 - Detail

### Trigger: Webhooks by Zapier - Catch Hook

- URL registered in Vapi at the **assistant level** (assistant-level overrides phone number level)
- Vapi fires this URL for every call event - filter immediately to avoid spurious runs

### Filter: message.type = end-of-call-report

- Condition: the field containing `end-of-call-report` (search "type" in field list - it appears as **"Message Type"**) exactly matches `end-of-call-report`
- **Gotcha:** Zapier's field list shows flattened names. Do not use a generic "Field 1" placeholder - find the field whose sample value IS `end-of-call-report`.

### Action 1: Code by Zapier - Extract Deal ID

- App: **Code by Zapier**, Action: **Run Javascript**
- Input: `compound_id` mapped to `message > call > metadata > hubspot_deal_id`
- Code:
```javascript
const parts = inputData.compound_id.split('-');
output = [{deal_id: parts[0]}];
```
- **Why this exists:** Zapier's HubSpot "Updated Deal" trigger returns a compound string `dealId-stageId-timestampMs` (e.g. `58208201305-<your-hubspot-deal-stage-id>-1780009240278`). The real deal ID is the first segment. The last segment is a Unix timestamp in milliseconds - NOT the deal ID.
- Output: `deal_id` = plain numeric deal ID (e.g. `58208201305`)

### Action 2: AI by Zapier - Generate Structured Call Note

- App: **AI by Zapier**, Action: **Analyze & Transform**
- Input: `message > transcript` from the webhook trigger (NOT `message > summary`)
- Prompt:
> "You are a sales ops assistant writing a HubSpot call note for an Account Executive. Based on the transcript below, write a clean, professional summary using these exact headings. Do NOT quote the prospect verbatim - paraphrase their meaning in clear, professional language:
>
> **Pain Point:** [What problem or frustration the prospect described. 1–2 clean sentences.]
>
> **Objection / Hesitation:** [Any concern or pushback they raised. Write "None raised" if there wasn't one.]
>
> **Qualification Signal:** [What confirmed they are a fit - decision-making authority, budget, relevant business type, expressed interest.]
>
> **Recommended Next Step:** [Specific action for the AE - who should reach out, what to discuss, what to offer. Do not include contact details such as email addresses or phone numbers - the AE will find those in HubSpot.]
>
> **Disposition:** Qualified
>
> Transcript: [mapped field]"

### Action 3: HubSpot - Create Note Engagement

- App: **HubSpot**, Action: **Create Engagement**
- Type: **NOTE** (not CALL)
- Note body: full output from AI by Zapier step
- dealIds: set to **Custom mode** (click `[...]` → Custom) and map `deal_id` from the Code step
  - **Gotcha:** In default line-item mode, Zapier passes a stray index `1` alongside the deal ID, causing HubSpot to reject the association. Custom mode passes a single plain value.
- haltIfAssociationsErrors: **true** (surfaces real errors instead of silently dropping the deal link)

### Filter: successEvaluation = true

- Condition: `message > analysis > successEvaluation` (text) exactly matches `true`
- Unqualified calls get their HubSpot note logged but do not trigger an AE Slack alert

### Action 4: HubSpot - Get Deal

- App: **HubSpot**, Action: **Find Deal**
- Deal ID: `deal_id` from Code by Zapier step
- Used to retrieve the deal name for the Slack message

### Action 5: Slack - Send Channel Message

- App: **Slack**, Action: **Send Channel Message**
- Channel: `#ae-leads`
- Message:
```
🤖 AI SDR Qualified Lead

*Contact:* [message > call > customer > name]
*Deal:* [deal name from Get Deal step]

*Pain Point:* [AI output - Pain Point section]
*Objection:* [AI output - Objection / Hesitation section]
*Qualification Signal:* [AI output - Qualification Signal section]
*Next Step for AE:* [AI output - Recommended Next Step section]

HubSpot Deal: https://app.hubspot.com/contacts/<your-hubspot-portal-id>/deal/[deal_id from Code step]

React to claim this lead or reply in thread.
```

---

## Vapi Assistant System Prompt Variables

The system prompt references these variables - all must be passed from Zap 1:

| Variable | Source | Passed via |
|----------|--------|-----------|
| `{{customer.name}}` | HubSpot contact firstname + lastname | `customer.name` in POST body |
| `{{customer.email}}` | HubSpot contact email | `customer.email` in POST body |
| `{{company}}` | HubSpot contact company field | `assistantOverrides.variableValues.company` |
| `{{leadSource}}` | HubSpot contact hs_lead_source | `assistantOverrides.variableValues.leadSource` |

---

## Gotchas & Lessons Learned

### 1. Polling delay on HubSpot trigger
Zapier's HubSpot "Updated Deal" trigger polls every 5–15 minutes - not instant. For a live demo this is noticeable. **Fix when needed:** replace with a HubSpot Workflow that POSTs to a Zapier Catch Hook on stage change. This gives 1–3 second triggering. Not implemented yet.

### 2. HubSpot compound deal ID
The `hubspot_deal_id` value coming through Zapier is `dealId-stageId-timestampMs`, not a plain deal ID. The first segment is the real deal ID. Solved with a Code by Zapier step using `split('-')[0]`.

### 3. Zapier filter field mapping
When setting up Filter steps, Zapier may default to a generic "Field 1" placeholder with no data. Always find the field by its sample value in the dropdown - for `end-of-call-report`, search "type" and look for the field whose sample value matches.

### 4. HubSpot dealIds in line-item mode
The HubSpot Create Engagement `dealIds` field defaults to line-item mode, which injects a stray index `1` alongside the deal ID. HubSpot receives two values and fails the association. Switch to Custom mode to pass a single plain value.

### 5. AI by Zapier input must be transcript, not summary
Feeding `message.summary` (Vapi's pre-generated summary) into AI by Zapier produces vague, repetitive output - the AI just echoes the summary back. Always use `message.transcript` (the full raw conversation) as the input.

### 6. Vapi filter field - multiple event types
Vapi fires multiple event types to the same webhook URL. Without the `end-of-call-report` filter, every transcript chunk and status event triggers Zap 2. This wastes tasks and creates duplicate notes.

### 7. Email and company not passed to Vapi by default
The Vapi POST body initially only included `number` and `name`. The AI assistant was hallucinating email addresses because `{{customer.email}}` was blank. Always pass `email` in the customer object and custom variables via `assistantOverrides.variableValues`.

### 8. Task usage per call
- Zap 1: ~4 tasks (Filter + Contact lookup + Formatter + Webhook POST)
- Zap 2: ~7 tasks qualified (Filter + Code + AI + HubSpot Note + Filter + Get Deal + Slack)
- Zap 2: ~4 tasks unqualified (Filter + Code + AI + HubSpot Note - Slack skipped)

---

## Open Decisions

- [ ] Replace HubSpot polling trigger with HubSpot Workflow → Catch Hook for instant triggering
- [ ] Add duplicate-call prevention (HubSpot deal property gate on Zap 1)
- [ ] Which HubSpot phone field to prefer: `mobilephone` vs `phone`?
