---
name: dco-system-guide
description: >
  Complete operating guide for Digital Cove Ops (DCO) outreach system.
  Load this whenever Simone asks about DCO, the outreach system, sending emails,
  scraping leads, checking results, or anything related to Digital Cove's
  cold outreach pipeline. Contains all endpoints, credentials, business rules,
  and Cowork workflows in one place.
---

# Digital Cove Ops — Cowork Operating Guide

This document is the single source of truth for running the DCO cold outreach
pipeline via Cowork. Everything you need: system overview, all API endpoints,
business rules, and step-by-step workflows.

---

## System Overview

Digital Cove sells a Google Business Profile (GBP) automation service to local
businesses (florists, barbers, hair salons). Clients send photos via WhatsApp,
we generate AI captions and post them to Google automatically.

The outreach pipeline finds businesses that:
- Are on Google Maps with 15–150 reviews and 3.8★+
- Have not been posting to GBP regularly (our product solves their problem)
- Have a real contact email we can reach

We email them a 3-step drip sequence (day 1, day 5, day 12) via Resend.
Resend free tier: **100 emails/day max. Never exceed this.**

---

## Infrastructure

| Service | URL / Details |
|---|---|
| Backend (ops) | `https://app.thedigitalcove.co.uk` |
| Frontend (landing pages) | `https://thedigitalcove.co.uk` |
| Admin panel | `https://app.thedigitalcove.co.uk/admin/outreach` |
| Database | Supabase project `uevsgwozfcuttayhplzj` |
| Email sending | Resend (free tier — 3,000/month, 100/day) |
| Hosting | Vercel (team: `team_l8PGGaIxGVbyDHwXHc64AOnb`) |

**Auth header for all API calls:**
```
Authorization: Bearer dc-outreach-2026
```

---

## Landing Pages (confirmed live)

Only use these three niches — they have real landing pages:

| Niche | Landing page | Email template group |
|---|---|---|
| `florists` | `thedigitalcove.co.uk/florists` | warm |
| `barbers` | `thedigitalcove.co.uk/barbers` | warm |
| `salons` | `thedigitalcove.co.uk/salons` | warm |

Do **not** outreach for other niches until landing pages are built for them.

---

## All API Endpoints

### 1. Scrape leads automatically (Places API — has small cost)

```
POST https://app.thedigitalcove.co.uk/api/outreach/scrape-leads
Authorization: Bearer dc-outreach-2026
Content-Type: application/json
```

**Body:**
```json
{
  "niche": "barbers",
  "city": "Edinburgh",
  "limit": 12,
  "pages": 1
}
```

**What it does:** Searches Google Maps via Places API, filters 15–150 reviews
/ 3.8★+, fetches each business website, extracts email (tries homepage then
/contact page), inserts into DB. Costs ~£0.80 per 6-city run.

**Prefer the Cowork browser method instead (£0 cost) — see workflow below.**

**Response:**
```json
{
  "success": true,
  "added": 5,
  "no_website": 3,
  "no_email_found": 4,
  "blocked_emails": 2,
  "added_businesses": ["Salon A", "Salon B", ...]
}
```

---

### 2. Import leads from CSV (Cowork's primary import method — free)

```
POST https://app.thedigitalcove.co.uk/api/outreach/import-csv
Authorization: Bearer dc-outreach-2026
Content-Type: text/csv
[body: raw CSV content]
```

**CSV format** (exact column names required):
```csv
business_name,email,location,niche,google_rating,google_review_count,notes
Fallon and Mann Barbers,hello@fallonandmannbarbers.co.uk,Edinburgh,barbers,5.0,60,last post 45 days ago
The Barber Club,thebarberclub.dan@gmail.com,Edinburgh,barbers,5.0,30,no GBP posts found
```

Required: `business_name`, `email`, `niche`, `location`
Optional: `google_rating`, `google_review_count`, `notes`

Valid niches: `florists` `barbers` `salons` (and others, but only these 3 have landing pages)

**The endpoint automatically:**
- Validates email format
- Rejects Wix/Sentry/placeholder emails
- Skips duplicates (same email already in system)
- Returns row-by-row detail on anything rejected

**Response:**
```json
{
  "success": true,
  "imported": 8,
  "skipped_duplicates": 2,
  "invalid": 1,
  "invalid_rows": [
    { "row": 4, "reason": "Invalid email: user@domain.com", "data": "Rum Barber" }
  ]
}
```

**How to call from browser console on `app.thedigitalcove.co.uk`:**
```javascript
const csv = `business_name,email,location,niche,google_rating,google_review_count
Fallon and Mann Barbers,hello@fallonandmannbarbers.co.uk,Edinburgh,barbers,5.0,60`;

fetch('/api/outreach/import-csv', {
  method: 'POST',
  headers: {
    'Content-Type': 'text/csv',
    'Authorization': 'Bearer dc-outreach-2026'
  },
  body: csv
}).then(r => r.json()).then(console.log);
```

---

### 3. View all leads

```
GET https://app.thedigitalcove.co.uk/api/outreach/leads
Authorization: Bearer dc-outreach-2026
```

Returns all leads with status, niche, location, sequence step, etc.

---

### 4. Send a batch (Step 1, 2, or 3)

```
POST https://app.thedigitalcove.co.uk/api/outreach/send-batch
Authorization: Bearer dc-outreach-2026
Content-Type: application/json
```

**Body:**
```json
{
  "step": 1,
  "limit": 50
}
```

Optional filters:
- `"niche": "barbers"` — only send to one niche
- `"variant": "a"` — force a specific template variant (a/b/c)
- `"limit": 20` — cap the batch size

**Step logic:**
- `step: 1` — sends to all leads with `status=new` and `sequence_step=0`
- `step: 2` — sends to leads 5+ days after step 1 was sent
- `step: 3` — sends to leads 12+ days after step 2 was sent

**IMPORTANT — daily limit:**
Resend free tier = 100 emails/day hard cap.
Always check how many have been sent today before triggering a batch.
Check from browser console on `app.thedigitalcove.co.uk`:
```javascript
// Check how many sent today (last 24h)
fetch('/api/outreach/leads').then(r => r.json()).then(d => {
  const today = new Date();
  today.setHours(0,0,0,0);
  const sent = d.filter(l =>
    l.last_email_sent_at &&
    new Date(l.last_email_sent_at) > today
  );
  console.log(`Sent today: ${sent.length} / 100 limit`);
});
```

**Response:**
```json
{
  "success": true,
  "sent": 8,
  "failed": 0,
  "step": 1
}
```

**How to call from browser console:**
```javascript
fetch('/api/outreach/send-batch', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': 'Bearer dc-outreach-2026'
  },
  body: JSON.stringify({ step: 1, limit: 50 })
}).then(r => r.json()).then(console.log);
```

---

### 5. Open tracking (fires automatically)

```
GET https://app.thedigitalcove.co.uk/api/outreach/track/open?ref={leadId}&s={step}
```

Embedded as a 1×1 pixel in every HTML email. Never call manually.

---

### 6. Click tracking (fires automatically)

```
GET https://app.thedigitalcove.co.uk/api/outreach/track/click?ref={leadId}&s={step}&u={encodedUrl}
```

Wraps every link in the email. Logs the click then redirects to the landing page.
Never call manually.

---

### 7. Mark a lead as converted

```
POST https://app.thedigitalcove.co.uk/api/outreach/convert
Authorization: Bearer dc-outreach-2026
Content-Type: application/json
```

```json
{
  "leadId": "uuid-here",
  "channel": "cold_email"
}
```

Use when a lead signs up. Marks `converted_at` timestamp in the DB.

---

### 8. Unsubscribe a lead

```
GET https://app.thedigitalcove.co.uk/unsubscribe?ref={leadId}
```

Sets `unsubscribed=true` and stops all future emails to that lead.
Every email footer includes this link.

---

## Drip Cron (runs automatically)

Vercel runs a daily cron at **9am** that automatically:
- Sends step 2 to all leads where step 1 was sent 5+ days ago
- Sends step 3 to all leads where step 2 was sent 12+ days ago

You don't need to trigger this manually. It's configured in `vercel.json`.
To check if it ran: look at Vercel runtime logs for `[Cron] Outreach followup complete`.

---

## Email Templates

There are 27 templates: 3 niche groups × 3 variants (A/B/C) × 3 steps.

**Groups:**
- `warm` — florists, barbers, salons, beauty, tattoo, dog-groomers
- `trades` — electricians, plumbers, cleaners, estate-agents
- `health` — dentists, vets, personal-trainers, restaurants

**Variants rotate** A→B→C→A→B→C across the batch automatically.

**Subjects used today (examples):**
- A: `"your Google listing, Medusa"` / `"noticed something about Fratelli Barbers"`
- B: `"barbers in Edinburgh — thought you should know"` / `"what other salons are doing"`
- C: `"quick question, Roku"` / `"ROKU — question about ROKU Salon"`

**FROM:** `Chloe from Digital Cove <chloe@thedigitalcove.co.uk>`
**REPLY-TO:** `SalesSupport@thedigitalcove.co.uk` (M365 inbox — all replies land here)

---

## Email Quality Rules

NEVER import or send to:
- `user@domain.com`, `info@example.com`, `test@test.com` (placeholders)
- Any `@sentry-next.wixpress.com` address (Wix internal monitoring)
- Any `@webador.com`, `@squarespace.com`, `@shopify.com` (website builder emails)
- Any `@wix.com`, `@godaddy.com`, `@hubspot.com` (platform emails)
- Any email with a hex string before the `@` (Sentry monitoring pattern)
- Any email containing `&quot;`, `\"`, or HTML entities
- Any email with a TLD shorter than 2 characters

The import endpoint enforces these automatically, but apply them manually
when building CSVs in Cowork too.

---

## Cowork Workflow — Lead Scraping (£0 cost, preferred method)

This is the primary way to find new leads. Costs nothing. Higher quality
than the API scraper because a real browser extracts emails from rendered pages.

### Before you start

Ask Simone:
1. Which niche? (`florists` / `barbers` / `salons`)
2. Which city? (`Glasgow` / `Edinburgh` / `Dundee` / `Aberdeen` / `Stirling` / `Perth` / `Paisley`)
3. How many leads? (default 15)
4. Check how many emails remain today (100 - already sent)

### Step 1 — Open Google Maps

Navigate to `https://maps.google.com` in Chrome.
Search: `[niche] in [city]` — e.g. `barbers in Edinburgh`

### Step 2 — Filter results

For each result in the left panel:

**Skip immediately if:**
- Fewer than 15 reviews OR more than 150 reviews
- Less than 3.8 stars
- Chain brand (TONI&GUY, Regis, Supercuts, Great Clips)
- "Permanently closed" or "Temporarily closed"

**Click the listing, then:**
1. Check the "Updates" or "Posts" tab
   - Posted in the last 14 days → SKIP (already active on GBP)
   - Last post 30+ days ago, or no posts → GOOD TARGET
2. Note: business name, rating, review count, days since last post
3. Click the "Website" link → open in new tab

### Step 3 — Find the email

With the business website open in a real browser:

1. Scan for a mailto link or visible email
2. If not on homepage, try in order:
   - `/contact`
   - `/contact-us`
   - `/about`
   - `/about-us`
3. **Accept** if the email ends in a real business domain
4. **Reject** if:
   - Domain is wix.com, squarespace.com, sentry.io, webador.com, etc.
   - Email is user@domain.com or any obvious placeholder
   - Long hex string before the @ (Sentry monitoring address)
   - Contains HTML entities (&quot; etc.)

**Tips for hard cases:**
- Booking system only (Fresha, Treatwell, Booksy) → usually no email → skip
- Check their Instagram bio if website has no email
- Facebook "About" section often has email even when website doesn't

### Step 4 — Build the CSV

Record each valid lead in this format:

```csv
business_name,email,location,niche,google_rating,google_review_count,notes
Fallon and Mann Barbers,hello@fallonandmannbarbers.co.uk,Edinburgh,barbers,5.0,60,last post 45 days ago
The Barber Club,thebarberclub.dan@gmail.com,Edinburgh,barbers,5.0,30,no GBP posts found
Half-Cut Barbershop,booking@halfcutbarbershop.co.uk,Edinburgh,barbers,4.9,121,never posted
```

### Step 5 — Show Simone for review

Display the table. Ask:
> "Here are [N] leads for [niche] in [city]. Any to remove before I import?"

Wait for confirmation or removals.

### Step 6 — Save CSV

Save to desktop as:
`dco-leads-[niche]-[city]-[YYYY-MM-DD].csv`

### Step 7 — Import

Navigate to `https://app.thedigitalcove.co.uk/admin/outreach` in Chrome.
Open browser console (Cmd+Option+J on Mac / F12 on Windows).

Run:
```javascript
const csv = `[paste full CSV content here]`;
fetch('/api/outreach/import-csv', {
  method: 'POST',
  headers: {
    'Content-Type': 'text/csv',
    'Authorization': 'Bearer dc-outreach-2026'
  },
  body: csv
}).then(r => r.json()).then(d => {
  console.log('Imported:', d.imported);
  console.log('Skipped duplicates:', d.skipped_duplicates);
  console.log('Invalid:', d.invalid_rows);
});
```

### Step 8 — Report to Simone

```
Done! Results for [niche] in [city]:
✅ Imported: [N] new leads
⏭  Skipped: [N] duplicates (already in system)
❌ Rejected: [N] rows — [reasons if any]

Ready to send. You have [X] daily email slots remaining today.
Shall I trigger a send now, or wait until tomorrow's batch?
```

---

## Cowork Workflow — Triggering a Send

After importing leads (or any time Simone wants to send):

### Check daily count first

In browser console on `app.thedigitalcove.co.uk`:
```javascript
fetch('/api/outreach/leads').then(r => r.json()).then(leads => {
  const today = new Date();
  today.setHours(0,0,0,0);
  const todaySends = leads.filter(l =>
    l.last_email_sent_at && new Date(l.last_email_sent_at) >= today
  ).length;
  const newReady = leads.filter(l => l.status === 'new' && l.sequence_step === 0).length;
  console.log(`Sent today: ${todaySends}/100`);
  console.log(`Ready to send (step 1): ${newReady}`);
});
```

### Trigger the send

```javascript
const remaining = 100 - [todaySends]; // replace with actual count
fetch('/api/outreach/send-batch', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': 'Bearer dc-outreach-2026'
  },
  body: JSON.stringify({ step: 1, limit: remaining })
}).then(r => r.json()).then(console.log);
```

**The batch sends 1 email per second** — if sending 30 leads it takes ~30 seconds.
The response only arrives after all emails are sent, so wait for it.

### Report back

```
Batch complete:
✅ Sent: [N] emails
❌ Failed: [N] (if any — these are usually bad email addresses)
Remaining daily limit: [100 - total] slots

Step 2 drip will fire automatically in 5 days for today's leads.
Step 3 fires automatically in 12 days.
```

---

## Cowork Workflow — Checking Results

Simone can ask "how are the emails doing?" or "show me open rates".

Navigate to `https://app.thedigitalcove.co.uk/admin/outreach` — the admin
panel shows the full sends log with open/click tracking.

Or query the DB directly:
```javascript
// Check open rate for today's sends
fetch('/api/outreach/leads').then(r => r.json()).then(leads => {
  const contacted = leads.filter(l => l.status === 'contacted');
  console.log('Total contacted:', contacted.length);
  // Opened and clicked stats come from the outreach_sends table
  // — view at /admin/outreach
});
```

---

## Cowork Workflow — Checking for Replies

Prospect replies go to `SalesSupport@thedigitalcove.co.uk` (M365 inbox).
Cowork cannot check this inbox directly.
Tell Simone to check it manually — any reply from a business is warm interest.

If Simone gets a positive reply:
1. Mark the lead as converted via the API:
```javascript
fetch('/api/outreach/convert', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': 'Bearer dc-outreach-2026'
  },
  body: JSON.stringify({ leadId: 'UUID_HERE', channel: 'cold_email' })
}).then(r => r.json()).then(console.log);
```
2. Note: the lead UUID can be found in the admin panel or by searching the leads list

---

## Key Business Rules

| Rule | Detail |
|---|---|
| Daily send limit | 100 emails/day (Resend free tier) — hard cap |
| Target: reviews | 15–150 Google reviews |
| Target: rating | 3.8★ or above |
| Target: GBP activity | No posts in 30+ days (or never posted) |
| Skip: chains | TONI&GUY, Regis, Supercuts, Great Clips, etc. |
| Sequence timing | Step 2: 5 days after step 1. Step 3: 12 days after step 2 |
| Drip cron | Runs daily at 9am automatically — no manual trigger needed |
| Geographic model | One business per postcode area per niche (exclusivity) |
| Niches with pages | florists, barbers, salons ONLY until more pages built |
| Pricing | £50/month recurring (after 15-day trial at £15) |

---

## Common Tasks — Quick Reference

| What Simone says | What to do |
|---|---|
| "scrape leads for barbers in Glasgow" | Run Cowork browser workflow, NOT the API scraper |
| "send today's emails" | Check daily count, trigger send-batch step 1 |
| "how many sent today" | Check via browser console snippet above |
| "show me the results" | Navigate to `/admin/outreach` |
| "add this lead manually" | Import single-row CSV via import-csv endpoint |
| "mark [business] as converted" | Use convert endpoint with their lead UUID |
| "remove a lead" | Update status to 'skip' in admin panel or via Supabase |
| "check build status" | Vercel project `prj_OunOw4tQpl0FdT6xneN5uFd6EsjS`, team `team_l8PGGaIxGVbyDHwXHc64AOnb` |
| "check the DB" | Supabase project `uevsgwozfcuttayhplzj` |

---

## Known Issues & Fixes Applied

| Issue | Status |
|---|---|
| Wix/Sentry emails slipping through | Fixed — blocklist in import-csv and scraper |
| firstName showing "Salon" or "blunted" | Fixed — smart extraction skips generic words |
| Landing URL showing as tracking URL in email | Fixed — clean URL displayed, tracking in href |
| City field showing postcode ("Edinburgh EH1 3EB") | Fixed — extractCity strips postcode |
| US leads appearing in scrape results | Fixed — skip leads with non-UK postcodes |
| Vercel timeout on large batches | Known — batch of 30+ takes 30s+, browser tab must stay open |

---

## What's Coming Next

- **Google GBP API quota** — case `2-9265000041135` pending (7–10 days from Apr 7). Once approved, GBP post activity checking becomes automated.
- **Rank Scout** — next product: local businesses get a free Google Maps visibility report, converting to subscribers. Built after GBP quota unblocked.
- **More landing pages** — dog groomers, electricians, dentists etc. once conversion data from florists/barbers/salons is in.
- **Self-serve OAuth** — customers connect their own Google account. Currently Simone connects manually via admin dashboard.
