# Task 03 — Integration Design

Connecting the OrthoNow consultation landing page to HubSpot, WhatsApp (Karix) and Google Ads. On a
form submit, three things must happen automatically: create-or-update the HubSpot contact, send a
WhatsApp confirmation within 2 minutes, and fire the `consultation_form_submitted` Google Ads
conversion. The 300–400 word answer is below; supporting detail (diagram, payload, edge cases)
follows as an appendix.

---

## The answer (≈370 words)

**How I'd architect it.** The landing page POSTs the form to a small backend of our own — an AWS
Lambda behind API Gateway in ap-south-1 — not a HubSpot native embed, Forms API, or Zapier/Make. This
flow needs real logic (phone-based dedup, retries, a 2-minute SLA, idempotency) that an embed can't
express and that Zapier/Make bury behind opaque, hard-to-monitor steps. Order: browser →
our `/api/leads` → **(1)** upsert the HubSpot contact via the CRM API, **(2)** drop a message on an
SQS queue that a worker uses to call Karix's WhatsApp API, **(3)** fire the Google Ads conversion. For
Ads I'd route the `consultation_form_submitted` event through dataLayer → GTM → GA4 and import it, so
there's one clean definition rather than a brittle on-page tag.

The HubSpot step is where the actual thinking is. **HubSpot deduplicates on email — not phone — and
we don't collect email**, so a naive "create contact" would duplicate every returning patient.
Instead the backend first calls the CRM **Search API on the phone property**; match → update, no
match → create, setting Name, Phone, Clinic Preference, Source = 'Google Ads - Consultation Landing
Page', and Lead Status = 'New Enquiry'. If two people submit the same phone with different names, I
don't overwrite the existing name — I create a separate contact, associate both to that number, and
flag `needs_review` for ops. Silently merging two patients is worse than a duplicate.

**Biggest failure point + fallback.** The synchronous dependency on HubSpot — if it's slow or down,
leads vanish. Fallback: write the raw lead to our own store *first* (write-ahead), make the CRM and
WhatsApp calls asynchronous through the queue with exponential-backoff retries, and send exhausted
messages to a dead-letter queue for replay. Nothing is lost, and phone/`lead_id` idempotency stops
retries creating duplicates.

**The 2-minute WhatsApp SLA.** It breaks from Karix latency or outage, template rate-limits, an
unapproved template, or a queue backlog upstream. Since the send is queued, I timestamp at submit and
measure enqueue-to-delivered latency, with CloudWatch/Grafana alarming on p95 and queue age *before*
the 2 minutes, a DLQ-depth alarm, and Karix delivery receipts confirming actual delivery — not just
"sent."

---

## Appendix — supporting detail

### End-to-end data flow

Everything runs in **AWS `ap-south-1` (Mumbai)** — lowest latency to the Bengaluru / Hyderabad /
Chennai audience, and keeps health-related personal data in-region (relevant under India's DPDP Act).

```
Browser  (OrthoNow booking page — our Task 2 index.html)
   │  form: Name · 10-digit mobile · Clinic Preference
   │
   ├─(a) dataLayer.push: consultation_form_submitted
   │        ─► GTM ─► GA4 ─► Google Ads (import consultation_form_submitted)   [measurement path]
   │
   └─(b) HTTPS POST /api/leads  (same validated payload)                       [fulfilment path]
        ▼
   AWS API Gateway  (TLS, throttling, AWS WAF)
        ▼
   Backend orchestrator — AWS Lambda (Node/TypeScript), ap-south-1
        │  1. re-validate; normalise "9812345678" → E.164 "+919812345678"  (the dedup key)
        │  2. write raw lead to DynamoDB FIRST, PK = lead_id  (write-ahead → never lost)
        │  3. lead_id is the idempotency key end-to-end
        ▼
   HubSpot CRM API v3
        │  4. POST /crm/v3/objects/contacts/search  → filter on phone = "+919812345678"
        │  5. found? PATCH (update) : POST (create)
        │     set: Name, Phone, Clinic Preference,
        │          Source = "Google Ads - Consultation Landing Page",
        │          Lead Status = "New Enquiry"
        ▼
   Amazon SQS  (+ Dead Letter Queue)  ── publish "SendWhatsApp", then return 200 to the browser
        ▼
   SQS consumer  (AWS Lambda)
        ▼
   Karix WhatsApp Business API  — send an approved confirmation template
        │  "Hi Ananya, we've received your OrthoNow enquiry for Indiranagar. A coordinator
        │   will call you shortly to confirm your consultation."
        ▼   (must complete ≤ 2 min)
   CloudWatch + Grafana  — p95 send latency, queue age, DLQ depth, alarms
```

### Concrete payload our page sends to `/api/leads`

Derived directly from the Task 2 form and the `consultation_form_submitted` dataLayer event:

```json
{
  "lead_id": "ORTHO-LEAD-2026-000482",
  "name": "Ananya Rao",
  "phone": "9812345678",
  "clinic_preference": "Bengaluru — Indiranagar",
  "source": "Google Ads - Consultation Landing Page",
  "lead_status": "New Enquiry",
  "submission_time": "2026-07-03T09:41:12.874Z"
}
```

### Why a direct API call, not the alternatives

| Option | Why rejected |
|--------|--------------|
| **HubSpot native embed** | Posts browser → HubSpot directly. No control over phone-dedup (HubSpot keys on email; we collect none), nowhere to enqueue the WhatsApp step or enforce the 2-min SLA, no retry/DLQ, and the lead exists only in HubSpot. |
| **HubSpot Forms API** | Still makes HubSpot the orchestrator; doesn't solve phone-dedup and gives no queue/SLA/DLQ machinery. |
| **Zapier / Make** | Opaque, per-task pricing; weak retry/dead-letter control; no CloudWatch-grade metrics to defend a 2-minute SLA; fragile to add custom dedup + idempotency. |
| ✅ **Direct API → custom backend** | We own validation, phone normalisation, dedup, idempotency, the queue, retries, DLQ, monitoring and the SLA — and the lead is durably ours regardless of any vendor's uptime. |

### Named stack

- **Frontend:** our Task 2 vanilla-JS page → `POST /api/leads`.
- **Edge / API:** AWS API Gateway + WAF → AWS Lambda (Node/TypeScript), **ap-south-1**.
- **Durable store:** DynamoDB (write-ahead lead log), PK `lead_id`.
- **CRM:** HubSpot CRM API v3 (Search + Create/Update) via a private-app token; searched on **phone** in E.164.
- **Queue:** Amazon SQS + dedicated Dead Letter Queue; idempotency keyed on `lead_id`.
- **WhatsApp:** Karix WhatsApp Business API (approved confirmation template).
- **Analytics / Ads:** GTM → GA4 → Google Ads import of `consultation_form_submitted`.
- **Ops / security:** CloudWatch alarms + Grafana; correlation-ID logging; AWS Secrets Manager for the HubSpot token and Karix key; in-region residency for DPDP.

### Edge-case matrix — shared / ambiguous phone numbers (the dedup trap)

| Scenario | Search-by-phone result | System behaviour | Ops resolution |
|----------|------------------------|------------------|----------------|
| New number | 0 matches | Create contact | none |
| Returning patient, same name | 1 match | Update contact | none |
| Same phone, **different name** | 1 match | Create a **new** contact, associate to the phone, flag `needs_review` | Ops decides merge vs. keep-separate from booking history |
| >1 match already exists | 2+ matches | Attach to most-recently-active; flag `needs_review` | Ops merges duplicates |

### Idempotency & the Dead Letter Queue

Every lead carries a `lead_id`. The HubSpot upsert and the Karix send both key on it, so a retried
message is a no-op if the work already succeeded. Messages that exhaust retries move to a DLQ with
full context; an alarm on DLQ depth pages on-call, and messages replay once HubSpot/Karix recover —
zero lead loss, zero duplicates.

### Why GA4-imported conversion beats a direct Google Ads tag

A hard-coded Ads tag on the thank-you state double-counts on refresh, can't see server-side events,
and creates a second definition of "a lead" that never reconciles with GA4. Routing dataLayer → GTM →
GA4 → Google Ads import gives one authoritative, deduplicated definition of `consultation_form_submitted`
and lets bidding optimise toward it cleanly.
