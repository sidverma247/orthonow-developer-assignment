# Task 01 — Complete GTM Event Schema

> Every event I would implement for the OrthoNow consultation campaign, in one structured table.
> For each event: **Event Name**, **Trigger Type**, **Key Parameters** (minimum three per event), and
> **the GA4 report or audience it feeds**. The schema is built around one north-star metric —
> consultation bookings — with supporting micro-conversions that explain *why* bookings happen.

All 13 events are implemented and observable in the [Task 2 landing page](../task-02-landing-page/index.html)
(open the console and interact), and pre-wired in the importable
[`GTM_Container_OrthoNow.json`](GTM_Container_OrthoNow.json).

---

## The event schema

| # | Event Name | Trigger Type | Key Parameters (min. 3) | GA4 Report / Audience it feeds |
|---|------------|--------------|-------------------------|--------------------------------|
| 1 | `booking_started` | Custom Event (dataLayer) — booking widget opens | `entry_point`, `device_type`, `page_location` | **Report:** Funnel Exploration (entry step / denominator). **Audience:** *"Started booking — not confirmed"* → remarketing. |
| 2 | `booking_step_complete` | Custom Event (dataLayer) — each step's valid "Continue" | `step_number`, `step_name`, `clinic_location`, `specialty` | **Report:** Funnel Exploration (middle steps, drop-off). **Audience:** *"Reached step 2 / step 3"*. |
| 3 | `booking_confirmed` | Custom Event (dataLayer) — booking persisted (has `booking_id`) | `booking_id`, `clinic_location`, `specialty`, `value`, `currency` | **Report:** Conversions / Key Events + Funnel final step. **Audience:** *"Converters"* (exclude from prospecting). Also the **imported Google Ads conversion**. |
| 4 | `booking_error` | Custom Event (dataLayer) — error/catch branch of booking flow | `error_type`, `error_message`, `step_number`, `clinic_location` | **Report:** Custom exploration + DebugView (diagnostic; alert on spikes). No audience. |
| 5 | `call_now_click` | Click — auto-event on `a[href^="tel:"]` | `link_url`, `link_location`, `page_location` | **Report:** Engagement → Events. **Audience:** *"High-intent callers"* → remarketing / call-focused ads. |
| 6 | `whatsapp_click` | Click — auto-event on `a[href*="wa.me"]` | `link_url`, `link_location`, `page_location` | **Report:** Engagement → Events. **Audience:** *"WhatsApp responders"*. |
| 7 | `patient_guide_submit` | Custom Event (dataLayer) — guide gate form submit | `guide_name`, `pain_area`, `city` | **Report:** Events / Lead report. **Audience:** *"Guide leads — nurture"* (content remarketing). |
| 8 | `patient_guide_download` | Click / File download — link click on the `.pdf` | `file_name`, `guide_name`, `pain_area` | **Report:** Engagement → File downloads (confirms delivery vs. submit). |
| 9 | `clinic_location_view` | Element Visibility (≥50% of `.clinic-card`) or Custom Event | `clinic_location`, `city`, `page_location` | **Report:** Engagement by location (local budget). **Audience:** *"Interested in <city>"* → geo remarketing. |
| 10 | `blog_scroll_25` | Scroll Depth — auto-event, 25% (blog pages only) | `percent_scrolled`, `article_title`, `pain_area`, `page_path` | **Report:** Content engagement. Separates bounces from readers. |
| 11 | `blog_scroll_50` | Scroll Depth — auto-event, 50% | `percent_scrolled`, `article_title`, `pain_area`, `page_path` | **Report:** Content engagement. **Audience:** *"Engaged readers"* → booking-CTA remarketing. |
| 12 | `blog_scroll_75` | Scroll Depth — auto-event, 75% | `percent_scrolled`, `article_title`, `pain_area`, `page_path` | **Report:** Content engagement. **Audience:** *"Deep readers"*. |
| 13 | `blog_scroll_90` | Scroll Depth — auto-event, 90% | `percent_scrolled`, `article_title`, `pain_area`, `page_path` | **Report:** Content engagement. **Audience:** *"Finished the article"* (highest-intent content audience). |

> **Feature → events map.** Appointment booking (3 steps) = events 1–4; Call Now buttons = 5;
> WhatsApp widget = 6; Patient Guide download = 7–8; Clinic location views = 9; Blog article reads =
> 10–13.

---

## Trigger types used, and why

**Custom Event (dataLayer) — events 1, 2, 3, 4, 7.** Fire on an explicit
`window.dataLayer.push({ event: "..." })` from the developer. Used for anything GTM **cannot reliably
infer from the DOM**: a *completed* step, a *persisted* booking, a *validated* form submit, an error
branch. Only the application code knows a booking actually saved — so it announces it. This is the
backbone of the schema.

**Click — events 5, 6, 8.** GTM's built-in Click listener fires on elements matching a CSS selector
or URL pattern (`tel:`, `wa.me`, `.pdf`). Ideal where the click itself is the signal; no developer
code needed, so marketing can add tracked links without a deploy.

**Scroll Depth — events 10–13.** GTM's Scroll-Depth listener fires at configured vertical
percentages, scoped to blog templates by a page-path condition. One listener, four thresholds → four
events differentiated by `percent_scrolled`.

**Element Visibility — event 9.** Fires when a clinic card crosses a visibility threshold — "was this
clinic actually seen?" — without a developer push. (A dataLayer push is the alternative when it's a
true interaction like opening a clinic detail.)

### Why the booking events must be dataLayer pushes (not auto-detected)

A modern booking widget is a single-page interaction: no page loads between steps, no native
per-step `submit` event. GTM can see a *click* on "Continue," but not whether it was *valid*, which
*step* it completed, or what was *selected* — that state lives in JavaScript. So each meaningful
step-completion is announced with a structured `dataLayer.push()`. GTM listens; GA4 records; Funnel
Exploration sequences. Full per-step JSON is in [Booking_Funnel.md](Booking_Funnel.md).

---

## Naming conventions (so the schema scales)

- **`snake_case`, verb-led** event names, matching GA4's recommended-event style and keeping BigQuery exports clean.
- **Events describe *what happened*; parameters describe *context*.** No `booking_step_1` / `booking_step_2` — one `booking_step_complete` event with a `step_number` parameter. Fewer events, and GA4's Funnel Exploration does the sequencing.
- **Every custom parameter is registered as a GA4 event-scoped custom dimension** so it's queryable in Explorations, not just visible in DebugView.
- **No PII in GA4.** `user_name` / `phone_number` are captured for the CRM (Task 3), never mapped into GA4 parameters.

---

## Extended reference (custom dimensions, conversion & business value)

The 4-column table above is the deliverable; this appendix adds the operational detail behind it.

| Event | Business Purpose | Custom Dimensions to register | Google Ads Conversion |
|-------|------------------|-------------------------------|-----------------------|
| `booking_started` | Top-of-funnel intent; funnel denominator | `entry_point` | — |
| `booking_step_complete` | Step-by-step progression / drop-off | `step_number`, `step_name`, `clinic_location`, `specialty` | — |
| `booking_confirmed` | **Primary conversion** — a booking exists | `clinic_location`, `specialty`, `booking_id` | ✅ Imported (value-based) |
| `booking_error` | Surfaces failures that silently kill bookings | `error_type`, `step_number` | — |
| `call_now_click` | Phone intent (top path for 40+) | `link_location` | — (intent, not outcome) |
| `whatsapp_click` | Low-friction contact channel | `link_location` | — |
| `patient_guide_submit` | Content lead capture / nurture list | `guide_name`, `pain_area` | — (optional secondary) |
| `patient_guide_download` | Confirms asset delivery | `guide_name` | — |
| `clinic_location_view` | Local interest → budget allocation | `clinic_location`, `city` | — |
| `blog_scroll_25/50/75/90` | Content engagement depth | `article_title`, `pain_area` | — |

**Why only `booking_confirmed` is imported to Google Ads** — and why call / WhatsApp / PDF /
clinic-view / scroll are deliberately not — is argued in
[Google_Ads_Conversion.md](Google_Ads_Conversion.md). The funnel drop-off analysis and per-step
`dataLayer` JSON are in [Booking_Funnel.md](Booking_Funnel.md).
