# Task 01 — GTM Event Schema

A plain-English guide to how OrthoNow measures its website — written so you can read it top to
bottom and then explain it to someone who has never touched analytics.

**The whole point in one sentence:** the campaign spends money to get people onto the site, and this
schema lets us *see* which of those people book a consultation — and where the rest drop off — so we
stop guessing and start fixing.

---

## Explain-it-in-30-seconds

> "Every time a visitor does something that matters — opens the booking form, taps *Call*, downloads
> a guide, finishes an article — the page quietly writes a note about it. Google Tag Manager reads
> those notes and passes them to Google Analytics, which turns them into reports and audiences. The
> single most important note is *'a booking was confirmed'* — that's the one we send to Google Ads so
> its bidding chases real bookings, not cheap clicks."

---

## How the tracking works (the mental model)

Four moving parts, in order. If you understand this chain, you understand the whole schema:

```
  1. The visitor          2. The dataLayer        3. GTM              4. GA4 / Google Ads
  does something    →     (the page's notepad) →  (the dispatcher) →  (the reports + the ad robot)
```

| Part | Plain-English job | The one-line analogy |
|------|-------------------|----------------------|
| **The visitor action** | Clicks, submits, scrolls, books. | Someone doing something in a shop. |
| **The `dataLayer`** | A little notepad on the page where the developer writes a structured note: *"event: booking_confirmed, clinic: Indiranagar…"* | A sticky note the shop assistant jots down. |
| **GTM (Google Tag Manager)** | Watches the notepad. When a note it recognises appears, it forwards it to the right place. We configure this without touching code. | The dispatcher who reads the notes and radios them in. |
| **GA4 & Google Ads** | GA4 stores every note and builds reports/audiences. Google Ads takes *one* note — the confirmed booking — and learns from it. | Head office: keeps the records, and trains the ad-buying robot. |

**The key idea to land with anyone:** GTM can *see* clicks and scrolls on its own, but it **cannot
see inside a JavaScript booking form** — it doesn't know a step was completed or a booking was saved.
So for those moments, the developer has to write the note explicitly (`dataLayer.push`). Everything
else, GTM can pick up by itself.

---

## The events, grouped by what they're *for*

Ten events, in three buckets. This grouping is the easiest way to explain the schema — lead with the
bucket, not the event name.

### 🟢 MONEY — the booking funnel (the events we actually get paid on)

These trace a person from *"started booking"* to *"booked."* This is where optimisation effort goes.

- **`booking_started`** — someone opened the booking form. *Tells us:* how many people even begin. Without it we'd have no denominator.
- **`booking_step_complete`** — they finished a step (1, 2 or 3). *Tells us:* exactly which step people give up on.
- **`booking_confirmed`** — the booking was saved. **This is the goal.** *Tells us:* a real appointment exists — and it's the one signal we send to Google Ads.
- **`booking_error`** — something broke mid-booking. *Tells us:* whether a drop-off is the user hesitating or our system failing.

### 🔵 CONTACT — they reached out another way

Not everyone uses the form. Older or in-a-hurry visitors call or WhatsApp instead.

- **`call_now_click`** — tapped a phone number. *Tells us:* how much demand goes down the phone line.
- **`whatsapp_click`** — opened the WhatsApp chat. *Tells us:* appetite for a low-pressure channel.

*(Important nuance to explain: a tap is **interest**, not a booking — we count it, but we don't treat
it as a conversion. More on that in the Google Ads section.)*

### 🟡 INTEREST — warming up, not ready yet

People researching their knee/back pain. Great remarketing fuel; not conversions.

- **`patient_guide_submit`** — gave their details to download a guide. *Tells us:* a nurture lead.
- **`patient_guide_download`** — the PDF actually downloaded. *Tells us:* the content was delivered (submit ≠ download).
- **`clinic_location_view`** — looked at a specific clinic. *Tells us:* which city/area is drawing interest → where to put local budget.
- **`blog_scroll`** — read 25 / 50 / 75 / 90% of an article. *Tells us:* who's genuinely engaged vs. who bounced.

---

## The full schema table (the deliverable)

Every event, its GTM trigger, at least three parameters, and the GA4 report or audience it feeds —
plus a plain-English column so anyone can follow it.

| Event Name | Trigger Type | Key Parameters (min. 3) | Feeds (GA4 report / audience) | In plain English |
|------------|--------------|-------------------------|-------------------------------|------------------|
| `booking_started` | Custom Event (page writes the note when the form opens) | `entry_point`, `device_type`, `page_location` | Funnel Exploration (start) · Audience *"Started, didn't finish"* | "Someone opened the booking form." |
| `booking_step_complete` | Custom Event (one per completed step) | `step_number`, `step_name`, `clinic_location`, `specialty` | Funnel Exploration (middle steps) · Audience *"Reached step 2/3"* | "They finished a step of the form." |
| `booking_confirmed` | Custom Event (fired when the server saves the booking) | `booking_id`, `clinic_location`, `specialty`, `value`, `currency` | Key Events / Conversions · Audience *"Booked"* · **→ Google Ads** | "A real appointment now exists." |
| `booking_error` | Custom Event (fired in the error branch) | `error_type`, `error_message`, `step_number`, `clinic_location` | Debug dashboard (alert on spikes) | "The booking broke — here's why." |
| `call_now_click` | Click — GTM sees a tap on a `tel:` link | `link_url`, `link_location`, `page_location` | Engagement → Events · Audience *"High-intent callers"* | "They tapped a phone number." |
| `whatsapp_click` | Click — GTM sees a tap on the `wa.me` link | `link_url`, `link_location`, `page_location` | Engagement → Events · Audience *"WhatsApp responders"* | "They opened WhatsApp chat." |
| `patient_guide_submit` | Custom Event (or Form Submit) on the gate form | `guide_name`, `pain_area`, `phone_provided` | Lead report · Audience *"Guide leads — nurture"* | "They gave details for a free guide." |
| `patient_guide_download` | Click / File-download on the `.pdf` | `file_name`, `guide_name`, `pain_area` | Engagement → File downloads | "The guide actually downloaded." |
| `clinic_location_view` | Element Visibility (demo) / Page View on `/clinics/*` (live site) | `clinic_location`, `city`, `page_location` | Engagement by location · Audience *"Interested in <city>"* | "They looked at a specific clinic." |
| `blog_scroll` | Scroll Depth — fires at 25 / 50 / 75 / 90% | `percent_scrolled`, `article_title`, `pain_area`, `page_path` | Content engagement · Audience *"Engaged readers"* | "They read this much of an article." |

> **Parameters, in plain terms:** the *event* is the headline ("a booking was confirmed"); the
> *parameters* are the details that make it useful ("…at Indiranagar, for a knee, worth ₹800"). Every
> parameter here is registered in GA4 as a **custom dimension** so it can be filtered and grouped in
> reports.

---

## The booking funnel, and how we see where people quit

### Why the page has to "announce" each step

A booking form built in JavaScript changes on screen without loading a new page. GTM watches for page
loads and clicks — so between "step 1" and "step 2" it sees **nothing**. It has no idea a step was
completed or what the person chose. So we make the page **say it out loud**: at the end of each step
the developer runs one `dataLayer.push()` with a note. GTM hears the note; GA4 records it; the funnel
report lines them up in order. That's the entire trick.

### What the page says at each step (the real notes)

**Step 1 — picked a clinic and what's wrong.** GTM listens with a Custom Event trigger on
`booking_step_complete`; the `step_number` tells the steps apart.

```json
{
  "event": "booking_step_complete",
  "step_number": 1,
  "step_name": "location_specialty_selected",
  "clinic_location": "{{clinic name}}",
  "specialty": "{{specialty selected}}"
}
```

**Step 2 — entered contact details and a preferred date.** *Same* trigger — we don't need a new one.

```json
{
  "event": "booking_step_complete",
  "step_number": 2,
  "step_name": "contact_and_date_entered",
  "clinic_location": "{{clinic name}}",
  "specialty": "{{specialty selected}}",
  "appointment_date": "{{YYYY-MM-DD}}",
  "phone_provided": true
}
```

**Step 3 — reviewed the booking.** Still the same trigger.

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

**Confirmation — the booking is saved.** A *separate* note, fired from the server's success response,
carrying the real `booking_id` (which proves it saved and stops double-counting).

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

> **Why one `booking_step_complete` event with a `step_number`, instead of three different events?**
> Fewer things to explain, fewer triggers to maintain, and GA4 can still separate the steps using the
> number. It's the same reason we don't name files `photo1`, `photo2`, `photo3`.

### Seeing the drop-off in GA4 (Funnel Exploration)

In **GA4 → Explore → Funnel exploration**, build a **closed** funnel (people must go in order):

| Funnel step | GA4 condition |
|-------------|---------------|
| 1. Started | event = `booking_started` |
| 2. Step 1 done | `booking_step_complete` AND `step_number` = 1 |
| 3. Step 2 done | `booking_step_complete` AND `step_number` = 2 |
| 4. Step 3 done | `booking_step_complete` AND `step_number` = 3 |
| 5. Confirmed | event = `booking_confirmed` |

Add a **breakdown by `clinic_location`** and a **segment by `device_type`**, and GA4 draws the
drop-off between every step.

**Read it like a story:**

| Step | People here | Made it to next step | Lost here |
|------|------------:|---------------------:|----------:|
| Started | 4,000 | 78% | — |
| Step 1 done | 3,120 | 63% | 22% |
| Step 2 done | 1,950 | 80% | **37% ← biggest leak** |
| Step 3 done | 1,560 | 84% | 20% |
| **Confirmed** | **1,310** | — | 16% |

The story this tells: *"We lose the most people at the contact-details step — and on mobile it's
worse. That's not a reason to buy more ads; it's a reason to fix that form."* Cross-check the final
16% against `booking_error` and you know whether it's hesitation or a bug. That's the whole value of
the schema in one sentence: **it turns 'bookings are expensive' into a specific, fixable to-do.**

---

## The one conversion we send to Google Ads

**We import exactly one: `booking_confirmed`.**

The plain-English reason: Google Ads' bidding is a robot that learns from whatever you feed it. Feed
it *real bookings* and it goes and finds more people like the ones who book. Feed it *cheap actions*
and it goes and finds more of those — and you get a dashboard full of "conversions" and an empty
appointment book.

| If we imported… | The robot would learn to find… | Why that's bad |
|-----------------|-------------------------------|----------------|
| `call_now_click` / `whatsapp_click` | people who **tap** a button | A tap isn't a booking — we don't know it connected. |
| `patient_guide_download` | people who grab a **free PDF** | Cheap and easy — it would flood us with non-buyers. |
| `clinic_location_view` | people who **scroll past** a clinic | Almost pure noise. |
| `blog_scroll` | people who **read an article** | Engagement, not intent to book. |
| ✅ `booking_confirmed` | people who **actually book** | The real outcome — and it carries `value`, so Ads can chase higher-value bookings (tROAS). |

We send it from **GA4 into Google Ads** (not a separate Ads tag) so there's one agreed definition of
"a booking," no double-counting, and value-based bidding is available.

---

## Design choices worth knowing (so you can defend them)

- **Blog scroll is one event (`blog_scroll`) with a `percent_scrolled` parameter**, not four separate `blog_scroll_25/50/75/90` events. It's cleaner to explain and matches how GA4 natively handles scroll. (The original brief listed the four thresholds separately — this is the same information, modelled better.)
- **`clinic_location_view` becomes `clinic_page_view` on the real site.** OrthoNow has nine real clinic pages, so in production this is a normal page view on `/clinics/*`. On our single-page demo we approximate it with an Element-Visibility trigger — same event, different plumbing.
- **`phone_provided` is a boolean, never the number.** We record *that* someone gave a phone, not the phone itself — a useful signal without putting personal data in analytics.
- **No PII in GA4, ever.** Names and phone numbers go to the CRM (see Task 3). GA4 only gets non-personal context like clinic, specialty and value.

---

## Mini-glossary (hand this to anyone who's lost)

| Term | What it means |
|------|---------------|
| **dataLayer** | A small list on the page where the developer writes structured notes about what's happening. |
| **event** | One note — the name of a thing that happened (`booking_confirmed`). |
| **parameter** | A detail attached to the note (`clinic_location: "Indiranagar"`). |
| **GTM (Tag Manager)** | The tool that watches the notes and forwards them — configured without code changes. |
| **trigger** | The rule in GTM that says "when *this* note appears, do something." |
| **GA4** | Google Analytics — stores the notes and turns them into reports and audiences. |
| **audience** | A saved group of people (e.g. "started booking but didn't finish") we can re-target with ads. |
| **conversion** | The action that counts as success. Here: `booking_confirmed`. |

All ten events are implemented and observable in the [Task 2 landing page](../task-02-landing-page/index.html) —
open the browser console and interact with the page to watch the notes appear.
