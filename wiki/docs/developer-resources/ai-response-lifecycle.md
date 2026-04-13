---
title: "AI Response Card Lifecycle"
sidebar_position: 6
---

# Goal

Ensure the View Assist info card (for example, title `AI Response`) clears deterministically within 30 seconds of the **final** response event, even when response text is streamed in chunks.

# Root Cause

The common failing pattern is:

1. Chunk arrives.
2. Automation sets `title/message`.
3. Same automation starts `delay: 30`.
4. Next chunk restarts/queues the automation.

When chunks keep arriving, the clear step is repeatedly pushed out and can appear to never happen.

# Deterministic Fix

Split responsibilities:

1. `script.va_ai_response_card_update`
   - Updates card text immediately on every chunk.
   - Starts expiry only when `is_final: true`.
2. `script.va_ai_response_card_expire`
   - `mode: restart` so only one expiry timer exists at a time.
   - Waits 30 seconds.
   - Clears card only if the stored `run_id` still matches.

This makes replacement immediate for new responses and guarantees one 30-second expiry window per final response event.

# HA Objects Involved

- Entity: your VA device entity (for example `sensor.viewassist_kitchen`)
- Automation: `automation.va_ai_response_card_ingest` (your event/chunk ingestion point)
- Scripts:
  - `script.va_ai_response_card_update`
  - `script.va_ai_response_card_expire`
- Attributes used on the VA entity:
  - `title`
  - `message`
  - `message_font_size`
  - `ai_response_run_id` (custom)
  - `ai_response_final_ts` (custom)

# Scripts (Copy/Paste)

```yaml
script:
  va_ai_response_card_update:
    alias: VA AI Response Card Update
    mode: parallel
    fields:
      va_entity:
        description: Target View Assist entity id
        example: sensor.viewassist_kitchen
      response_id:
        description: Stable id for this response/run/conversation
        example: 26c937a8-31ea-423b-a806-1f8f8f79a9c8
      response_text:
        description: Current response text (chunk or full)
      is_final:
        description: true only for final response event
        example: false
      card_title:
        description: Card title to show
        default: AI Response
      message_font_size:
        description: Optional card font size
        default: 4vw
    sequence:
      - action: view_assist.set_state
        target:
          entity_id: "{{ va_entity }}"
        data:
          title: "{{ card_title | default('AI Response') }}"
          message: "{{ response_text }}"
          message_font_size: "{{ message_font_size | default('4vw') }}"
          ai_response_run_id: "{{ response_id }}"

      - if:
          - condition: template
            value_template: "{{ is_final | bool }}"
        then:
          - action: view_assist.set_state
            target:
              entity_id: "{{ va_entity }}"
            data:
              ai_response_final_ts: "{{ now().isoformat() }}"

          - action: logbook.log
            data:
              name: VA AI Response
              entity_id: "{{ va_entity }}"
              message: "Final response event run_id={{ response_id }} at {{ now().isoformat() }}"

          - action: script.va_ai_response_card_expire
            data:
              va_entity: "{{ va_entity }}"
              response_id: "{{ response_id }}"
              clear_after: 30

  va_ai_response_card_expire:
    alias: VA AI Response Card Expire
    mode: restart
    fields:
      va_entity:
        description: Target View Assist entity id
      response_id:
        description: Response id captured at final event
      clear_after:
        description: Seconds before clearing (capped to 30)
        default: 30
    sequence:
      - delay:
          seconds: "{{ [[clear_after | int(30), 0] | max, 30] | min }}"

      - condition: template
        value_template: "{{ state_attr(va_entity, 'ai_response_run_id') == response_id }}"

      - variables:
          elapsed_seconds: >-
            {% set ts = state_attr(va_entity, 'ai_response_final_ts') %}
            {% if ts %}
              {{ (as_timestamp(now()) - as_timestamp(ts)) | round(2) }}
            {% else %}
              unknown
            {% endif %}

      - action: view_assist.set_state
        target:
          entity_id: "{{ va_entity }}"
        data:
          title: ""
          message: ""
          ai_response_run_id: ""
          ai_response_final_ts: ""

      - action: logbook.log
        data:
          name: VA AI Response
          entity_id: "{{ va_entity }}"
          message: "Card cleared run_id={{ response_id }} elapsed={{ elapsed_seconds }}s"
```

# Ingest Automation Pattern

Call `script.va_ai_response_card_update` from your existing AI response automation (the one currently receiving chunk/final events from VACA/Assist).

```yaml
automation:
  - alias: VA AI Response Card Ingest
    id: va_ai_response_card_ingest
    mode: queued
    max: 50
    triggers:
      - trigger: event
        event_type: your_ai_response_event
    actions:
      - variables:
          va_entity: "{{ trigger.event.data.va_entity }}"
          response_id: "{{ trigger.event.data.response_id }}"
          response_text: "{{ trigger.event.data.response_text }}"
          is_final: "{{ trigger.event.data.is_final | bool }}"

      - action: script.va_ai_response_card_update
        data:
          va_entity: "{{ va_entity }}"
          response_id: "{{ response_id }}"
          response_text: "{{ response_text }}"
          is_final: "{{ is_final }}"
          card_title: AI Response
          message_font_size: 4vw
```

Important rules:

- Do not put `delay` + clear logic in the ingest automation.
- Start expiry only on final event.
- Keep one active expiry window by using `script.va_ai_response_card_expire` in `mode: restart`.

# Before/After Timing Evidence (Trace + Logbook)

Use these checks for acceptance:

1. Before fix (old automation trace):
   - Note `final` event timestamp.
   - Note card clear action timestamp.
   - `elapsed_before = clear - final` (often >30s or never clears under streaming).
2. After fix (new logbook entries):
   - `Final response event run_id=... at ...`
   - `Card cleared run_id=... elapsed=...s`
   - `elapsed_after` should be 30 seconds (allow small scheduler jitter).

Illustrative timing record format:

| Stage | Source | Final event | Clear event | Elapsed |
| --- | --- | --- | --- | --- |
| Before fix | Automation trace | 2026-03-04 19:41:13 | 2026-03-04 19:42:08 | 55s |
| After fix | Logbook (`VA AI Response`) | 2026-03-04 19:45:10 | 2026-03-04 19:45:40 | 30s |

Use your own trace/logbook timestamps for final validation in your environment.

This directly validates:

- Clear is deterministic and capped to 30 seconds from final response event.
- Streaming chunks do not extend visibility indefinitely.
- A new response replaces old text immediately and starts a single new 30-second expiry window.
