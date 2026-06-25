# Google Ads Conversion — What to Import, and Why

> Google Ads' bidding is only as good as the signal you feed it. Import the wrong events and Smart
> Bidding optimises toward cheap, low-value actions and quietly wastes budget. This document explains
> the single conversion OrthoNow should import — and, just as importantly, the five events it should
> **not**.

## The recommendation

**Import exactly one event as the primary Google Ads conversion: `booking_confirmed`.**

Import it via the **GA4 → Google Ads conversion import** path (not a hard-coded Google Ads tag on the
page). The reasoning for that plumbing choice is in [Integration_Architecture.md](../task-03-integration-design/Integration_Architecture.md#google-ads-attribution),
but the summary: one measurement layer (GA4) owns the definition of "a booking," and Ads consumes it —
so the numbers reconcile and there's a single source of truth.

### Why `booking_confirmed` is the right conversion

- **It is the actual business outcome.** The campaign goal is *consultation bookings*. `booking_confirmed` fires only when a booking is persisted (has a server `booking_id`) — it is the closest measurable event to revenue.
- **It carries value.** With `value` + `currency`, we can move from "maximise conversions" to **value-based bidding (tROAS)**, letting Google favour higher-value specialties/clinics.
- **It's deduplicated.** The `booking_id` acts as the order/transaction id, so refreshes or retries don't inflate the count.
- **It's low-volume, high-fidelity.** Smart Bidding learns fastest from a signal that reliably correlates with money. A confirmed booking does; a scroll event doesn't.

`patient_guide_submit` may *optionally* be imported as a **secondary** conversion (observation only,
not the bid target) if OrthoNow runs a content/nurture campaign — but it must never share the bidding
target with `booking_confirmed`, or the algorithm will chase cheap PDF leads.

---

## Why NOT these events

Each of the following is valuable **in GA4** for analysis and remarketing — but importing it as a
Google Ads *conversion* would corrupt bidding.

### ❌ `call_now_click`
A **click on a `tel:` link is intent, not outcome.** We don't know if the call connected, how long it
lasted, or whether it produced a booking. If imported, Smart Bidding would optimise for people who
*tap* a phone number — trivially cheap and easy to inflate — instead of people who actually book. The
right treatment: keep it as a GA4 event and (if call volume is high) measure real bookings via
call-tracking that ties a *completed, qualified* call back to a conversion, not the tap.

### ❌ `whatsapp_click`
Same failure mode as call clicks — it measures the *start* of a conversation, not its result. A
person can tap "WhatsApp us," send nothing, and never book. Optimising toward it teaches Google to
find tappers, not patients. Useful as an engagement/remarketing signal only.

### ❌ `patient_guide_download`
Downloading a free PDF is **top-of-funnel curiosity**, an order of magnitude cheaper and more frequent
than a booking. Import it as the bid target and Google will flood the account with guide-grabbers
because they're easy to get — cost-per-booking rises even as "conversions" look great. It's a content
KPI, not a campaign conversion.

### ❌ `clinic_location_view`
Viewing a clinic (a visibility/interest event) is **passive** — it doesn't even require an
interaction. As a conversion it's almost pure noise: bidding would reward any traffic that scrolls
past a clinic card. Keep it for geo-interest analysis and city-level remarketing.

### ❌ `blog_scroll_25 / 50 / 75 / 90`
Scroll depth is an **engagement micro-signal**, not a business result. Someone reading 90% of an
article has read an article — nothing more. These events are extremely high-volume and only weakly
correlated with booking. Importing any of them would drown the true conversion signal and push
bidding toward content readers. Their real value is building "engaged reader" **remarketing
audiences** that we then show a booking-focused ad to — the scroll is the *targeting* input, the
booking is still the *conversion*.

---

## The principle in one line

> **Optimise bidding toward the outcome that makes money (`booking_confirmed`); use everything else as
> analysis and remarketing signals — never as the bid target.**

Mixing micro-conversions into the primary conversion is the most common way healthcare/lead-gen
accounts silently waste spend: the dashboard shows thousands of "conversions," CPA looks fantastic,
and actual bookings flatline. One clean, valued, deduplicated conversion keeps Google's algorithm and
OrthoNow's revenue pointed at the same target.
