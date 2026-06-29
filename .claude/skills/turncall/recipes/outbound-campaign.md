# Recipe: outbound campaign (leads → call by logic)

Your app holds leads. You decide who to call and when; TurnCall places the call and
reports the outcome. The decision is the **logic seam** — everything else is the loop
below.

## Prerequisites
- A published agent (`POST /v1/agents` → `/publish`).
- A bound outbound number — `POST /v1/phone-numbers` with `routing_target_type: "agent"`
  pointing at that agent. Keep its `data.id` → this is your `from_number_id`.
- A webhook subscription for `call.ended` (`POST /v1/webhooks`).

## The seam — replace this, keep everything else

```python
def pick_next_lead(leads):
    # EXAMPLE — replace with your logic (priority, scheduling, retry backoff, timezone).
    callable_now = [l for l in leads if l.status == "pending" and l.attempts < 3]
    return min(callable_now, key=lambda l: (l.priority, l.last_attempt), default=None)
```
Your real logic lives here and nowhere else: business hours, do-not-call, retry windows,
A/B of agents. The loop doesn't change.

## The loop

1. **Pick** a lead: `lead = pick_next_lead(leads)`. If `None`, idle.
2. **Place the call** — carry your id in `metadata` so you can correlate on the way back:
   ```python
   POST /v1/calls/outbound
   {"to_number": lead.phone, "from_number_id": FROM_ID, "metadata": {"lead_id": lead.id}}
   ```
   Mark the lead `dialing`, bump `attempts`. The response `data.id` is the `call_id`.
3. **React on `call.ended`** (webhook). Read `payload.metadata.lead_id` to find the lead,
   then branch on `payload.ended_reason`:
   - `customer_ended_call` / `assistant_ended_call` → completed; inspect `payload.analysis`
     (e.g. `success_evaluation`, `structured_data`) to set the disposition.
   - `customer_did_not_answer` / `customer_busy` / `voicemail` → reschedule (your backoff).
   - `telephony_failed` / `pipeline_error` → retry or flag for review.
4. **Pace yourself.** Cap concurrent in-flight calls (count leads in `dialing`) so you don't
   blow past provider limits or staff capacity. Add the lead back to the pool when it ends.

## Notes
- **Dedupe** on `event_id` — `call.ended` can be redelivered; don't double-count an outcome.
- Don't poll `GET /v1/calls/{id}` in a loop; the `call.ended` webhook is the signal. Use the
  GET endpoints only for debugging or a one-off backfill.
- Want the agent to leave a voicemail? Configure that on the agent (voicemail detection),
  not in this loop — the loop only decides *who* to call.
- Compliance (consent, DNC, calling hours) is your `pick_next_lead` logic's job, and it is
  not optional. The platform won't enforce it for you.
