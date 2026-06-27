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
working-professional audience on a phone. On a **valid** form submission it pushes one event to
`window.dataLayer` (`consultation_form_submitted`) — the analytics contract GTM consumes — then
swaps the form for a thank-you state. It logs `window.dataLayer` to the console so the behaviour is
visible in a demo.

## Page structure

| Section | Purpose (CRO) |
|---------|---------------|
| Sticky header + phone | Persistent call path; `tel:` link is a tracked interaction. |
| Hero | Specific, doubt-addressing headline + single primary CTA to the form. |
| Trust strip | Real-feeling proof (years/clinics/rating) — no fake testimonials. |
| Why OrthoNow | Three benefit cards framed around the audience's scheduling reality. |
| What the consult covers | Sets expectations → reduces "is it worth it?" friction. |
| Consultation form | Three fields only (name, phone, clinic) — minimum friction. |
| FAQ | Native `<details>` — kills the top objections (referral, cost, surgery). |
| Footer | Contact + cities. |
| Sticky mobile CTA | Always-visible Call / Book bar on mobile. |

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
