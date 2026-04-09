---
name: dco-system-guide
description: >
  Complete operating guide for Digital Cove Ops (DCO) outreach system.
  Load this whenever Simone asks about DCO, the outreach system, sending emails,
  scraping leads, checking results, or anything related to Digital Cove's
  cold outreach pipeline. Contains all MCP tools, endpoints, credentials,
  business rules, and Cowork workflows in one place.
---

# Digital Cove Ops — Cowork Operating Guide

Single source of truth for running the DCO cold outreach pipeline.
Use **MCP tools directly** wherever possible — they are faster, more reliable,
and don't require navigating the browser console.

---

## Available MCPs — What Each One Does for DCO

### Supabase MCP — database reads and writes
**⚠️ CRITICAL: ALWAYS use `project_id: uevsgwozfcuttayhplzj` explicitly.**
The bare `supabase:execute_sql` (lowercase, no project_id) connects to a
DIFFERENT project (FanRink) and must NEVER be used for DCO work.
Always use the capitalised `Supabase:execute_sql` with the project_id.

Use for: checking leads, importing leads, checking send results, updating
lead status, marking conversions, skip/unsubscribe operations.

### Vercel MCP — deployments and logs
- `Vercel:list_deployments` — check if latest code is live in production
- `Vercel:get_deployment_build_logs` — debug a failed build
- `Vercel:get_runtime_logs` — see what happened when emails were sent
- `Vercel:get_project` — confirm project config

DCO project ID: `prj_OunOw4tQpl0FdT6xneN5uFd6EsjS`
Team ID: `team_l8PGGaIxGVbyDHwXHc64AOnb`

### GitHub MCP — code changes
- `github:push_files` — update skill files or fix code
- `github:get_file_contents` — read current code before editing
- `github:create_or_update_file` — update a single file

DCO backend repo: `DevSim1/Digitalcove-ops` (branch: `claude/setup-nextjs-app-router-z9yaE`)
Skills repo: `DevSim1/claude-skills` (branch: `main`)

### Claude in Chrome — browser automation
Use for: Google Maps scraping (the £0 lead finding method), navigating
the admin panel, triggering the send-batch API (the only thing that
still needs the browser, since it calls Resend).

### Stripe MCP — subscription management
Use for: checking if a customer paid, creating prices, listing subscriptions.
DCO product is ListingPilot (needs renaming to reflect Digital Cove branding).

---

## Infrastructure

| Service | Detail |
|---|---|
| Backend | `https://app.thedigitalcove.co.uk` |
| Frontend | `https://thedigitalcove.co.uk` |
| Admin panel | `https://app.thedigitalcove.co.uk/admin/outreach` |
| Supabase project | `uevsgwozfcuttayhplzj` (68 tables) |
| Vercel project | `prj_OunOw4tQpl0FdT6xneN5uFd6EsjS` |
| Vercel team | `team_l8PGGaIxGVbyDHwXHc64AOnb` |
| Email sending | Resend — 3,000/month, **100/day hard cap** |
| API auth header | `Authorization: Bearer dc-outreach-2026` |

---

## Landing Pages (confirmed live — only use these 3 niches)

| Niche | Landing page |
|---|---|
| `florists` | `thedigitalcove.co.uk/florists` |
| `barbers` | `thedigitalcove.co.uk/barbers` |
| `salons` | `thedigitalcove.co.uk/salons` |

Do not outreach for other niches until their landing pages are built.

---

## MCP Workflows — Use These First

### Check how many emails sent today

Use **Supabase MCP** — no browser needed:

```sql
-- How many sent today
SELECT COUNT(*) as sent_today
FROM outreach_sends
WHERE created_at > NOW() - INTERVAL '24 hours'
  AND error IS NULL;

-- How many new leads ready to send (step 1)
SELECT COUNT(*) as ready_to_send
FROM outreach_leads
WHERE status = 'new'
  AND sequence_step = 0
  AND unsubscribed = false;
```

Call with:
`Supabase:execute_sql` → `project_id: uevsgwozfcuttayhplzj`

---

### Import leads directly to database

After Simone approves the leads table from the Google Maps scrape,
insert directly via **Supabase MCP** — no CSV endpoint needed:

```sql
INSERT INTO outreach_leads (
  business_name, email, location, niche,
  google_rating, google_review_count,
  status, sequence_step, unsubscribed, source
) VALUES
  ('Fallon and Mann Barbers', 'hello@fallonandmannbarbers.co.uk',
   'Edinburgh', 'barbers', 5.0, 60, 'new', 0, false, 'cowork'),
  ('The Barber Club', 'thebarberclub.dan@gmail.com',
   'Edinburgh', 'barbers', 5.0, 30, 'new', 0, false, 'cowork')
-- add more rows here
;
```

**Before inserting, always check for duplicates:**
```sql
SELECT email FROM outreach_leads
WHERE email IN (
  'hello@fallonandmannbarbers.co.uk',
  'thebarberclub.dan@gmail.com'
);
```
Only insert emails not already in that list.

**Email blocklist — never insert these domains:**
`sentry-next.wixpress.com`, `wixpress.com`, `webador.com`, `squarespace.com`,
`shopify.com`, `wix.com`, `godaddy.com`, `domain.com`, `example.com`,
`hubspot.com`, `noreply.com`

Also never insert: `user@domain.com`, any email with a hex string before `@`,
any email containing `&quot;` or HTML entities.

---

### Check send results and open rates

Use **Supabase MCP**:

```sql
-- Full send summary for today
SELECT
  ol.business_name,
  ol.niche,
  ol.location,
  os.variant,
  os.subject,
  os.error,
  os.opened_at,
  os.clicked_at,
  os.created_at
FROM outreach_sends os
JOIN outreach_leads ol ON ol.id = os.lead_id
WHERE os.created_at > NOW() - INTERVAL '24 hours'
ORDER BY os.created_at DESC;

-- Open rate summary
SELECT
  COUNT(*) as total_sent,
  COUNT(opened_at) as opened,
  COUNT(clicked_at) as clicked,
  ROUND(COUNT(opened_at)::numeric / COUNT(*) * 100, 1) as open_rate_pct
FROM outreach_sends
WHERE error IS NULL
  AND created_at > NOW() - INTERVAL '7 days';
```

---

### Mark a lead as converted

Use **Supabase MCP** when a prospect signs up:

```sql
UPDATE outreach_leads
SET
  converted_at = NOW(),
  status = 'converted',
  conversion_channel = 'cold_email'
WHERE email = 'hello@example.co.uk';
```

---

### Skip or unsubscribe a lead

Use **Supabase MCP**:

```sql
-- Skip (stop emails, keep in system)
UPDATE outreach_leads SET status = 'skip'
WHERE email = 'hello@example.co.uk';

-- Unsubscribe (they asked to be removed)
UPDATE outreach_leads
SET unsubscribed = true, status = 'skip'
WHERE email = 'hello@example.co.uk';
```

---

### Check pipeline health

Use **Supabase MCP** for a full snapshot:

```sql
SELECT
  status,
  niche,
  COUNT(*) as count
FROM outreach_leads
GROUP BY status, niche
ORDER BY niche, status;
```

---

### Check deployment status

Use **Vercel MCP** — `Vercel:list_deployments`:
- `projectId: prj_OunOw4tQpl0FdT6xneN5uFd6EsjS`
- `teamId: team_l8PGGaIxGVbyDHwXHc64AOnb`

Latest deployment should show `state: READY` and `target: production`.
If it shows `state: ERROR`, use `Vercel:get_deployment_build_logs` to debug.

---

### Check Vercel runtime logs (did emails actually send?)

Use **Vercel MCP** — `Vercel:get_runtime_logs`:
- `projectId: prj_OunOw4tQpl0FdT6xneN5uFd6EsjS`
- `teamId: team_l8PGGaIxGVbyDHwXHc64AOnb`
- `query: "Outreach"` — filters to outreach-related log lines
- `since: "1h"` — last hour

Look for lines like:
- `[Outreach] ✓ step1(a) → email@domain.co.uk` — successful send
- `[Outreach] ✗ email@domain.co.uk` — failed send
- `[Cron] Outreach followup complete` — drip cron ran successfully

---

### Update the skills files

Use **GitHub MCP** — `github:push_files` or `github:create_or_update_file`:
- Owner: `DevSim1`
- Repo: `claude-skills`
- Branch: `main`
- Always fetch the current SHA first with `github:get_file_contents` before updating

---

## Trigger a Send (still needs Chrome — calls Resend)

The send-batch endpoint is the one thing that must go through the browser,
because it calls Resend's API to actually deliver emails. Supabase MCP
can't do that.

### Step 1 — Check daily count first (Supabase MCP)
```sql
SELECT COUNT(*) as sent_today
FROM outreach_sends
WHERE created_at > NOW() - INTERVAL '24 hours'
  AND error IS NULL;
```

### Step 2 — Navigate to the app and trigger via browser console

Use **Claude in Chrome** to navigate to `https://app.thedigitalcove.co.uk`,
then run in the console:

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

Adjust `limit` so that `sent_today + limit <= 100`.

The batch sends at 1 email/second — 30 leads takes ~30 seconds.
Wait for the JSON response before reporting back.

### Step 3 — Verify with Supabase MCP
```sql
SELECT COUNT(*) as just_sent
FROM outreach_sends
WHERE created_at > NOW() - INTERVAL '5 minutes'
  AND error IS NULL;
```

---

## Cowork Workflow — Lead Scraping (£0, preferred method)

Everything here uses **Claude in Chrome** for the browser work,
then **Supabase MCP** to import. No API cost. Higher email quality.

### Before starting — ask Simone:
1. Niche: `florists` / `barbers` / `salons`
2. City: Glasgow / Edinburgh / Dundee / Aberdeen / Stirling / Perth / Paisley
3. How many leads? (default 15)

Then check daily send budget (Supabase MCP):
```sql
SELECT COUNT(*) as sent_today FROM outreach_sends
WHERE created_at > NOW() - INTERVAL '24 hours' AND error IS NULL;
```

### Step 1 — Open Google Maps (Chrome MCP)
Navigate to `https://maps.google.com`.
Search: `[niche] in [city]` — e.g. `barbers in Edinburgh`.

### Step 2 — Filter results (Chrome MCP)
For each listing in the panel:

**Skip immediately if:**
- Under 15 or over 150 reviews
- Under 3.8 stars
- Chain brand (TONI&GUY, Regis, Supercuts, Great Clips, etc.)
- Closed permanently or temporarily

**Click listing → check Posts/Updates tab:**
- Posted in last 14 days → SKIP (already active, not our target)
- Last post 30+ days ago, or no posts at all → GOOD TARGET ✓

**Click Website → open in new tab:**

### Step 3 — Find the email (Chrome MCP)
With real browser open on the business website:

1. Scan for mailto link or visible email on homepage
2. If none found, try: `/contact`, `/contact-us`, `/about`, `/about-us`
3. **Accept** if ends in a real business domain
4. **Reject** if:
   - Domain: `wix.com`, `squarespace.com`, `sentry.io`, `webador.com`, etc.
   - Is `user@domain.com` or any placeholder
   - Has a long hex string before the `@`
   - Contains `&quot;` or other HTML entities

**Tips:**
- Booking-system-only sites (Fresha, Treatwell, Booksy) → usually no email → skip
- Check Instagram bio or Facebook About if website has no email

### Step 4 — Build the leads list
Record each valid lead. Show Simone the table:

```
Business Name            | Email                              | Rating | Reviews | Last GBP Post
Fallon and Mann Barbers  | hello@fallonandmannbarbers.co.uk   | 5.0    | 60      | 45 days ago
The Barber Club          | thebarberclub.dan@gmail.com        | 5.0    | 30      | never
Half-Cut Barbershop      | booking@halfcutbarbershop.co.uk    | 4.9    | 121     | never
```

Ask: "Here are [N] leads for [niche] in [city]. Any to remove before I import?"

### Step 5 — Import via Supabase MCP (not the API)
After Simone confirms, insert directly:

```sql
INSERT INTO outreach_leads (
  business_name, email, location, niche,
  google_rating, google_review_count,
  status, sequence_step, unsubscribed, source
) VALUES
  ('Fallon and Mann Barbers', 'hello@fallonandmannbarbers.co.uk',
   'Edinburgh', 'barbers', 5.0, 60, 'new', 0, false, 'cowork'),
  ('The Barber Club', 'thebarberclub.dan@gmail.com',
   'Edinburgh', 'barbers', 5.0, 30, 'new', 0, false, 'cowork')
;
```

### Step 6 — Report back
```
Done! Imported [N] leads for [niche] in [city].
✅ Added: [N] new leads
⏭  Skipped: [N] duplicates
❌ Rejected: [N] — [reasons]

Daily budget: [sent_today]/100 used. [remaining] slots left today.
Shall I trigger a send now, or wait?
```

---

## API Endpoints (for reference — use MCPs above where possible)

These are still valid but most tasks are better handled via MCP directly.

| Endpoint | When to use |
|---|---|
| `POST /api/outreach/send-batch` | Trigger email sends — still needs Chrome |
| `POST /api/outreach/scrape-leads` | API-based scraping — costs ~£0.80/run, use only if asked |
| `POST /api/outreach/import-csv` | CSV import fallback — use Supabase MCP instead |
| `POST /api/outreach/convert` | Mark converted — use Supabase MCP instead |
| `GET /api/outreach/leads` | List leads — use Supabase MCP instead |

All endpoints: `Authorization: Bearer dc-outreach-2026`

---

## Email Templates

27 templates: 3 groups × 3 variants (A/B/C) × 3 steps.

**Groups:** `warm` (florists, barbers, salons), `trades`, `health`
**Variants** rotate A→B→C automatically across each batch.
**FROM:** `Chloe from Digital Cove <chloe@thedigitalcove.co.uk>`
**REPLY-TO:** `SalesSupport@thedigitalcove.co.uk` (M365 — all replies land here)

**Drip timing:**
- Step 1: immediately on import + manual send trigger
- Step 2: 5 days after step 1 (cron fires automatically at 9am daily)
- Step 3: 12 days after step 2 (cron fires automatically at 9am daily)

---

## Key Business Rules

| Rule | Detail |
|---|---|
| Daily send limit | **100 emails/day — hard cap, never exceed** |
| Target: reviews | 15–150 Google reviews |
| Target: rating | 3.8★ or above |
| Target: GBP activity | No posts in 30+ days, or never posted |
| Skip: chains | TONI&GUY, Regis, Supercuts, Great Clips, etc. |
| Active niches | florists, barbers, salons ONLY |
| Pricing | £50/month (after 15-day trial at £15) |
| Exclusivity | One business per postcode area per niche |

---

## Quick Reference — MCP to Use for Each Task

| What Simone says | MCP to use | Action |
|---|---|---|
| "scrape leads for barbers in Edinburgh" | Chrome MCP | Google Maps browser workflow |
| "import these leads" | Supabase MCP | INSERT into outreach_leads |
| "send today's emails" | Chrome MCP | /api/outreach/send-batch via console |
| "how many sent today" | Supabase MCP | COUNT from outreach_sends |
| "show open rates" | Supabase MCP | Query outreach_sends with opened_at |
| "check build status" | Vercel MCP | list_deployments |
| "why did the build fail" | Vercel MCP | get_deployment_build_logs |
| "check if emails sent" | Vercel MCP | get_runtime_logs query="Outreach" |
| "mark [business] converted" | Supabase MCP | UPDATE outreach_leads |
| "remove/skip a lead" | Supabase MCP | UPDATE status='skip' |
| "unsubscribe a lead" | Supabase MCP | UPDATE unsubscribed=true |
| "update the skill file" | GitHub MCP | push_files to claude-skills/main |
| "fix a bug in the code" | GitHub MCP | push_files to Digitalcove-ops |
| "check Stripe subscriptions" | Stripe MCP | list_subscriptions |
| "pipeline overview" | Supabase MCP | GROUP BY status, niche |

---

## Known Issues & Fixes

| Issue | Status |
|---|---|
| Wix/Sentry emails slipping through scraper | Fixed — blocklist in scraper + import |
| firstName showing "Salon" or "blunted" | Fixed — smart word skipping in names.ts |
| Landing URL visible as tracking URL | Fixed — clean URL in body, tracking in href |
| City field showing postcode | Fixed — extractCity strips postcode |
| US leads appearing | Fixed — skip non-UK postcodes |
| Vercel 60s timeout on large batches | Known — keep Chrome tab open during send |

---

## What's Coming Next

- **GBP API quota** — case `2-9265000041135` (submitted ~Apr 7, 7–10 days). Once approved, GBP posting activity checking becomes automated.
- **Rank Scout** — next product after GBP quota unblocked. Local businesses get a free Google Maps visibility report, converting to subscribers.
- **More landing pages** — dog groomers, electricians, dentists etc.
- **Self-serve OAuth** — customers connect their own Google account. Currently Simone connects manually via admin dashboard.
