# FanRink Hockey Domain Skill — v5.1 MERGED (30 Apr 2026)

> **Read this before any prompt involving**: leagues, teams, game data, scores, schedules, seasons, national teams, player identities, **rec leagues, beer leagues, college hockey, floating teams**, or any team without a league affiliation.

---

## Core Architecture

**One auth user → one profile → many identities** (fan, creator, player, team, league, brand, standard). Users switch identities rather than holding separate profiles.

**Community types:**
- **Fanzones** — platform-created, one per team, P0 for beta. Path: /fanzone/[slug]
- **Locker Rooms** — user-created, P1/P2

**Do NOT conflate Fanzones with Team Profile pages.** They are separate surfaces.

**Teams have ONE primary_league_id** but compete in multiple competitions.

**CRITICAL NEW RULE**: `primary_league_id` **CAN BE NULL** for national teams, floating teams, and some rec teams.

---

## Key Identifiers

- **Supabase project**: atveiabuhjtmbxestdzu
- **Vercel project**: prj_alpfjSXXUbD0ULLm5oY1edo7eXEu
- **GitHub repo**: DevSim1/fanrink-next
- **Admin account**: supportteam@fanrink.com (UUID: 3ed1047f-2b84-4b48-9f72-c1310c7eedd4)
- **Beta access code**: BETAFAN (9,999 max uses)
- **TheSportsDB premium key**: 485341 (images from r2.thesportsdb.com)
- **API-Sports Hockey key**: f99dd2df2f359d938898b39b52c9b5fc (100 req/day)
- **Email provider**: Resend (NOT Sendgrid)
- **Admin email**: Microsoft/Outlook (NOT Gmail)

---

## Data Sources by League

| League | Source | Cost | Notes |
|--------|--------|------|-------|
| NHL | api-web.nhle.com | Free | Official, no key needed |
| AIHL | theaihl.com/esportsdesk | Free | Stores aihl_game_id in meta |
| EIHL | TheSportsDB via cron-refresh | Premium key | |
| PWHL | sync-pwhl-scores | Unknown | Audit needed |
| SHL | TheSportsDB | Premium key | ID: 4384 |
| DEL | TheSportsDB | Premium key | ID: 4386 |
| Liiga | TheSportsDB | Premium key | ID: 4387 |
| KHL | TheSportsDB | Premium key | ID: 4920. Legal: do NOT crawl without consent |
| AHL | NOT YET INGESTED | TBD | Needs new sync-ahl-events function |

**Free NHL API (no key)**:
- GET https://api-web.nhle.com/v1/schedule/{YYYY-MM-DD}
- GET https://api-web.nhle.com/v1/gamecenter/{gameId}/landing
- GET https://api-web.nhle.com/v1/standings/now

---

## DB Critical Rules

- **NEVER** use FROM (VALUES ...) JOIN teams UPDATE pattern — catastrophic cross-join overwrite risk. Always use individual `UPDATE teams SET description = '...' WHERE name = 'X'` statements.
- CREATE POLICY IF NOT EXISTS is invalid — wrap in DO block with EXCEPTION WHEN duplicate_object.
- supabase:apply_migration for schema changes (tracked). execute_sql for read-only only.
- `leagues` table uses `abbreviation` NOT `short_name`.
- Team crests use `strBadge` NOT `strLogo`. Images from `r2.thesportsdb.com`.
- `sync_runs` is the canonical cron logging table.
- All commits that don't need a Vercel build must include `[skip ci]`.
- `posts.author_id` used before migration 20260125000007; `user_id` only from that version onward.
- `is_placeholder = true` for ghost teams (NOT is_hidden).
- privacy_level, dm_privacy, follow_approval columns DO NOT YET EXIST on profiles (27 Apr 2026). Add via migration before using.

---

## DB Schema — New Tables (added 26 Apr 2026)

- **team_memberships**: team_id, user_id, role (owner/admin/player/fan enum), status (active/pending/rejected enum)
- **team_invites**: team_id, invite_type (enum), invite_code, invited_email, is_placeholder
- **job_runs**: job_name, status (enum), started_at, finished_at
- **post_translations**: post_id, language, translated_content (AI cache)
- **promoted_posts**: post_id, budget_pence, spend_pence, daily_cap_pence, cpm_pence, targeting fields

PostgreSQL ENUMs: membership_role, membership_status, claim_status, invite_type, job_status

Private schema: private.is_team_role() security definer helper

---

## Season Windows 2025-26

| League | Status |
|--------|--------|
| NHL | Playoffs ACTIVE (Apr 2026) |
| AIHL | LIVE Apr–Aug 2026 (10 teams) |
| EIHL | ENDED 19 Apr 2026 |
| PWHL | ENDED 26 Apr 2026. Playoffs not in DB. |
| SHL/DEL/Liiga/KHL | 2024-25 data only. 2025-26 missing. |
| AHL | Not ingested. |

---

## Cron Jobs

All live-sync crons gate on `has_active_games(league_abbr)` function.

- **sync-nhl-scores** (live): every 2min during game windows
- **sync-pwhl-scores** (live): every 5min during game windows
- **sync-aihl-scores** (live): every 2min weekends
- **cron-refresh** (EIHL live): every 2min 14:00-21:00 UTC
- **cleanup-job-tables**: 03:00 UTC daily

---

## Security Architecture

- **private** schema for all security definer functions (NOT exposed schema)
- `private.is_team_role()` wraps auth.uid() in SELECT for RLS
- Views bypass RLS without `security_invoker = true`
- Claim approval = atomic SQL transaction
- Notification dedup keys: `reply:{commentId}:{userId}` pattern

---

## Vercel / GitHub Patterns

- Poll `Vercel:get_deployment` with specific deployment ID (not list_deployments) for build tracking
- Builds take 3–5 minutes
- GitHub MCP auth expires in long sessions. If it fails, use Chrome extension to push via GitHub web editor.
- PAT for Claude Code on MacBook valid until July 2026 (regenerated 27 Apr 2026)
- Preview branches replay all migrations from scratch — fix source migration files directly

---

## Working Model

- Solo non-technical founder (Simone)
- One prompt → one PR → green → merge → next. Never stack.
- Claude drives decisions and prepares next steps proactively
- QA by Cowork (mobile browser testing)
- DB fixes via MCP directly (no PR, preserves GitHub Actions)
- Audit always precedes implementation
- **CLAUDE.md** in fanrink-next root is auto-read by Claude Code
- **docs/ai/NEXT_PROMPT.md** is the single source of truth for work queue

---

# Hockey Domain Knowledge

## Purpose

This is the canonical hockey knowledge reference for Claude when working on FanRink. It contains verified facts about leagues, competitions, structures, rankings, team data, and **special team types** (rec leagues, beer leagues, college, national teams, floating teams).

Always consult this before writing team/league descriptions, seeding data, or making assumptions about competition structures or team types.

---

## The Multi-Competition Reality

**Critical concept**: A hockey team's `primary_league_id` is their *home* league — but teams compete in MULTIPLE competitions simultaneously. Never assume a team only plays in one competition.

### Examples by league type:

**EIHL (UK)** — every team plays 3 competitions per season:
- EIHL League Championship (regular season)
- EIHL Challenge Cup (cup competition, starts September)
- EIHL Playoffs (April, weekend tournament at Motorpoint Arena Nottingham)
- EIHL regular season champion also qualifies for the **Champions Hockey League**

**NHL** — teams play:
- NHL Regular Season (82 games, October–April)
- NHL Playoffs (16 teams, April–June, Stanley Cup)
- Some players/teams participate in: 4 Nations Face-Off, NHL All-Star Game, Winter Classic

**SHL / DEL / Liiga / Swiss NL / Czech Extraliga** — top clubs also enter:
- **Champions Hockey League (CHL)** — 24 clubs from Europe's top leagues, Aug–Feb
- Domestic cup competitions (varies by country)
- Some leagues have promotion/relegation playoffs

**KHL** — 22 clubs across Russia, Belarus, China, Kazakhstan:
- Regular season (68 games, Sep–Mar)
- Gagarin Cup Playoffs (Mar–May)
- No promotion/relegation (closed league)

**National teams** — players compete for clubs AND country:
- IIHF World Championship (men's: May; women's: April)
- Winter Olympics (every 4 years, February)
- World Junior Championship (U20, December–January)
- 4 Nations Face-Off (NHL players, February)

---

## FanRink Database Architecture for Competitions

```
teams.primary_league_id → home/primary league (single FK, CAN BE NULL)
team_competitions → many-to-many: team ↔ competition (empty, needs seeding)
team_league_memberships → many-to-many: team ↔ league (empty, needs seeding)
competitions → specific competition instances (3 rows: EIHL League, Challenge Cup, Playoffs)
leagues → 96 leagues total
```

**Rule**: When showing a team's competitions to a user, always think beyond `primary_league_id`. The `team_competitions` table is where multi-competition memberships live once seeded.

**CRITICAL**: `primary_league_id` **CAN BE NULL** for:
- **National teams** (no home league)
- **Floating teams** (tournament-only, exhibition)
- **Some rec teams** (community-organized, no formal league)

---

## Team Type Taxonomy

**CRITICAL**: Different team types require different data models and UX patterns. **Not every team has a `primary_league_id`.**

### Professional Teams
- **Governance**: Formal league authority
- **Characteristics**: Player contracts, salaries, official stats
- **Examples**: NHL, EIHL, SHL, Liiga, DEL, Swiss NL
- **Data Quality**: High - official APIs/feeds available
- **Verification**: Not required (leagues verify themselves)
- **primary_league_id**: REQUIRED (cannot be NULL)

### Semi-Professional Teams
- **Governance**: Regional league governance
- **Characteristics**: Mix of paid/amateur players
- **Examples**: NIHL, lower-tier European leagues
- **Data Quality**: Medium - some official data, some self-reported
- **Verification**: League-level verification
- **primary_league_id**: REQUIRED (cannot be NULL)

### National Teams
- **Governance**: IIHF / national federations
- **Characteristics**: 
  - `primary_league_id` = **NULL** (no home league)
  - Only compete in tournaments
  - Players represent country, not club
- **Examples**: Team USA, Team Canada, Team Finland
- **Data Quality**: High - official IIHF data
- **Critical Rule**: National team identity is SEPARATE from club identity
- **primary_league_id**: **NULL** (no home league)

### University/College Teams

**US Structure:**
- NCAA Division I (Atlantic Hockey, Big Ten, ECAC, Hockey East, NCHC, WCHA)
- NCAA Division III (multiple conferences)
- ACHA D1/D2/D3 (club hockey)

**UK Structure:**
- BUIHA (British Universities Ice Hockey Association)
- University club teams (may also compete in NIHL)

**Key Rule**: Universities can field MULTIPLE teams in DIFFERENT structures
- Example: Edinburgh University could have BOTH a BUIHA team AND a separate NIHL club team
- These are DIFFERENT teams with DIFFERENT rosters, not the same team in two leagues

**Data Model Impact:**
- Need to distinguish varsity vs club
- Allow same institution to have multiple team entities
- Don't assume "Edinburgh University" means one team
- **primary_league_id**: REQUIRED (NCAA league, BUIHA league, or NIHL league)

### Rec / Beer League Teams

**Definition**: Community-organized, self-governed hockey teams

**Characteristics:**
- Self-reported scores (need verification flow)
- Fluid rosters (players can move mid-season, no registration deadlines)
- Flexible seasons (no fixed calendar, can start/end anytime)
- No official stats (community-maintained, optional)
- Trust-based verification (not official)
- Variable skill levels (A/B/C/D divisions common)

**Examples:**
- England Ice Hockey "Rec registered teams"
- Adult hobby leagues
- Weekend beer leagues
- Corporate leagues

**Data Model Differences:**
```sql
-- Rec teams may have:
primary_league_id: NULL or "community_league_id"
scores: self_reported: true, verification_required: true
roster_deadline: NULL (no deadline)
official_stats: false
league_governance: "community_organized"
```

**UX Implications:**
- Fixture creation: Either team can add, need confirmation from opponent
- Score submission: Need verification flow (opponent confirms)
- Stats: Mark as "unofficial" or "community stats"
- Roster: Allow mid-season changes, no transfer windows

**Rec League Progression Path:**
Some rec teams "graduate" to formal league status:
- Start as community-organized
- Gain formal league membership
- `primary_league_id` changes from NULL → league_id
- Governance type changes from "community_organized" → "league_organized"

**primary_league_id**: **NULL OR community league ID** (optional)

### Floating Teams

**Definition**: Teams that exist without permanent league membership

**Types:**

1. **Exhibition Opponents**
   - One-off games
   - No league affiliation
   - Example: NHL team plays exhibition vs college team

2. **Tournament Guests**
   - Invited to specific events
   - No regular season
   - Example: Team invited to Christmas tournament

3. **Travelling Select Teams**
   - Represent region, not league
   - Play multiple opponents
   - Example: Scottish Select team

4. **Ghost Teams** (FanRink-specific)
   - Opponent placeholders before identity confirmed
   - Created during fixture invite flow
   - Mapped to real team when opponent accepts
   - `is_placeholder: true` in database

**Rules:**
- Can have `primary_league_id = NULL`
- Can participate in competitions without league membership
- May have temporary status that converts to permanent
- Need to track invitation/acceptance state

**Data Model:**
```sql
team_type: 'floating'
primary_league_id: NULL (allowed)
is_placeholder: boolean
invitation_status: enum('pending', 'accepted', 'confirmed')
```

**primary_league_id**: **NULL** (no permanent league)

### Junior Teams

**Categories:**
- Major Junior: OHL, WHL, QMJHL (age 16-20, CHL)
- Tier II Junior: NAHL, BCHL
- Tier III Junior: various regional leagues
- Age-specific: U20, U19, U16, U14, U12, U10

**Special Rules:**
- Players often move between teams mid-season (trades, loans)
- Draft systems (OHL Priority Selection)
- Overage player limits
- NCAA eligibility considerations

**primary_league_id**: REQUIRED (junior league)

---

## League Authority Models

Different leagues have different governance structures that affect data modeling:

### Federation-Governed
- **Authority**: National federation (e.g., IIHF, national hockey federations)
- **Examples**: National teams, IIHF tournaments
- **Data Source**: Central federation APIs
- **Verification**: Federation-level

### League-Organized
- **Authority**: Centralized league office
- **Examples**: NHL, EIHL, SHL
- **Data Source**: League official APIs/sites
- **Verification**: League-level

### Association-Registered
- **Authority**: Registration body, but teams self-govern
- **Examples**: England Ice Hockey Rec teams
- **Characteristics**: 
  - Teams register but manage themselves
  - No central fixtures/results
  - Self-reported data
- **Data Source**: Team self-reporting
- **Verification**: Peer verification (opponent confirms)

### Community-Organized
- **Authority**: None (informal)
- **Examples**: Beer leagues, pickup games
- **Data Source**: Participant self-reporting
- **Verification**: Trust-based

### Unaffiliated
- **Authority**: None
- **Examples**: Floating teams, exhibition-only
- **Data Source**: Event organizers
- **Verification**: Event-specific

---

## Global League Calendar (Monthly Operating Guide)

This tells you **what should be "alive" in FanRink each month**:

### January - Peak Multi-League Month
**Active**: NHL, AHL, EIHL, SHL, Liiga, DEL, National League, ICEHL, Asia League, PWHL, SDHL, Auroraliiga, DFEL, Swiss Women's League, WNIHL, AWIHL, OHL/WHL/QMJHL/USHL, UK junior leagues  
**Operational Meaning**: Most score/standings/community traffic lives here

### February - Olympic Break Risk
**Active**: Same as January but compressed around Olympic windows in 2026  
**Operational Meaning**: Needs stronger schedule change detection, postponement handling

### March - Playoff Transitions
**Active**: European regular seasons finish, playoffs begin; CHL junior regular seasons end; DFEL playoffs; SDHL/Auroraliiga/Swiss women's title windows; NCAA women's Frozen Four; WNIHL run-in  
**Operational Meaning**: Playoff brackets, format changes, qualification rules matter more than raw schedule ingest

### April - North Playoffs + South Season Start
**Active**: NHL/AHL/EIHL/DEL/SHL/CHL playoffs; PWHL regular season ends (25 Apr) → playoffs start (30 Apr); AIHL starts mid-month; WNIHL still active; NCAA men's Frozen Four  
**Operational Meaning**: Best engagement month - playoff threads in north, fresh regular-season in Australia

### May - Bridge Month
**Active**: Northern leagues wind down; WNIHL finals (late May); NZIHL opens (30 May); UK junior nationals  
**Operational Meaning**: Fewer central scoreboards, more event coverage and community stories

### June - Southern Hemisphere Lead
**Active**: AIHL/NZIHL; AWIHL off-season; light women's club hockey globally  
**Operational Meaning**: Lean into AIHL/NZIHL, transfers, pre-season planning, fan content

### July - Off-Season Build
**Active**: AIHL/NZIHL live; northern leagues publish schedules  
**Operational Meaning**: Source discovery, parser testing, season-bootstrap jobs

### August - Switch-Over Month
**Active**: AIHL regular season (through 30 Aug); NZIHL (into mid-Aug); CHL club competition starts (28 Aug); German DNL starts (end Aug); UK junior/women's calendars lock  
**Operational Meaning**: Southern live coverage overlaps with northern pre-season setup

### September - Hardest Schedule Integrity Month
**Active**: NHL camps/pre-season; AHL pre-season; European men's leagues begin; EIHL/NIHL begin; Auroraliiga, WNIHL, DFEL start; SDHL already in-season; OHL/WHL/QMJHL/USHL/DEB juniors all start  
**Operational Meaning**: Most leagues boot at once - highest schedule ingestion complexity

### October - Global Feed Density Begins
**Active**: NHL opens (7 Oct); AHL opens (10 Oct); European leagues in normal cadence; women's European leagues continue; WNIHL continues; NCAA men's/women's live  
**Operational Meaning**: First month where "global open-ice feed" feels truly dense

### November - Women's Coverage Improves
**Active**: Northern pro schedules in full flow; PWHL begins (21 Nov); AWIHL begins (early Nov); juniors in full swing  
**Operational Meaning**: Women's coverage breadth improves (PWHL + AWIHL online)

### December - High-Volume Month
**Active**: Almost every northern club league; PWHL neutral-site window (Dec-Apr); PWHL/WNIHL/AWIHL/European women's leagues; juniors packed before holiday breaks  
**Operational Meaning**: High-volume but manageable if feed-based

---

## UK Hockey Structure (Critical for Beta Launch)

### Hierarchy
```
Ice Hockey UK (governing body)
├── EIHL (top pro tier, 10 teams)
├── England Ice Hockey
│   ├── NIHL National (10 teams, second tier below EIHL)
│   ├── NIHL 1 North/South
│   ├── NIHL 2 North/South
│   ├── WNIHL (senior + U16)
│   ├── England Juniors (U19, U16, U14, U12, U10)
│   └── England Rec (registered teams)
└── Scottish Ice Hockey
    ├── SNL (Scottish National League - Scotland's premier)
    ├── Scottish Junior leagues
    └── Scottish Rec

Independent:
├── BUIHA (British Universities)
└── BIPHA (British Inline Puck Hockey)
```

### UK Season Dates (2025-26)
**NIHL 1/2**: 6 Sep 2025 – 22 Mar 2026 (regular) + 28 Mar – 19 Apr 2026 (playoffs)  
**WNIHL**: 6 Sep 2025 – 10 May 2026 (regular) + championship weekends late May  
**England Juniors**: U19/U16 to 17 May; U14/U12 to 10 May; U10 to 31 May  
**EIHL**: Oct–Apr (10 teams: Belfast Giants, Cardiff Devils, Coventry Blaze, Dundee Stars, Fife Flyers, Glasgow Clan, Guildford Flames, Manchester Storm, Nottingham Panthers, Sheffield Steelers)

### Scottish Clubs Referenced in Memories
Aberdeen Lynx, Dundee Rockets, Edinburgh Capitals, Kirkcaldy Kestrels, Kilmarnock Thunder, North Ayrshire Wild, Paisley Pirates, Solway Sharks, Whitley Warriors

**CRITICAL**: EIHL regular season champion qualifies for CHL (Champions Hockey League) - this is an example of multi-competition participation

---

## Women's Hockey Priority Stack

### Strategic Importance
1. Growing faster than many legacy men's leagues
2. Social/community layer matters disproportionately
3. Data landscape still open enough to be differentiated

### Recommended First Wave
**PWHL + WNIHL + AWIHL** because:
- PWHL: Premium North American competition
- WNIHL: Very relevant UK grassroots/senior (Sep-May, fits beta)
- AWIHL: Southern-hemisphere fills calendar gap (Nov-Feb)

### Next Priority Additions
**SDHL + DFEL** because:
- Centralised official hubs
- Predictable season patterns

### Full Women's League Reference

| League | Geography | Season | Late-Apr 2026 Status |
|--------|-----------|--------|----------------------|
| PWHL | US/Canada | Nov-Apr + playoffs | Playoffs start 30 Apr 2026 |
| SDHL | Sweden | Sep-Mar/Apr | Playoff/finals end-state |
| Auroraliiga | Finland | Sep-Feb + playoffs | Season concluded |
| Swiss Women's League | Switzerland | Autumn-late winter + playoffs | Championship decided Mar 2026 |
| DFEL | Germany | Sep-Feb + playoffs to early Apr | Finals 21 Mar, champion by 4 Apr 2026 |
| WNIHL | UK | Sep-May | Finals weekend 23-25 May 2026 |
| AWIHL | Australia | Nov-Feb + playoffs early Mar | 2025-26 finished 1 Mar 2026 |
| EWHL/AWHL | Central Europe | Autumn-March | Champions decided |
| Czech Extraliga žen | Czech Republic | Autumn-winter | Standings available |
| NCAA women | US college | Autumn-March | Frozen Four 20-22 Mar 2026 |

---

## Database Schema Requirements

**Team Type Column:**
```sql
CREATE TYPE team_type AS ENUM (
  'professional',      -- NHL, EIHL, SHL, etc.
  'semi_professional', -- NIHL, lower-tier European
  'national',          -- Country teams (Olympics, World Championships)
  'university',        -- NCAA, BUIHA
  'junior',            -- OHL, WHL, QMJHL, age-group
  'recreational',      -- Beer leagues, pickup-style
  'floating'           -- Tournament-only, exhibition, unattached
);

ALTER TABLE teams ADD COLUMN team_type team_type;
```

**League Governance Column:**
```sql
CREATE TYPE league_governance AS ENUM (
  'federation_governed',   -- IIHF, national federations
  'league_organized',      -- NHL, EIHL have central authority
  'association_registered', -- England Ice Hockey Rec - teams register but self-govern
  'community_organized',   -- Beer leagues, no formal authority
  'unaffiliated'           -- Floating teams, exhibition only
);

ALTER TABLE leagues ADD COLUMN governance_type league_governance;
```

**Games Verification Columns:**
```sql
ALTER TABLE games ADD COLUMN is_self_reported boolean DEFAULT false;
ALTER TABLE games ADD COLUMN verification_status varchar;
```

---

## Database Migration Checklist

When adding team type support, run these migrations in order:

**1. Add ENUMs:**
```sql
CREATE TYPE team_type AS ENUM (
  'professional', 'semi_professional', 'national', 
  'university', 'junior', 'recreational', 'floating'
);

CREATE TYPE league_governance AS ENUM (
  'federation_governed', 'league_organized', 
  'association_registered', 'community_organized', 'unaffiliated'
);
```

**2. Add columns:**
```sql
ALTER TABLE teams ADD COLUMN team_type team_type;
ALTER TABLE leagues ADD COLUMN governance_type league_governance;
ALTER TABLE games ADD COLUMN is_self_reported boolean DEFAULT false;
ALTER TABLE games ADD COLUMN verification_status varchar;
```

**3. Set defaults for existing data:**
```sql
-- Professional teams (have official leagues)
UPDATE teams SET team_type = 'professional' 
WHERE primary_league_id IN (
  SELECT id FROM leagues WHERE abbreviation IN 
  ('NHL', 'EIHL', 'SHL', 'DEL', 'Liiga', 'Swiss NL', 'Czech ELH', 'KHL')
);

-- National teams (no primary_league_id, country names)
UPDATE teams SET team_type = 'national'
WHERE name LIKE 'Team %' OR name IN ('USA', 'Canada', 'Finland');

-- Junior teams
UPDATE teams SET team_type = 'junior'
WHERE primary_league_id IN (
  SELECT id FROM leagues WHERE abbreviation IN ('OHL', 'WHL', 'QMJHL', 'USHL')
);

-- Rest default to semi_professional for now
UPDATE teams SET team_type = 'semi_professional' WHERE team_type IS NULL;
```

**4. Set league governance:**
```sql
UPDATE leagues SET governance_type = 'league_organized'
WHERE abbreviation IN ('NHL', 'EIHL', 'SHL', 'DEL', 'Liiga');

UPDATE leagues SET governance_type = 'federation_governed'
WHERE name LIKE '%IIHF%' OR name LIKE '%National%';

UPDATE leagues SET governance_type = 'association_registered'
WHERE name LIKE '%Rec%' OR name LIKE '%Beer%';
```

---

## IIHF World Rankings (May 2025)

### Men's Rankings
1. USA (3,985)
2. Switzerland (3,975)
3. Canada (3,935)
4. Sweden (3,915)
5. Czechia (3,860)
6. Finland (3,780)
7. Germany (3,710)
17. United Kingdom (3,100)

**Note**: Russia/Belarus not participating

### Women's Rankings
1. USA (4,150)
2. Canada (4,140)
3. Finland (3,930)
4. Czechia (3,920)
5. Switzerland (3,855)
6. Sweden (3,730)

### Recent Tournament Results

**2026 Olympics Men**: USA Gold, Canada Silver, Finland Bronze  
**2026 Olympics Women**: USA Gold, Canada Silver, Switzerland Bronze  
**2025 WC Men**: USA 1st, Switzerland 2nd, Sweden 3rd  
**2025 WC Women**: USA 1st, Canada 2nd

---

## Top League Structures (Summary)

### NHL (North America)
32 teams (25 USA, 7 Canada), 82-game season, 16-team playoffs, closed league

### KHL (Russia/Eurasia)
22 clubs across Russia, Belarus, China, Kazakhstan, 68-game season, Gagarin Cup

### SHL (Sweden)
14 clubs, promotion/relegation, CHL qualification for top clubs

### Liiga (Finland)
15 clubs, one of Europe's strongest leagues, CHL qualification

### DEL (Germany)
15 clubs, closed league, CHL qualification for champion

### Swiss NL
14 clubs, 52-game season, promotion/relegation with Swiss League

### Czech Extraliga
14 clubs, one of Europe's strongest leagues

### EIHL (UK)
10 teams, 3 competitions per season (League, Cup, Playoffs), CHL qualification

### NIHL National (England)
10 teams, second tier below EIHL

### PWHL (Women's)
8 teams across North America, top level women's professional

### AIHL (Australia)
10 teams, Apr-Sep season (Southern Hemisphere), Goodall Cup

---

## FanRink Data State (as of Apr 2026)

**Teams:**
- Total in-scope: 468
- All 468 have `primary_league_id` ✅ **(NOTE: This will change - some teams can have NULL)**
- Have descriptions: ~55 (NHL ×18, KHL ×16, SHL ×8, PWHL ×8, national teams ×5)
- Have logo_url: 434
- Have arena_name: 297
- Have founded_year: 398
- **Have team_type**: YES ✅ (column exists in DB)

**Leagues:**
- Total: 96
- All 96 have descriptions ✅
- **Have governance_type**: NO ❌ (needs migration)

**Games:**
- NHL: 1,157 games
- EIHL: 856 games
- **Have is_self_reported**: NO ❌ (needs migration)
- **Have verification_status**: NO ❌ (needs migration)

---

## Enrichment Pipeline Notes

### Wikipedia Enrichment Rules

**Always reject a page if it contains:**
- "is a professional ice hockey league"
- "The Kontinental Hockey League (KHL"
- "The National Hockey League (NHL"
- "organized by the International Ice Hockey Federation"
- "is the top tier of" / "is the second tier of"

**Only accept if it contains:**
- "is a professional ice hockey team"
- "is a junior ice hockey team"
- "hockey club based in"
- "play their home games"
- "was founded in" / "were founded in"
- AND the team name appears in first 300 characters

### Supabase FK Ambiguity Fix

Always use explicit FK hint when joining teams to leagues:

```typescript
// CORRECT
.select('id, name, leagues!primary_league_id(name)')

// WRONG — causes "more than one relationship found" error
.select('id, name, leagues(name)')
```

### Edge Function Limits

- Supabase Edge Function hard timeout: **150 seconds**
- Safe limit per run: **limit: 10** (completes in ~75s)
- CORS issue: browser calls from fanrink.com blocked (OPTIONS → 405)
- Must call from server-side (curl, Node script) not browser fetch

---

## Edge Cases That Destroy Trust

These feel "rare" but silently break product trust:

1. **Stale JWT after claim approval** - permissions don't update
2. **Duplicate ghost teams** for same opponent
3. **Team chosen on prelaunch but lost** due to cookie expiry
4. **DST changes** making game cards display on wrong day
5. **Acceptance flow** attaching fixture to wrong canonical team
6. **Translation refusal/parse failure** leaving UI spinner stuck
7. **Queue backlog** growing invisibly (no mission-control page)
8. **Rec team score disputes** - both teams submit different scores
9. **University team confusion** - varsity vs club teams with same name
10. **Floating team permanence** - ghost team never converted to real team
11. **National team with league** - setting primary_league_id on national team

---

## Quick Decision Framework

**When designing any feature involving teams/leagues/competitions, ask:**

1. Does this respect that teams compete in multiple competitions?
2. Does this handle timezone correctly (UTC storage, IANA zones, locale rendering)?
3. Does this account for season windows varying by hemisphere?
4. Does this work for both men's and women's hockey?
5. Does this handle UK's federated structure (EIHL/England Ice Hockey/Scottish Ice Hockey)?
6. Will this work when southern-hemisphere leagues are live and northern are off-season?
7. Does this respect the multi-competition reality (EIHL teams play League + Cup + Playoffs)?
8. **Does this handle special team types correctly (rec, beer, college, national, floating)?**
9. **Does this account for different league governance models (federation, league, association, community)?**
10. **If this involves user-generated data (rec leagues), is there a verification flow?**
11. **Does this allow primary_league_id to be NULL where appropriate?**

**If any answer is unclear, reference this skill before implementing.**

---

## Hockey Terminology Reference

| Term | Meaning |
|------|---------|
| Hat trick | 3 goals by one player in a game |
| Power play | Team plays with man advantage |
| Penalty kill | Short-handed team defending |
| Gordie Howe hat trick | Goal + assist + fight |
| Zamboni | Ice resurfacing machine |
| Crease | Blue area in front of goal |
| Five-hole | Gap between goalie's legs |
| Breakaway | Player alone vs goalie |
| Overtime (OT) | Extra period to decide tie |
| Shootout (SO) | Penalty shots if OT scoreless |
| Gagarin Cup | KHL championship |
| Stanley Cup | NHL championship |
| Goodall Cup | AIHL championship |

---

## League Type Taxonomy

| Type | Description | Examples |
|------|-------------|----------|
| PRO | Fully professional domestic top tier | SHL, Liiga, DEL, Swiss NL, EIHL |
| INTERNATIONAL | Cross-border professional league | NHL, KHL, CHL, PWHL |
| SEMI_PRO | Mixed economics, lower tiers | NIHL, AIHL |
| JUNIOR | Under-20/major junior | OHL, WHL, QMJHL, USHL |
| COLLEGE | University programs | NCAA Division I |
| WOMENS | Women's leagues | PWHL, SDHL, WNIHL |
| INTERNATIONAL_TOURNAMENT | IIHF events, Olympics | Olympics, IIHF WC, World Juniors |

---

**Version**: v5.1 MERGED (30 Apr 2026)  
**Last Updated**: Combined v4 operational details with v5 team type taxonomy. Added explicit NULL rules for primary_league_id for national/floating/rec teams. Ready for production use.
