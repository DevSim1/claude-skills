# FanRink High-Value Skills Index

Curated skills for building FanRink — the mobile-first social network for hockey fans.

## Quick Install (Claude Code)

```bash
# Install all FanRink-relevant skills at once
npx skills add wondelai/skills/hooked-ux
npx skills add wondelai/skills/improve-retention
npx skills add wondelai/skills/contagious
npx skills add wondelai/skills/system-design
npx skills add wondelai/skills/refactoring-ui
npx skills add wondelai/skills/jobs-to-be-done
npx skills add wondelai/skills/storybrand-messaging
npx skills add wondelai/skills/ios-hig-design
npx skills add wondelai/skills/ux-heuristics
npx skills add wondelai/skills/microinteractions
```

Or via Claude Code Plugin Marketplace:
```
/plugin marketplace add wondelai/skills
/plugin install ux-design@wondelai-skills
/plugin install marketing-cro@wondelai-skills
/plugin install systems-architecture@wondelai-skills
```

---

## Priority 1: Engagement & Retention

### hooked-ux
**Purpose:** Hook Model framework for habit-forming products (Trigger → Action → Variable Reward → Investment)

**FanRink Application:**
- Design the feed refresh loop (internal trigger: "what's happening with my team?")
- Variable rewards: live scores, breaking news, hot takes from other fans
- Investment: following teams, posting reactions, earning XP

**Example prompts:**
- "Audit FanRink's onboarding for Hook Model completion. Use hooked-ux skill."
- "Design the internal trigger map for hockey fans checking the app. Use hooked-ux skill."
- "What variable rewards should we show in the My Rink feed? Use hooked-ux skill."

---

### improve-retention
**Purpose:** Fogg Behavior Model (B=MAP) for diagnosing why users don't return

**FanRink Application:**
- Day 1/7/30 retention diagnostics
- Friction audit on signup → team selection → first post flow
- Push notification timing and content

**Example prompts:**
- "Our Day-7 retention is 20%. Diagnose using B=MAP. Use improve-retention skill."
- "Audit the team selection step for friction using the Ability Chain. Use improve-retention skill."
- "Design a Tiny Habits recipe for fans checking live scores daily. Use improve-retention skill."

---

### contagious
**Purpose:** STEPPS framework for word-of-mouth and virality

**FanRink Application:**
- Social currency: exclusive beta access, verified fan status
- Triggers: game day notifications, playoff brackets
- Emotion: clutch goals, rivalry wins, heartbreak losses
- Public: shareable goal clips, bracket predictions
- Stories: "I called it" moments fans can share

**Example prompts:**
- "Design a referral mechanic using STEPPS for FanRink beta. Use contagious skill."
- "How do we build social currency into fan profiles? Use contagious skill."
- "Create shareable moments around playoff brackets. Use contagious skill."

---

## Priority 2: Architecture & Scalability

### system-design
**Purpose:** Scalable distributed systems design

**FanRink Application:**
- Feed architecture for 100K+ MAU
- Real-time score updates (WebSocket vs SSE vs polling)
- Caching strategy for team/game data
- Database partitioning for multi-league support

**Example prompts:**
- "Design the feed generation system for FanRink. Use system-design skill."
- "How do we scale live score updates to 50K concurrent users? Use system-design skill."
- "Calculate capacity needs for 100K DAU with 10 page views each. Use system-design skill."

---

## Priority 3: UI/UX Quality

### refactoring-ui
**Purpose:** Practical UI design for developers

**FanRink Application:**
- Mobile-first card design for feed items
- Color palette that works with team colors
- Typography scale for stats vs commentary
- Dark mode implementation

**Example prompts:**
- "Audit the game card component for visual hierarchy. Use refactoring-ui skill."
- "Design a color system that complements team brand colors. Use refactoring-ui skill."
- "Fix the spacing on the fanzone member list. Use refactoring-ui skill."

---

### ios-hig-design
**Purpose:** Native iOS patterns (applies to mobile-first web too)

**FanRink Application:**
- Navigation patterns (tab bar, nav stack)
- Touch targets for mobile
- Pull-to-refresh, infinite scroll
- Safe areas and responsive layout

**Example prompts:**
- "Review the FanRink tab bar against HIG guidelines. Use ios-hig-design skill."
- "What's the correct gesture for switching between My Rink and Open Ice? Use ios-hig-design skill."

---

### ux-heuristics
**Purpose:** Nielsen's 10 heuristics + Krug's usability principles

**FanRink Application:**
- Onboarding usability audit
- Error states and empty states
- Navigation clarity ("trunk test")
- Form usability (registration, posting)

**Example prompts:**
- "Run the Trunk Test on the FanRink home screen. Use ux-heuristics skill."
- "Audit the post creation flow for usability issues. Use ux-heuristics skill."

---

### microinteractions
**Purpose:** Design triggers, feedback, loops for polish

**FanRink Application:**
- Like/react animations
- Pull-to-refresh feedback
- Score update animations
- Loading states that build anticipation

**Example prompts:**
- "Design the microinteraction for liking a post. Use microinteractions skill."
- "Create a loading state for live scores that builds anticipation. Use microinteractions skill."

---

## Priority 4: Product Strategy

### jobs-to-be-done
**Purpose:** Understand why fans "hire" FanRink

**FanRink Application:**
- Core jobs: "Help me feel connected to my team when I can't be at the rink"
- Competing alternatives: Reddit, Twitter, team subreddits, WhatsApp groups
- Interview questions for beta users

**Example prompts:**
- "Write JTBD interview questions for FanRink beta users. Use jobs-to-be-done skill."
- "What jobs compete with FanRink that aren't other hockey apps? Use jobs-to-be-done skill."

---

### storybrand-messaging
**Purpose:** Clear brand positioning using story structure

**FanRink Application:**
- App Store description
- Landing page copy
- Onboarding messaging
- Email sequences

**Example prompts:**
- "Create a StoryBrand brand script for FanRink. Use storybrand-messaging skill."
- "Write a one-liner for FanRink's App Store listing. Use storybrand-messaging skill."

---

## Also Useful

| Skill | Use Case |
|-------|----------|
| `mom-test` | Beta user interview questions |
| `hundred-million-offers` | Premium tier pricing |
| `one-page-marketing` | Go-to-market plan |
| `lean-startup` | MVP scoping |
| `clean-architecture` | Code structure |
| `release-it` | Production stability patterns |

---

## Installation in Claude Code

From your fanrink-next repo:

```bash
# Install skills.sh if not already installed
npm install -g skills

# Add wondelai marketplace
skills marketplace add wondelai/skills

# Install individual skills
skills add wondelai/skills/hooked-ux
skills add wondelai/skills/improve-retention
skills add wondelai/skills/contagious
# ... etc
```

Skills will be installed to `.claude/skills/` in your repo and automatically used by Claude Code.

---

## See Also

- [fanrink-hockey-domain.md](./fanrink-hockey-domain.md) — Hockey-specific domain knowledge
- [skills-resource-links.md](./skills-resource-links.md) — Full index of 300+ skills
- [wondelai/skills](https://github.com/wondelai/skills) — Source repo for these skills
- [skills.wondel.ai](https://skills.wondel.ai) — Browse all 41 wondelai skills
