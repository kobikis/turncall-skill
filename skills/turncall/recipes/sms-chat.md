# Recipe: text conversations (SMS / Chat API)

Same agents, no audio. Two entry points; pick by where the customer is.

## SMS (customer texts your number)

TurnCall handles the conversation end-to-end: inbound text → session lookup/create →
agent reply → outbound text. You don't run the loop. To stand it up:

1. Bind the number with SMS enabled and an agent:
   ```json
   POST /v1/phone-numbers
   {"external_number_sid": "PN...", "e164_number": "+1...",
    "routing_target_type": "agent", "routing_target_id": "<agent_id>", "sms_enabled": true}
   ```
2. The agent's `system_prompt` *is* your behavior. There's no per-message seam in the basic
   case — the agent decides each reply.
3. Observe via webhooks: `session.created`, `chat.created` (each message, with `role` +
   `content` + `channel`), `session.updated`/`session.deleted`. Use these to log, analyze,
   or hand off — not to generate replies (TurnCall already sent them).

Sessions auto-expire after 24h of inactivity and resume within that window for the same
(customer, your-number) pair.

## Chat API (your app drives the conversation)

When the text lives in *your* UI (web widget, in-app), call the Chat API directly and render
the reply yourself:

```json
POST /v1/chat
{"agent_id": "<id>", "message": "I need to reschedule", "session_id": "<id?>"}
-> {"reply": "...", "chat_id": "...", "session_id": "..."}
```
Thread context with **either** `session_id` (group) **or** `previous_chat_id` (linear chain) —
never both. List/inspect: `GET /v1/chat/sessions`, `/sessions/{id}/messages`.

## The seam (if you need one)

Text apps usually have no dialer/router seam — the "logic" is the agent prompt. If you do need
to route a text conversation (e.g. different agent by keyword or customer), do it in *your*
code before calling `POST /v1/chat` (pick the `agent_id`), the same shape as
[inbound-routing](inbound-routing.md)'s `route_call`. SMS routing-by-logic would use the same
`routing_target_type: "webhook"` call-init mechanism as voice.

## Notes
- Don't build your own SMS retry/session store — TurnCall owns session lifecycle. Mirror it
  from the `session.*` events if you need a local view.
- Same agent can serve voice and text; the prompt should read well spoken *and* typed.
