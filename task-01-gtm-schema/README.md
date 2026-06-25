# Task 01 — GTM / GA4 Event Schema

The measurement plan for the OrthoNow consultation campaign. Read the three documents in this order:

1. **[GTM_Event_Schema.md](GTM_Event_Schema.md)** — the full event table (13 events), naming
   conventions, trigger-type explanations, and why GTM can't auto-detect a multi-step form.
2. **[Booking_Funnel.md](Booking_Funnel.md)** — the 3-step booking funnel instrumented end to end:
   user action → frontend code → GTM listener → GA4 tag → captured variables, with **real
   `dataLayer` JSON** for every step and a worked GA4 Funnel Exploration (drop-off analysis).
3. **[Google_Ads_Conversion.md](Google_Ads_Conversion.md)** — why only `booking_confirmed` is
   imported to Google Ads and why call / WhatsApp / PDF / clinic-view / scroll are deliberately not.

## The one thing to take away

The schema is built on one design decision: **the developer owns a small, structured `dataLayer`;
GTM owns the tags.** Events describe *what happened* (`booking_step_complete`), parameters describe
*context* (`step_number`, `clinic_location`). That keeps the event list short, lets GA4's Funnel
Exploration do the sequencing, and lets marketing change GA4/Ads tags without a code deploy.

## Assets

`assets/` holds the funnel and event-flow diagrams referenced by the docs. Export them as:

- `ga4-funnel-diagram.png` — the 5-step Funnel Exploration view.
- `event-flow-diagram.png` — dataLayer → GTM → GA4 → Google Ads flow.
