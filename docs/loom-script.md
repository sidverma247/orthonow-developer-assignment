# Loom Walkthrough Script (≤ 8 minutes)

A minute-by-minute script for recording the submission walkthrough. Keep it calm and specific — show,
don't tell. Have the repo open in an editor and `index.html` open in a browser with DevTools ready.

---

## 0:00–0:45 — Opening

> "Hi, I'm Priyanshi. This is my submission for the OrthoNow developer assignment. The campaign goal
> is simple — get working professionals in Bengaluru, Hyderabad and Chennai to book an orthopaedic
> consultation. I split the work into the three tasks the brief asks for: a GTM/GA4 measurement
> schema, a self-contained landing page with real tracking, and a CRM integration design. I'll walk
> the repo, demo the page live including the dataLayer, then talk through the integration. Let's go."

## 0:45–1:30 — Repo walkthrough

- Show the folder tree in the editor.
> "Each task is its own folder with its own README, so any reviewer can open one in isolation.
> `docs/` holds cross-cutting stuff — testing, PageSpeed notes, this script. The landing page is a
> single `index.html`: no frameworks, no CDN, no external assets. That's a deliberate constraint from
> the brief and it's also what makes it fast."

## 1:30–3:00 — Task 1: GTM / GA4 schema

- Open `GTM_Event_Schema.md`, scroll the table.
> "Thirteen events, one design rule: the developer owns a small structured dataLayer, GTM owns the
> tags. Notice there's no `booking_step_1` and `booking_step_2` — it's one `booking_step_complete`
> event with a `step_number` parameter. That keeps the event list short and lets GA4's Funnel
> Exploration do the sequencing."
- Open `Booking_Funnel.md`, show the JSON and the drop-off table.
> "Here's the real dataLayer JSON for each step, and a worked funnel. The biggest leak is step 2 to
> step 3 — slot selection — which tells marketing to add evening and weekend slots, not to spend more
> on ads. Every number here is a fixable, re-measurable decision."
- Open `Google_Ads_Conversion.md`.
> "And I only import one conversion into Google Ads — `booking_confirmed` — because it's the actual
> outcome and it carries value. Call clicks, WhatsApp clicks, PDF downloads and scroll are intent
> signals; importing them would teach Google to chase cheap taps instead of bookings."

## 3:00–5:30 — Task 2: landing page live demo

- Show the page in the browser; scroll top to bottom.
> "Mobile-first healthcare page. One H1, one primary CTA colour, honest copy — the headline addresses
> the real fear, 'before anyone mentions surgery.' Trust numbers are specific but not fake
> testimonials. Three-field form only: name, phone, clinic."
- **Open DevTools → Console. Type `window.dataLayer`** → show it's empty.
> "Nothing has fired on load — that's important."
- Submit the empty form.
> "Inline errors, no alert, no reload, focus jumps to the first bad field. Accessible errors with
> `role=alert`."
- Type an invalid phone, then fix it.
> "Non-digits are stripped live, and the error clears the moment it's valid."
- Fill valid data and submit.
> "Form is replaced by a personalised thank-you, focus moves there for screen-reader users. And now
> in the console —"
- **Show the logged `window.dataLayer`** with the `consultation_form_submitted` object.
> "One clean event, fired only after validation passes. `form_name`, clinic preference, ISO
> timestamp — exactly what GTM needs. This is the single-screen version of the `booking_confirmed`
> event from Task 1."
- Briefly show Lighthouse result (or the saved screenshot).
> "90-plus mobile across all four categories — mostly free, because there's nothing external to load."

## 5:30–7:00 — Task 3: integration design

- Open `Integration_Architecture.md`, show the ASCII flow.
> "Browser to a custom backend, not Zapier — because I need dedup, retries, an SLA and idempotency,
> and connectors hide all of that. The key trap: HubSpot deduplicates by email, but our patients book
> by phone. So I search HubSpot by the phone property first, then update or create."
- Point at the shared-phone matrix.
> "If two people share a phone with different names, I don't overwrite — I create, associate, and
> flag `needs_review` for ops. Data integrity over false tidiness."
> "WhatsApp confirmations go through a queue so a Karix spike or outage can't blow the 2-minute SLA;
> failures land in a dead-letter queue, and CloudWatch/Grafana alert on queue age. Google Ads gets
> the conversion through GA4, not a direct tag, so there's one deduplicated source of truth."

## 7:00–8:00 — Closing

> "So: a page engineered to convert and to measure, a schema that turns the funnel into ranked fixes,
> and an integration that's resilient where real systems break. Everything in the repo runs as-is —
> open the HTML, submit, watch the dataLayer. Thanks for watching, happy to dig into any part."

---

## Expected interviewer questions & suggested answers

**Q. Why no framework for the landing page?**
> The brief required it, but I'd choose the same here anyway: it's one page with one form. A framework
> adds a runtime, a build step and network weight for zero benefit. Vanilla keeps LCP fast and the
> tracking contract obvious.

**Q. Why push `phone_number` into the dataLayer — isn't that PII in analytics?**
> On the demo page I push it to prove the contract, and I log it for the reviewer. In production I'd
> keep PII out of GA4 — the phone/name go to the CRM via the backend (Task 3), and GA4 only gets
> non-PII context like `clinic_preference`. I'd also gate everything behind Consent Mode.

**Q. Why is `booking_confirmed` fired from the API success callback, not on the confirmation page load?**
> Because a page can load without the booking actually persisting. Firing on the server's success
> response (with a real `booking_id`) means we only count bookings that exist, and the id gives us
> deduplication for retries and refreshes.

**Q. HubSpot dedupes by email — what if you only have a phone?**
> That's the core trap. I search the `phone` property via the CRM Search API before writing. Found →
> update; not found → create. For a shared phone with a different name I create a new contact and flag
> it rather than overwrite the wrong person.

**Q. How do you guarantee the 2-minute WhatsApp SLA?**
> I decouple it: the API returns immediately and drops a message on a queue; a consumer calls Karix
> with retries. Monitoring watches p95 send latency and queue age, alerting before the SLA is at risk.
> Exhausted messages go to a DLQ so nothing is lost.

**Q. Why import the conversion through GA4 instead of a direct Google Ads tag?**
> One source of truth. A direct tag double-counts on refresh, can't see server-side events, and
> creates a second definition of "a booking" that never matches GA4. GA4 → Ads import dedupes on
> `booking_id`, works server-side, and unlocks value-based bidding.

**Q. What would you build next?**
> Wire a real GTM container and publish the tags, build the multi-step booking widget the Task 1
> schema describes, implement the Task 3 backend endpoint, and add Lighthouse-CI + axe to the GitHub
> Action so quality can't regress.
