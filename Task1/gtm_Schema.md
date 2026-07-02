# Task 01 — GTM Event Schema 


## DELIVERABLE 1: Complete GTM Event Schema

| S.no. | Event Name | Trigger Type | Key Parameters | GA4 Report |
|---|---|---|---|---|
| 1 | booking_step_complete | Custom Event — fires on `dataLayer.push` | `step_number` (1/2/3), `step_name`, `clinic_location`, `specialty` | Funnel Exploration → "Booking Funnel"; Audience: "Started Booking" |
| 2 | booking_confirmed | Custom Event — fires on `dataLayer.push` after step 3 backend confirmation | `clinic_location`, `specialty`, `booking_id`, `preferred_date` | GA4 Conversion Event; Google Ads import |
| 3 | call_now_click | Click Trigger — Just Links, matching `tel:` href pattern | `page_location`, `click_source` (homepage / clinic-page / landing-page), `clinic_location` | Engagement Report; Audience: "High-Intent Callers" |
| 4 | whatsapp_chat_open | Click Trigger — Just Links, matching `wa.me` href | `page_location`, `widget_type` (floating-button), `click_source` | Engagement Report; Audience: "WhatsApp Intent" for remarketing |
| 5 | guide_download_requested | Custom Event — fires on gated form submit (before PDF unlocks) | `form_location`, `lead_source`, `has_phone` (boolean — no raw PII) | Lead Gen Report; Audience: "Content Downloaders" for remarketing |
| 6 | guide_download_complete | File Download Trigger (Auto-Event) or Click Trigger on PDF link | `file_name`, `file_extension`, `link_url` | Engagement Report; cross-referenced with `guide_download_requested` to measure gate conversion rate |
| 7 | clinic_page_view | Page View Trigger — filtered by URL path `/clinics/*` or History Change for JS routing | `clinic_name`, `clinic_city`, `page_path` | Pages & Screens Report with custom dimension `clinic_name`; Audience per city for geo-targeted remarketing |
| 8 | blog_scroll_depth | Scroll Depth Trigger — 25 / 50 / 75 / 90% thresholds | `scroll_percentage`, `article_title`, `article_category`, `page_path` | Engagement Report; Audience: "Engaged Readers 75%+" for content remarketing |
| 9 | consultation_form_submitted | Custom Event — fires on validated form submit on landing page | `form_location`, `campaign_source`, `clinic_preference`, `has_phone` | GA4 Conversion Event; secondary/observation conversion in Google Ads |

---

## DELIVERABLE 2: 3-Step Booking Form: Funnel Drop-Off Tracking

### Why custom dataLayer pushes are required

GTM cannot natively detect a step change inside a JavaScript-driven multi-step form. There is no new URL, no full page reload, and no DOM event GTM listens to by default. The only way to track each step is for the **front-end developer** to manually fire a `window.dataLayer.push(...)` call inside the app's step-transition logic — GTM simply listens for what gets pushed.

**Division of responsibility:**
- **Front-end developer** writes the `dataLayer.push` inside each step's transition handler (after client-side validation passes)
- **GTM owner (me)** configures the Custom Event trigger and GA4 event tag to capture it, and maps the variables

---

### GTM Trigger Setup 

| Trigger Setting | Value |
|---|---|
| Trigger Type | Custom Event |
| Event Name | `booking_step_complete` |
| This trigger fires on | All Custom Events (filtered by step in the GA4 tag if needed) |

A second Custom Event trigger for `booking_confirmed` covers step 3 confirmation separately.

---

### dataLayer JSON — Step 1 (Location + Specialty Selected)

```json
{
  "event": "booking_step_complete",
  "step_number": 1,
  "step_name": "location_specialty_selected",
  "clinic_location": "Indiranagar - Bengaluru",
  "specialty": "Knee & Joint"
}
```

**When it fires:** Inside the "Next" button handler of step 1, after client-side validation passes and before the UI advances to step 2.

---

### dataLayer JSON — Step 2 (Contact Details Entered)

```json
{
  "event": "booking_step_complete",
  "step_number": 2,
  "step_name": "contact_details_entered",
  "clinic_location": "Indiranagar - Bengaluru",
  "has_phone": true
}
```

**When it fires:** Inside the "Next" button handler of step 2, after validation of name/phone/date passes.

**Note:** Raw phone number and name are intentionally excluded — only a boolean `has_phone` is pushed. No PII enters the dataLayer, keeping GA4 data collection PDPA/DPDP compliant.

---

### dataLayer JSON — Step 3 (Booking Confirmed)

```json
{
  "event": "booking_confirmed",
  "step_number": 3,
  "step_name": "booking_confirmed",
  "clinic_location": "Indiranagar - Bengaluru",
  "specialty": "Knee & Joint",
  "booking_id": "ORTHO-20260702-00312"
}
```

**When it fires:** After the backend confirms the booking and returns a `booking_id` — not on the button click, so we only count real confirmed bookings.

---

### Surfacing Drop-Off in GA4 Funnel Exploration

1. Go to **Explore → Funnel Exploration**, create new exploration
2. Add steps:
   - Step 1: Event = `booking_step_complete`, Parameter `step_number` = `1`
   - Step 2: Event = `booking_step_complete`, Parameter `step_number` = `2`
   - Step 3: Event = `booking_confirmed`
3. Set funnel type to **Open** initially (to see overall drop-off regardless of session), then switch to **Closed** to see strict sequential completion rate
4. Add **`clinic_location`** as a breakdown dimension — if drop-off is concentrated at one clinic, it's likely a clinic-specific availability issue, not a UX issue
5. The visualization will show exact % drop from step 1→2 and 2→3, which is only possible because of the manual dataLayer pushes above

---

## DELIVERABLE 3: Google Ads Conversion Import

**Chosen conversion: `booking_confirmed`**

The right conversion action to import into Google Ads is `booking_confirmed` — the event that fires after the backend confirms a real appointment and returns a booking ID at step 3 of the booking form. It is the closest proxy to actual business value available in the tracking setup, and it gives Smart Bidding the cleanest possible signal to find more people who will do the same thing.

--`booking_step_complete` at step 1 fires the moment someone selects a clinic and a specialty. That is far too early in the journey — it would count every curious visitor who clicks through a few dropdowns as a conversion, massively inflating numbers and training the algorithm to find people who browse, not people who book.

--`consultation_form_submitted` from the landing page is a 2-field form fill — just a name and a phone number. It captures intent, not commitment. In Indian healthcare lead gen especially, form fill rates can be high while actual appointment show-up rates are much lower. Optimising toward this would drive cheap leads that never turn into patients.

--`call_now_click` is a click event, not a conversation. We have no visibility into whether the person who clicked actually spoke to someone, booked anything, or hung up immediately. It is too unverified to bid against.

--`whatsapp_chat_open` has the same problem — opening a WhatsApp link is a single tap, not a signal of real intent. It would be one of the noisiest events in the entire schema to import as a conversion.

--`guide_download_complete` and `guide_download_requested` are content engagement events. They indicate interest in orthopaedic information, not intent to book a consultation. Optimising toward them would bring in an audience of researchers, not patients.

--`blog_scroll_depth` and `clinic_page_view` are purely informational signals — awareness-stage behaviour with no direct link to booking intent.




