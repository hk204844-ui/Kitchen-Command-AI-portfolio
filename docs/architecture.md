# Kitchen Command AI — Architecture (portfolio summary)

## System context

```
                    ┌─────────────────────────────────────────┐
                    │           Caller (PSTN)                 │
                    └────────────────────┬────────────────────┘
                                         │
                                         ▼
                              ┌──────────────────────┐
                              │   Retell AI (Maya)   │
                              │  LLM + custom tools  │
                              └──────────┬───────────┘
                                         │ HTTPS tool calls
         ┌───────────────────────────────┼───────────────────────────────┐
         │                               ▼                               │
         │              ┌────────────────────────────────┐                   │
         │              │  Voice API (Express / Railway)│                   │
         │              │  /webhook/inbound-call         │                   │
         │              │  /webhook/call-events          │                   │
         │              │  /session/start, /restaurant/* │                   │
         │              │  /cart/*, /order/confirm        │                   │
         │              └───────────────┬────────────────┘                   │
         │                              │                                   │
         │                              ▼                                   │
         │              ┌────────────────────────────────┐                   │
         │              │         Supabase Postgres       │                   │
         │              │  restaurants · menu_items       │                   │
         │              │  call_sessions · orders         │                   │
         │              │  call_registry · kitchen_terminal│                  │
         │              └───────────────┬────────────────┘                   │
         │                              │ Realtime + RLS                    │
         │              ┌───────────────┴────────────────┐                   │
         │              ▼                                ▼                   │
         │   ┌─────────────────────┐          ┌─────────────────────┐      │
         │   │ Kitchen tablet PWA  │          │ Admin (platform)    │      │
         │   │ /kitchen kanban     │          │ /admin/restaurants  │      │
         │   │ /orders · /settings │          │ create venue+menu   │      │
         │   └─────────────────────┘          └─────────────────────┘      │
         └──────────────────────────────────────────────────────────────────┘
```

## Inbound call flow (latency-optimized)

1. **Ring** — Retell hits `POST /webhook/inbound-call` before the agent answers.
2. **Preload** — API resolves restaurant by `to_number`, loads menu/hours into cache, optional ring hold.
3. **Connect** — Webhook returns `call_inbound` dynamic variables (`restaurant_id`, `restaurant_name`).
4. **`call_started`** — `POST /webhook/call-events` creates DB session + registry while caller hears first audio.
5. **Agent tools** — Parallel `session_start` + `restaurant_init` (cache hit, single-digit ms).
6. **Order** — `add_item` / `remove_item` mutate `call_sessions.cart`; `confirm_order` writes `orders` row.
7. **Kitchen** — Supabase Realtime pushes new `pending` order to tablet; staff accept → prepare → ready.

## Security & multi-tenancy

| Concern | Approach |
|---------|----------|
| Tool auth | Internal API (Retell-only); service role on server |
| Kitchen users | Supabase Auth + `kitchen_terminal` 1:1 restaurant link |
| Admin users | `app_metadata.role = admin`; RLS policies on `restaurants`, `menu_items` |
| Order isolation | RLS `user_can_access_restaurant(restaurant_id)` on `orders` |
| Secrets | Railway + Vercel env; never in portfolio repos |

## Data model (high level)

- **`restaurants`** — `twilio_number`, hours, `is_open`, `busy_mode`, trading days
- **`menu_items`** — voice agent reads via `restaurant_init`
- **`call_sessions`** — cart JSON keyed by Retell `call_id`
- **`orders`** — confirmed items, `pickup_time`, `channel: voice`
- **`call_registry`** — maps `call_id` → restaurant + Twilio SID for escalate

## Escalation path

Agent invokes `escalate_to_human` → API logs reason → Twilio transfer to owner (when SID/account alignment allows). Retell native transfer recommended for Retell-managed numbers.

## Design principles

| Principle | Implementation |
|-----------|----------------|
| **Server resolves IDs** | Never trust `{{placeholders}}` from Retell tool payloads |
| **Warm before speak** | Webhook + cache so first agent turn is fast |
| **Omnichannel consistency** | Same order row powers voice confirm and kitchen UI |
| **Fail closed** | Closed restaurants blocked at confirm; no silent fallback to wrong venue |

## Extension points

- Per-restaurant owner transfer numbers (today: shared `TWILIO_OWNER_NUMBER`)
- SMS order confirmation via Twilio
- Analytics dashboard on `orders` + `escalations`

## Related coursework context (ICT304)

Platform aligns with **enterprise integration** themes: single repository (Supabase), capability-based restaurant onboarding (admin), and **CRM-style** customer journey from voice capture through kitchen fulfillment—comparable to omnichannel retail CRM case studies (capture → personalise → fulfil across channels).
