# Task 03 — Integration Architecture

## Data flow

```
Browser
   ↓  (form submit / dataLayer event)
Landing Page
   ↓  HTTPS POST /api/leads
Backend API  (Node.js / AWS Lambda behind API Gateway)
   ↓  search by phone
HubSpot CRM Search API
   ↓  create or update
HubSpot Contact  (upsert)
   ↓  publish LeadCreated
Message Queue  (SQS + Dead Letter Queue)
   ↓  consumer
Karix REST API  (WhatsApp confirmation, ≤2 min SLA)
   ↓
GA4  (server-side booking_confirmed)
   ↓  conversion import
Google Ads
```

## The 350-word design

OrthoNow's lead pipeline runs through a thin **custom backend API** rather than an embedded HubSpot
form or a no-code connector. A custom backend is preferred because the flow needs deduplication
logic, retries, an SLA-bound WhatsApp step and idempotency — orchestration a HubSpot Forms embed
cannot express and that **Zapier/Make** hide behind opaque, hard-to-monitor steps with per-task
pricing and no dead-letter control. Owning the backend means owning observability and failure
handling, which is the whole game in a lead system.

The trap is HubSpot's deduplication. **HubSpot deduplicates contacts by email, not phone.** Our
audience books by phone and many omit email, so blindly creating contacts would spawn duplicates. The
backend therefore first calls the **CRM Search API filtering on the `phone` property**; if a contact
is found it **updates**, otherwise it **creates**. When two different people share one phone (a
couple, a shared family line), search returns an existing record and we'd overwrite the wrong name. We
avoid silent corruption by treating phone as non-unique: on a name mismatch we don't overwrite — we
create a new contact, associate both to the shared phone, and flag `needs_review` so the **operations
team** merges or separates them in HubSpot using booking context. Data integrity beats false
tidiness.

**Failure point — HubSpot outage.** The upsert is wrapped in exponential-backoff retries; on
exhaustion the message lands in a **Dead Letter Queue** rather than being lost. An **idempotency key**
(`booking_id`) makes retries safe — no duplicate contacts or double WhatsApp sends. Everything is
logged with correlation ids and monitored.

**WhatsApp SLA.** Confirmations must send within **two minutes**. Karix calls run through the queue
so a spike, rate-limit, API outage or latency can't block the request thread; consumers retry with
backoff. **CloudWatch/Grafana** track queue age and send latency, alerting if p95 nears the SLA.

**Google Ads.** The page pushes to the `dataLayer`; **GTM → GA4** records `booking_confirmed`, which
is **imported into Google Ads** as the conversion. This beats a direct Ads tag: one deduplicated
source of truth, server-side reliability, and value-based bidding.

---

## Notes on the design decisions

### <a id="google-ads-attribution"></a>Why GA4-imported conversion beats a direct Google Ads tag

A hard-coded Google Ads conversion tag on the confirmation page double-counts on refresh, can't see
server-side bookings, and creates a *second*, divergent definition of "a booking" that never
reconciles with GA4. Routing **dataLayer → GTM → GA4 → Google Ads import** gives one authoritative
definition, deduplication on `booking_id`, resilience (the conversion can be sent server-side even if
the browser closes), and access to GA4's richer attribution. Marketing and analytics argue about one
number instead of two.

### Edge-case matrix — shared / ambiguous phone numbers

| Scenario | Search-by-phone result | System behaviour | Ops resolution |
|----------|------------------------|------------------|----------------|
| New number | 0 matches | Create contact | none |
| Returning patient, same name | 1 match | Update contact | none |
| Same phone, **different name** | 1 match | Create new contact, associate to phone, flag `needs_review` | Ops decides merge vs. keep-separate using booking history |
| >1 match already exists | 2+ matches | Attach booking to most-recently-active; flag `needs_review` | Ops merges duplicates |

### Idempotency & the Dead Letter Queue

Every lead carries a `booking_id`. The HubSpot upsert and the Karix send both key on it, so a retried
message is a no-op if the work already succeeded. Messages that exhaust retries move to a **DLQ** with
full context; an alarm on DLQ depth pages on-call, and messages can be replayed once HubSpot recovers
— zero lead loss, zero duplicates.
