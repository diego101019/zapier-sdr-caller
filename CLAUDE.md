# Zapier SDR Caller — Project Context

## Purpose

Replicating an existing Make.com AI SDR outbound call workflow using Zapier as the orchestration layer. This is a no-code configuration project — no application code, only Zapier Zap config, Vapi assistant config, and HubSpot config.

**Motivation:** Platform evaluation conducted as part of consultancy work — comparing Zapier vs. Make.com as orchestration layers for AI sales automation workflows. A Make.com version of this workflow was already running in production for a client; the Zapier build was done in parallel to assess ease-of-use, task economics, and integration depth, to inform future client platform recommendations.

## Existing Make.com Flow (source of truth)

1. HubSpot deal moves to a specific pipeline stage
2. Make detects the stage change and triggers an outbound call via Vapi
3. Vapi's AI agent runs a qualifying conversation with the prospect
4. After the call, Vapi POSTs the transcript + outcome to a Make webhook
5. Make reads the result and creates a call note on the HubSpot deal

## Tools in Use

| Tool | Role |
|------|------|
| Zapier | Orchestration (replacing Make.com) |
| Vapi | AI voice agent — existing assistant already configured |
| HubSpot | CRM — deal stages, contacts, call notes |

## Zapier Architecture (Planned)

### Zap 1 — HubSpot Deal Stage → Trigger Vapi Call

**Trigger:** HubSpot — Deal Stage Changed (or "Updated Deal" filtered by stage)
**Actions:**
1. HubSpot — Get Contact associated with the deal (to get phone number)
2. Webhooks by Zapier — POST to Vapi `/call` API endpoint
   - Pass: phone number, assistant ID, metadata (deal ID, contact ID)

### Zap 2 — Vapi Webhook → HubSpot Call Note

**Trigger:** Webhooks by Zapier — Catch Hook (receives Vapi `call.ended` event)
**Actions:**
1. Formatter by Zapier — parse/extract transcript summary and disposition
2. HubSpot — Create Engagement (Call) on the deal
   - Set note body to transcript summary + AI disposition

## Key Data Flow

```
HubSpot Deal (stage change)
  → deal ID, associated contact ID
    → contact phone number (via HubSpot lookup)
      → Vapi call triggered (with deal ID in metadata)
        → call ends, Vapi webhook fires (with deal ID in payload)
          → HubSpot call note created on deal
```

The `deal ID` passed as Vapi call metadata is the linchpin — it's how Zap 2 knows which HubSpot deal to update when the call finishes.

## Known Gaps & Gotchas

See `docs/architecture-notes.md` for the full gap analysis.

## Files

- `CLAUDE.md` — this file
- `docs/architecture-notes.md` — detailed gap analysis and decision log
