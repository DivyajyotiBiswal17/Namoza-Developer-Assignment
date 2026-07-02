# Task 03 — Integration Design 
## OrthoNow Landing Page → HubSpot → WhatsApp → Google Ads


## End-to-End Architecture

When the patient submits the consultation form on the landing page, the flow works as follows:

**Step 1 — Form Submit (via Landing Page)**
The form fires two things simultaneously on validated submit: the client-side GTM dataLayer push (`consultation_form_submitted`) which handles the Google Ads conversion via the GTM container already on the page, and a POST request to a serverless backend function (hosted on Vercel or AWS Lambda) carrying Name, Phone, Clinic Preference, and UTM parameters from the URL.

**Step 2 — HubSpot Contact Create/Update (Direct API)**
The backend function calls HubSpot's **Contacts Search API** first (`POST /crm/v3/objects/contacts/search`), filtering on the `phone` property to check if a contact with that number already exists. Based on the result it either creates a new contact (`POST /crm/v3/objects/contacts`) or updates the existing one (`PATCH /crm/v3/objects/contacts/{id}`), setting Name, Phone, Clinic Preference, Source = `Google Ads - Consultation Landing Page`, and Lead Status = `New Enquiry`.

I chose a **direct API call** over native HubSpot embed, Zapier, or Make for one reason: control. Native embeds force email as the identifier, which doesn't apply here. Zapier and Make introduce polling latency of 1–15 minutes on standard plans — that alone would blow the 2-minute WhatsApp SLA before anything else goes wrong. A direct API call is synchronous, fast, and puts the dedup logic fully in our hands.

**Step 3 — WhatsApp Message via Karix (fired in parallel with Step 2)**
The backend function calls Karix's send-message API immediately after the form data is received — in parallel with the HubSpot write, not after it. Waiting for HubSpot to confirm before triggering WhatsApp would add unnecessary latency. The message uses a pre-approved WhatsApp Business template populated with the patient's name and clinic preference.

**Step 4 — Google Ads Conversion**
Handled client-side via GTM as described in Step 1. No server-side call needed here since the GTM container is already on the landing page and fires `consultation_form_submitted` on submit.


## Biggest Failure Point + Fallback

**Failure Point:** The serverless backend function going down

Every step in this integration — the HubSpot write, the Karix WhatsApp call, and the UTM parameter capture — runs through the single backend function. If that function fails (cold start timeout, deployment error, Vercel/AWS Lambda outage), the patient sees a successful form submission on screen but their data is never captured anywhere. The lead is permanently lost with no trace.

**Fallback:**  This includes two layers-

-First, if the backend call fails, the landing page automatically retries the request up to 3 times using exponential backoff — waiting 1 second before the first retry, 2 seconds before the second, and 4 seconds before the third. This handles temporary blips like a cold start or a brief network timeout without the patient ever noticing. 

-Second, every form submission simultaneously sends the raw data (Name, Phone, Clinic Preference, timestamp) to a separate Google Sheet via Google Apps Script — completely independent of the main backend. This acts as a dead letter queue: even if the primary backend is fully down and all 3 retries fail, the lead data is sitting in the sheet and the team gets an email alert from Apps Script so they can process it manually and ensure no patient is lost.


## WhatsApp 2-Minute SLA — What Could Break It and How to Monitor

**Risks:**
- Karix API downtime or rate limiting
- WhatsApp template not pre-approved by Meta, or business account quality score degraded — Meta can throttle or pause message delivery without warning
- Backend function timeout while waiting on the HubSpot write, if WhatsApp is fired sequentially instead of in parallel (fixed by the parallel execution in Step 3 above)
- Patient submitted a landline number or a number with no WhatsApp account — message will fail silently on Karix's end

**Monitoring:**
Every form submission writes a log entry with a `submitted_at` timestamp. Every successful Karix API response writes a `whatsapp_sent_at` timestamp against the same record. A lightweight cron job runs every 1 minutes checking for any record where `whatsapp_sent_at` is null or where the gap between `submitted_at` and `whatsapp_sent_at` exceeds 60 seconds — and fires a Slack or email alert to the team immediately. This gives the team a manual fallback window to call the patient before the lead goes cold.