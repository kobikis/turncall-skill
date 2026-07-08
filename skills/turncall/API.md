# TurnCall API — builder reference

Curated for building apps. For any field not covered here, the bundled
[openapi.json](openapi.json) is the field-exact source (authoritative copy:
`docs/openapi.json` in the TurnCall repo).

Base URL: your TurnCall host, e.g. `http://localhost:8090` (dev) or your deployment.
All paths below are under `/v1` unless noted.

## Auth

```
Authorization: Bearer tc_...
```
Project-scoped: every object you create/read belongs to the API key's project.

Response envelope: `{"success": true, "data": ...}` on success;
`{"success": false, "error": "...", "code": "..."}` on error.

To bootstrap a key (admin):
```
POST /v1/projects                {"name": "..."}                       -> data.id
POST /v1/api-keys?project_id=ID  {"name": "...", "role": "admin"}      -> data.raw_key  (shown once)
```

## Agents

```
POST /v1/agents                 {"name": "...", "config": { ... }}     -> draft agent
POST /v1/agents/{id}/publish                                           -> live (required before calls)
PUT  /v1/agents/{id}            {"config": { ... }}                    -> edit a draft
GET  /v1/agents/{id}
POST /v1/agents/{id}/rollback                                          -> restore previous version
```

Minimal `config`:
```json
{
  "system_prompt": "You are ...",
  "first_message": "Hello! ...",
  "llm": {"provider": "openai", "model": "gpt-4o-mini"}
}
```
Common extras: `tools` (built-in + webhook + MCP), `analysis` (post-call summary/scoring),
`stt`/`llm`/`tts` provider blocks, or `s2s` for a native speech-to-speech model
(`pipeline_mode: "s2s"`; `s2s.base_url` targets an OpenAI-Realtime gateway like Grok).
Reusable structured extractions attach via `analysis.takeaway_ids` (define them at
`/v1/takeaways`; results land in `call.ended` under `analysis.takeaways`). See openapi
`AgentConfig` for the full surface.

## Phone numbers

Bind a number you own (on your telephony provider) to an agent or a routing webhook:
```json
POST /v1/phone-numbers
{
  "external_number_sid": "PN...",          // provider's number SID
  "e164_number": "+15551234567",
  "routing_target_type": "agent",          // "agent" | "webhook"
  "routing_target_id": "<agent_id>",       // when target_type=agent
  "server_url": "https://you.example/route",// when target_type=webhook (call-init)
  "sms_enabled": true                       // also handle SMS on this number
}
```
- `routing_target_type: "agent"` — inbound calls always use that agent. Simple.
- `routing_target_type: "webhook"` — inbound calls trigger **call-init** (below): your
  server decides the agent per call. This is how you route by logic.

Returns `data.id` — the **phone-number id** you pass as `from_number_id` for outbound.

Update a binding in place (id + call-init `server_url_secret` stay stable — no unbind/rebind):
```
PUT /v1/phone-numbers/{id}   {"routing_target_id": "...", "server_url": "...", "sms_enabled": true}
```

Weighted A/B routing (split inbound traffic across agents):
```
PUT /v1/phone-numbers/{id}/routing   {"weights": [{"agent_id": "...", "weight": 70}, ...]}  // sum 100
```

## Outbound calls

```json
POST /v1/calls/outbound
{
  "to_number": "+15557654321",
  "from_number_id": "<phone-number-id>",     // the bound number's id, NOT the E.164 string
  "metadata": {"lead_id": "L-123"}            // echoed back on call.ended for correlation
}
```
The agent that answers is the one bound to `from_number_id`. Returns the call record
(`data.id` = call_id). Drive *which* number you dial from your own logic (see the
outbound recipe).

## call-init (inbound routing decision)

When a number has `routing_target_type: "webhook"`, TurnCall POSTs to your `server_url`
**before** the agent is selected, and waits (~5s) for your answer.

Request TurnCall sends you:
```json
{
  "message": {
    "type": "call-init",
    "timestamp": "2026-06-29T...",
    "call": {"id": "<call_id>", "provider_call_id": "CA...", "type": "inboundPhoneCall"},
    "phoneNumber": {"number": "+15551234567"},   // the number that was called (yours)
    "customer": {"number": "+15559876543"}        // the caller
  }
}
```
(`call.type` ∈ `inboundPhoneCall` | `inboundWhatsAppCall` | `webrtc`.)

Your response picks the agent and optionally injects data:
```json
{
  "agent_id": "<uuid>",                          // OR "agent": { ...inline config }
  "variables": {"name": "Jane", "tier": "gold"}, // template vars for the prompt
  "metadata": {"crm_id": "C-123"},               // stored on the call, echoed on call.ended
  "dynamic_data": {"knowledge_context": "..."}    // prepended to the system prompt
}
```
If you can't decide in time, return an `agent_id` for a sensible default — never leave the
caller hanging.

## Webhooks (events)

Subscribe:
```json
POST /v1/webhooks   {"url": "https://you.example/hook", "events": ["*"]}
```
`events` is a list of event types, or `["*"]` for all.

Every delivery is a signed envelope (event-specific data under `payload`):
```json
{
  "event": "call.ended",
  "project_id": "...", "call_id": "...", "session_id": null,
  "agent_id": "...", "event_id": "...", "timestamp": "...",
  "payload": { ... }
}
```
Headers: `X-TurnCall-Signature` (HMAC-SHA256), `X-TurnCall-Timestamp`, `X-TurnCall-Event`.
Verify by computing HMAC-SHA256 over the raw body with your signing secret and comparing
to the header. `event_id` is stable across retries — **use it to dedupe**.

Events you'll actually handle:

| event | when | key payload |
|---|---|---|
| `call.started` | call connected | `agent_id`, `from_number`, `to_number` |
| `call.ended` | call finished (after recording + analysis) | `status`, `ended_reason`, `duration_ms`, `metadata`, `transcript`, `summary`, `analysis`, `recording_url` |
| `transfer.answered` | a transfer's destination answered | `target_number`, `answered_by` (`human`/`machine`) |
| `call.transferred` | transfer initiated | `target_number`, `transfer_mode` |
| `tool.result` | a tool ran | `tool_name`, `arguments`, `result` |
| `transcript.final` | each finished utterance | `role` (`customer`/`assistant`), `text` |
| `session.created` / `chat.created` | SMS/chat activity | `session_id`, `channel`, `content` |

`ended_reason` values: `customer_ended_call`, `assistant_ended_call`,
`customer_did_not_answer`, `customer_busy`, `voicemail`, `transferred`,
`pipeline_error`, `telephony_failed`, `unknown`.

## Live call control

Act on an in-progress call (from a tool the agent calls, or your backend out-of-band):
```
POST /v1/calls/{id}/transfer   {"target_number": "+1...", "transfer_mode": "cold"|"warm",
                                "transfer_message": "...", "briefing": "..."|{"from_summary": true},
                                "fallback_message": "..."}
POST /v1/calls/{id}/handoff    {"target_agent_id": "...", "reason": "...", "context_payload": {...}}
POST /v1/calls/{id}/end        {"reason": "..."}
POST /v1/calls/{id}/dtmf       {"digits": "123#"}
```
Warm transfer + `fallback_message` require the TurnCall server to have `PUBLIC_BASE_URL` set.

## Reading call data

```
GET /v1/calls/{id}                 -> status, numbers, duration, timestamps
GET /v1/calls/{id}/transcript      -> the transcript
GET /v1/calls/{id}/events          -> ordered event log (your debugger)
GET /v1/calls/{id}/analysis        -> post-call analysis (or {"status":"pending"})
```
