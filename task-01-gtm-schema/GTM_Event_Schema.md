# Task 01 — GTM Event Schema

**OrthoNow consultation campaign — measurement plan.**
This document is the complete Task 1 deliverable in three parts:

1. **[The complete GTM event schema](#1-complete-gtm-event-schema)** — every event I would implement, as a structured table (Event Name · Trigger Type · Key Parameters · GA4 report/audience).
2. **[Booking funnel drop-off tracking](#2-booking-funnel-drop-off-3-step-form)** — for the 3-step form: the GTM trigger at each step, the **actual JSON** `dataLayer` push, and how the step-level drop-off is surfaced in GA4 Funnel Exploration.
3. **[The one Google Ads conversion to import](#3-google-ads-conversion-to-import)** — and why that one over the others.

All 13 events are implemented and observable in the [Task 2 landing page](../task-02-landing-page/index.html) — open the browser console and interact.

---

## 1. Complete GTM event schema

| # | Event Name | Trigger Type | Key Parameters (min. 3) | GA4 Report / Audience it feeds |
|---|------------|--------------|-------------------------|--------------------------------|
| 1 | `booking_started` | Custom Event (dataLayer) — booking widget opens | `entry_point`, `device_type`, `page_location` | **Report:** Funnel Exploration (entry step / denominator). **Audience:** *"Started booking — not confirmed"* → remarketing. |
| 2 | `booking_step_complete` | Custom Event (dataLayer) — each step's valid "Continue" | `step_number`, `step_name`, `clinic_location`, `specialty` | **Report:** Funnel Exploration (middle steps, drop-off). **Audience:** *"Reached step 2 / step 3"*. |
| 3 | `booking_confirmed` | Custom Event (dataLayer) — booking persisted (server returns `booking_id`) | `booking_id`, `clinic_location`, `specialty`, `value`, `currency` | **Report:** Conversions / Key Events + Funnel final step. **Audience:** *"Converters"* (exclude from prospecting). **→ imported Google Ads conversion.** |
| 4 | `booking_error` | Custom Event (dataLayer) — error/catch branch of booking flow | `error_type`, `error_message`, `step_number`, `clinic_location` | **Report:** Custom exploration + DebugView (diagnostic; alert on spikes). No audience. |
| 5 | `call_now_click` | Click — auto-event on `a[href^="tel:"]` | `link_url`, `link_location`, `page_location` | **Report:** Engagement → Events. **Audience:** *"High-intent callers"* → call-focused remarketing. |
| 6 | `whatsapp_click` | Click — auto-event on `a[href*="wa.me"]` | `link_url`, `link_location`, `page_location` | **Report:** Engagement → Events. **Audience:** *"WhatsApp responders"*. |
| 7 | `patient_guide_submit` | Custom Event (dataLayer) — guide gate form submit | `guide_name`, `pain_area`, `city` | **Report:** Events / Lead report. **Audience:** *"Guide leads — nurture"* (content remarketing). |
| 8 | `patient_guide_download` | Click / File download — link click on the `.pdf` | `file_name`, `guide_name`, `pain_area` | **Report:** Engagement → File downloads (confirms delivery vs. submit). |
| 9 | `clinic_location_view` | Element Visibility (≥50% of `.clinic-card`) or Custom Event | `clinic_location`, `city`, `page_location` | **Report:** Engagement by location (local budget). **Audience:** *"Interested in <city>"* → geo remarketing. |
| 10 | `blog_scroll_25` | Scroll Depth — auto-event, 25% (blog pages only) | `percent_scrolled`, `article_title`, `pain_area`, `page_path` | **Report:** Content engagement — separates bounces from readers. |
| 11 | `blog_scroll_50` | Scroll Depth — auto-event, 50% | `percent_scrolled`, `article_title`, `pain_area`, `page_path` | **Report:** Content engagement. **Audience:** *"Engaged readers"* → booking-CTA remarketing. |
| 12 | `blog_scroll_75` | Scroll Depth — auto-event, 75% | `percent_scrolled`, `article_title`, `pain_area`, `page_path` | **Report:** Content engagement. **Audience:** *"Deep readers"*. |
| 13 | `blog_scroll_90` | Scroll Depth — auto-event, 90% | `percent_scrolled`, `article_title`, `pain_area`, `page_path` | **Report:** Content engagement. **Audience:** *"Finished the article"* (highest-intent content audience). |

**Feature → events map:** Appointment booking (3 steps) = events 1–4 · Call Now buttons = 5 · WhatsApp widget = 6 · Patient Guide download = 7–8 · Clinic location views = 9 · Blog article reads = 10–13.

### Trigger types, and why each is used

- **Custom Event (dataLayer)** — events 1–4, 7. Fire on an explicit `dataLayer.push({ event: "..." })` from the developer. Used for anything GTM **cannot infer from the DOM**: a *completed* step, a *persisted* booking, a *validated* submit, an error branch. Only the app code knows a booking actually saved, so it announces it.
- **Click (auto-event)** — events 5, 6, 8. GTM's Click listener fires on elements matching a CSS selector / URL pattern (`tel:`, `wa.me`, `.pdf`). No developer code; marketing adds tracked links without a deploy.
- **Scroll Depth (auto-event)** — events 10–13. One listener, four thresholds, scoped to blog templates by a page-path condition; differentiated by `percent_scrolled`.
- **Element Visibility** — event 9. Fires when a clinic card crosses a visibility threshold — "was this clinic seen?" — with no developer push.

### Conventions

`snake_case`, verb-led event names (GA4 recommended style). Events describe *what happened*; parameters describe *context* (so it's **one** `booking_step_complete` event with a `step_number` parameter, not `booking_step_1`/`booking_step_2`). Every custom parameter is registered as a GA4 **event-scoped custom dimension** so it's queryable in Explorations. **No PII in GA4** — `user_name`/`phone_number` go to the CRM, never into GA4 parameters.

---

## 2. Booking funnel drop-off (3-step form)

The booking widget is a single-page interaction: no page loads between steps and no native per-step `submit` event. GTM can see a *click* on "Continue," but not whether it was *valid*, which *step* it completed, or what was *selected* — that state lives in JavaScript. So **the developer announces each step completion with a structured `dataLayer.push()`**; GTM listens, GA4 records, and Funnel Exploration sequences. That is exactly how step-level drop-off becomes measurable.

The three steps and confirmation:

```
Open widget          →  booking_started
Step 1: clinic+specialty →  booking_step_complete (step_number = 1)
Step 2: name+phone+date  →  booking_step_complete (step_number = 2)
Step 3: review & confirm →  booking_step_complete (step_number = 3)
Booking saved            →  booking_confirmed
```

### What fires at each step (GTM trigger + actual JSON)

**On widget open — `booking_started`.**
GTM trigger: **Custom Event**, condition `Event equals booking_started`. Fires the GA4 tag *"GA4 – booking_started"*. This is the funnel's denominator.

```json
{
  "event": "booking_started",
  "entry_point": "hero_cta",
  "device_type": "mobile",
  "page_location": "https://www.orthonow.example/book"
}
```

**Step 1 complete (location + specialty selected).**
GTM trigger: **Custom Event**, condition `Event equals booking_step_complete`. The step is identified by the `step_number` parameter — the same trigger handles all three steps.

```json
{
  "event": "booking_step_complete",
  "step_number": 1,
  "step_name": "location_specialty_selected",
  "clinic_location": "{{clinic name}}",
  "specialty": "{{specialty selected}}"
}
```

**Step 2 complete (contact details + preferred date entered).**
GTM trigger: the **same** `Event equals booking_step_complete` Custom Event trigger.

```json
{
  "event": "booking_step_complete",
  "step_number": 2,
  "step_name": "contact_and_date_entered",
  "clinic_location": "{{clinic name}}",
  "specialty": "{{specialty selected}}",
  "appointment_date": "{{YYYY-MM-DD}}"
}
```

**Step 3 complete (booking reviewed).**
GTM trigger: the **same** `Event equals booking_step_complete` Custom Event trigger.

```json
{
  "event": "booking_step_complete",
  "step_number": 3,
  "step_name": "booking_reviewed",
  "clinic_location": "{{clinic name}}",
  "specialty": "{{specialty selected}}",
  "appointment_date": "{{YYYY-MM-DD}}"
}
```

**Confirmation — `booking_confirmed` (the conversion).**
GTM trigger: **Custom Event**, condition `Event equals booking_confirmed`. Fired from the **success callback** of the booking API — using the server `booking_id` proves the record persisted and gives a dedup / order id.

```json
{
  "event": "booking_confirmed",
  "booking_id": "ORTHO-2026-000482",
  "clinic_location": "Indiranagar",
  "specialty": "Knee pain",
  "appointment_date": "2026-07-09",
  "value": 800,
  "currency": "INR"
}
```

> **Why one event with a `step_number` parameter (not three separate events)?** It keeps the schema small and lets GA4's Funnel Exploration define each step as a condition on that parameter — so adding or reordering steps never means creating new events or GTM triggers.

### Surfacing step-level drop-off in GA4 Funnel Exploration

In **GA4 → Explore → Funnel exploration**, build a **closed** funnel (each step must happen in order) with these five steps:

| Funnel step | Condition |
|-------------|-----------|
| 1. Booking started | event = `booking_started` |
| 2. Step 1 complete | event = `booking_step_complete` **AND** `step_number` = 1 |
| 3. Step 2 complete | event = `booking_step_complete` **AND** `step_number` = 2 |
| 4. Step 3 complete | event = `booking_step_complete` **AND** `step_number` = 3 |
| 5. Booking confirmed | event = `booking_confirmed` |

Add a **breakdown** by `clinic_location`, a **segment** by `device_type`, and enable *"Show elapsed time"* to spot steps where users stall. GA4 renders the completion and abandonment rate between every step.

**Worked example — where the funnel leaks:**

| Step | Event | Users | Step conversion | Drop-off | % of started |
|------|-------|------:|----------------:|---------:|-------------:|
| 1 | `booking_started` | 4,000 | — | — | 100% |
| 2 | Step 1 complete | 3,120 | 78.0% | 22.0% (880) | 78.0% |
| 3 | Step 2 complete | 1,950 | 62.5% | **37.5% (1,170)** | 48.8% |
| 4 | Step 3 complete | 1,560 | 80.0% | 20.0% (390) | 39.0% |
| 5 | `booking_confirmed` | 1,310 | 84.0% | 16.0% (250) | **32.8%** |

**How marketing acts on it:** the biggest single leak is **step 2 → step 3 (37.5%)** — the contact-details/date step. Broken down by `device_type` it's usually worse on mobile, pointing at a UX/keyboard problem or too few convenient dates rather than "spend more on ads." The final `booking_confirmed` drop (16%) is cross-referenced with `booking_error` to tell hesitation apart from a technical failure. The `clinic_location` breakdown shows which clinic under-converts so slot capacity and budget can be reallocated. Every fix is re-measurable in the same report.

---

## 3. Google Ads conversion to import

**Import exactly one conversion: `booking_confirmed`** — via the **GA4 → Google Ads conversion import** (not a hard-coded Ads tag).

**Why `booking_confirmed` and not the others:**

- **It is the business outcome.** The campaign goal is *consultation bookings*; `booking_confirmed` fires only when a booking is persisted (has a `booking_id`). It's the closest measurable event to revenue.
- **It carries value.** `value` + `currency` enable **value-based bidding (tROAS)**, so Google can favour higher-value clinics/specialties.
- **It's deduplicated.** `booking_id` acts as the order id, so refreshes/retries don't inflate the count.
- **It's high-fidelity, low-volume.** Smart Bidding learns fastest from a signal that reliably correlates with money.

**Why not the micro-conversions** (all valuable in GA4 for analysis/remarketing, but wrong as the bid target):

| Event | Why it's excluded as the Ads conversion |
|-------|-----------------------------------------|
| `call_now_click` / `whatsapp_click` | A tap is **intent, not outcome** — no proof the call/chat connected or booked. Bidding would optimise for cheap tappers. |
| `patient_guide_download` | Downloading a free PDF is **top-of-funnel curiosity** — far cheaper and more frequent than a booking; it would flood the account with guide-grabbers. |
| `clinic_location_view` | A **passive visibility** event — near-pure noise as a conversion. |
| `blog_scroll_25/50/75/90` | **Engagement micro-signals**, extremely high-volume and weakly correlated with booking; they'd drown the real conversion. |

> **Principle:** optimise bidding toward the outcome that makes money (`booking_confirmed`); use everything else as analysis and remarketing signals — never as the bid target. Importing `booking_confirmed` from GA4 (rather than a direct Ads tag) also gives one deduplicated source of truth, server-side reliability, and value-based bidding.
