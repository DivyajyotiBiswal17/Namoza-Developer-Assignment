# Task 03 — Integration Design 
## OrthoNow Landing Page → HubSpot → WhatsApp → Google Ads

---

## End-to-End Architecture

When the patient submits the consultation form on the landing page, the flow works as follows:

**Step 1 — Form Submit (Landing Page)**
The form fires two things simultaneously on validated submit: the client-side GTM dataLayer push (`consultation_form_submitted`) which handles the Google Ads conversion via the GTM container already on the page, and a POST request to a serverless backend function (hosted on Vercel or AWS Lambda) carrying Name, Phone, Clinic Preference, and UTM parameters from the URL.

**Step 2 — HubSpot Contact Create/Update (Direct API, not native embed or Zapier)**
The backend function calls HubSpot's **Contacts Search API** first (`POST /crm/v3/objects/contacts/search`), filtering on the `phone` property to check if a contact with that number already exists. Based on the result it either creates a new contact (`POST /crm/v3/objects/contacts`) or updates the existing one (`PATCH /crm/v3/objects/contacts/{id}`), setting Name, Phone, Clinic Preference, Source = `Google Ads - Consultation Landing Page`, and Lead Status = `New Enquiry`.

I chose a **direct API call** over native HubSpot embed, Zapier, or Make for one reason: control. Native embeds force email as the identifier, which doesn't apply here. Zapier and Make introduce polling latency of 1–15 minutes on standard plans — that alone would blow the 2-minute WhatsApp SLA before anything else goes wrong. A direct API call is synchronous, fast, and puts the dedup logic fully in our hands.

**Step 3 — WhatsApp Message via Karix (fired in parallel with Step 2)**
The backend function calls Karix's send-message API immediately after the form data is received — in parallel with the HubSpot write, not after it. Waiting for HubSpot to confirm before triggering WhatsApp would add unnecessary latency. The message uses a pre-approved WhatsApp Business template populated with the patient's name and clinic preference.

**Step 4 — Google Ads Conversion**
Handled client-side via GTM as described in Step 1. No server-side call needed here since the GTM container is already on the landing page and fires `consultation_form_submitted` on submit.

---

## Biggest Failure Point + Fallback

**The biggest failure point is HubSpot's contact deduplication — which defaults to email, not phone.**

OrthoNow's form collects only Name and Phone, no email. HubSpot will not deduplicate on phone number by default. If two submissions come in with the same phone number — a common scenario in Indian healthcare where a family member books on behalf of a patient, or the same person submits twice — HubSpot will silently create two separate contact records with identical phone numbers, both tagged New Enquiry. This fragments the patient's history and breaks any downstream follow-up logic.

**Fallback:** the backend function always runs the Contacts Search API call first before deciding to create or update. If a phone match is found, it updates the existing record and appends the new submission as a timeline note (name, clinic preference, timestamp) rather than overwriting the contact name. This puts dedup logic in code we control, completely bypassing HubSpot's email-based default matching.

---

## WhatsApp 2-Minute SLA — What Could Break It and How to Monitor

**Risks:**
- Karix API downtime or rate limiting
- WhatsApp template not pre-approved by Meta, or business account quality score degraded — Meta can throttle or pause message delivery without warning
- Backend function timeout while waiting on the HubSpot write, if WhatsApp is fired sequentially instead of in parallel (avoided by the parallel execution in Step 3 above)
- Patient submitted a landline number or a number with no WhatsApp account — message will fail silently on Karix's end

**Monitoring:**
Every form submission writes a log entry with a `submitted_at` timestamp. Every successful Karix API response writes a `whatsapp_sent_at` timestamp against the same record. A lightweight cron job runs every 2 minutes checking for any record where `whatsapp_sent_at` is null or where the gap between `submitted_at` and `whatsapp_sent_at` exceeds 120 seconds — and fires a Slack or email alert to the team immediately. This gives the team a manual fallback window to call the patient before the lead goes cold.