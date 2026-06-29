---
name: turncall
description: Build apps on top of the TurnCall voice API — outbound calling and lead dialers, inbound call routing, webhook and event handling, and live call control. Use when wiring an app to TurnCall, placing TurnCall outbound calls by some logic, routing TurnCall inbound calls by some logic, handling TurnCall webhooks, or configuring TurnCall call-init.
---

TurnCall is an API-only voice-agent platform. Your app never touches audio — it tells TurnCall **which agent** handles a call and reacts to what happens. Two directions, and every app is some mix of them:

- **You → TurnCall** (REST): create agents, bind numbers, place outbound calls, control live calls.
- **TurnCall → you** (HTTP callbacks): **webhooks** (fire-and-forget events) and **call-init** (synchronous — TurnCall asks your server which agent to use, mid-ring).

## Before wiring anything

Read [API.md](API.md) — auth, the builder endpoints, and the webhook/call-init shapes. For a field-exact schema, consult the bundled [openapi.json](openapi.json) (a snapshot; the authoritative spec is `docs/openapi.json` in the TurnCall repo — regenerate API.md and this snapshot from it when the API changes).

## The logic seam

Every app has exactly one spot that is **yours** — a decision the platform can't make. Each recipe isolates it as a single replaceable function with a worked example. Replace the body; keep the wiring around it.

- Outbound: `pick_next_lead(leads)` — who to call next.
- Inbound: `route_call(call) -> agent_id` — which agent answers.

Resist spreading business logic across the integration. If it's not the seam, it's plumbing the recipe already gives you.

## Pick a recipe

| You're building | Recipe |
|---|---|
| Leads → call them by some logic (campaign / dialer) | [recipes/outbound-campaign.md](recipes/outbound-campaign.md) |
| Inbound call → route by some logic (router) | [recipes/inbound-routing.md](recipes/inbound-routing.md) |
| Text conversation (SMS / Chat API) | [recipes/sms-chat.md](recipes/sms-chat.md) |

## Shared setup — every recipe starts here

1. **Auth.** Every request carries `Authorization: Bearer tc_...` (a project-scoped key). All data is scoped to that key's project.
2. **Create + publish an agent.** `POST /v1/agents`, then `POST /v1/agents/{id}/publish`. A **draft** agent cannot take calls — publishing is not optional.
3. **Wire the feedback loop.** Subscribe to webhooks (`POST /v1/webhooks`) and treat `call.ended` as the authoritative end-of-call signal. `GET /v1/calls/{id}/events` is your debugger.

Keep TurnCall as the source of truth for **call state**; your app owns **leads, routing rules, and history**, and reacts to TurnCall's events. Don't mirror call state in your DB — read it back from the events.
