# Task 01 — GTM / GA4 Event Schema

> **Scope.** A complete measurement plan for the OrthoNow consultation campaign: every event the
> site should emit, how GTM triggers it, how it maps to GA4, which parameters travel with it, and
> what marketing does with the result. Designed around one north-star metric — **consultation
> bookings** — with supporting micro-conversions that explain *why* bookings do or don't happen.

## Naming & design conventions

Before the table, the rules the schema follows (so it stays maintainable as the site grows):

- **`snake_case` event names**, verb-led where there's an action (`booking_started`, `call_now_click`). This matches GA4's own recommended-event style and keeps BigQuery exports clean.
- **Events describe *what happened*, parameters describe *the context*.** We never bake context into the event name (no `booking_step_1`, `booking_step_2` — it's one `booking_step_complete` event with a `step_number` parameter). This keeps the number of events small and lets GA4's Funnel Exploration do the sequencing.
- **The developer owns the `dataLayer`; GTM owns the tags.** The front-end pushes structured, predictable objects. GTM contains *zero* business logic beyond reading them. This separation means marketing can add/rename GA4 tags without a code deploy.
- **Every custom parameter is registered as a GA4 custom dimension** (event-scoped) so it's queryable in Explorations, not just visible in DebugView.

---

## The event schema

| # | Event Name | Business Purpose | Trigger Type | Trigger Details | GTM Trigger (condition) | Key Parameters | GA4 Event | Custom Dimensions | Google Ads Conversion | Audience | GA4 Report |
|---|------------|------------------|--------------|-----------------|-------------------------|----------------|-----------|-------------------|-----------------------|----------|------------|
| 1 | `booking_started` | Measures top-of-funnel intent — how many people begin the booking flow. Denominator for funnel completion. | Custom Event (dataLayer) | Dev pushes when the booking widget opens / step 1 renders. | `Event equals booking_started` | `entry_point` (hero/sticky/nav), `device_type` | `booking_started` | `entry_point` | — (micro-conversion, not imported) | "Started but didn't book" (for remarketing) | Funnel Exploration – step 1 |
| 2 | `booking_step_complete` | Tracks progression through each funnel step so we can see exactly where people drop. | Custom Event (dataLayer) | Dev pushes on each step's "continue" / valid step submit. | `Event equals booking_step_complete` | `step_number`, `step_name`, `clinic_location`, `specialty`, `slot_type` | `booking_step_complete` | `step_number`, `step_name`, `clinic_location`, `specialty` | — | "Reached step 2/3" audiences | Funnel Exploration – middle steps |
| 3 | `booking_confirmed` | **The primary conversion.** A consultation is actually booked. | Custom Event (dataLayer) | Dev pushes on the confirmation screen after the booking is persisted (has a booking id). | `Event equals booking_confirmed` | `booking_id`, `clinic_location`, `specialty`, `appointment_date`, `value`, `currency` | `booking_confirmed` (marked **Key Event**) | `clinic_location`, `specialty`, `booking_id` | ✅ **Imported to Google Ads** | Converters (exclude from prospecting) | Funnel exploration – final step; Conversions report |
| 4 | `booking_error` | Surfaces technical/UX failures that silently kill bookings (payment fail, slot gone, API error). | Custom Event (dataLayer) | Dev pushes in the catch/error branch of the booking flow. | `Event equals booking_error` | `error_type`, `error_message`, `step_number`, `clinic_location` | `booking_error` | `error_type`, `step_number` | — | — (diagnostic) | Custom report / DebugView; alert if spikes |
| 5 | `call_now_click` | Phone is a top conversion path for 40+ users. Measures call intent. | Click (auto-event) | GTM listens for clicks on `a[href^="tel:"]`. | `Click Element matches CSS a[href^="tel:"]` | `link_url` (number), `link_location` (header/sticky/footer/thankyou) | `call_now_click` | `link_location` | — (see Google_Ads_Conversion.md for why) | "High-intent callers" | Engagement / Events |
| 6 | `whatsapp_click` | Measures WhatsApp as an alternative low-friction contact channel. | Click (auto-event) | GTM listens for clicks on `a[href*="wa.me"]` / `a[href*="api.whatsapp.com"]`. | `Click Element matches CSS a[href*="wa.me"], a[href*="whatsapp"]` | `link_url`, `link_location` | `whatsapp_click` | `link_location` | — | "WhatsApp responders" | Engagement / Events |
| 7 | `patient_guide_submit` | Lead-gen for the top-of-funnel content offer (e.g. "Knee pain recovery guide"). Builds a nurture list. | Custom Event (dataLayer) | Dev pushes on successful guide-request form submit. | `Event equals patient_guide_submit` | `guide_name`, `pain_area`, `city` | `patient_guide_submit` | `guide_name`, `pain_area` | — (soft conversion; optionally a *secondary* Ads conversion) | "Guide leads – nurture" | Events / Lead report |
| 8 | `patient_guide_download` | Confirms the asset was actually delivered (submit ≠ download). Content-quality signal. | Click / File download | GTM link-click trigger on the PDF `href` (or GA4 enhanced-measurement file download). | `Click URL ends with .pdf` AND contains guide slug | `file_name`, `guide_name`, `pain_area` | `file_download` (GA4 recommended) | `guide_name` | — | — | Engagement / File downloads |
| 9 | `clinic_location_view` | Measures interest in a specific clinic/city — informs local budget allocation. | Element visibility / Custom Event | GTM Element-Visibility trigger when a clinic card enters viewport, OR dev push on clinic-detail open. | `Percent Visible ≥ 50` on `.clinic-card` | `clinic_location`, `city` | `clinic_location_view` | `clinic_location`, `city` | — | "Interested in <city>" (geo remarketing) | Engagement by city |
| 10 | `blog_scroll_25` | Content engagement (early). Distinguishes bounces from genuine readers. | Scroll Depth (auto-event) | GTM Scroll-Depth trigger, vertical, 25%. | `Scroll Depth Threshold equals 25` (on blog templates only) | `page_path`, `article_title`, `pain_area` | `scroll` (with `percent_scrolled=25`) | `article_title`, `pain_area` | — | — | Content engagement |
| 11 | `blog_scroll_50` | Mid-article engagement — a meaningful read. | Scroll Depth (auto-event) | GTM Scroll-Depth trigger, 50%. | `Scroll Depth Threshold equals 50` | `page_path`, `article_title`, `pain_area` | `scroll` (`percent_scrolled=50`) | `article_title`, `pain_area` | — | "Engaged readers" (remarket booking CTA) | Content engagement |
| 12 | `blog_scroll_75` | Deep engagement — strong content-to-consult candidates. | Scroll Depth (auto-event) | GTM Scroll-Depth trigger, 75%. | `Scroll Depth Threshold equals 75` | `page_path`, `article_title`, `pain_area` | `scroll` (`percent_scrolled=75`) | `article_title`, `pain_area` | — | "Deep readers" | Content engagement |
| 13 | `blog_scroll_90` | Near-complete read — highest-intent content audience for a booking nudge. | Scroll Depth (auto-event) | GTM Scroll-Depth trigger, 90%. | `Scroll Depth Threshold equals 90` | `page_path`, `article_title`, `pain_area` | `scroll` (`percent_scrolled=90`) | `article_title`, `pain_area` | — | "Finished the article" (highest-intent remarketing) | Content engagement |

> The landing page in **Task 2** implements a simplified single-screen version of this flow and emits
> `consultation_form_submitted` — the one-step analogue of `booking_confirmed`. The full multi-step
> schema above is what the production booking widget would emit.

---

## Trigger types explained

Understanding *why* each trigger type is used matters more than the table itself:

**Custom Event (dataLayer) triggers — events 1, 2, 3, 4, 7, (9).**
These fire on a `window.dataLayer.push({ event: "..." })` from the developer. We use them for
anything GTM **cannot reliably infer from the DOM**: the completion of a step, a confirmed booking, a
successful (validated + persisted) form submit, an error branch. GTM has no way to know a booking was
*saved* to the database — only the application code knows that — so the app announces it explicitly.
This is the backbone of the schema.

**Click triggers (auto-event) — events 5, 6, 8.**
GTM's built-in Click listener fires on element clicks matching a CSS selector or URL pattern. Perfect
for outbound intents where the *click itself* is the signal: `tel:` links, `wa.me` links, PDF
downloads. No developer code required — the marketer configures the selector in GTM. This keeps the
code lean and lets marketing add new tracked links without a deploy.

**Scroll-Depth triggers (auto-event) — events 10–13.**
GTM's Scroll-Depth listener fires at configured vertical percentages. Scoped to blog page templates
via a page-path condition so we don't record scroll on the landing page (where scroll depth is
meaningless). One listener, four thresholds → four events differentiated by `percent_scrolled`.

**Element-Visibility trigger — event 9 (alternative implementation).**
Fires when an element crosses a visibility threshold (e.g. a clinic card 50% in view). Useful for
"was this section actually seen?" without any developer push. We list a dataLayer alternative too,
because for a true *interaction* (opening a clinic detail) a developer push is cleaner than inferring
intent from visibility.

---

## Why GTM cannot auto-detect a multi-step booking form

This is the single most important reason the developer must own the `dataLayer` — see
[`Booking_Funnel.md`](Booking_Funnel.md) for the full walkthrough and real JSON. In short: a modern
booking widget is a single-page interaction with no page loads between steps and no native `submit`
event per step. GTM's DOM-based triggers can see a *click* on a "Continue" button, but they cannot
know whether that click was *valid*, which *step* it completed, or what the user *selected* — that
state lives in JavaScript, not the DOM. So each meaningful step-completion is announced with an
explicit, structured `dataLayer.push()`. GTM listens; GA4 records; Funnel Exploration sequences.

---

## Related documents

- **[Booking_Funnel.md](Booking_Funnel.md)** — the 3-step funnel: per-step ownership, real `dataLayer` JSON, and the GA4 Funnel Exploration setup with a worked drop-off table.
- **[Google_Ads_Conversion.md](Google_Ads_Conversion.md)** — why only `booking_confirmed` is imported to Google Ads, and why call/WhatsApp/PDF/clinic-view/scroll are deliberately *not*.
