# Task 03 — Integration Architecture

> Answers to the Task 3 brief, question by question. Q1 (end-to-end architecture and the connector
> choice) is below; further questions are appended as they're answered.

---

## Question 1 — How would you architect this integration end-to-end?

### Decision first: a direct API call to our own backend

For OrthoNow I'd build a **custom backend that our booking page calls directly (HTTPS POST), which
then talks to HubSpot, the queue and Karix via their REST APIs** — not the HubSpot native embed, not
the Forms API, not Zapier/Make. The reason in one line, grounded in *this* project: **our audience is
28–50 working professionals who book a ₹800 orthopaedic consultation by phone (many never give an
email), against a 2-minute WhatsApp SLA — so dedup-by-phone, idempotency and no-lost-leads are the
whole job, and that is orchestration logic only a backend I control can express.** A connector hides
exactly the machinery this project lives or dies on.

### The end-to-end flow, in order, with the actual tech at each step

Everything runs in **AWS `ap-south-1` (Mumbai)** — lowest latency to the Bengaluru / Hyderabad /
Chennai audience and keeps health-related personal data in-region (relevant under India's **DPDP Act**).

```
Browser  (OrthoNow booking page, vanilla JS — our Task 2 index.html)
   │  form: name · 10-digit mobile · clinic (9 across 3 cities) · specialty · preferred date
   │
   ├─(a) dataLayer.push: consultation_form_submitted + booking_confirmed
   │        ─► GTM ─► GA4 ─► Google Ads (import booking_confirmed)           [measurement path]
   │
   └─(b) HTTPS POST /api/leads  (the same validated payload)                 [fulfilment path]
        ▼
   AWS API Gateway   (TLS, throttling, AWS WAF)
        ▼
   Backend orchestrator — AWS Lambda (Node/TypeScript), region ap-south-1
        │  1. re-validate; normalise "9812345678" → E.164 "+919812345678"  (the dedup key)
        │  2. write raw lead to DynamoDB FIRST, PK = booking_id  (write-ahead → never lost)
        │  3. booking_id "ORTHO-2026-000482" is the idempotency key end-to-end
        ▼
   HubSpot CRM API v3
        │  4. POST /crm/v3/objects/contacts/search  → filter on phone = "+919812345678"
        │  5. found? PATCH (update) : POST (create)   [dedup-by-phone — see Q2 / edge-case matrix]
        ▼
   Amazon SQS  (+ Dead Letter Queue)  ── publish "SendWhatsApp", then return 200 to the browser
        ▼
   SQS consumer  (AWS Lambda)
        ▼
   Karix WhatsApp Business REST API  — send an approved confirmation template
        │  "Hi Ananya, your OrthoNow consultation at Indiranagar on 9 Jul is being confirmed…"
        ▼   (must complete ≤ 2 min)
   CloudWatch + Grafana  — metrics, p95 send latency, queue age, DLQ depth, alarms
```

**Concrete payload our page sends to `/api/leads`** (derived directly from the Task 2 form + the
`booking_confirmed` dataLayer event):

```json
{
  "booking_id": "ORTHO-2026-000482",
  "name": "Ananya Rao",
  "phone": "9812345678",
  "clinic_location": "Bengaluru — Indiranagar",
  "specialty": "Knee pain",
  "appointment_date": "2026-07-09",
  "value": 800,
  "currency": "INR",
  "source": "consultation_landing_page"
}
```

### Why that order matters (for OrthoNow specifically)

1. **Measurement and fulfilment split at the browser** — a slow HubSpot call never delays the GA4 `booking_confirmed` event, and an ad-blocker on GTM never loses the actual lead, because the lead goes to our own `/api/leads`, not to a third-party tag.
2. **Persist the raw lead before calling anyone.** DynamoDB write-ahead (keyed on `booking_id`) means even if HubSpot *and* Karix are both down, the booking still exists and is replayable — non-negotiable when the "lead" is a patient expecting a call back.
3. **HubSpot upsert is synchronous** (fast; the browser waits ~300 ms), but **the Karix WhatsApp send is queued** — a Karix spike, +91 template rate-limit or outage must not block the request or start eating the 2-minute SLA. The API returns `200` the instant the message is enqueued.
4. **`booking_id` is one idempotency key across the whole chain** — DynamoDB PK, HubSpot search fallback, SQS dedup and the Karix send — so a retry or a double-tap on "Confirm" can't create a duplicate contact or a second WhatsApp.

### Why Karix (and not a generic global provider)

Karix is an India-based CPaaS with a mature WhatsApp Business API and strong local deliverability for
`+91` numbers — the right fit for an all-India (Bengaluru/Hyderabad/Chennai) patient base where
template approval, sender registration and latency are all India-specific. The architecture treats it
as a swappable REST integration behind the queue, so a provider change never touches the rest of the
pipeline.

### The connector justification — why not the alternatives

| Option | Why I rejected it |
|--------|-------------------|
| **HubSpot native form embed** | Posts straight from the browser to HubSpot. Zero control over dedup (HubSpot dedupes by **email**, our audience is **phone-first**), nowhere to enqueue the WhatsApp step or enforce the 2-minute SLA, no idempotency/retry/DLQ, and the lead exists *only* in HubSpot — if HubSpot is down, it's gone. Fine for a brochure page; wrong for an orchestrated pipeline. |
| **HubSpot Forms API** (server-side submit to HubSpot's forms endpoint) | Better than the embed, but still makes HubSpot the orchestrator. It doesn't solve phone-dedup (still email-keyed), gives us no queue/SLA/DLQ machinery, and couples our success path to HubSpot's uptime. |
| **Zapier / Make** | Opaque, per-task pricing that balloons at lead volume; clumsy control over retries and dead-lettering; not version-controlled or unit-testable; and you can't get CloudWatch-grade metrics to defend a 2-minute SLA. Layering custom dedup-by-phone + idempotency into a no-code tool is fragile. Great for a prototype, not a healthcare lead system with an SLA. |
| ✅ **Direct API call → custom backend** | We own validation, phone normalisation, dedup-by-phone, idempotency, the queue, retries, DLQ, monitoring and the SLA. It's testable, version-controlled, observable, and the lead is durably ours regardless of any one vendor's uptime. The only cost is running a small service — cheap next to the requirements it satisfies. |

### Named stack, at a glance

- **Frontend:** our Task 2 vanilla-JS booking page → `POST /api/leads` (name · +91 mobile · clinic · specialty · date).
- **Edge / API:** AWS API Gateway + WAF → AWS Lambda (Node/TypeScript), region **ap-south-1 (Mumbai)**.
- **Durable store:** DynamoDB (write-ahead lead log), partition key `booking_id`.
- **CRM:** HubSpot CRM API v3 (Search + Create/Update contacts) via a private-app token; searched on the **phone** property in E.164.
- **Queue:** Amazon SQS + dedicated Dead Letter Queue; idempotency keyed on `booking_id`.
- **WhatsApp:** Karix WhatsApp Business REST API (approved confirmation template).
- **Analytics / Ads:** GTM → GA4 → Google Ads import of `booking_confirmed` (optionally GA4 Measurement Protocol server-side for reliability).
- **Ops / security:** CloudWatch alarms + Grafana dashboards; structured logs with correlation IDs; AWS Secrets Manager for the HubSpot token and Karix key; in-region data residency for DPDP.

### Analysis — why this design fits OrthoNow (trade-offs and risks)

**Why it fits.** Every choice tracks a specific fact about this campaign. The audience is *phone-first
and email-optional*, so **dedup must key on phone**, which HubSpot won't do natively — that single
fact is what rules out the embed and the Forms API. A booked consultation is *worth ₹800 and a
patient's trust*, so **losing a lead is unacceptable** — hence the DynamoDB write-ahead and the DLQ.
The confirmation is *promised within 2 minutes over WhatsApp to +91 numbers*, so **the send is queued
and monitored** and the provider (Karix) is India-appropriate. The data is *health-related personal
data of Indian residents*, so **ap-south-1 residency and secrets management** matter under the DPDP
Act.

**The honest trade-off.** A custom backend is more to build and run than a Zapier zap or a HubSpot
embed — we own a Lambda, a queue, a table and their monitoring. That cost is justified here because
the requirements (phone-dedup, SLA, zero lead loss) are *exactly* the things no-code tools do poorly;
for a lower-stakes newsletter signup I'd happily use the embed. Scale is not the driver — a few
hundred leads a day is trivial for this stack — **reliability and control are**.

**Residual risks and how the design absorbs them.** HubSpot outage → retries then DLQ + replay (lead
already safe in DynamoDB). Karix outage or template rate-limit → queued, retried with backoff, alarmed
on queue age before the SLA is breached. Shared/ambiguous phone numbers → never overwrite; create +
associate + `needs_review` for ops (see the edge-case matrix). Duplicate submits / refreshes →
`booking_id` idempotency makes them no-ops. The one thing outside our control — a *user* mistyping
their own number — is mitigated by the page's `^[6-9]\d{9}$` validation and the WhatsApp confirmation
itself acting as a delivery check.

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
