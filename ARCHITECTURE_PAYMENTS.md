# LivingIQ — Auth & Payments Architecture Reference

Consolidated reference for implementation. Covers: component placement, the
two core workflows (login handoff, payment/checkout), the full use-case ×
layer matrix, and every failure-handling decision made along the way.

**Companion docs**, not duplicated here: `livingiq-auth/README.md` (API
reference, setup), `livingiq-auth/ARCHITECTURE.md` (why Neon over Supabase,
why sliding sessions), `dealiq/README.md` (DealIQ-specific setup, pipeline
detail).

---

## 1. Component placement

Two services, each with its own Neon database. Nothing is shared between
the databases directly — every cross-service interaction goes through an
HTTP API call, never a shared table or connection.

```
┌─────────────────────────────────────────────────────────────┐
│  livingiqweb.com (static hub)                                │
│  Just links. No backend logic of its own.                    │
└───────────────────────────┬───────────────────────────────────┘
                             │ login link
                             ▼
┌─────────────────────────────────────────────────────────────┐
│  livingiq-auth  ·  auth.livingiqweb.com                      │
│                                                                │
│  ┌─────────────────────┐    ┌──────────────────────────────┐ │
│  │  Identity            │    │  Payments relay               │ │
│  │  OTP, sessions, JWT  │    │  Dodo credentials, webhook,   │ │
│  │                       │    │  appRegistry, retry/replay    │ │
│  └─────────────────────┘    └──────────────────────────────┘ │
└──────┬───────────────┬──────────────────┬─────────────────────┘
       │                │                  │
       ▼                ▼                  ▼
   Twilio          Dodo Payments      Neon (auth data)
   SMS OTP         hosted checkout    7 tables

                             │ session cookie (down)
                             │ checkout + relay calls (both ways)
                             ▼
┌─────────────────────────────────────────────────────────────┐
│  dealiq  ·  dealiq.livingiqweb.com                            │
│                                                                │
│  ┌─────────────────────┐    ┌──────────────────────────────┐ │
│  │  Deal engine          │    │  Payments client               │ │
│  │  intake, 4-pass       │    │  checkout proxy, relay        │ │
│  │  Claude pipeline       │    │  receiver, own payments table │ │
│  └─────────────────────┘    └──────────────────────────────┘ │
└───────────────────────────┬───────────────────────────────────┘
                             ▼
                        Neon (deal data)
                        5 tables
```

**The one thing worth internalizing from this diagram:** DealIQ has no
direct line to Twilio or Dodo — only `livingiq-auth` does. DealIQ never
holds a Dodo API key, a webhook secret, or Twilio credentials. It only
ever talks to its own database and to `livingiq-auth`'s internal API.

### Why split this way

- **Identity and payments are centralized** because they involve shared,
  sensitive, cross-app credentials (JWT secret, Dodo API key, Twilio
  credentials) and a hard external constraint: Dodo caps webhook endpoints
  at 5 per account — a distributed approach (each app holding its own Dodo
  credentials) hits a wall by the 6th LivingIQ app.
- **Deal data and payment status are NOT centralized** — each app owns its
  own domain data. `livingiq-auth` relays a verified payment event to
  DealIQ, but DealIQ decides what that payment unlocks and stores that
  decision itself. `livingiq-auth` never knows or cares what a `dealId`
  actually is.

---

## 2. Workflow 1 — Login handoff

1. User clicks an app card on `livingiqweb.com`
2. Redirected to `auth.livingiqweb.com/login?redirect_to=<app-domain>`
3. User completes OTP verification (Twilio)
4. `livingiq-auth` sets `liq_access` + `liq_refresh` cookies, scoped to
   `.livingiqweb.com` — this domain scoping is what lets every LivingIQ
   subdomain read the same session without a second login
5. Redirected back to the originating app, already authenticated

**Layers touched:** Browser, `livingiq-auth`, Twilio, Neon (auth). DealIQ
itself is not involved in this workflow at all — it only ever *validates*
the resulting JWT locally, using a shared secret, never calling
`livingiq-auth` to check.

---

## 3. Workflow 2 — Checkout and payment confirmation

1. User clicks "Unlock" on a DealIQ report → `POST /api/payments/checkout/:reportCode`
2. DealIQ's backend calls `livingiq-auth`'s `POST /api/payments/checkout`
   (internal key, server-to-server, never a browser call) — `appName` is
   stamped into the metadata server-side, so DealIQ can't spoof another
   app's identity
3. `livingiq-auth` creates a Dodo checkout session, returns the
   `checkoutUrl` — browser redirects there
4. User pays on Dodo's hosted checkout (IP detection + currency
   localization happen here — Adaptive Currency, zero custom code)
5. Dodo sends its webhook to `livingiq-auth` (the *only* registered
   endpoint, regardless of how many LivingIQ apps exist)
6. Signature verified → event logged to `liq_payment_events`
   **before** anything else happens, so it's never lost even if DealIQ is
   down at this exact moment
7. Event relayed to DealIQ's `POST /api/internal/payment-events`, with
   retries (1s / 3s / 8s backoff) — decoupled from Dodo's own webhook
   response, so a slow/down app can never make Dodo think the webhook
   failed
8. DealIQ marks the payment `succeeded` in its own `payments` table

**If the relay hasn't landed by the time the user returns from checkout:**
`report.html` polls its own state a few times (~8s total), then falls
back to `GET /api/payments/recheck/:reportCode`, which asks
`livingiq-auth` to check with Dodo directly and self-heals DealIQ's row.

**If that recheck itself is inconclusive** (network failure, ambiguous
Dodo status) — the UI explicitly does *not* assume "safe to pay again". It
shows a distinct warning and a "check again" action instead of silently
re-offering checkout.

**Layers touched:** Browser, DealIQ, `livingiq-auth`, both Neon databases,
Dodo. Twilio and Claude are not involved in this workflow at all.

---

## 4. Use case × layer matrix

| Use case | Browser | DealIQ | livingiq-auth | Neon (dealiq) | Neon (auth) | Dodo | Twilio | Claude API |
|---|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
| Login (OTP) | ✓ | – | ✓ | – | ✓ | – | ✓ | – |
| Submit job offer | ✓ | ✓ | – | ✓ | – | – | – | ✓ |
| View free report | ✓ | ✓ | – | ✓ | – | – | – | – |
| Start checkout | ✓ | ✓ | ✓ | ✓ | – | ✓ | – | – |
| Payment webhook + relay | – | ✓ | ✓ | ✓ | ✓ | ✓ | – | – |
| Uncertain-payment recheck | ✓ | ✓ | ✓ | ✓ | – | ✓ | – | – |
| Retry failed analysis | ✓ | ✓ | – | ✓ | – | – | – | ✓ |
| Manual event replay (admin) | – | ✓ | ✓ | ✓ | ✓ | – | – | – |

**What this table confirms about the architecture:**
- `livingiq-auth` and Twilio only ever appear together — nothing that
  touches Twilio ever also touches Dodo directly, and vice versa
- Claude API only appears in the two pipeline-running use cases
  (submit, retry) — payment machinery never calls Claude, analysis never
  calls Dodo
- "Payment webhook + relay" is the busiest row (6 of 8 layers) — the one
  workflow that's inherently cross-service, and correspondingly the one
  most worth monitoring
- The browser is absent from exactly two rows: the webhook (Dodo calls
  `livingiq-auth` directly) and manual replay (an admin curl command, not
  product UI)

---

## 5. Failure handling, by layer

| Failure | Handled by | Mechanism |
|---|---|---|
| Invalid webhook signature | `livingiq-auth` → `dodo-webhook.js` | Rejects with 400 |
| Duplicate webhook delivery | `livingiq-auth` → `relay.js` `logEvent()` | `dodo_event_id` unique constraint, `ON CONFLICT DO NOTHING` |
| DealIQ unreachable during relay | `livingiq-auth` → `relay.js` `relayWithRetries()` | 3 retries, 1s/3s/8s backoff; event already logged before this starts |
| Hanging outbound call (Dodo or relay) | `livingiq-auth` & `dealiq` → `fetchWithTimeout.js` | Hard 10s timeout via `AbortController` |
| DealIQ's DB write fails on relay receipt | `dealiq` → `internal.js` | Returns 500 → tells the relay to retry |
| Uncaught exception / unhandled rejection | Each service independently | `process.on(...)` handlers log and exit; Render/Vercel auto-restarts |
| Unhandled route error | Each service independently | Last-resort Express error middleware |
| All 3 relay retries exhausted | Manual, via `livingiq-auth` | `GET /api/payments/events/failed` + `POST /api/payments/events/:id/replay` |
| Pipeline pass fails (extraction/benchmark/risk/verdict) | `dealiq` → `deals.js` | Labeled error (`"extraction pass failed: ..."`), deal marked `failed`, user-facing retry available |
| Recheck itself can't confirm status | `dealiq` → `payments.js` `/recheck` | Returns `{ paid: false, confirmed: false }` — UI shows a distinct warning, never implies safe to pay again |

**The one known, deliberately deferred gap:** if all 3 relay retries fail
and nobody happens to revisit the report page, nothing automatically
retries later — no scheduled sweep exists yet. Money is never at risk
(Dodo and `liq_payment_events` both already have the record); fulfillment
can silently stall until the manual replay endpoint is used. Worth
building an automatic sweep once real payment volume makes this a live
risk rather than a theoretical one.

---

## 6. Open items before going live

- [ ] Confirm Dodo's actual payment-status string values (`succeeded`,
      `failed`, `cancelled`, `expired`, or different) against real test-mode
      responses — the `/recheck` terminal-state check currently assumes
      these without verified documentation
- [ ] Register the single Dodo webhook endpoint:
      `https://auth.livingiqweb.com/api/payments/webhooks/dodo`
- [ ] Enable Adaptive Currency in the Dodo dashboard (Settings → Business)
- [ ] Create DealIQ's Product in Dodo, set `DODO_DEALIQ_PRODUCT_ID`
- [ ] Confirm `JWT_SECRET` and `INTERNAL_APP_API_KEY` match exactly across
      both services' environments
- [ ] Add DealIQ's relay URL to `livingiq-auth/server/lib/appRegistry.js`
      (already done in code — confirm it's deployed)
- [ ] Test the full loop once in Dodo test mode before flipping to live:
      login → submit → checkout → webhook → relay → unlock
