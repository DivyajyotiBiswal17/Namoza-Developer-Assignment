# Task 01 — GTM Event Schema 


## DELIVERABLE 1: Complete GTM Event Schema

| # | Event Name | Trigger Type (GTM) | Key Parameters | GA4 Report |
|---|---|---|---|---|
| 1 | `booking_step_complete` | Custom Event — fires on `dataLayer.push` written by front-end dev at each step transition | `step_number` (1/2/3), `step_name`, `clinic_location`, `specialty` | Funnel Exploration → "Booking Funnel"; Audience: "Started Booking" |
| 2 | `booking_confirmed` | Custom Event — fires on `dataLayer.push` after step 3 backend confirmation | `clinic_location`, `specialty`, `booking_id`, `preferred_date` | GA4 Conversion Event; Google Ads import (see Section 3) |
| 3 | `call_now_click` | Click Trigger — Just Links, matching `tel:` href pattern | `page_location`, `click_source` (homepage / clinic-page / landing-page), `clinic_location` | Engagement Report; Audience: "High-Intent Callers" |
| 4 | `whatsapp_chat_open` | Click Trigger — Just Links, matching `wa.me` href | `page_location`, `widget_type` (floating-button), `click_source` | Engagement Report; Audience: "WhatsApp Intent" for remarketing |
| 5 | `guide_download_requested` | Custom Event — fires on gated form submit (before PDF unlocks) | `form_location`, `lead_source`, `has_phone` (boolean — no raw PII) | Lead Gen Report; Audience: "Content Downloaders" for remarketing |
| 6 | `guide_download_complete` | File Download Trigger (Auto-Event) or Click Trigger on PDF link | `file_name`, `file_extension`, `link_url` | Engagement Report; cross-referenced with `guide_download_requested` to measure gate conversion rate |
| 7 | `clinic_page_view` | Page View Trigger — filtered by URL path `/clinics/*` or History Change for JS routing | `clinic_name`, `clinic_city`, `page_path` | Pages & Screens Report with custom dimension `clinic_name`; Audience per city for geo-targeted remarketing |
| 8 | `blog_scroll_depth` | Scroll Depth Trigger — 25 / 50 / 75 / 90% thresholds | `scroll_percentage`, `article_title`, `article_category`, `page_path` | Engagement Report; Audience: "Engaged Readers 75%+" for content remarketing |
| 9 | `consultation_form_submitted` | Custom Event — fires on validated form submit on landing page (see Task 02) | `form_location`, `campaign_source`, `clinic_preference`, `has_phone` | GA4 Conversion Event; secondary/observation conversion in Google Ads |

---

## DELIVERABLE 2: 3-Step Booking Form: Funnel Drop-Off Tracking

### Why custom dataLayer pushes are required

GTM **cannot natively detect** a step change inside a JavaScript-driven multi-step form. There is no new URL, no full page reload, and no DOM event GTM listens to by default. The only way to track each step is for the **front-end developer** to manually fire a `window.dataLayer.push(...)` call inside the app's step-transition logic — GTM simply listens for what gets pushed.

**Division of responsibility:**
- **Front-end developer** writes the `dataLayer.push` inside each step's transition handler (after client-side validation passes)
- **GTM owner (me)** configures the Custom Event trigger and GA4 event tag to capture it, and maps the variables

---

### GTM Trigger Setup (same trigger, used for all 3 steps)

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

**Dev brief for step 2 (verbatim brief to give the front-end team):**
> "In your step 2 transition handler — the function that runs when the user clicks Next and all fields pass validation — add this before you call the API or advance the UI:
> ```js
> window.dataLayer = window.dataLayer || [];
> window.dataLayer.push({
>   event: 'booking_step_complete',
>   step_number: 2,
>   step_name: 'contact_details_entered',
>   clinic_location: formState.clinicLocation,
>   has_phone: !!formState.phone
> });
> ```
> Fire it once per step only — guard against re-fires if the user navigates back and forward within the form. Do not include the raw phone number or name."

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

**Import: `booking_confirmed`**

**Why this one:**

| Event | Why not chosen |
|---|---|
| `consultation_form_submitted` | Captures lead intent only — a 2-field form fill is a soft signal. Optimizing toward this trains Smart Bidding to find people who fill forms, not people who actually book |
| `booking_step_complete` (step 1 or 2) | Mid-funnel, leaky — step 1 fires on anyone who selects a clinic, massively overcounting low-intent traffic |
| `call_now_click` | Unverified intent — a click is not a conversation, and we have no visibility into whether the call converted |
| `booking_confirmed` | Fires only after backend confirms a real appointment with a `booking_id`. Closest event to actual business value. Gives Smart Bidding a clean, high-intent signal to optimize toward |
