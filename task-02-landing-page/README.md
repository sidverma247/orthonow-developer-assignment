# Task 02 — Landing Page

A single, self-contained `index.html` — HTML5 + CSS3 + vanilla JavaScript. No frameworks, no CDNs,
no external fonts or images, no build step. Open it in a browser and it works.

## How to run

```bash
open index.html                 # macOS — or double-click the file
# or serve it for a clean Lighthouse run:
python3 -m http.server 8080     # then http://localhost:8080
```

## What it does

A conversion-focused consultation-booking page for OrthoNow's campaign, built for the 28–50
working-professional audience on a phone. It is also a **live demo of the entire Task 1 measurement
plan**: every tracked interaction pushes the exact `dataLayer` event documented in
[`../task-01-gtm-schema/`](../task-01-gtm-schema/), and each push is mirrored to the console so the
behaviour is visible without a GTM container loaded.

## Implemented features → events fired

| Feature | Behaviour | `dataLayer` event(s) |
|---------|-----------|----------------------|
| **3-step booking funnel** | Open widget → choose clinic + specialty → name/phone/date → review & confirm. Real inline validation, a server-style `booking_id` on confirm. | `booking_started`, `booking_step_complete` (×3), `booking_confirmed`, `booking_error`, `consultation_form_submitted` |
| **Call Now buttons** | In header, hero, each clinic card, footer and the sticky bar. | `call_now_click` (with `link_location`) |
| **WhatsApp widget** | Floating button that opens a `wa.me` deep link in a new tab. | `whatsapp_click` |
| **Patient Guide** | Gated by name + phone; on submit a **valid PDF is generated in-browser** (correct xref byte offsets — no external file) and downloaded. | `patient_guide_submit`, `patient_guide_download` |
| **Clinic locations (9)** | 3 cities × 3 clinics, rendered from one data source. "View clinic" expands details. | `clinic_location_view` (with `clinic_location`, `city`) |
| **Blog article** | Full read with a sticky progress bar; thresholds fire once each. | `blog_scroll_25/50/75/90` |

> **Production note.** The click/scroll/visibility events (`call_now_click`, `whatsapp_click`,
> `patient_guide_download`, `blog_scroll_*`, `clinic_location_view`) are captured by GTM's built-in
> auto-event triggers in the real container. The page *also* pushes them so it's a self-contained,
> observable demo — in production you'd keep one source per event to avoid double-counting. The
> `booking_*`, `patient_guide_submit` and `consultation_form_submitted` events are always explicit
> developer-owned pushes.

## Page structure

| Section | Purpose (CRO) |
|---------|---------------|
| Sticky header + Call Now | Persistent call path; tracked `call_now_click`. |
| Hero | Doubt-addressing headline; primary CTA opens the booking widget. |
| Why OrthoNow | Three benefit cards framed around the audience's scheduling reality. |
| Clinic locations | Nine clinics; view/expand + per-card Call button. |
| 3-step booking widget | The full funnel — minimum friction, one step at a time. |
| Patient Guide | Lead magnet; gated in-browser PDF download. |
| Blog article | Content engagement + scroll tracking + a booking CTA. |
| FAQ | Native `<details>` — kills the top objections (referral, cost, surgery). |
| Footer + sticky CTA + WhatsApp float | Always-available conversion paths. |

## The tracking contract

```javascript
window.dataLayer = window.dataLayer || [];
window.dataLayer.push({
  event: "consultation_form_submitted",
  form_name: "consultation_booking",
  user_name: name,
  phone_number: phone,
  clinic_preference: clinic,
  submission_time: new Date().toISOString()
});
console.log(window.dataLayer);
```

Fires **only after validation passes** — never on page load. This is the single-screen analogue of
the multi-step `booking_confirmed` event documented in [Task 1](../task-01-gtm-schema/Booking_Funnel.md).

## Validation rules

| Field | Rule | Error behaviour |
|-------|------|-----------------|
| Name | Required, ≥ 2 chars | Inline message, `aria-invalid`, focus on submit |
| Phone | Required, `^[6-9]\d{9}$` (10-digit Indian mobile) | Non-digits stripped live; inline error |
| Clinic | Required (must pick an option) | Inline error |

No `alert()`. Errors use `role="alert"` + `aria-invalid` so screen readers announce them, and submit
moves focus to the first invalid field.

## Accessibility

- Skip link, semantic landmarks, labelled form controls, `aria-describedby` wiring for hints/errors.
- Visible focus rings, 48px minimum tap targets, AA-contrast palette.
- Thank-you panel is `role="status"` + `aria-live="polite"` and receives focus on success.
- Respects `prefers-reduced-motion`.

## Performance

- Zero network requests beyond the document → fast LCP. System fonts → no font swap/CLS.
- Minimal DOM, inlined critical CSS/JS (single file), no render-blocking third parties.
- See [`../docs/pagespeed-notes.md`](../docs/pagespeed-notes.md) for the Lighthouse strategy and
  expected 90+ mobile scores.

## Why this page should convert better than a 2.1% baseline

- **Visual hierarchy.** One H1, one primary CTA colour, generous whitespace — the eye goes hero → CTA → form with nothing competing.
- **Reduced friction.** Three fields, no account, no payment to book, live phone formatting, "under a minute" reassurance.
- **Single CTA.** Every CTA points to the same action (book). Call/WhatsApp are secondary, not competing conversions.
- **Trust.** Specific numbers, "surgery is the last resort," transparent pricing, no invented testimonials — credibility for a health decision.
- **Mobile usability.** Sticky Call/Book bar, thumb-sized targets, numeric keypad for phone, no horizontal scroll. The audience is on phones between meetings; the page is built for that thumb.

## Screenshots

Export to `assets/`: `desktop-preview.png`, `mobile-preview.png`, `pagespeed-mobile.png`.
