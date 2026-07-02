# Task 01 – GTM Event Schema

## 1. GTM Event Schema

| Event Name               | Trigger Type                | Key Parameters (Min. 3)                                        | GA4 Report / Audience                   |
| ------------------------ | --------------------------- | -------------------------------------------------------------- | --------------------------------------- |
| `booking_started`        | Custom Event (`dataLayer`)  | `entry_point`, `device_type`, `page_location`                  | Funnel Exploration                      |
| `booking_step_complete`  | Custom Event (`dataLayer`)  | `step_number`, `step_name`, `clinic_location`, `specialty`     | Funnel Exploration                      |
| `booking_confirmed`      | Custom Event (`dataLayer`)  | `booking_id`, `clinic_location`, `preferred_date`, `specialty` | Conversions Report, Google Ads          |
| `booking_error`          | Custom Event                | `step_number`, `error_type`, `error_message`                   | DebugView                               |
| `call_now_click`         | Click Trigger (`tel:` link) | `page_location`, `clinic_location`, `button_position`          | Engagement Report, High-Intent Audience |
| `whatsapp_click`         | Click Trigger (`wa.me`)     | `page_location`, `clinic_location`, `device_type`              | Engagement Report, Remarketing Audience |
| `patient_guide_submit`   | Form Submission             | `guide_name`, `patient_name`, `phone_number`                   | Lead Generation Report                  |
| `patient_guide_download` | Link Click (PDF)            | `guide_name`, `file_name`, `page_location`                     | Events Report                           |
| `clinic_location_view`   | Page View                   | `clinic_location`, `city`, `page_title`                        | Pages & Screens Report                  |
| `blog_scroll_25`         | Scroll Trigger (25%)        | `article_title`, `scroll_depth`, `page_location`               | Engagement Report                       |
| `blog_scroll_50`         | Scroll Trigger (50%)        | `article_title`, `scroll_depth`, `page_location`               | Engagement Report                       |
| `blog_scroll_75`         | Scroll Trigger (75%)        | `article_title`, `scroll_depth`, `page_location`               | Engaged Readers Audience                |
| `blog_scroll_90`         | Scroll Trigger (90%)        | `article_title`, `scroll_depth`, `page_location`               | Highly Engaged Readers Audience         |

---

# 2. Booking Form Funnel Tracking

The booking form is a custom 3-step form, so the **frontend developer** should push a `dataLayer` event whenever a user successfully completes a step. GTM listens for these custom events and sends them to GA4 for funnel analysis.

### Step 1 – Select Clinic & Specialty

**GTM Trigger:** Custom Event → `booking_step_complete`

```json
{
  "event": "booking_step_complete",
  "step_number": 1,
  "step_name": "location_specialty_selected",
  "clinic_location": "Indiranagar",
  "specialty": "Orthopaedics"
}
```

### Step 2 – Enter Patient Details

**GTM Trigger:** Custom Event → `booking_step_complete`

```json
{
  "event": "booking_step_complete",
  "step_number": 2,
  "step_name": "patient_details_entered",
  "clinic_location": "Indiranagar",
  "specialty": "Orthopaedics",
  "preferred_date": "2026-07-20"
}
```

### Step 3 – Booking Confirmed

**GTM Trigger:** Custom Event → `booking_step_complete`

```json
{
  "event": "booking_step_complete",
  "step_number": 3,
  "step_name": "booking_confirmed",
  "clinic_location": "Indiranagar",
  "specialty": "Orthopaedics",
  "booking_id": "BK104582"
}
```

---

## GA4 Funnel Exploration

Create a **Closed Funnel Exploration** in GA4 using the following steps:

| Funnel Step | Event                   | Filter            |
| ----------- | ----------------------- | ----------------- |
| Step 1      | `booking_step_complete` | `step_number = 1` |
| Step 2      | `booking_step_complete` | `step_number = 2` |
| Step 3      | `booking_step_complete` | `step_number = 3` |

### Example Funnel

| Step   | Users | Drop-off |
| ------ | ----: | -------: |
| Step 1 | 1,000 |        — |
| Step 2 |   700 |      30% |
| Step 3 |   520 |    25.7% |

This report helps identify where users leave the booking process so improvements can be made to increase completed appointments.

---

# 3. Google Ads Conversion Recommendation

I would import **`booking_confirmed`** as the Google Ads conversion because it represents a successfully completed consultation booking, which is OrthoNow's primary business goal. Events such as **Call Now**, **WhatsApp Click**, **Patient Guide Download**, or **Blog Scroll** indicate user interest but do not confirm a booking. Optimizing campaigns for **`booking_confirmed`** gives Google Ads a stronger conversion signal and improves Smart Bidding performance.
