# Booking Funnel — Step-by-Step Tracking

> How the multi-step consultation booking flow is instrumented end to end: what the user does, what
> the frontend developer writes, how GTM listens, which GA4 tag fires, and which variables are
> captured. Includes **real, copy-pasteable `dataLayer` JSON** for every step, and the GA4 Funnel
> Exploration setup marketing uses to act on it.

The production booking widget is a three-step flow followed by a confirmation:

```
Step 1: Choose city + clinic + specialty   →  booking_step_complete (step 1)
Step 2: Choose date + time slot            →  booking_step_complete (step 2)
Step 3: Enter contact details              →  booking_step_complete (step 3)
Confirmation: booking saved                →  booking_confirmed  (PRIMARY conversion)
```

`booking_started` fires once when the widget first opens, giving us the funnel's true denominator.

---

## Step 1 — Location & specialty

**User action.** The visitor opens the booking widget and selects a city, a specific clinic, and
the specialty (e.g. Orthopaedics → knee). They tap **Continue**.

**Frontend implementation.** The booking component holds step state in JavaScript. When the user
taps Continue, the component first *validates* the selections (all three chosen). Only if valid does
it push to the `dataLayer` and advance the UI to step 2. The validation gate is critical — we never
record a step as "complete" if it wasn't.

**Which developer writes the dataLayer.** The **frontend developer** who owns the booking widget.
The push lives in the component's step-transition handler, right after validation passes and right
before the UI advances — so the event and the state change are atomic.

**How GTM listens.** A **Custom Event trigger** with condition `Event equals booking_step_complete`
fires a GA4 Event tag. Because all three steps share one event name, the same trigger handles every
step; the `step_number` parameter differentiates them downstream.

**Which GA4 tag fires.** A single **GA4 Event tag** named *"GA4 – booking_step_complete"*, event
name `booking_step_complete`, with the parameters mapped from Data Layer Variables.

**Which variables are captured.** `step_number`, `step_name`, `clinic_location`, `specialty`.
(`slot_type` is added from step 2 onward.)

```json
{
  "event": "booking_step_complete",
  "step_number": 1,
  "step_name": "location_specialty_selected",
  "clinic_location": "Indiranagar",
  "specialty": "Orthopaedics"
}
```

---

## Step 2 — Date & time slot

**User action.** The visitor picks an available appointment date and a time slot (e.g. an after-work
7:30 pm slot), then taps **Continue**.

**Frontend implementation.** The widget validates that a bookable slot is selected (and still
available), then pushes and advances. `slot_type` (e.g. `weekday_evening`) is captured because *when*
people book is a real optimisation lever for a working-professional audience.

**Which developer writes the dataLayer.** Same frontend developer / booking component — the step-2
transition handler.

**How GTM listens.** The same `Event equals booking_step_complete` Custom Event trigger. No new
trigger needed — this is the payoff of the "one event, `step_number` parameter" design.

**Which GA4 tag fires.** The same *"GA4 – booking_step_complete"* tag.

**Which variables are captured.** `step_number` (=2), `step_name`, `clinic_location`, `specialty`,
`slot_type`, `appointment_date`.

```json
{
  "event": "booking_step_complete",
  "step_number": 2,
  "step_name": "slot_selected",
  "clinic_location": "Indiranagar",
  "specialty": "Orthopaedics",
  "slot_type": "weekday_evening",
  "appointment_date": "2026-07-09"
}
```

---

## Step 3 — Contact details

**User action.** The visitor enters name, mobile number and (optionally) reason for visit, then taps
**Confirm booking**.

**Frontend implementation.** The widget validates the contact fields (name present, 10-digit Indian
mobile). On valid input it pushes `booking_step_complete` for step 3 and then calls the backend to
persist the booking. It does **not** fire `booking_confirmed` yet — confirmation is only pushed once
the backend returns a real `booking_id` (see next section). This ordering prevents counting a
"confirmed" booking that actually failed server-side.

**Which developer writes the dataLayer.** Frontend developer for the step-3 completion push; the
`booking_confirmed` push is fired from the success callback of the booking API request.

**How GTM listens.** Step-3 completion → the shared `booking_step_complete` trigger. Confirmation →
a separate `Event equals booking_confirmed` trigger.

**Which GA4 tag fires.** *"GA4 – booking_step_complete"* for the step; *"GA4 – booking_confirmed"*
(marked as a **Key Event**) for the confirmation.

**Which variables are captured.** Step 3: `step_number` (=3), `step_name`, plus everything carried
forward. We deliberately **do not** put PII (name, phone) into GA4 parameters — that data goes to the
CRM via the backend (Task 3), not into analytics.

```json
{
  "event": "booking_step_complete",
  "step_number": 3,
  "step_name": "contact_details_entered",
  "clinic_location": "Indiranagar",
  "specialty": "Orthopaedics",
  "slot_type": "weekday_evening",
  "appointment_date": "2026-07-09"
}
```

---

## Confirmation — `booking_confirmed` (primary conversion)

**User action.** The backend saves the booking and returns a confirmation; the user sees the
"You're booked" screen.

**Frontend implementation.** In the **success callback** of the booking API call, the app pushes
`booking_confirmed` with the server-generated `booking_id`. Using the server id (a) proves the
booking really persisted and (b) gives us an idempotency/dedup key so a page refresh on the
confirmation screen doesn't double-count (GTM/GA4 can dedupe on `booking_id`, and Google Ads uses it
as the transaction/order id). `value` + `currency` let us attribute revenue.

**How GTM listens.** `Event equals booking_confirmed` Custom Event trigger.

**Which GA4 tag fires.** *"GA4 – booking_confirmed"*, marked as a GA4 **Key Event**, and the source
of the imported Google Ads conversion.

**Which variables are captured.** `booking_id`, `clinic_location`, `specialty`, `appointment_date`,
`value`, `currency`.

```json
{
  "event": "booking_confirmed",
  "booking_id": "ORTHO-2026-000482",
  "clinic_location": "Indiranagar",
  "specialty": "Orthopaedics",
  "appointment_date": "2026-07-09",
  "value": 800,
  "currency": "INR"
}
```

---

## GA4 Funnel Exploration

In GA4 → **Explore → Funnel exploration**, we build a **closed** funnel with these ordered steps:

1. `booking_started`
2. `booking_step_complete` where `step_number = 1`
3. `booking_step_complete` where `step_number = 2`
4. `booking_step_complete` where `step_number = 3`
5. `booking_confirmed`

Add a **breakdown** by `clinic_location` (and a segment by `device_type`) to see where each city and
device drops. Enable "Show elapsed time" to spot steps where users stall.

### Worked example — where the money leaks

| Step | Event | Users | Step-to-step conversion | Drop-off | Completion rate (of started) |
|------|-------|------:|------------------------:|---------:|-----------------------------:|
| 1 | `booking_started` | 4,000 | — | — | 100% |
| 2 | Step 1 complete (location/specialty) | 3,120 | 78.0% | 22.0% (880) | 78.0% |
| 3 | Step 2 complete (slot selected) | 1,950 | 62.5% | 37.5% (1,170) | 48.8% |
| 4 | Step 3 complete (contact entered) | 1,560 | 80.0% | 20.0% (390) | 39.0% |
| 5 | `booking_confirmed` | 1,310 | 84.0% | 16.0% (250) | **32.8%** |

### How marketing uses this

The **step 2 → step 3 slot-selection drop (37.5%)** is the biggest single leak — more people
abandon at "pick a date/time" than anywhere else. That points to a *supply* or *UX* problem: too few
convenient slots for a working audience, or a clunky slot picker. Marketing's response isn't "spend
more on ads" — it's "add weekday-evening/weekend slots and simplify the picker," then re-measure. The
`slot_type` breakdown confirms whether evening/weekend slots convert better, justifying the fix.

The final `booking_confirmed` drop (16%) overlaps with `booking_error` events — cross-referencing the
two tells us whether that last leak is user hesitation (fixable with reassurance copy) or a technical
failure (fixable by engineering). The `clinic_location` breakdown shows which city/clinic underperforms
so budget and slot capacity can be reallocated. In short: the funnel turns "our cost-per-booking is
too high" into a *specific, ranked list of fixes*, and every fix is re-measurable in the same report.
