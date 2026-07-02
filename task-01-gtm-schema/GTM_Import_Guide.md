# GTM Container — Import & Setup Guide

`GTM_Container_OrthoNow.json` is a **real, importable Google Tag Manager container export** built
directly from the [event schema](GTM_Event_Schema.md). Import it into any Web container and you get
the variables, triggers and tags pre-wired — you only need to plug in three IDs.

## What's inside

| Type | Count | Details |
|------|------:|---------|
| Built-in variables | 14 | Page URL/Path, Click Element/URL/Text/Classes/ID/Target, Scroll Depth Threshold/Units/Direction, Event |
| User-defined variables | 22 | 3 constants (GA4 ID, Ads ID, Ads label) + 19 Data Layer Variables |
| Triggers | 11 | 6 custom-event, 3 link-click (call / WhatsApp / PDF), 1 element-visibility (clinic), 1 scroll (blog 25/50/75/90) |
| Tags | 14 | Google tag (GA4 base), Conversion Linker, 11 GA4 event tags, 1 Google Ads conversion |

Every GA4 event tag maps its parameters from Data Layer Variables exactly as the schema specifies,
and `booking_confirmed` is the one flagged as the primary conversion.

## How to import

1. In **Google Tag Manager**, open (or create) a **Web** container.
2. **Admin → Import Container**.
3. Choose the file `GTM_Container_OrthoNow.json`.
4. Select a workspace (a **new workspace** is safest).
5. Choose **Merge → Rename conflicting tags/triggers/variables** for an existing container, or
   **Overwrite** for a brand-new empty one.
6. Click **Confirm**. GTM shows a preview of everything that will be added — you should see 14 tags,
   11 triggers and 22 variables.

## Post-import setup (3 values to replace)

The container ships with placeholders so it imports cleanly. Update these constants under
**Variables → User-Defined**:

| Variable | Placeholder | Replace with |
|----------|-------------|--------------|
| `CONST - GA4 Measurement ID` | `G-XXXXXXXXXX` | Your GA4 Measurement ID |
| `CONST - Google Ads Conversion ID` | `AW-XXXXXXXXXX` | Your Google Ads conversion ID |
| `CONST - Google Ads Conversion Label` | `AbC-D_efG-h12_34-567` | The conversion label for the "Consultation booked" action |

Then:

1. In **GA4 → Admin → Events**, mark `booking_confirmed` as a **Key Event**.
2. In **GA4 → Admin → Custom definitions**, register the custom dimensions (`step_number`,
   `step_name`, `clinic_location`, `specialty`, `booking_id`, `entry_point`, `error_type`,
   `article_title`, `pain_area`, `link_location`) so they're queryable in Explorations.
3. **Recommended:** import `booking_confirmed` into Google Ads **from GA4** (see
   [Google_Ads_Conversion.md](Google_Ads_Conversion.md)) and then **pause or delete the client-side
   `Google Ads - Booking Conversion` tag** — keep only one path to avoid double-counting. The
   client-side `awct` tag is included only as a fallback.

## Verify it works

1. Enable **Preview** (Tag Assistant) and load the site.
2. Fire an event — e.g. submit the [Task 2 landing page](../task-02-landing-page/index.html) form.
3. In Tag Assistant you should see the `consultation_form_submitted` event and the
   **GA4 - consultation_form_submitted** tag fire, with `form_name` and `clinic_preference` populated.
4. Trigger a `tel:` link → **GA4 - call_now_click** fires.
5. On a confirmed booking → **GA4 - booking_confirmed** and the Google Ads conversion fire, and
   `booking_id` appears as the order id.
6. Check **GA4 → DebugView** to confirm the events and parameters land in GA4.

## Notes

- `accountId`/`containerId` are `0` and the public ID is `GTM-XXXXXXX` on purpose — GTM assigns real
  IDs on import, so the file is portable into any account.
- PII is deliberately never mapped to GA4: `consultation_form_submitted` sends only `form_name` and
  `clinic_preference`, not the visitor's name or phone (those go to the CRM via the Task 3 backend).
- The `awct` conversion tag and the GA4-import path are mutually exclusive by design — pick one.
