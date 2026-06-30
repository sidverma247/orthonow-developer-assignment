# Assignment Overview

## The brief

**Company.** OrthoNow — a multi-city orthopaedic clinic group (healthcare).

**Campaign.** *"Get an expert orthopaedic opinion — book your consultation at OrthoNow."*

**Audience.** Working professionals, age 28–50, in **Bengaluru, Hyderabad and Chennai**.

**Pain points.** Knee pain, back pain, neck pain, sports injuries.

**Primary goal.** Generate consultation bookings.

## What was asked, and where it lives

| Task | Deliverable | Location |
|------|-------------|----------|
| 1 | GTM event schema + booking funnel + Google Ads conversion choice | [`task-01-gtm-schema/`](../task-01-gtm-schema/) |
| 2 | One self-contained landing page (HTML/CSS/JS) with real `dataLayer` tracking | [`task-02-landing-page/index.html`](../task-02-landing-page/index.html) |
| 3 | ~350-word integration architecture (backend → HubSpot → Karix → GA4 → Ads) | [`task-03-integration-design/`](../task-03-integration-design/) |

## How each task maps to the business goal

The campaign spends money to buy clicks; the only thing that matters is turning those clicks into
booked consultations and knowing which spend worked. The three deliverables are the three links in
that chain:

1. **Task 2 converts** the click — a fast, trustworthy, low-friction page whose single job is to get
   a working professional to book, and which emits a clean tracking event when they do.
2. **Task 1 measures** the conversion — a funnel schema that shows exactly where prospective patients
   drop, which channel/clinic performs, and what to feed Google Ads so bidding chases real bookings.
3. **Task 3 fulfils** the conversion — a resilient backend that lands the lead in HubSpot without
   duplicates, sends a WhatsApp confirmation inside 2 minutes, and reports the conversion back to
   Google Ads through GA4.

Together they close the loop: *acquire → convert → measure → fulfil → optimise.*

## Design constraints honoured

- **Landing page:** no frameworks, no libraries, no CDN, no external fonts/images, no server. One file.
- **Analytics:** the `dataLayer` push fires only after successful validation, never on load.
- **Integration:** custom backend (not Zapier/Make), HubSpot dedup-by-phone trap explicitly handled,
  2-minute WhatsApp SLA, idempotency + DLQ, GA4-imported Ads conversion.
