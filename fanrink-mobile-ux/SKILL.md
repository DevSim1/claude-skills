# FanRink Mobile UX/UI Skill

## Purpose
This skill defines the non-negotiable mobile UX/UI standards for FanRink.
90% of FanRink users are on mobile — primarily iPhone (Safari) and Android (Chrome).
Every UI decision must pass a "does this feel good on a 390px iPhone screen?" test.
This skill is read before writing or reviewing ANY component that has a visual output.

---

## The Golden Rule

**If it looks fine on desktop but feels squashed, tiny, or awkward on mobile — it is broken.**

FanRink is a mobile-first social app competing with Instagram, Twitter/X, and TikTok in the
attention economy. Users will leave the moment something feels unpolished on their phone.
Desktop is secondary. Always design for the thumb first.

---

## Target Device Profiles

### Primary (design for these first)
| Device | Screen width | Browser | Notes |
|--------|-------------|---------|-------|
| iPhone 15 / 15 Pro | 393px logical | Safari iOS 17+ | Most common beta user device |
| iPhone 14 / 14 Pro | 390px logical | Safari iOS 16+ | High volume |
| iPhone SE (3rd gen) | 375px logical | Safari iOS 15+ | Smallest target — test here |
| iPhone 12/13 | 390px logical | Safari iOS 15+ | High volume |

### Secondary
| Device | Screen width | Browser |
|--------|-------------|--------|
| Samsung Galaxy S23/S24 | 360–393px | Chrome Android |
| Pixel 7/8 | 393px | Chrome Android |
| iPad Mini | 744px | Safari iOS |

### Desktop (never break, but not primary)
| Device | Screen width | Browser |
|--------|-------------|--------|
| MacBook / Windows | 1280px+ | Chrome/Firefox/Safari |

---

## Safe Area and Viewport Rules

### iPhone Safe Areas — Always Handle These
```css
/* Bottom safe area — CRITICAL for bottom tab bar */
paddingBottom: 'env(safe-area-inset-bottom, 0px)'

/* Top safe area — for full-bleed headers */
paddingTop: 'env(safe-area-inset-top, 0px)'

/* Dynamic Island / Notch handling */
viewportFit: 'cover'  /* in Next.js viewport export */
```

### The Bottom Tab Bar Problem
FanRink has a fixed bottom tab bar. **Every scrollable page must have bottom padding** that
accounts for BOTH the tab bar height AND the iPhone safe area inset:
```
pb-[calc(var(--bottom-tabbar-height, 64px) + env(safe-area-inset-bottom, 0px) + 16px)]
```
If a page's content gets hidden behind the bottom tab bar — this padding is missing.

### Viewport Meta — Already Set in IceLayout
```typescript
export const viewport: Viewport = {
  width: 'device-width',
  initialScale: 1,
  viewportFit: 'cover',
};
```
Never remove `viewportFit: 'cover'` — it enables proper safe area handling on iPhone.

---

## Touch Target Standards

### Minimum Sizes — Never Go Below These
| Element | Min width | Min height | Why |
|---------|-----------|------------|-----|
| Bottom nav tab | 44px | 52px | Fat finger safety + iOS HIG |
| Button (primary) | 44px | 48px | iOS minimum |
| Button (secondary/ghost) | 44px | 44px | Absolute minimum |
| Icon button | 44px | 44px | Wrap icon in 44px container |
| List item / card tap area | full width | 52px+ | Generous tap target |
| Input field | full width | 48px | Comfortable typing |
| Checkbox / toggle | 44px | 44px | Include label in tap area |

### Implementation Pattern
```tsx
// WRONG — icon is 20px, untappable
<button><Icon size={20} /></button>

// CORRECT — 44px tap target wrapping small icon
<button className="flex items-center justify-center w-11 h-11 rounded-full">
  <Icon size={20} />
</button>
```

### Tap Target Spacing
Adjacent tap targets should have at least 8px gap between them.
On a bottom tab bar with 5 items, each item needs ≥ 60px width (390px ÷ 5 = 78px max).
**Never have more than 5 items in a bottom tab bar including the FAB spacer.**

---

## Typography Standards for Mobile

### Font Size Minimums — Never Go Below These
| Context | Min size | Tailwind class | Why |
|---------|----------|---------------|-----|
| Body text / post content | 14px | `text-sm` | Readable without zooming |
| Secondary / metadata | 12px | `text-xs` | Absolute minimum |
| Labels / captions | 11px | `text-[11px]` | Acceptable for nav labels only |
| Never use | <11px | `text-[10px]` | Too small on mobile |
| Headings | 18px+ | `text-lg`+ | Clear visual hierarchy |
| Section titles | 16px | `text-base` | Minimum for section headers |

### Line Height
Always use `leading-relaxed` (1.625) or `leading-normal` (1.5) for body text.
Never use `leading-tight` for text longer than 1 line on mobile.

### Font Weight
Primary content: `font-medium` (500) or `font-semibold` (600)
Secondary/metadata: `font-normal` (400) at reduced opacity (`text-white/60`)
Never use `font-thin` or `font-extralight` for content text on mobile — illegible

---

## Spacing and Layout Standards

### Horizontal Padding
Page-level horizontal padding on mobile: **16px minimum** (`px-4`)
Never let text touch the edge of the screen.
Full-bleed images and cards are fine, but text inside them needs inner padding.

### Card and List Item Padding
Minimum inner padding for interactive cards: `p-4` (16px all sides)
For dense lists: `px-4 py-3` (16px horizontal, 12px vertical)
Never less than `p-3` for any tappable card.

### Vertical Spacing Between Sections
Mobile feed sections: `space-y-4` (16px) to `space-y-6` (24px)
Desktop can use `space-y-6` to `space-y-8`
Do not let sections bleed into each other without clear visual separation.

### Gap Between Related Elements
Inline tag groups: `gap-2` (8px)
Card grids: `gap-3` (12px) on mobile, `gap-4` (16px) on desktop
Bottom nav tabs: distribute with `flex-1` — do not set fixed widths

---

## Bottom Navigation Standards

### FanRink Bottom Tab Bar Rules
```
Max items: 5 (including FAB spacer slot)
Recommended layout: [Ice] [Fanzone] [+FAB] [Alerts] [Profile]
Never more than 4 navigable tabs + 1 FAB slot
```

### Tab Item Requirements
```tsx
// Each tab item must have:
className="flex flex-col items-center gap-0.5 flex-1 min-h-[52px] py-2 px-2"

// Icon: 22–24px, not smaller
<Icon size={22} strokeWidth={active ? 2.5 : 1.5} />

// Label: 11px minimum
<span className="text-[11px] font-medium">{label}</span>
```

### Bottom Bar Height
Target height: **56–64px** content area + safe-area inset
Do not let the bar be so tall it eats content, or so short it's hard to tap.

### Active State
Active tab: `text-fr-ice` (the brand ice blue)
Inactive tab: `text-white/40` — clearly visually subordinate
Active icon strokeWidth: 2.5 (bolder)
Inactive icon strokeWidth: 1.5 (thinner)

### Badge Positioning
```tsx
// Badge on notification bell — top-right corner of the icon container
<span className="absolute top-1 right-1 flex h-4 min-w-[1rem] items-center justify-center
  rounded-full bg-gradient-to-r from-rose-500 to-amber-400 px-1 text-[9px] font-bold text-white">
  {count > 99 ? '99+' : count}
</span>
```
Badge must never overlap adjacent tabs or be clipped by the container.

---

## Loading and Skeleton States

### Rules for Skeleton States
1. **Never show a skeleton longer than 3 seconds** — after 3s, show empty state or error
2. **Never show a skeleton for a section that has no data** — return null instead
3. Skeleton colours: `bg-white/10 animate-pulse` — subtle, not jarring black boxes
4. Skeleton dimensions must match actual content dimensions
5. Show max 3 skeleton cards at once — more is overwhelming

### The Black Box Problem (avoid this)
```tsx
// WRONG — dark skeleton on dark background = invisible/broken-looking
<div className="bg-black animate-pulse" />

// CORRECT — slightly lighter than background, clearly a placeholder
<div className="bg-white/10 animate-pulse rounded-xl" />
```

### Timeout Pattern for Conditional Sections
```tsx
// For sections that only show when data exists:
const [timedOut, setTimedOut] = useState(false);
useEffect(() => {
  const t = setTimeout(() => setTimedOut(true), 3000);
  return () => clearTimeout(t);
}, []);

if (!loading && data.length === 0) return null;
if (timedOut && loading) return null;  // Give up after 3s
```

---

## Auth State and Loading Flashes

### The "Sign In to Post" Flash Problem
FanRink uses client-side auth (AuthContext). On page load:
1. `authLoading = true` for ~200-500ms
2. During this window, `isAuthenticated = false`
3. Components that check `isAuthenticated` show signed-out state briefly
4. This looks broken — user is actually logged in

### Fix Pattern — Always Use This
```tsx
const { user, loading: authLoading } = useAuth();

// Show nothing (or neutral state) while auth is loading
if (authLoading) {
  return <NeutralPlaceholder />; // No lock icon, no "sign in" text
}

// Only show gated state when confirmed not authenticated
if (!user) {
  return <SignInPrompt />;
}

// Show authenticated state
return <AuthenticatedComponent />;
```

### BottomTabBar Visibility
`BottomTabBar` returns null when `!authed || loading`.
This is correct — but ensure the page content doesn't jump when the bar appears.
Use the bottom padding on all pages regardless of auth state.

---

## Performance Rules That Affect Mobile UX

### Never Mount Heavy Components Simultaneously
On mobile CPUs (A15/A16 chips are fast, but mid-range Android is not), mounting 5+ heavy
components at once causes visible UI freeze — the thread blocks.

**Rule: maximum 2 heavy components mounting at the same time.**

Heavy components on FanRink:
- `LiveScoreTicker` — opens Supabase Realtime channel
- `LiveGameThreads` — opens Supabase Realtime channel
- `GamesToWatch` — multiple DB queries
- `SuggestedFollows` — DB queries
- `LockerRoomFeed` — RPC + multiple enrichment queries
- Any component that calls `supabase.channel()` on mount

### Lazy Loading Pattern — Use for Below-Fold Components
```tsx
import { lazy, Suspense } from 'react';

// Heavy component → lazy load
const LiveScoreTicker = lazy(() => import('@/components/games/LiveScoreTicker'));
const GamesToWatch = lazy(() => import('@/components/ice/GamesToWatch'));

// In JSX:
<Suspense fallback={null}>
  <LiveScoreTicker teamIds={teamIds} />
</Suspense>
```

### Realtime Channels — 1 Per Feature Maximum
Every `supabase.channel()` call establishes a WebSocket subscription.
On mobile, multiple open channels drain battery and cause connection pressure.
- Bottom tab bar: 1 channel for notifications
- Live game thread: 1 channel per active game
- Do not open channels in components that mount on every page load
- Always clean up channels in useEffect cleanup

---

## Navigation Patterns

### Bottom Tab Bar — The Primary Navigation on Mobile
FanRink uses a fixed bottom tab bar (`BottomTabBar.tsx`) on mobile.
Desktop uses a top horizontal nav bar.

**Navigation hierarchy:**
```
Level 0: Bottom tab bar — The Ice, Fanzone, [+FAB], Alerts, Profile
Level 1: Sub-tabs within a section — e.g. Open Ice | Games | My Rink pills inside The Ice
Level 2: Individual pages/modals — team page, game detail, settings
```

**What goes in the bottom bar:** Primary destinations only. Features users return to every session.
**What does NOT go in the bottom bar:** Secondary features (Locker Rooms, Search, Settings)

### In-Section Tab Pills
For secondary navigation within a section (e.g. Open Ice / Games / My Rink):
```tsx
// Pills must have:
- Full-width scrollable container: overflow-x-auto scrollbar-hide
- Pill padding: px-4 py-2 min-h-[36px]
- Active pill: bg-fr-ice text-black (high contrast)
- Inactive pill: bg-white/10 text-white/70
- Font: text-sm font-medium
```

### The FAB (Floating Action Button)
FAB sits in the centre slot of the bottom bar, raised above it.
```tsx
className="absolute left-1/2 -translate-x-1/2 -top-5
  h-14 w-14 rounded-full bg-fr-ice text-frost-black
  shadow-lg shadow-fr-ice/30 ring-2 ring-frost-black/50
  active:scale-95 transition-transform z-20"
```
FAB must always be 56px (w-14 h-14). Never smaller — it's the primary create action.

---

## Forms and Inputs on Mobile

### Input Field Standards
```tsx
// Text input — always at least 48px tall
className="w-full h-12 px-4 rounded-xl bg-white/10 border border-white/20
  text-white placeholder:text-white/40 text-sm
  focus:outline-none focus:border-fr-ice/50"

// Never use height < 44px for any input
```

### Keyboard Avoidance
iOS Safari lifts the viewport when the keyboard opens.
Inputs near the bottom of the screen get pushed up awkwardly.
- Place important inputs in the upper half of screens
- For bottom-of-screen inputs (e.g. comment box), use `position: fixed` + safe-area padding
- Test with keyboard open — does the input remain visible?

### Preventing Zoom on Input Focus (iOS Safari)
iOS Safari auto-zooms if input font-size < 16px.
```tsx
// Always use text-base (16px) or add this meta tag approach:
// In the input itself:
className="text-base"  // not text-sm for inputs
// OR handle globally in viewport meta:
// maximum-scale=1 (but this disables user zoom — use sparingly)
```

---

## Scroll and Gesture Standards

### Horizontal Scroll Rails (e.g. ClipsRail, FollowChips)
```tsx
// Standard horizontal scroll rail pattern:
className="flex gap-3 overflow-x-auto pb-2 -mx-4 px-4 scrollbar-hide"
// -mx-4 px-4 = bleed to screen edge, content starts at padding
// scrollbar-hide = no ugly scrollbar on iOS/Android
// pb-2 = space for shadow/content below scroll area
```

### Pull to Refresh
Do not use custom pull-to-refresh on mobile — iOS Safari has native pull-to-refresh.
Conflicting implementations cause gesture wars.
Current FanRink implementation hides PullToRefreshPuck on mobile (`hidden sm:block`) — correct.

### Preventing Overscroll
For modals and sheets that should not trigger background scroll:
```tsx
// On the overlay/backdrop:
onTouchMove={(e) => e.preventDefault()}
```

---

## Modal and Sheet Patterns on Mobile

### Bottom Sheet (preferred for mobile)
For actions, confirmations, and overflow menus:
- Use a bottom sheet, not a centred modal — bottom sheets feel native on mobile
- Sheet handle: `w-10 h-1 bg-white/30 rounded-full mx-auto mb-4`
- Sheet max-height: `max-h-[85vh]` — never full height (leaves context visible)
- Backdrop: semi-transparent dark overlay, tappable to close
- Animate from bottom: `translate-y-full` → `translate-y-0`

### Centred Modal (desktop-preferred)
Use centred modals for:
- Complex forms (image editors, post composer on desktop)
- Confirmations with significant consequences
On mobile, centred modals should become bottom sheets below `sm:` breakpoint.

### Image Crop Modal (FanRink specific)
`src/components/ui/ImageCropModal.tsx` — canvas-based editor with Crop/Adjust/Filter tabs.
On mobile: renders as bottom sheet. On desktop: centred modal.
Always test with EXIF-rotated iPhone photos (portrait images appear sideways without correction).

---

## Empty States — Mobile UX Best Practice

### Empty State Requirements
Every section that can be empty MUST have an empty state.
Empty states on mobile must:
1. **Explain what's missing** — "Your rink is empty"
2. **Explain why** — "Follow teams and fans to see their content here"
3. **Provide a clear CTA** — a button that solves the problem
4. **Use an icon/illustration** — visual anchor, not just text

### Empty State Structure
```tsx
<div className="rounded-2xl border border-white/10 bg-white/5 px-6 py-10 text-center">
  <div className="mx-auto mb-4 flex h-16 w-16 items-center justify-center rounded-full bg-fr-ice/15">
    <SomeIcon className="h-8 w-8 text-fr-ice" />
  </div>
  <h3 className="mb-2 text-lg font-semibold text-white">Short clear title</h3>
  <p className="text-sm text-white/60 max-w-sm mx-auto mb-6">
    One or two sentences max. Tell them what to do.
  </p>
  <Link href="/somewhere" className="inline-flex items-center gap-2 px-5 py-2.5
    bg-fr-ice text-black font-semibold rounded-xl text-sm min-h-[44px]">
    Action label
  </Link>
</div>
```

---

## Colour and Contrast Standards

### FanRink Colour System
```
bg-frost-black     = #0A0E14  (primary background)
bg-arctic-navy     = #0D1B2A  (secondary background)
bg-fr-dark         = #111827  (card background)
text-fr-ice        = #7FD7FF  (brand accent — ice blue)
text-fr-white      = #F5F9FF  (primary text)
text-white/70      = secondary text
text-white/50      = tertiary/placeholder
text-white/40      = inactive nav items, disabled state
text-white/30      = very subtle hints
```

### Contrast Rules for Mobile
Dark mode only — FanRink is a dark app.
Minimum contrast ratio: 4.5:1 for normal text, 3:1 for large text (WCAG AA).
Do not use `text-white/20` or lower for any readable text.
Never put `text-white/40` text on a `bg-white/10` background — too low contrast.

### Border Visibility
`border-white/10` — very subtle, card separation on dark BG ✅
`border-white/20` — input fields, interactive elements ✅
`border-fr-ice/30` — hover states, active focus ✅
`border-white/5` — barely visible dividers ✅

---

## Safari and Chrome Mobile Gotchas

### iOS Safari Specific Issues

**1. 100vh includes the address bar height**
Use `100dvh` (dynamic viewport height) instead of `100vh` on iOS Safari 16+
Or use `min-h-screen` with Tailwind (handled correctly on mobile).

**2. Input zoom (already covered in Forms section)**
Font-size < 16px triggers auto-zoom. Always use `text-base` on inputs.

**3. Position: fixed and keyboard**
iOS Safari moves fixed elements when the keyboard opens.
Test any fixed element (bottom bar, FAB, sticky inputs) with keyboard open.

**4. Backdrop blur support**
`backdrop-blur` works on iOS Safari 14+ — safe to use.
The FanRink frosted glass nav style (`backdrop-blur-xl`) is supported.

**5. Smooth scrolling**
iOS Safari supports smooth scrolling natively.
Do not use JS-based scroll animation — conflicts with native scroll inertia.

**6. WebKit touch callout**
Long-press on images triggers iOS image save dialog.
For cards with images: `style={{ WebkitTouchCallout: 'none' }}` if needed.

### Chrome Android Specific Issues

**1. Address bar resize**
Chrome on Android resizes the viewport when the address bar hides/shows on scroll.
This causes layout jumps. Use `dvh` units or be careful with full-height layouts.

**2. Tap highlight**
Chrome shows a blue/grey tap highlight by default.
```css
-webkit-tap-highlight-color: transparent;
```
FanRink likely has this globally — verify in global CSS.

**3. Overscroll bounce**
Chrome Android does not have iOS rubber-band overscroll.
Do not rely on overscroll effects for UX.

---

## Component-Specific Standards

### Feed Cards (Post.tsx)
- Min height: no minimum — let content breathe
- Horizontal padding: `px-4` inside card
- Avatar size: 40px (`w-10 h-10`) on mobile
- Username: `text-sm font-medium`
- Post content: `text-sm leading-relaxed`
- Reaction bar: min-h-[44px] for tappable area

### Team/Profile Headers
- Banner: aspect ratio 3:1 on mobile (e.g. `aspect-[3/1]`)
- Avatar overlapping banner: `-mt-10` with `border-4 border-frost-black`
- Username: `text-xl font-bold` (minimum)

### Game Cards
- Score text: `text-2xl font-bold` — big and readable at a glance
- Team names: `text-sm font-medium`
- Status pill (LIVE/FINAL): clear contrast, `text-xs font-semibold`

### SearchBar
- Height: min 44px
- Icon: left-side search icon inside input, not outside
- Placeholder: `text-white/40`
- Background: `bg-white/10 border border-white/20`

---

## Pre-Ship Mobile Checklist

Before any PR that touches UI components, verify:

### Layout
- [ ] No content hidden behind bottom tab bar (bottom padding set)
- [ ] No content hidden behind iPhone Dynamic Island (top safe area)
- [ ] Page scrolls correctly on iPhone SE (375px) — most constrained target
- [ ] No horizontal overflow (content doesn't cause left-right scroll)

### Touch
- [ ] All tappable elements are ≥ 44px in both dimensions
- [ ] Bottom nav tabs are ≥ 52px tall
- [ ] No two tap targets closer than 8px to each other
- [ ] FAB remains 56px (w-14 h-14) — never shrink it

### Text
- [ ] No text smaller than 11px anywhere
- [ ] Input fields use text-base (16px) to prevent iOS zoom
- [ ] Contrast ratio passes 4.5:1 for all body text
- [ ] No text touching screen edges (px-4 minimum)

### Loading States
- [ ] Skeleton states resolve or time out within 3 seconds
- [ ] Sections with no data return null (no ghost sections)
- [ ] Auth loading state handled — no false "sign in" flash

### Performance
- [ ] No more than 2 heavy components mounting simultaneously
- [ ] Below-fold components are lazy-loaded
- [ ] Realtime channels cleaned up in useEffect return
- [ ] No infinite loops in useEffect dependencies

### Safari/Chrome
- [ ] Tested in iOS Safari (not just Chrome — they behave differently)
- [ ] Keyboard opens without breaking layout of focused input
- [ ] No position: fixed elements broken by keyboard
- [ ] Bottom bar still visible/correct with iPhone address bar showing

---

## FanRink-Specific Patterns to Always Follow

### 1. The Ice Layout Pattern
The Ice section uses IceChrome → SecondaryBar → TabPills on mobile.
Always ensure new Ice sub-pages are registered in `ICE_TABS` in IceChrome.tsx.
New Ice sub-pages should NOT show the MobileHeader unless they are deep sub-pages
(team detail, league detail) — main Ice pages start directly with content.

### 2. Route Consistency
Mobile nav items in `navigation.ts` MUST point to routes inside `/ice` where possible.
Navigating outside `/ice` from the bottom bar causes a full AppShell re-render.
This is jarring on mobile and can feel like a crash.
- ✅ `/ice/my-rink` (stays in Ice layout, feels smooth)
- ❌ `/locker-rooms` (different layout, full re-render, feels like a crash)

### 3. Auth Guard Pattern in Ice Layout
The `/ice` route group uses `RequireAuth` + `IceChrome`.
Never duplicate auth checking inside individual Ice page components.
Relying on the layout-level auth guard is correct.

### 4. CreateFab Visibility
`CreateFab` is hidden on Ice pages (`!isIcePage` in AppShell).
Ice pages use the centre FAB in BottomTabBar instead.
Do not show both — it creates two create buttons on screen.

### 5. Notification Badge Clear Pattern
The Alerts/notifications badge in BottomTabBar MUST reset when `/notifications` is visited:
```tsx
useEffect(() => {
  if (pathname === '/notifications') {
    const t = setTimeout(() => setUnreadNotifs(0), 1000);
    return () => clearTimeout(t);
  }
}, [pathname]);
```

---

## What Good Mobile UX Looks Like on FanRink

The reference bar is **Instagram Stories + Twitter/X feed + ESPN scores app** — combined.
- Content is immediate — no blank states, no loading confusion
- Every tap does something obvious
- Text is large enough to read without glasses
- The feed feels fast even on a 3-year-old iPhone
- Navigating around feels like a native app, not a website
- Nothing looks squashed or truncated
- The bar at the bottom always works, always responds immediately

If any component makes the app feel like a website instead of an app on mobile — fix it.
