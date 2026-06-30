# Testing Guide

How to verify every part of the landing page: rendering, validation, the GTM `dataLayer` push, and
Lighthouse. No tooling required beyond a browser (and optionally Python for a local server).

## 1. Open the page

```bash
# Direct
open task-02-landing-page/index.html            # macOS
# xdg-open task-02-landing-page/index.html        # Linux
# start task-02-landing-page/index.html           # Windows

# Or served (recommended for Lighthouse / Best Practices)
cd task-02-landing-page && python3 -m http.server 8080
# → http://localhost:8080
```

## 2. Test validation (no JS knowledge needed)

| Step | Action | Expected result |
|------|--------|-----------------|
| 2.1 | Click **Request my appointment** with all fields empty | Three inline errors appear; page does **not** reload; **no** `alert()`; focus jumps to the Name field |
| 2.2 | Type `1` in Name, submit | "That name looks too short." |
| 2.3 | Type `12345` in Phone | "Enter a valid 10-digit mobile number." |
| 2.4 | Type letters in Phone | Non-digits are stripped as you type; field never exceeds 10 digits |
| 2.5 | Type `9812345678` in Phone | Phone error clears immediately |
| 2.6 | Leave clinic on the placeholder, submit | "Please choose a clinic." |
| 2.7 | Fill Name=`Ananya Rao`, Phone=`9812345678`, Clinic=any, submit | Form disappears; thank-you panel appears with a personalised message; focus lands on it |

## 3. Test the GTM `dataLayer` push

1. Open **DevTools → Console** *before* submitting.
2. Submit the form with **valid** data (step 2.7).
3. The console logs the `window.dataLayer` array. Expand it and confirm one object:

```js
{
  event: "consultation_form_submitted",
  form_name: "consultation_booking",
  user_name: "Ananya Rao",
  phone_number: "9812345678",
  clinic_preference: "Bengaluru — Indiranagar",
  submission_time: "2026-07-02T09:41:12.874Z"
}
```

4. **Negative test — confirm it does NOT fire on load or on invalid submit.** Reload the page and in
   the Console run:

```js
window.dataLayer            // → [] (empty on load)
```

   Then submit an *invalid* form and re-check — still empty. Only a *valid* submit pushes the event.

### Optional: verify in a real GTM container

1. Add your GTM snippet to `<head>`/`<body>` (or use **GTM Preview / Tag Assistant**).
2. In Preview mode, submit the form.
3. Confirm `consultation_form_submitted` appears in the event stream and your GA4 tag fires with the
   mapped parameters.

## 4. Test accessibility quickly

- **Keyboard only:** Tab through the page — the skip link appears first; every control is reachable
  with a visible focus ring; submit with `Enter`.
- **Screen reader (VoiceOver ⌘+F5 / NVDA):** errors are announced on submit; the thank-you panel is
  announced when it appears.
- **axe DevTools** (browser extension): run a scan → expect zero critical issues.

## 5. Run Lighthouse

See [`pagespeed-notes.md`](pagespeed-notes.md). Short version:

```bash
cd task-02-landing-page && python3 -m http.server 8080
# Chrome DevTools → Lighthouse → Mobile → Analyze
```

Expect ≥ 90 across Performance, Accessibility, Best Practices and SEO. Save the screenshot to
`task-02-landing-page/assets/pagespeed-mobile.png`.

## 6. Take screenshots for the repo

- Desktop full-page → `task-02-landing-page/assets/desktop-preview.png`
- Mobile (DevTools device toolbar, e.g. iPhone 14) → `task-02-landing-page/assets/mobile-preview.png`
- Lighthouse result → `task-02-landing-page/assets/pagespeed-mobile.png`
