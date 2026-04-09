---
name: dco-lead-scraper
description: >
  Scrapes Google Maps for local business leads for Digital Cove outreach — zero API cost.
  Searches a niche + city, filters by review count and rating, visits each business website
  in the real browser to find contact emails, checks GBP posting activity, then saves a
  reviewed CSV and imports it to the DCO backend.
  Trigger when Simone says things like: "scrape leads for [niche] in [city]",
  "find me barbers in Edinburgh", "get salon leads", "cowork scrape", "find leads",
  "run the lead scraper", or any request to find new outreach prospects.
---

# DCO Lead Scraper — Cowork Shortcut

Finds quality local business leads for Digital Cove cold outreach using Google Maps
directly in the browser. No Places API cost. Real browser = real emails, no Wix/Sentry junk.

## Inputs

Ask Simone before starting:

1. **Niche** — one of: `florists` `barbers` `salons` (these have confirmed landing pages)
2. **City** — e.g. `Glasgow`, `Edinburgh`, `Aberdeen`, `Dundee`, `Stirling`, `Perth`
3. **How many leads?** — default 15 (stays well within 100/day Resend limit)

If Simone says something like "barbers Edinburgh" — no need to ask, just start.

## Target Profile (filter criteria)

Only collect businesses that match ALL of these:
- ⭐ Rating: **3.8 stars or above**
- 📊 Reviews: **between 15 and 150**
- 📅 GBP activity: **not posting regularly** (ideally no posts in 30+ days, or no posts at all)
- 🚫 Skip chains: TONI&GUY, Supercuts, Regis, Great Clips, etc.
- 🚫 Skip businesses with 0 reviews or closed status

## Step 1 — Open Google Maps and search

1. Navigate to `https://maps.google.com`
2. Search for: `[niche] in [city]` — e.g. `barbers in Edinburgh`
3. Look at the results panel on the left

## Step 2 — Filter and collect candidates

For each result in the list:

1. Read the **star rating** and **review count** from the listing
2. **Skip immediately** if:
   - Fewer than 15 reviews OR more than 150 reviews
   - Less than 3.8 stars
   - Name looks like a chain (TONI&GUY, Regis, etc.)
   - Marked as "Permanently closed" or "Temporarily closed"
3. **Click the listing** to open the detail panel
4. Check **"Updates" or "Posts"** tab if visible — note when they last posted
   - If they posted in the last 14 days → skip (they're already active, not our target)
   - If no posts visible, or last post 30+ days ago → good target
5. Note down: business name, star rating, review count, days since last post (or "no posts found")
6. Click **"Website"** link if available — open in a new tab

## Step 3 — Find the email (real browser, free)

With the business website open:

1. **Scan the page** for a mailto link or visible email address
2. If not on homepage → try these paths in order:
   - `/contact`
   - `/contact-us`
   - `/about`
   - `/about-us`
3. Look for an email that:
   - Ends in a real domain (not `@wix.com`, `@squarespace.com`, `@sentry.io`, etc.)
   - Is NOT `user@domain.com`, `info@example.com`, or any obvious placeholder
   - Is NOT a Sentry/monitoring address (long hex string before `@`)
4. If a **real business email is found** → add to the leads table
5. If no email found after checking contact page → mark as "no email" and skip

## Step 4 — Build the leads table

As you collect each valid lead, add a row to a running table:

```
business_name | email | location | niche | google_rating | google_review_count | notes
```

Example:
```
Fallon and Mann Barbers | hello@fallonandmannbarbers.co.uk | Edinburgh | barbers | 5.0 | 60 | last post: 45 days ago
The Barber Club         | thebarberclub.dan@gmail.com      | Edinburgh | barbers | 5.0 | 30 | no GBP posts found
Half-Cut Barbershop     | booking@halfcutbarbershop.co.uk  | Edinburgh | barbers | 4.9 | 121| last post: never
```

Keep going until you have the requested number of leads (default 15) or you've
checked all results on the first page of Maps.

## Step 5 — Show Simone the table for review

Display the complete table clearly. For each lead show:
- Business name, email, rating, reviews, last GBP post date
- Any notes (e.g. "personal hotmail — might be owner-direct")

Ask:
> "Here are [N] leads for [niche] in [city]. Any you want to remove before I import?"

Wait for Simone's response. She may:
- Say "looks good, import them all"
- Say "remove [business name]"
- Ask to check a specific business more carefully

Apply any removals, then proceed.

## Step 6 — Save as CSV

Save the approved leads to a CSV file on the desktop:

**Filename:** `dco-leads-[niche]-[city]-[YYYY-MM-DD].csv`

**Format** (exact column names, comma-separated):
```csv
business_name,email,location,niche,google_rating,google_review_count,notes
Fallon and Mann Barbers,hello@fallonandmannbarbers.co.uk,Edinburgh,barbers,5.0,60,last post 45 days ago
The Barber Club,thebarberclub.dan@gmail.com,Edinburgh,barbers,5.0,30,no GBP posts found
```

Open the file briefly so Simone can see it saved correctly.

## Step 7 — Import to DCO backend

POST the CSV to the import endpoint:

```
POST https://app.thedigitalcove.co.uk/api/outreach/import-csv
Authorization: Bearer dc-outreach-2026
Content-Type: text/csv
[body: the CSV content]
```

Do this via the browser console on `app.thedigitalcove.co.uk`:

```javascript
const csv = `[paste full CSV content here]`;
const resp = await fetch('/api/outreach/import-csv', {
  method: 'POST',
  headers: {
    'Content-Type': 'text/csv',
    'Authorization': 'Bearer dc-outreach-2026'
  },
  body: csv
});
const result = await resp.json();
console.log(result);
```

Read the response and report back:
- `imported` — how many were added
- `skipped_duplicates` — already in the system
- `invalid` — anything rejected and why

## Step 8 — Report back

Give Simone a clean summary:

```
Done! Imported [N] leads for [niche] in [city].

✅ Added: [N] new leads
⏭️  Skipped: [N] duplicates (already in system)
❌ Rejected: [N] rows (if any, explain why)

Ready to send — you have [X] daily sends remaining today.
```

## Email quality rules (enforce strictly)

NEVER import an email that:
- Contains `wix`, `sentry`, `squarespace`, `webador`, `godaddy`, `shopify`, `hubspot` in the domain
- Is `user@domain.com`, `info@example.com`, `test@test.com` or any placeholder
- Has a long hex string before the `@` (Sentry monitoring address)
- Has an invalid TLD (less than 2 characters after the final `.`)
- Contains `&quot;`, `\"`, or HTML entities

If in doubt, skip the lead. A missed lead costs nothing. A bad email wastes our daily send limit.

## Tips for finding emails faster

- **Instagram link on website?** Check their Instagram bio — often has an email
- **Booking system only?** (Fresha, Treatwell, Booksy) — no email usually. Skip.
- **Facebook page?** Facebook About section often has email even when website doesn't
- **Google Maps "Message" button only?** No website = skip unless email found elsewhere
- If website redirects to booking platform with no contact page → skip

## Valid niches and their landing page URLs

| Niche | Landing page | Template group |
|---|---|---|
| `florists` | `thedigitalcove.co.uk/florists` | warm |
| `barbers` | `thedigitalcove.co.uk/barbers` | warm |
| `salons` | `thedigitalcove.co.uk/salons` | warm |

Only use these three niches until more landing pages are built.

## API reference

**Import endpoint:**
```
POST https://app.thedigitalcove.co.uk/api/outreach/import-csv
Authorization: Bearer dc-outreach-2026
```

**Check existing leads:**
```
https://app.thedigitalcove.co.uk/admin/outreach
```

**Trigger send after import:**
```javascript
fetch('/api/outreach/send-batch', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': 'Bearer dc-outreach-2026'
  },
  body: JSON.stringify({ step: 1, limit: 50 })
}).then(r => r.json()).then(console.log)
```
