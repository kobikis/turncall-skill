# Recipe: inbound routing (call → route by logic)

A call comes in; you decide which agent answers — per call, from live context (the caller's
number, the time, a CRM lookup). That decision is the **logic seam**. TurnCall asks you for
it synchronously via **call-init**.

## Two ways to route — pick one

- **By logic (this recipe): `routing_target_type: "webhook"`.** TurnCall POSTs call-init to
  your `server_url` mid-ring and uses the agent you return. Full control per call.
- **Static split (no code): weighted routing.** If "route by logic" just means an A/B/% split
  across agents, skip the webhook: `PUT /v1/phone-numbers/{id}/routing` with weights summing
  to 100. Deterministic by caller number. Use this when there's no real decision to make.

## Setup (webhook routing)
1. Published agents for each destination.
2. Bind the number to your server:
   ```json
   POST /v1/phone-numbers
   {"external_number_sid": "PN...", "e164_number": "+1...",
    "routing_target_type": "webhook", "server_url": "https://you.example/route"}
   ```
   The response includes a signing secret — store it to verify call-init requests.

## The seam — replace this, keep the endpoint

```python
def route_call(caller_number, called_number):
    # EXAMPLE — replace with your logic (CRM lookup, business hours, language, VIP tier).
    if is_after_hours():
        return {"agent_id": AFTERHOURS_AGENT}
    customer = crm_lookup(caller_number)            # your system
    if customer and customer.tier == "gold":
        return {"agent_id": VIP_AGENT,
                "variables": {"name": customer.name, "tier": "gold"},
                "metadata": {"crm_id": customer.id}}
    return {"agent_id": DEFAULT_AGENT}
```
Return an `agent_id` (or an inline `agent` config), plus optional `variables` (filled into
the prompt) and `metadata` (echoed back on `call.ended`).

## The call-init endpoint

```python
@app.post("/route")
def route(request):
    verify_signature(request)                       # X-TurnCall-Signature, your stored secret
    msg = request.json["message"]
    decision = route_call(
        caller_number=msg["customer"]["number"],
        called_number=msg["phoneNumber"]["number"],
    )
    return decision                                  # {agent_id | agent, variables, metadata, ...}
```

Hard rules:
- **Answer fast** — TurnCall waits ~5s, then gives up on you. Do slow CRM work async; keep the
  decision path quick.
- **Always return a usable agent** — on any error or timeout in your logic, fall back to a
  default `agent_id`. A missing/late answer means dead air for the caller.
- **Verify the signature** before trusting the request (it's a public endpoint).

## After the call
Subscribe to `call.ended`; `payload.agent_id` tells you who actually handled it (handoffs are
reflected), `payload.metadata` carries the `crm_id` you injected, and `payload.analysis`/
`summary` give the outcome to write back to your CRM.
