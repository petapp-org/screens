# Screen 30: Shops

## Overview

The **"Shops" tab** — third tab in the bottom navigation. A location-aware marketplace hub for pet shops. **This phase ships a placeholder only**: the feature is **"Coming Soon"** in every city; the screen renders the shared root-tab header, a city location bar, and a Coming-Soon body.

Requires login. If not logged in → tapping the Shops tab redirects to Login (same rule as My Pets / Services / More — see `screen_1_home_explore.md` → Bottom Navigation auth rules).

Like the **More** tab, the screen is scoped to a **selected city** shown in the location bar; on first open the app auto-detects the nearest city and the user can change it. Because the whole feature is Coming Soon, the body shows the same "not available" state regardless of which city is selected (Shops availability is independent of a city's More-status this phase).

---

## UI Layout

```
[Header]
  Title: "Shops"
  Right actions: [Search icon] | [Messages icon ✉] | [Notifications icon 🔔] | [Profile avatar]

[Location bar]
  [📍 icon]  Hà Nội, VN                                 [Change]

[Coming Soon body — fills the rest of the screen]
            ( 📍 pin glyph in a soft circle )

                     Coming Soon!
        We are currently not available in this region.

                 [ Change Location ]

[Bottom Navigation]
  My Pets | Explore | Shops (active) | Services | More
```

---

## Components

### 1. Header

**Identical to the My Pets header (`screen_8`)** — the canonical root-tab header: Search icon · Messages icon `✉` (red dot when any unread chat, opens `screen_10`) · Notifications icon `🔔` (red dot when any unread activity, opens `screen_22`) · Profile avatar. No back button (it's a root tab).

> The mockup image shows a single chat icon (pre-dating the Messages/Notifications split); this screen uses the **canonical 4-icon header** to stay consistent with Explore / My Pets / More.

---

### 2. Location Bar

**Identical to the More tab's location bar** (`screen_17` → Location Bar): shows the **selected city** as `"{cityShortName}, {countryCode}"` (e.g. `"Hà Nội, VN"`) with a **[Change]** button that opens the **Choose Your City** bottom sheet. Auto-detection (nearest city on first open), client-side persistence, and the picker all follow `screen_17` exactly.

- Data source: `Cities query (CA)` (`screen_17`) — the same cached list powers the picker and nearest detection.
- **No coming-soon city lock distinction here** — every city renders the Coming-Soon body this phase (the feature itself isn't live anywhere), so the picker is purely for showing/changing the city label.

---

### 3. Coming Soon Body

Fills the area below the location bar:

| Element | Content |
|---------|---------|
| Icon | A muted location-pin glyph in a soft circle |
| Title | **"Coming Soon!"** |
| Subtitle | *"We are currently not available in this region."* |
| **[Change Location]** button | Opens the same **Choose Your City** bottom sheet as the location bar's **[Change]** |

- The body is **static** this phase — picking another city changes the location-bar label but the body stays Coming Soon (no Shops data anywhere yet).

---

## API Endpoints Required

> All calls go to `POST /graphql`. **Auth:** Required (the Shops tab requires login).
> **No new query** — reuses `Cities (CA)` (`screen_17`) for the location bar / Choose Your City picker. There is **no Shops-listing query** this phase (the feature is a placeholder).

---

## User Flow Diagrams

### Open Shops tab

```
User taps Shops tab
  ├─ [not logged in] → redirect to Login
  └─ [logged in]
        └─> resolve selected city (auto-detect nearest / stored — same as screen_17)
              └─> render header + location bar + Coming-Soon body
```

### Change location

```
Tap [Change] (location bar)  OR  [Change Location] (body)
  └─> Choose Your City sheet (Cities CA, cached)
        └─ pick a city → update location-bar label (body stays Coming Soon)
```

---

## Edge Cases & Notes

| Case | Expected Behaviour |
|------|--------------------|
| Not logged in | Shops tab redirects to Login |
| Location permission denied / unavailable | Default to a city (same fallback as `screen_17`); body still Coming Soon |
| Any city selected | Coming-Soon body shown (feature not live anywhere this phase) |
| City list fetch fails | Fall back to default city label; Coming-Soon body unaffected |

---

## Decisions Log

| # | Question | Decision |
|---|----------|----------|
| 1 | Scope this phase | **Placeholder only** — Coming Soon in every city; no Shops data/listing query |
| 2 | Header | Reuse the canonical root-tab header (`screen_8`); 4-icon (mockup's single chat icon predates the Messages/Notifications split) |
| 3 | Location bar | Reuse the More tab's location bar + Choose Your City + `Cities (CA)` (`screen_17`) — referenced, not redefined |
| 4 | City vs body | Body is Coming Soon regardless of selected city; the picker only changes the label |
| 5 | Auth | Requires login (bottom-nav rule, `screen_1`) |

---

## Open Items (next steps)

- **Shops feature** (listings / shop detail / per-city availability) — future phase; this screen is the placeholder until then.
