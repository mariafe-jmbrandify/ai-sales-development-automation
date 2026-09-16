# AI SDR Automation System — Standard Operating Procedure

**System:** AI SDR (Sales Development Representative) Automation System
**Stack:** Apify → Make.com → Google Sheets → GoHighLevel → Vapi AI → Twilio → OpenAI
**Owner:** Maria Fe Blanca (JM Brandify)

---

## 1. Purpose & Scope

This SOP covers day-to-day operation, monitoring, and change management for the AI SDR pipeline — from Google Maps lead scraping through AI voice qualification to appointment booking in GoHighLevel. It does not cover initial platform setup (assumed already provisioned) or client-specific onboarding contracts.

**Current production status:** Phases 1–5 (Lead Acquisition → Call Outcome Processing) are live. Phase 6 (SMS Follow-Up) is a **planned enhancement, not yet live** — do not assume SMS follow-ups are running for leads who don't answer until this is explicitly built and confirmed.

---

## 2. Platform Ownership & Access

| Platform | Role in system | Access needed by |
|---|---|---|
| Apify | Scrapes business leads from Google Maps | Whoever configures/runs scrape jobs |
| Make.com | Orchestrates every hand-off between platforms | Whoever maintains scenarios |
| Google Sheets | Temporary lead staging before CRM entry | Make.com service account |
| GoHighLevel (GHL) | CRM — contacts, opportunities, pipeline, calendar | Sales team + automation |
| Vapi AI | AI voice agent that calls and qualifies leads | Whoever maintains the voice agent config/prompts |
| Twilio | Phone numbers + SMS | Vapi (calling) + Make.com (SMS phase, once live) |
| OpenAI | Conversation logic behind the Vapi agent | Whoever maintains prompts |

Keep API keys/credentials for all seven platforms in a shared secrets manager, not hardcoded into Make.com scenarios — a leaked or expired key in any one of these breaks the whole chain, since each phase depends on the previous one's output.

---

## 3. Routine Operating Tasks

| Cadence | Task | Where |
|---|---|---|
| Daily | Check Make.com scenario run history for errors across all active scenarios | Make.com |
| Daily | Review new leads created in GHL — confirm dedupe (Search Existing Contact) is working, no duplicate contacts | GoHighLevel |
| Daily | Check Vapi call logs for failed/errored calls (distinct from "no answer") | Vapi AI dashboard |
| Weekly | Review pipeline stage distribution — are leads moving from Interested → Appointment Booked at an expected rate, or piling up in one stage? | GoHighLevel |
| Weekly | Spot-check 3–5 AI call transcripts for qualification quality and objection handling | Vapi AI |
| Monthly | Review scrape targets/verticals — are they still relevant to active client work? | Apify |
| Monthly | Rotate/verify expiry dates on API keys (Twilio, OpenAI, Vapi, Apify, GHL) | Secrets manager |

---

## 4. Standard Procedures

### 4.1 Adding a new lead source or scrape target
1. Configure the new Apify actor/search parameters (business category, location).
2. Run a small test scrape (10–20 records) before a full run — verify the four required fields (Business Name, Phone Number, Website, City) are populating correctly.
3. Confirm the test batch flows correctly through Make.com → Google Sheets → GHL before scaling up the scrape volume.

### 4.2 Onboarding a new client vertical
1. Duplicate the existing Make.com scenario set rather than editing the live one directly.
2. Update the Vapi prompt/script for the new vertical's qualification questions.
3. Confirm the GHL pipeline stages match the new vertical's sales process — don't assume the existing pipeline stages generalize.
4. Run a full test lead end-to-end (scrape → CRM → AI call → outcome) before pointing real scrape volume at it.

### 4.3 Updating the AI voice agent's prompt/script
1. Make the change in a Vapi test assistant, never the production assistant directly.
2. Run at least 3 test calls covering: an interested prospect, an objection scenario, and a not-interested prospect.
3. Confirm the webhook payload Vapi sends to Make.com on call completion hasn't changed shape — a prompt change that alters what data Vapi extracts (e.g. renames a qualification field) will break the Make.com routing logic downstream (Section 4.4 in the workflow mapping doc).
4. Promote to production only after all three test scenarios route correctly in GHL.

### 4.4 Handling a stuck or failed Make.com scenario run
1. Open the scenario's run history and identify the exact module that failed.
2. Check whether the failure is upstream data (e.g. a malformed lead record from Apify/Sheets) or a downstream API issue (GHL/Vapi/Twilio rate limit or auth failure).
3. For a data issue: fix or discard the specific bad record; don't rerun the whole batch blind.
4. For an API/auth issue: check the credential first before assuming it's a logic bug — expired GHL or Vapi tokens are the most common cause of a sudden full-scenario failure.

### 4.5 Handling Vapi call failures / no-answer volume spikes
1. Distinguish "no answer" (expected, routine) from "call errored" (a system problem) in the Vapi dashboard — don't treat them the same.
2. A sudden spike in "no answer" is often a Twilio number reputation/spam-flagging issue, not a targeting problem — check Twilio's number health before re-targeting leads.
3. A spike in "call errored" points to a Vapi/OpenAI-side issue (API downtime, rate limit, or a broken prompt change) — check Section 4.3's change log first.

---

## 5. Troubleshooting Guide

| Symptom | Likely cause | Fix |
|---|---|---|
| Leads not appearing in Google Sheets | Apify scrape failed or returned empty results | Check Apify run log; verify search parameters still return results on Google Maps |
| Duplicate contacts in GHL | "Search Existing Contact" step matching on the wrong field (e.g. name instead of phone) | Verify the Make.com dedupe module's match field |
| Opportunities created but AI never calls | Trigger between "Watch Contact Updated" and the HTTP request to Vapi isn't firing | Check the Make.com webhook/watch trigger status — it may be paused or hit a rate limit |
| AI calls happen but outcomes don't update GHL | Webhook from Vapi to Make.com is failing, or the payload shape changed | Check Make.com's incoming webhook log; compare payload structure against Section 4.3 |
| Appointment booked in Vapi but no calendar event in GHL | Vapi's calendar-check/booking tool call to GHL failed silently | Check Vapi's tool-call logs for the booking step specifically, not just the overall call outcome |
| Leads stuck in "Not Interested" with no nurture activity | Lead nurture workflow in GHL isn't enrolling contacts, or Phase 6 (SMS) is assumed live but isn't built yet | Confirm nurture workflow trigger; confirm Phase 6 status before troubleshooting SMS as if it exists |

---

## 6. Change Management

- No change to a live Make.com scenario, GHL workflow, or Vapi prompt happens directly in production. Duplicate first, test, then promote.
- Any change to what data Vapi extracts from a call (new qualification field, renamed field) must be checked against every downstream Make.com module that reads that field — this is the single most common way this system breaks silently.
- Log every production change (what, why, who, when) somewhere durable — even a simple dated entry in a shared doc. Without this, diagnosing "when did this start happening" across seven platforms is very slow.

---

## 7. KPIs to Monitor

| Metric | Where measured | Why it matters |
|---|---|---|
| Leads scraped per run | Apify | Volume going into the top of the funnel |
| Contact rate (calls connected / calls attempted) | Vapi + Twilio | Catches number reputation/deliverability problems early |
| Qualification rate (interested / connected) | GHL pipeline | Signal on lead targeting and AI script quality |
| Booking rate (booked / interested) | GHL calendar | Signal on the AI agent's scheduling flow specifically |
| Time-to-first-contact (lead created → first call attempt) | GHL + Vapi timestamps | The core efficiency claim of this system — should be minutes, not hours |
