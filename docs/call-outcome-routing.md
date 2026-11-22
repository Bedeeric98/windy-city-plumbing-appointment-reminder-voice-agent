# Windy City Plumbing — Call Outcome Routing Reference

Call-outcome routing for WCP. WCP is an appointment-reminder/confirmation agent, not a lead-qualification agent — its outcome vocabulary (`confirmed/rescheduled/rescheduled_unconfirmed/cancelled/busy/wrong_person/no_meaningful_conversation/do_not_call`) is structurally different from SunPath's, so this doc covers WCP specifics rather than assuming SunPath's routing applies.

GHL field IDs:
- `reminder_status` → `YOUR_GHL_FIELD_ID_REMINDER_STATUS`
- `remind_attempt_count` → `YOUR_GHL_FIELD_ID_REMIND_ATTEMPT`
- `call_in_progress` → `YOUR_GHL_FIELD_ID_CALL_IN_PROGRESS`
- `needs_human_followup` → `YOUR_GHL_FIELD_ID_NEEDS_HUMAN_FOLLOWUP`
- `service_type` → `YOUR_GHL_FIELD_ID_SERVICE_TYPE`
- `emergency` → `YOUR_GHL_FIELD_ID_EMERGENCY`

**Field-write convention differs from SunPath:** WCP's live GHL-update nodes write `{ "key": "...", "field_value": "..." }` (snake_case value key, field addressed by `key` not `id`) — not SunPath's `{ "id": "...", "fieldValue": "..." }`. Confirmed directly against live nodes, not assumed.

---

## EOCR Outcomes → GHL Fields (Outbound — `Route Outbound Call Outcome`, 8-way switch)

**Updated 2026-09-10:** added `rescheduled_unconfirmed`, WCP's analog of SunPath's `qualified_not_booked` — Telnyx's AI-generated call summary can hallucinate "rescheduled" language from a truncated conversation even when `reschedule_windycity` never actually fired or fired without a confirmed time. This outcome exists so the workflow doesn't have to trust the summary label alone for a state-changing outcome. Also added `Detect Mishearing-Driven Cancel Outbound`, inserted between the switch's `Cancelled` branch and its GHL update — WCP's analog of SunPath's `Detect Mishearing-Driven Disqualify`, re-targeted at `cancelled` (WCP's highest-cost misclassification risk, since WCP has no qualification concept to mirror `not_qualified` against).

| call_outcome | reminder_status value | needs_human_followup | remind_attempt_count | Notifications |
|---|---|---|---|---|
| `confirmed` | connected | false | Incremented (cap 2) | None |
| `rescheduled` | connected | false | Incremented (cap 2) | None |
| `rescheduled_unconfirmed` | rescheduled_unconfirmed | true | **Not incremented** | Slack + Gmail |
| `cancelled` | do_not_call ⚠️ known bug, see below | dynamic — read off `Detect Mishearing-Driven Cancel Outbound` | Incremented (cap 2) | None |
| `busy` | connected | true | Incremented (cap 2) | Slack + Gmail |
| `wrong_person` | connected | true | Incremented (cap 2) | Slack + Gmail |
| `no_meaningful_conversation` | connected | true | Incremented (cap 2) | Slack + Gmail |
| `do_not_call` | do_not_call | — (not written) | **Not incremented** | None |

**Confirmed directly against live node bodies (2026-09-14), not inferred from the template:**
- `rescheduled_unconfirmed` and `do_not_call` are the only two outcomes whose GHL-update node does **not** include a `remind_attempt_count` field at all — both are terminal/exception states where the normal reminder cadence no longer applies (a `do_not_call` contact shouldn't be re-attempted; a `rescheduled_unconfirmed` contact needs a human to actually book the new time, not another automated reminder attempt).
- `cancelled`'s `needs_human_followup` was changed 2026-09-10 from a hardcoded `"false"` to `"{{ $json.needs_human_followup }}"`, reading dynamically off the new upstream `Detect Mishearing-Driven Cancel Outbound` node.

**Inbound (`Route Inbound Call Outcome`) mirrors this exactly**, 8-way switch with the same rule set, reading `metadata_from`/`Set Inbound Contact & Insights Data` instead of `metadata_to`/`Set Outbound Contact & Insights Data`, and does not increment `remind_attempt_count` at all on any inbound outcome (a callback doesn't consume a scheduled-reminder attempt).

---

## No-Answer / Voicemail → GHL Fields (Real-time Call Status Handler)

| remind_attempt at time of call | reminder_status value | remind_attempt_count | needs_human_followup | Notifications |
|---|---|---|---|---|
| 0 (first call missed) | called_24 | 1 | false | None — GHL triggers 12hr workflow |
| 1 (second call missed) | called_12 | 2 | true | Slack + Gmail — both attempts exhausted |

WCP has only two reminder calls (24hr / 12hr before appointment) — **no Call 3**, unlike SunPath's three-call sequence. Confirmed against the master reference doc before building item 11's test utilities (SunPath's manual Call-3-dialer utility does not apply to WCP).

Voicemail (`AnsweredBy = machine_start`) is handled in the TeXML Router, not the Call Status Handler — same architecture as SunPath.

---

## GHL Update Bodies (live, verified 2026-09-14)

### Outbound EOCR — Confirmed / Rescheduled
```json
{
  "customFields": [
    { "key": "reminder_status", "field_value": "connected" },
    { "key": "needs_human_followup", "field_value": "false" },
    { "key": "call_in_progress", "field_value": "no" },
    { "key": "remind_attempt_count", "field_value": "={{ INCREMENT_EXPRESSION }}" }
  ]
}
```

### Outbound EOCR — Rescheduled Unconfirmed
```json
{
  "customFields": [
    { "key": "reminder_status", "field_value": "rescheduled_unconfirmed" },
    { "key": "needs_human_followup", "field_value": "true" },
    { "key": "call_in_progress", "field_value": "no" }
  ]
}
```
No `remind_attempt_count` field — this outcome needs a human to book the new time, not another automated attempt.

### Outbound EOCR — Cancelled
```json
{
  "customFields": [
    { "key": "reminder_status", "field_value": "do_not_call" },
    { "key": "needs_human_followup", "field_value": "={{ $json.needs_human_followup }}" },
    { "key": "call_in_progress", "field_value": "no" },
    { "key": "remind_attempt_count", "field_value": "={{ INCREMENT_EXPRESSION }}" }
  ]
}
```
⚠️ `reminder_status` hardcodes `do_not_call`, not `cancelled` — flagged, not fixed. `needs_human_followup` reads dynamically off `Detect Mishearing-Driven Cancel Outbound/Inbound`.

### Outbound EOCR — Busy / Wrong Person / No Meaningful Conversation
```json
{
  "customFields": [
    { "key": "reminder_status", "field_value": "connected" },
    { "key": "needs_human_followup", "field_value": "true" },
    { "key": "call_in_progress", "field_value": "no" },
    { "key": "remind_attempt_count", "field_value": "={{ INCREMENT_EXPRESSION }}" }
  ]
}
```

### Outbound EOCR — Do Not Call
```json
{
  "customFields": [
    { "key": "reminder_status", "field_value": "do_not_call" },
    { "key": "call_in_progress", "field_value": "no" }
  ]
}
```
No `needs_human_followup` or `remind_attempt_count` fields at all — a genuine opt-out needs neither a follow-up flag nor a reminder-attempt count once contact is permanently stopped.

### Call Status — First Call No Answer
```json
{
  "customFields": [
    { "key": "reminder_status", "field_value": "called_24" },
    { "key": "remind_attempt_count", "field_value": "1" },
    { "key": "call_in_progress", "field_value": "no" }
  ]
}
```

### Call Status — Second Call No Answer
```json
{
  "customFields": [
    { "key": "reminder_status", "field_value": "called_12" },
    { "key": "remind_attempt_count", "field_value": "2" },
    { "key": "needs_human_followup", "field_value": "true" },
    { "key": "call_in_progress", "field_value": "no" }
  ]
}
```

---

## GHL Note Headers by Outcome

| Outcome | Header |
|---|---|
| confirmed | ✅ Client confirmed appointment. |
| rescheduled | 🔄 Client rescheduled appointment. New time agreed on call. |
| rescheduled_unconfirmed | 🔄 Client discussed rescheduling but the call ended before a specific new time was confirmed and booked. Human follow-up required. |
| cancelled | ❌ Client cancelled appointment. |
| busy | ⚠️ Client was busy and needs a callback. Human follow-up required. |
| wrong_person | 👤 Wrong person answered. The intended contact was not available. |
| no_meaningful_conversation | 🔇 Call connected but no meaningful conversation took place. A human should follow up directly. |
| do_not_call | 🚫 Client opted out — do not contact again. |
| inbound confirmed | 📲 Inbound Call — Client called back and confirmed appointment. |
| inbound rescheduled | 📲 Inbound Call — Client called back and rescheduled appointment. |
| inbound rescheduled_unconfirmed | 📲 Inbound Call — Client called back and discussed rescheduling, but no specific new time was confirmed. Human follow-up required. |
| inbound cancelled | 📲 Inbound Call — Client called back and cancelled appointment. |
| inbound busy | 📲 Inbound Call — Client called back but was busy. Human follow-up required. |
| inbound wrong_person | 📲 Inbound Call — Wrong person answered the inbound call. |
| inbound no_meaningful_conversation | 📲 Inbound Call — Call connected but no useful conversation took place. |
| inbound do_not_call | 📲 Inbound Call — Client opted out — do not contact again. |

---

## GHL Note Body Template (WCP, live — confirmed via `Create Note GHL For Rescheduled Unconfirmed Route Outbound`)

```
[HEADER LINE]

Call SID: {{ call_control_id }}

Call Report -- {{ $now.setZone('America/Chicago').toFormat('cccc d LLLL yyyy, h:mm a') }} CT

Outcome: {{ call_outcome }}
Call Successful: {{ call_successful }}
Someone Home: {{ someone_home }}
Access Notes: {{ access_notes }}
Sentiment: {{ user_sentiment }}

Summary:
{{ telnyx_summary }}

Recording URL: {{ recording_url }}
```

WCP uses `America/Chicago` (Central Time) — confirm this against the actual client timezone before reusing this template verbatim on a future build; it is not a universal constant, just WCP's own value.

---

## Node count and verification

Outbound and Inbound routing switches are both 8-way (`Route Outbound/Inbound Call Outcome`). Full workflow: 223 nodes as of 2026-09-10. All GHL update bodies in this doc were pulled directly from the live workflow via `n8n_get_workflow` (filtered mode) on 2026-09-14, not copied from the template or from memory — including the `Rescheduled Unconfirmed`/`Do Not Call` no-`remind_attempt_count` detail, which was not previously documented anywhere.


