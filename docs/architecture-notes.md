# Architecture Notes & Gap Analysis

## Credentials & IDs

| Key | Value |
|-----|-------|
| Zap 2 Catch Hook URL | <your-zapier-webhook-url> |
| Vapi Assistant ID | <your-vapi-assistant-id> |
| Vapi Phone Number ID | <your-vapi-phone-number-id> |
| HubSpot Trigger Deal Stage ID | <your-hubspot-deal-stage-id> |

Register the Zap 2 URL in Vapi: Dashboard → Assistant → Server URL.

---

## Zap Architecture Overview

The workflow requires exactly **2 Zaps** plus a **Catch Hook URL** registered in Vapi.

```
┌─────────────────────────────────────────────────────────────┐
│  ZAP 1: "Trigger Vapi Call on Deal Stage Change"            │
│                                                             │
│  TRIGGER: HubSpot — Deal Stage Changed                      │
│  ACTION 1: HubSpot — Find Associated Contact                │
│  ACTION 2: Webhooks — POST to Vapi /call                    │
└─────────────────────────────────────────────────────────────┘
                          │
                          │  Vapi receives call request
                          │  Vapi dials prospect
                          │  Call ends
                          ▼
                   Vapi fires webhook
                          │
┌─────────────────────────────────────────────────────────────┐
│  ZAP 2: "Log Vapi Call Result to HubSpot"                   │
│                                                             │
│  TRIGGER: Webhooks — Catch Hook                             │
│  FILTER:   message.type = end-of-call-report                │
│  ACTION 1: AI by Zapier — generate call note from transcript│
│  ACTION 2: HubSpot — Create Call Engagement on Deal         │
│  FILTER:   successEvaluation = true                         │
│  ACTION 3: Slack — Send Channel Message to AE channel       │
└─────────────────────────────────────────────────────────────┘
```

---

## Zap 1 — Detail

### Trigger: HubSpot "Deal Stage Changed"

- Use the native HubSpot app in Zapier, trigger: **"Updated Deal"**
- Filter with a Zapier Filter step: only continue when `dealstage` equals your target stage ID
- HubSpot stage IDs are internal strings (e.g., `appointmentscheduled`), not display names — confirm yours in HubSpot Settings → Pipelines

### Action 1: Get Contact Phone Number

- **HubSpot — Find Associated Contact** (or "Get Contact by ID")
- The deal trigger gives you the deal ID but not the contact directly
- Use HubSpot's association lookup to get the contact, then pull `mobilephone` or `phone`
- If a deal has multiple contacts, this returns the first — decide upfront which phone field to use and whether to gate on "phone exists"

### Action 2: POST to Vapi `/call`

- Use **Webhooks by Zapier — Custom Request (POST)**
- Endpoint: `https://api.vapi.ai/call/phone`
- Headers:
  - `Authorization: Bearer <your_vapi_private_key>`
  - `Content-Type: application/json`
- Body (JSON):
  ```json
  {
    "assistantId": "<your_existing_assistant_id>",
    "customer": {
      "number": "<phone_from_step_2>",
      "name": "<contact_name_optional>"
    },
    "phoneNumberId": "<your_vapi_phone_number_id>",
    "metadata": {
      "hubspot_deal_id": "<deal_id_from_trigger>",
      "hubspot_contact_id": "<contact_id_from_step_2>"
    }
  }
  ```
- The `metadata` object rides along with the call and comes back in the Vapi webhook payload — this is how Zap 2 knows which deal to update.

---

## Zap 2 — Detail

### Trigger: Webhooks by Zapier — Catch Hook

- Create this Zap first to get the webhook URL
- Register that URL in Vapi: **Dashboard → Phone Numbers (or Assistant) → Server URL**
- Vapi sends multiple event types to this URL (`call.started`, `call.ended`, `transcript`, etc.)
- Add a Filter step: only continue when `message.type` equals `end-of-call-report` (Vapi's call-summary event)

### Vapi Webhook Payload (relevant fields)

```json
{
  "message": {
    "type": "end-of-call-report",
    "endedReason": "customer-ended-call",
    "transcript": "Agent: Hi, is this...\nCustomer: Yes...",
    "summary": "The prospect confirmed interest and agreed to a demo.",
    "analysis": {
      "summary": "...",
      "successEvaluation": "true"
    },
    "call": {
      "id": "vapi-call-id",
      "metadata": {
        "hubspot_deal_id": "12345678",
        "hubspot_contact_id": "87654321"
      }
    }
  }
}
```

- Extract `message.call.metadata.hubspot_deal_id` — this is the deal to update
- Extract `message.summary` or `message.analysis.summary` for the note body
- Extract `message.endedReason` for disposition tagging

### Action 1: AI by Zapier — Generate Call Note

- App: **AI by Zapier**, Action: **Analyze & Transform** (or equivalent prompt step)
- Inputs mapped from webhook: `message.transcript` and `message.summary`
- Prompt used:
  > "You are a sales ops assistant reviewing an AI SDR outbound call. Given the transcript and summary below, write a concise HubSpot call note in three parts: (1) the lead's key pain point or need, (2) any objection or hesitation they raised, and (3) the recommended next step for the account executive. Be specific, use the prospect's own words where possible, and keep the total note under 150 words."
- Output feeds directly into the HubSpot engagement body
- **Status: Built**

### Action 2: HubSpot — Create Call Engagement

- App: HubSpot, Action: **Create Engagement**
- Engagement type: **Call**
- Associate with: Deal ID (from `message.call.metadata.hubspot_deal_id`)
- Body: formatted note from Formatter step
- Duration: `message.durationSeconds` (convert to milliseconds for HubSpot: multiply by 1000)
- Direction: **Outbound**
- Status: **Completed**

### Filter: Only continue if qualified

- App: **Filter by Zapier**
- Condition: `message > analysis > successEvaluation` equals (text) `true`
- Unqualified calls get their HubSpot note logged but do not trigger an AE notification

### Action 3: Slack — Send Channel Message to AE Channel

- App: **Slack**, Action: **Send Channel Message**
- Channel: AE team channel (e.g. `#ae-leads`)
- Message template:
  ```
  🤖 AI SDR Qualified Lead

  *Contact:* {contact name}
  *Deal:* {deal name}
  *Outcome:* {message.endedReason}
  *Summary:* {AI-generated note from Action 1}

  HubSpot Deal: https://app.hubspot.com/contacts/{portal_id}/deal/{hubspot_deal_id}

  React to claim this lead or reply in thread.
  ```
- Hardcode your HubSpot portal ID; map deal ID from `message.call.metadata.hubspot_deal_id`
- Ensure the Zapier Slack bot is invited to the channel (`/invite @Zapier`) if it's private

---

## Gaps & Gotchas

### 1. HubSpot Deal Stage Trigger Reliability
- Zapier's HubSpot "Updated Deal" trigger polls every 5–15 minutes (not instant)
- Make.com also polls but their HubSpot module may behave differently
- **Mitigation:** Accept the polling delay, or use HubSpot Workflows to push to a Zapier Catch Hook (faster, event-driven)

### 2. HubSpot Deal → Contact Association
- Zapier's native HubSpot trigger for deals does NOT include associated contact data
- You need an extra "Find Associated Contact" step, which counts as an action in your Zap task usage
- If the deal has no associated contact, the Zap will error — add a Filter to halt early with a useful label

### 3. Phone Number Format
- Vapi expects E.164 format: `+15551234567`
- HubSpot stores phone numbers in various formats
- Use **Formatter by Zapier — Numbers — Format Phone Number** between the contact lookup and the Vapi POST

### 4. Vapi Webhook Registration
- Vapi sends all events (call started, transcript chunks, call ended) to a single server URL
- You must filter for `end-of-call-report` in Zap 2 or you'll get multiple Zap runs per call
- The server URL is set at the **assistant level** or **phone number level** in Vapi — assistant-level overrides phone-number-level

### 5. Mapping Vapi Payload in Zapier
- Zapier learns the webhook schema from a sample — you must make a real test call first to get Zapier to recognize the nested `message.call.metadata` fields
- Until you send a sample, you can't map those fields in subsequent steps
- Workaround: use **Webhooks — Catch Hook** and manually send a mock payload via curl/Postman to train Zapier

### 6. Duplicate Call Prevention
- If the deal stage changes multiple times (e.g., moved back and forward), Zap 1 fires again
- Consider adding a HubSpot deal property (e.g., `ai_call_triggered`) and filter in Zap 1: only proceed if that field is empty, then set it after triggering the call
- This requires an extra HubSpot "Update Deal" action at the end of Zap 1

### 7. Vapi `phoneNumberId` Requirement
- Outbound calls via Vapi API require a `phoneNumberId` (your Vapi-provisioned number)
- Find it in Vapi Dashboard → Phone Numbers — it's a UUID string

### 8. Task Usage / Zap Limits
- Each Zap run consumes tasks per action step
- Zap 1: ~3 tasks per call triggered (Filter + Contact lookup + Webhook POST)
- Zap 2: ~5 tasks per call completed (Filter + AI step + HubSpot engagement + Filter + Slack)
- Budget ~8 tasks per prospect called (qualified); ~3 for unqualified (Slack step skipped)

---

## Recommended Build Order

1. Build Zap 2 first (Catch Hook) to get the webhook URL
2. Register that URL in Vapi
3. Build Zap 1 and test with a real HubSpot deal stage change
4. Make a test call to verify the end-to-end flow
5. Send a test `end-of-call-report` payload to train Zapier's field mapper for Zap 2
6. Complete Zap 2 field mapping and test with live call

---

## Open Decisions

- [ ] Which HubSpot phone field to use: `mobilephone` vs `phone`?
- [ ] Should the Vapi webhook be set at assistant level or phone number level?
- [ ] Add duplicate-call prevention (deal property gate)?
- [ ] Should Zap 1 also update the deal stage after triggering the call?
