# Task 03 — Integration Architecture

> Answers to the Task 3 brief, question by question. Q1 (end-to-end architecture and the connector
> choice) is below; further questions are appended as they're answered.

---

## Question 1 — How would you architect this integration end-to-end?

### Decision first: a direct API call to our own backend

Of the options, I'd build a **custom backend that the landing page calls directly (HTTPS POST), which
then talks to HubSpot, the queue and Karix via their REST APIs** — not the HubSpot native embed, not
the Forms API, not Zapier/Make. The reason in one line: **every hard requirement in this brief —
deduplicate by phone, a 2-minute WhatsApp SLA, idempotency, no lost leads — is orchestration logic
that only a backend I control can express.** A connector hides exactly the machinery I need to own.

### The end-to-end flow, in order, with the actual tech at each step

```
Browser  (OrthoNow booking page, vanilla JS — Task 2)
   │
   ├─(a) dataLayer.push ─► GTM ─► GA4 ─► Google Ads (conversion import)      [measurement path]
   │
   └─(b) HTTPS POST /api/leads (JSON)                                        [fulfilment path]
        ▼
   AWS API Gateway   (TLS, throttling, WAF)
        ▼
   Backend orchestrator — AWS Lambda (Node/TypeScript)   [or ECS Fargate + Express]
        │  1. validate + normalise phone to E.164
        │  2. write raw lead to DynamoDB FIRST (write-ahead → nothing is ever lost)
        │  3. assign idempotency key (booking_id)
        ▼
   HubSpot CRM API v3
        │  4. POST /crm/v3/objects/contacts/search   → filter by phone
        │  5. found? PATCH (update) : POST (create)
        ▼
   Amazon SQS  (+ Dead Letter Queue)   ── publish "SendWhatsApp", then return 200 to the browser
        ▼
   SQS consumer  (AWS Lambda)
        ▼
   Karix WhatsApp Business REST API   — send confirmation template   (must complete ≤ 2 min)
        ▼
   CloudWatch + Grafana   — metrics, p95 send latency, queue age, alarms
```

### Why that order matters

1. **Measurement and fulfilment split at the browser** — a slow CRM never delays the analytics event, and an ad-blocker on GTM never loses the lead.
2. **Persist the raw lead before calling anyone.** The DynamoDB write-ahead step means that even if HubSpot *and* Karix are both down, the lead still exists and can be replayed. This is the single most important reliability decision.
3. **HubSpot upsert is synchronous** (it's fast; the browser can wait ~300 ms), but **the WhatsApp send is queued** — Karix latency, rate limits or an outage must not block the user's request or start eating the SLA clock. The API returns `200` the instant the message is enqueued.
4. **DLQ + idempotency (`booking_id`)** make every downstream step safely retryable — no duplicate contacts, no double WhatsApp sends.

### The connector justification — why not the alternatives

| Option | Why I rejected it |
|--------|-------------------|
| **HubSpot native form embed** | Posts straight from the browser to HubSpot. Zero control over dedup (HubSpot dedupes by **email**, our audience is **phone-first**), nowhere to enqueue the WhatsApp step or enforce the 2-minute SLA, no idempotency/retry/DLQ, and the lead exists *only* in HubSpot — if HubSpot is down, it's gone. Fine for a brochure page; wrong for an orchestrated pipeline. |
| **HubSpot Forms API** (server-side submit to HubSpot's forms endpoint) | Better than the embed, but still makes HubSpot the orchestrator. It doesn't solve phone-dedup (still email-keyed), gives us no queue/SLA/DLQ machinery, and couples our success path to HubSpot's uptime. |
| **Zapier / Make** | Opaque, per-task pricing that balloons at lead volume; clumsy control over retries and dead-lettering; not version-controlled or unit-testable; and you can't get CloudWatch-grade metrics to defend a 2-minute SLA. Layering custom dedup-by-phone + idempotency into a no-code tool is fragile. Great for a prototype, not a healthcare lead system with an SLA. |
| ✅ **Direct API call → custom backend** | We own validation, phone normalisation, dedup-by-phone, idempotency, the queue, retries, DLQ, monitoring and the SLA. It's testable, version-controlled, observable, and the lead is durably ours regardless of any one vendor's uptime. The only cost is running a small service — cheap next to the requirements it satisfies. |

### Named stack, at a glance

- **Frontend:** the Task 2 vanilla-JS page → `POST /api/leads`.
- **Edge / API:** AWS API Gateway → AWS Lambda (Node/TypeScript). (ECS Fargate + Express if a long-running service is preferred.)
- **Durable store:** DynamoDB (write-ahead lead log) — or RDS Postgres.
- **CRM:** HubSpot CRM API v3 (Search + Create/Update contacts) via a private-app token.
- **Queue:** Amazon SQS + dedicated Dead Letter Queue; idempotency keyed on `booking_id`.
- **WhatsApp:** Karix WhatsApp Business REST API (template message).
- **Analytics / Ads:** GTM → GA4 → Google Ads conversion import (optionally GA4 Measurement Protocol server-side for reliability).
- **Ops:** CloudWatch alarms + Grafana dashboards; structured logs with correlation IDs; AWS Secrets Manager for the HubSpot token and Karix key.

---

## Condensed design (≈350 words)

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

## Supporting notes

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
