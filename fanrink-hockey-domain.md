# FanRink Hockey Domain Skill — v4 (27 Apr 2026)

> Read this before any prompt involving: leagues, teams, game data, scores, schedules, seasons, national teams, or player identities.

---

## Core Architecture

**One auth user → one profile → many identities** (fan, creator, player, team, league, brand, standard).
Users switch identities rather than holding separate profiles.

**Community types:**
- **Fanzones** — platform-created, one per team, P0 for beta. Path: /fanzone/[slug]
- **Locker Rooms** — user-created, P1/P2

**Do NOT conflate Fanzones with Team Profile pages.** They are separate surfaces.
**Teams have ONE primary_league_id** but compete in multiple competitions.

---

## Key Identifiers

- Supabase project: atveiabuhjtmbxestdzu
- Vercel project: prj_alpfjSXXUbD0ULLm5oY1edo7eXEu
- GitHub repo: DevSim1/fanrink-next
- Admin account: supportteam@fanrink.com (UUID: 3ed1047f-2b84-4b48-9f72-c1310c7eedd4)
- Beta access code: BETAFAN (9,999 max uses)
- TheSportsDB premium key: 485341 (images from r2.thesportsdb.com)
- API-Sports Hockey key: f99dd2df2f359d938898b39b52c9b5fc (100 req/day)
- Email provider: Resend (NOT Sendgrid)
- Admin email is Microsoft/Outlook (NOT Gmail)

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

**Free NHL API (no key):**
- GET https://api-web.nhle.com/v1/schedule/{YYYY-MM-DD}
- GET https://api-web.nhle.com/v1/gamecenter/{gameId}/landing
- GET https://api-web.nhle.com/v1/standings/now

---

## DB Critical Rules

- NEVER use FROM (VALUES ...) JOIN teams UPDATE pattern — catastrophic cross-join overwrite risk. Always use individual UPDATE teams SET description = '...' WHERE name = 'X' statements.
- CREATE POLICY IF NOT EXISTS is invalid — wrap in DO block with EXCEPTION WHEN duplicate_object.
- supabase:apply_migration for schema changes (tracked). execute_sql for read-only only.
- leagues table uses abbreviation not short_name.
- Team crests use strBadge not strLogo. Images from r2.thesportsdb.com.
- sync_runs is the canonical cron logging table.
- All commits that don't need a Vercel build must include [skip ci].
- posts.author_id used before migration 20260125000007; user_id only from that version onward.
- is_placeholder = true for ghost teams (NOT is_hidden).
- privacy_level, dm_privacy, follow_approval columns DO NOT YET EXIST on profiles (27 Apr 2026). Add via migration before using.

---

## DB Schema — New Tables (added 26 Apr 2026)

- team_memberships: team_id, user_id, role (owner/admin/player/fan enum), status (active/pending/rejected enum)
- team_invites: team_id, invite_type (enum), invite_code, invited_email, is_placeholder
- job_runs: job_name, status (enum), started_at, finished_at
- post_translations: post_id, language, translated_content (AI cache)
- promoted_posts: post_id, budget_pence, spend_pence, daily_cap_pence, cpm_pence, targeting fields

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

All live-sync crons gate on has_active_games(league_abbr) function.
- sync-nhl-scores (live): every 2min during game windows
- sync-pwhl-scores (live): every 5min during game windows
- sync-aihl-scores (live): every 2min weekends
- cron-refresh (EIHL live): every 2min 14:00-21:00 UTC
- cleanup-job-tables: 03:00 UTC daily

---

## Security Architecture

- private schema for all security definer functions (NOT exposed schema)
- private.is_team_role() wraps auth.uid() in SELECT for RLS
- Views bypass RLS without security_invoker = true
- Claim approval = atomic SQL transaction
- Notification dedup keys: reply:{commentId}:{userId} pattern

---

## Hockey Rankings (2025-26)

Men's: 1=USA, 2=Switzerland, 3=Canada, 4=Sweden, 5=Czechia, 6=Finland, 7=Germany
2026 Olympics Men: USA Gold, Canada Silver, Finland Bronze
2025 WC Men: USA 1st, Switzerland 2nd, Sweden 3rd

Women's: 1=USA, 2=Canada, 3=Finland, 4=Czechia, 5=Switzerland, 6=Sweden
2026 Olympics Women: USA Gold, Canada Silver, Switzerland Bronze

---

## UK Hockey Structure

EIHL (10 teams, top pro tier) → NIHL National League (10 English) → NIHL Division 1 North (10 Scottish)
EIHL regular season champion qualifies for Champions Hockey League.
Scottish clubs: Aberdeen Lynx, Dundee Rockets, Edinburgh Capitals, Kirkcaldy Kestrels, Kilmarnock Thunder, North Ayrshire Wild, Paisley Pirates, Solway Sharks, Whitley Warriors.

---

## Vercel / GitHub Patterns

- Poll Vercel:get_deployment with specific deployment ID (not list_deployments) for build tracking
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
- CLAUDE.md in fanrink-next root is auto-read by Claude Code
- docs/ai/NEXT_PROMPT.md is the single source of truth for work queue
# FanRink Hockey Domain Knowledge

## Purpose
This file is the canonical hockey knowledge reference for Claude when working on FanRink. It contains verified facts about leagues, competitions, structures, rankings, and team data. Always consult this before writing team/league descriptions, seeding data, or making assumptions about competition structures.

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
teams.primary_league_id        → home/primary league (single FK)
team_competitions              → many-to-many: team ↔ competition (empty, needs seeding)
team_league_memberships        → many-to-many: team ↔ league (empty, needs seeding)
competitions                   → specific competition instances (3 rows: EIHL League, Challenge Cup, Playoffs)
leagues                        → 96 leagues total
```

**Rule**: When showing a team's competitions to a user, always think beyond `primary_league_id`. The `team_competitions` table is where multi-competition memberships live once seeded.

---

## IIHF World Rankings (Current as of 2026)

### Men's Rankings (published 2025-05-26)
| Rank | Country | Points |
|------|---------|--------|
| 1 | USA | 3,985 |
| 2 | Switzerland | 3,975 |
| 3 | Canada | 3,935 |
| 4 | Sweden | 3,915 |
| 5 | Czechia | 3,860 |
| 6 | Finland | 3,780 |
| 7 | Germany | 3,710 |
| 8 | Denmark | — |
| 9 | Slovakia | 3,595 |
| 17 | United Kingdom | 3,100 |

> Russia and Belarus: not participating in IIHF World Championship programme (points still tracked).

### Women's Rankings (published 2025-04-21)
| Rank | Country | Points |
|------|---------|--------|
| 1 | USA | 4,150 |
| 2 | Canada | 4,140 |
| 3 | Finland | 3,930 |
| 4 | Czechia | 3,920 |
| 5 | Switzerland | 3,855 |
| 6 | Sweden | 3,730 |

---

## Major Tournament Results (2025–2026)

### 2026 Winter Olympics — Men's Ice Hockey
- 🥇 **Gold**: USA (beat Canada OT in final)
- 🥈 **Silver**: Canada
- 🥉 **Bronze**: Finland (beat Slovakia 6–1)
- 4th: Slovakia

### 2026 Winter Olympics — Women's Ice Hockey
- 🥇 **Gold**: USA (beat Canada OT in final)
- 🥈 **Silver**: Canada
- 🥉 **Bronze**: Switzerland (beat Sweden OT)
- 4th: Sweden

### 2025 IIHF Men's World Championship
1. USA
2. Switzerland
3. Sweden

### 2025 IIHF Women's World Championship
1. USA
2. Canada

---

## Top League Structures by Country

### NHL (North America)
- 32 teams: 25 USA, 7 Canada
- 82-game regular season (October–April)
- 16-team playoffs → Stanley Cup (April–June)
- Closed league, no promotion/relegation
- Olympic break in February during Olympic years
- Division: Metropolitan, Atlantic (East), Central, Pacific (West)

### KHL (Russia/Eurasia)
- 22 clubs: Russia, Belarus, China, Kazakhstan (2025–26)
- 68-game regular season (Sep 5 – mid-March)
- Gagarin Cup Playoffs (Mar–May 23)
- Closed league
- Clubs: Ak Bars Kazan, Avangard Omsk, CSKA Moscow, SKA St Petersburg, Lokomotiv Yaroslavl, Metallurg Magnitogorsk, Dynamo Moscow, Salavat Yulaev Ufa, Traktor Chelyabinsk, Barys Astana, HC Sochi, Severstal, Neftekhimik, Amur Khabarovsk, Torpedo NN, Dinamo Minsk + others

### SHL (Sweden)
- 14 clubs
- Regular season Sep 13 – Mar 14
- Play-in round then playoffs (Mar–May)
- Promotion/relegation with HockeyAllsvenskan
- CHL qualification for top clubs
- Key clubs: Djurgårdens, Frölunda, Färjestad, Brynäs, Skellefteå, Rögle, Leksands, Linköping, Malmö, Växjö, HV71, Timrå, Örebro, Modo

### Liiga (Finland)
- 15 clubs
- Season: September–May
- One of Europe's strongest leagues
- CHL qualification

### DEL (Germany)
- 15 clubs, closed league
- Regular season Sep 9 – Mar 15
- Playoffs Mar 17 – May 7
- CHL qualification for champion

### Swiss National League (NL)
- 14 clubs
- 52-game regular season (Sep 9 – Mar)
- Play-in, playoffs, playout (promotion/relegation with Swiss League)
- Switzerland ranked 2nd in world — very strong league

### Czech Extraliga (Tipsport Extraliga)
- 14 clubs
- Season Sep 10 – Apr 30
- One of Europe's strongest leagues

### EIHL (Elite Ice Hockey League, UK)
- 10 clubs: Belfast Giants, Cardiff Devils, Coventry Blaze, Dundee Stars, Fife Flyers, Glasgow Clan, Guildford Flames, Manchester Storm, Nottingham Panthers, Sheffield Steelers
- 3 competitions per season: League Championship, Challenge Cup, Playoffs
- Regular season champion qualifies for CHL
- Season: October–April

### NIHL National League (England, second tier)
- 10 clubs: Leeds Knights, Sheffield Steeldogs, Swindon Wildcats, Telford Tigers, Basingstoke Bison, Peterborough Phantoms, Hull Seahawks, Milton Keynes Lightning, Romford Raiders, Bracknell Bees

### NIHL Division 1 North (Scotland)
- 10 clubs: Aberdeen Lynx, Dundee Rockets, Edinburgh Capitals, Kirkcaldy Kestrels, Kilmarnock Thunder, North Ayrshire Wild, Paisley Pirates, Solway Sharks, Solway Sharks SNL, Whitley Warriors

### CHL (Champions Hockey League)
- Pan-European club competition
- 24 clubs from SHL, Liiga, DEL, Swiss NL, Czech Extraliga, EIHL + others
- Group stage + knockouts (Aug–Feb)
- EIHL: regular season champion qualifies as "Challenger League" representative

### PWHL (Professional Women's Hockey League)
- 8 clubs across North America (expanding from 6)
- Boston Fleet, Minnesota Frost, Montreal Victoire, New York Sirens, Ottawa Charge, Seattle Tempest, Toronto Sceptres, Vancouver Rise
- Toronto won inaugural PWHL championship 2024
- Top level of professional women's hockey globally

### OHL / WHL / QMJHL (Canadian Hockey League — Junior)
- Three major junior leagues, together form the CHL
- Primary NHL Draft development pathway
- OHL: 20 clubs, Ontario + Michigan
- WHL: 22 clubs, Western Canada + Pacific Northwest USA
- QMJHL: 18 clubs, Quebec + Atlantic provinces
- Memorial Cup: annual CHL championship between the three league champions + host team

### AIHL (Australian Ice Hockey League)
- 10 clubs: Adelaide Adrenaline, Brisbane Lightning, Canberra Brave, Central Coast Rhinos, Melbourne Ice, Melbourne Mustangs, Newcastle Northstars, Perth Thunder, Sydney Bears, Sydney Ice Dogs
- Season April–September (Southern Hemisphere reversed calendar)
- Goodall Cup championship

### Asia League Ice Hockey
- International league: Japan, South Korea, China
- East Asia's top professional level
- Korean clubs: Anyang Halla, High1
- Japanese clubs: Nikkō Ice Bucks, Ōji Eagles, Tohoku Free Blades, Yokohama Grits

---

## FanRink Data State (as of March 2026)

### Teams
- **Total in-scope**: 468
- **All 468 have primary_league_id** ✅
- **Have descriptions**: ~55 (NHL x18, KHL x16, SHL x8, PWHL x8, national teams x5) — Wikipedia enrichment ongoing
- **Have logo_url**: 434
- **Have arena_name**: 297
- **Have founded_year**: 398
- **Have achievements**: 2 (needs backfill)
- **Have TheSportsDB ID**: 145

### Leagues
- **Total**: 96
- **All 96 have descriptions** ✅

### Games
- Only 2 leagues have actual game data: **NHL** (1,157 games) and **EIHL** (856 games)
- All other leagues have teams assigned but zero games — game ingestion is a future project

---

## Enrichment Pipeline Notes

### Wikipedia Enrichment Rules
**Always reject a page if it contains:**
- "is a professional ice hockey league"
- "The Kontinental Hockey League (KHL"
- "The National Hockey League (NHL"
- "The Swedish Hockey League (SHL"
- "The National League (NL) is a professional"
- "organized by the International Ice Hockey Federation"
- "series of tournaments"
- "is the top tier of" / "is the second tier of"

**Only accept if it contains (team-page signals):**
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

### Critical Migration Warning
**NEVER use `FROM (VALUES ...) JOIN teams` pattern for UPDATE statements.**
A missing WHERE condition causes ALL rows to get the same value (cross-join).
Always use individual `UPDATE teams SET ... WHERE name = 'X'` statements.

---

## Hockey Terminology Reference

| Term | Meaning |
|------|---------|
| Hat trick | 3 goals by one player in a game |
| Power play | Team plays with man advantage (opponent has penalty) |
| Penalty kill | Short-handed team defending during opponent's power play |
| Puck drop | Start of a game / the moment play begins |
| Gordie Howe hat trick | Goal + assist + fight in same game |
| Zamboni | Ice resurfacing machine (used between periods) |
| Crease | The blue area in front of goal — goalie's protected zone |
| Blue line | The lines that define the offensive/defensive zones |
| Icing | Shooting puck from behind centre line to behind opponent's goal line without touch |
| Offside | Player enters offensive zone before the puck |
| Checking | Legal body contact to separate opponent from puck |
| Enforcer | Player whose primary role is physical intimidation/fighting |
| Deke | Fake/feint move to get past a defender |
| Five-hole | Gap between a goalie's legs |
| Breakaway | Player with puck, no defenders, heads toward goal 1-on-1 with goalie |
| Penalty shot | 1-on-1 with goalie awarded for certain fouls |
| Overtime (OT) | Extra period to decide a tied game |
| Shootout (SO) | Tie-breaking series of penalty shots if OT is scoreless |
| Gagarin Cup | KHL championship trophy |
| Stanley Cup | NHL championship trophy (oldest trophy in pro sports) |
| Calder Cup | AHL championship trophy |
| Memorial Cup | CHL (junior) championship trophy |
| Goodall Cup | AIHL (Australia) championship trophy |
| President's Cup | QMJHL championship trophy |
| Robertson Cup | OHL championship trophy |
| Ed Chynoweth Cup | WHL championship trophy |

---

## League Type Taxonomy (for FanRink data model)

| Type | Description | Examples |
|------|-------------|---------|
| PRO | Fully professional domestic top tier | SHL, Liiga, DEL, Swiss NL, Czech Extraliga, EIHL |
| INTERNATIONAL | Cross-border professional league | NHL, KHL, CHL, Asia League, ICEHL, PWHL |
| SEMI_PRO | Mixed economics, lower tiers | NIHL, AIHL, most national second tiers |
| JUNIOR | Under-20/major junior | OHL, WHL, QMJHL, USHL |
| COLLEGE | University programs | NCAA Division I |
| WOMENS | Women's leagues | PWHL, SDHL, SDHL women, WNIHL |
| INTERNATIONAL_TOURNAMENT | IIHF events, Olympics | Olympics, IIHF WC, World Juniors, 4 Nations |
