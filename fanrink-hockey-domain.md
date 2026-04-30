# FanRink Hockey Domain Knowledge

**Purpose**: This skill provides comprehensive ice hockey domain knowledge for building FanRink correctly. It covers global league structures, competition models, season windows, team type taxonomy, and critical data modeling considerations.

**Use this skill when**: Designing database schemas, planning data ingestion, discussing league coverage, modeling team competitions, handling fixtures/schedules, making any product decisions requiring hockey domain knowledge, or working with special team types (rec leagues, beer leagues, college, national teams, floating teams).

---

## Core Data Model Principle

**CRITICAL**: Teams have ONE `primary_league_id` (their home league) but compete in MULTIPLE competitions.

```
Team → primary_league_id (home league)
Team → team_competitions (many)
Team → team_league_memberships (empty but exists)
```

### Why This Matters

- **EIHL teams** play: EIHL League Championship + EIHL Challenge Cup + EIHL Playoffs
- **NHL teams** play: Regular season + Playoffs + potentially 4 Nations/All-Star
- **SHL/DEL/Liiga top clubs** qualify for: Champions Hockey League (CHL)
- **KHL clubs** from: Russia, Belarus, China, Kazakhstan
- **National team players** compete for: Clubs AND national teams (Olympics, World Championships)

Never conflate `primary_league_id` with "only plays in one competition". The multi-competition reality is fundamental to hockey.

---

## Team Type Taxonomy

**CRITICAL**: Different team types require different data models and UX patterns.

### Professional Teams
- **Governance**: Formal league authority
- **Characteristics**: Player contracts, salaries, official stats
- **Examples**: NHL, EIHL, SHL, Liiga, DEL, Swiss NL
- **Data Quality**: High - official APIs/feeds available
- **Verification**: Not required (leagues verify themselves)

### Semi-Professional Teams
- **Governance**: Regional league governance
- **Characteristics**: Mix of paid/amateur players
- **Examples**: NIHL, lower-tier European leagues
- **Data Quality**: Medium - some official data, some self-reported
- **Verification**: League-level verification

### National Teams
- **Governance**: IIHF / national federations
- **Characteristics**: 
  - `primary_league_id` = NULL (no home league)
  - Only compete in tournaments
  - Players represent country, not club
- **Examples**: Team USA, Team Canada, Team Finland
- **Data Quality**: High - official IIHF data
- **Critical Rule**: National team identity is SEPARATE from club identity

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

## Season Windows 2025-26 Reference

**NHL**: Oct–Jun (82 games + playoffs)  
**KHL**: Sep 5–May 23 (68 games)  
**SHL**: Sep 13–May  
**DEL**: Sep 9–May 7  
**Swiss NL**: Sep 9–May (52 games)  
**Czech Extraliga**: Sep 10–Apr 30  
**Liiga**: Sep–May  
**EIHL**: Oct–Apr  
**CHL (Champions Hockey League)**: Aug–Feb  
**PWHL**: Nov 21–Apr 25 (regular season) → playoffs start Apr 30  
**SDHL**: Sep–Mar/Apr  
**AIHL**: Mid-Apr–30 Aug  
**NZIHL**: 30 May–mid-Aug  
**WNIHL**: Sep 6–May 10 (regular season) → finals May 23-25  

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

## IIHF Rankings (May 2025)

### Men's Rankings
1. USA
2. Switzerland
3. Canada
4. Sweden
5. Czechia
6. Finland
7. Germany

**Note**: Russia/Belarus not participating

### Women's Rankings
1. USA
2. Canada
3. Finland
4. Czechia
5. Switzerland
6. Sweden

### Recent Olympics Results
**2026 Olympics Men**: USA Gold, Canada Silver, Finland Bronze  
**2026 Olympics Women**: USA Gold, Canada Silver, Switzerland Bronze

**2025 World Championships Men**: USA 1st, Switzerland 2nd, Sweden 3rd  
**2025 World Championships Women**: USA 1st, Canada 2nd

---

## Global League Prioritisation Matrix (First Wave)

For a solo-builder product, choose leagues that maximize fan value per engineering hour:

### Recommended First Wave
1. **NHL** - Obvious
2. **UK stack** (EIHL, NIHL, WNIHL) - English-language, beta geography fit
3. **PWHL** - Women's premium, growing fast
4. **Australia stack** (AIHL, then AWIHL) - Southern hemisphere calendar fill
5. **Junior bundle** (USHL + reusable CHL parsers for OHL/WHL/QMJHL)

### Why This Bundle Works
- Fan demand balanced with engineering hours
- English-language UX
- Machine-readable sources available
- Year-round content calendar
- Source quality strong (USHL exposes RSS/Excel/calendar; EIHL has GameCentre; AIHL structured schedule; WNIHL/EIH exposes rules PDFs + event calendars)

### After First Wave
- **SHL** (Swedish league)
- **DEL** (German league)
- **Liiga** (Finnish league)
- **KHL** (with legal caution - site reserves rights, prohibits automatic extraction)
- **National League** (Swiss)

---

## Data Architecture Principles

### Database Schema Requirements

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

### Competitions Table
Currently has 3 rows:
1. EIHL League Championship
2. EIHL Challenge Cup
3. EIHL Playoffs

This illustrates the multi-competition model correctly.

### Team-Competition Relationships

**DO NOT conflate**:
- `primary_league_id` = home league affiliation
- `team_competitions` = actual competitions participated in (can be many)
- `team_league_memberships` = exists but empty (future multi-season membership tracking)

### Critical Update Pattern Anti-Pattern

**NEVER use VALUES JOIN UPDATE pattern for team descriptions** - caused catastrophic cross-join overwrite in past. Always use individual UPDATE statements:

```sql
-- CORRECT
UPDATE teams SET description = '...' WHERE name = 'Team X';
UPDATE teams SET description = '...' WHERE name = 'Team Y';

-- WRONG (catastrophic cross-join risk)
UPDATE teams t
SET description = v.desc
FROM (VALUES
  ('Team X', 'Desc X'),
  ('Team Y', 'Desc Y')
) v(name, desc)
WHERE t.name = v.name;
```

### Column Name Verification
Always verify before writing:
- `leagues` table uses `abbreviation` NOT `short_name`
- `provider_health` uses `provider` and `league_key`
- Team crests: use `strBadge` NOT `strLogo`
- Images: from `r2.thesportsdb.com` NOT `www.thesportsdb.com`
- Posts: `author_id` before migration `20260125000007`, `user_id` from that version onwards

### TheSportsDB Premium Key
`485341` - images from `r2.thesportsdb.com`

### API-Sports Hockey API Key
`f99dd2df2f359d938898b39b52c9b5fc` (100 req/day free tier)  
Used for: SHL, DEL, Liiga, PWHL, KHL ingestion

---

## Timezone & Localisation Critical Rules

### Database Storage
- PostgreSQL stores timestamps internally in **UTC**
- Use full **IANA timezone names** (e.g., `Europe/London`, `Australia/Sydney`)
- **DO NOT** model scheduling with `time with time zone` fields alone

### Client Rendering
- `Intl.DateTimeFormat` supports locale-sensitive rendering
- Use explicit `timeZone` handling
- Support both venue-local and user-local display
- Watch DST boundary cases

### Venue vs Team Timezone
- Prefer `venue.timezone` when available
- Fallback to `team.timezone` if venue missing
- Admin localisation dashboard should flag missing venue timezones
- **DST changes can make game cards display on wrong day** - test rigorously

### Examples
- UK: `Europe/London` (supports BST/GMT transitions)
- Australia: `Australia/Sydney`, `Australia/Melbourne`
- North America: `America/New_York`, `America/Chicago`, `America/Denver`, `America/Los_Angeles`

---

## Fanzones vs Locker Rooms (Never Conflate)

### Fanzones
- Platform-created
- One per team
- **P0 for beta** - most important community type
- Every team gets one automatically

### Locker Rooms
- User-created communities
- P1/P2 priority
- Custom topic-based communities
- Not tied to specific team

---

## Legal & Source Considerations

### KHL Specific Warning
KHL site **explicitly reserves rights and prohibits forms of automatic extraction without consent**. This is why country- and source-aware governance matters.

### General Source Priority
1. Official league APIs/feeds (when available)
2. Official schedule pages/GameCentres
3. RSS/calendar feeds
4. Structured HTML (with robots.txt compliance)
5. Browser automation (last resort, legal risk)

### Public Data Organization
Hockey data is NOT organized by neat league APIs. It's organized by:
- National federations
- Shared site vendors (HockeyTech, LeagueStat)
- Shared CMSs
- Recurring document types

**Better architecture**: One orchestrator per country/federation cluster, with league-specific parsers underneath

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

---

## Current FanRink DB State (as of memories)

- **468 in-scope teams**, all have `primary_league_id`
- **55 verified team descriptions** (NHL ×18, KHL ×16, SHL ×8, PWHL ×8, national teams ×5)
- **413 teams** still need Wikipedia enrichment
- **96 leagues** all have descriptions
- **434 teams** have `logo_url`
- **297 teams** have `arena_name`
- **398 teams** have `founded_year`
- **2 teams** have `achievements`

**Missing from schema (need to add):**
- `team_type` enum column on teams
- `governance_type` enum column on leagues
- `is_self_reported` boolean on games/scores
- `verification_status` enum on games (pending/confirmed/disputed)

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

**If any answer is unclear, reference this skill before implementing.**

---

## Database Migration Checklist for Team Type Support

When adding team type support, run these migrations in order:

1. **Add ENUMs:**
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

2. **Add columns:**
```sql
ALTER TABLE teams ADD COLUMN team_type team_type;
ALTER TABLE leagues ADD COLUMN governance_type league_governance;
ALTER TABLE games ADD COLUMN is_self_reported boolean DEFAULT false;
ALTER TABLE games ADD COLUMN verification_status varchar;
```

3. **Set defaults for existing data:**
```sql
-- Professional teams (have official leagues)
UPDATE teams SET team_type = 'professional' 
WHERE primary_league_id IN (
  SELECT id FROM leagues WHERE abbreviation IN 
  ('NHL', 'EIHL', 'SHL', 'DEL', 'Liiga', 'Swiss NL', 'Czech ELH', 'KHL')
);

-- National teams (no primary_league_id, country names)
UPDATE teams SET team_type = 'national'
WHERE name LIKE 'Team %' OR name IN ('USA', 'Canada', 'Finland', ...);

-- Junior teams
UPDATE teams SET team_type = 'junior'
WHERE primary_league_id IN (
  SELECT id FROM leagues WHERE abbreviation IN ('OHL', 'WHL', 'QMJHL', 'USHL')
);

-- Rest default to semi_professional for now
UPDATE teams SET team_type = 'semi_professional' WHERE team_type IS NULL;
```

4. **Set league governance:**
```sql
UPDATE leagues SET governance_type = 'league_organized'
WHERE abbreviation IN ('NHL', 'EIHL', 'SHL', 'DEL', 'Liiga');

UPDATE leagues SET governance_type = 'federation_governed'
WHERE name LIKE '%IIHF%' OR name LIKE '%National%';

UPDATE leagues SET governance_type = 'association_registered'
WHERE name LIKE '%Rec%' OR name LIKE '%Beer%';
```

---

**Version**: v5 (30 Apr 2026)
**Last Updated**: Added team type taxonomy, rec/beer league handling, college structure, floating teams definition, league governance models, and database migration checklist.
