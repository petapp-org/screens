# Screen 18: Lost Pets

## Overview

Full list of missing pets **in the selected city**, with a map preview and filter chips. This is the "View All" destination of the Lost Pets section on the **More** tab (`screen_17`).

Navigated to from:
- More tab → **Lost Pets** category icon, or
- More tab → Lost Pets section → **"View All →"**.

Both pass the **selected city** from the More tab's location bar. The screen is always scoped to that city (decision: city-scoped — see `screen_17` Decisions Log #5).

Requires login. Keeps the bottom navigation (More tab stays active) — it's a sub-screen of More.

---

## UI Layout

```
[Header]
  Left: Back button
  Center: "Lost Pets"

[Banner — "Have a missing pet?"]
  Have a missing pet?
  Report it so people nearby can help find them.        [Report]

[Map preview]
  ┌──────────────────────────────────────────────┐
  │  (map with pins at each report's last-seen)   │
  │                                  [Open map]   │
  │  ┌─────────────────┐                          │
  │  │ N missing nearby │                         │
  │  └─────────────────┘                          │
  └──────────────────────────────────────────────┘

[Filter chips — horizontal scroll]
  [All]  [Within 1km]  [5km]  [Cats]  [Dogs]  [Other]

[List — lost pet rows, infinite scroll]
  ┌────────────────────────────────────────────────────────┐
  │ [avatar]   Măng  (Minh's Family)                        │
  │ [Missing]  Pony · buckskin                              │
  │            Đà Lạt, VN                                   │
  ├────────────────────────────────────────────────────────┤
  │ [avatar]   Pudding  (Pudding's Family)                  │
  │ ...                                                     │
  └────────────────────────────────────────────────────────┘

[Bottom Navigation]
  My Pets | Explore | Shops | Services | More (active)
```

---

## Components

### 1. "Have a missing pet?" Banner

```
Have a missing pet?
Report it so people nearby can help find them.        [Report]
```

- **[Report] button** → opens the **Report Missing form** (the shared form defined in `screen_9` → Section 8), in **pet-selector mode**: a pet selector is shown at the top because there is no single pet in context here.
- This replaces the old mockup copy *"Report from your pet's page on My Pets."* — reporting now happens **inline** from here (decision: go straight to the form), not by sending the user away to My Pets.

**Pet selector (top of the form):**

```
Which pet is missing?
  [avatar]  Bụi   🔒
  [avatar]  Mèo
  [avatar]  Lúa
```

- Lists the pets of the user's **active family** only (the family currently active in Profile Settings, `screen_5`). Reporting is scoped to the active family — to report a pet in another family, switch the active family first.
- **Includes private pets** (`is_public = false`) — reporting a pet as missing **ignores the pet's privacy setting** (a private pet can still be reported and surfaced publicly in Lost Pets). Show the 🔒 marker for awareness, but it is selectable.
- **Excludes**: soft-deleted pets (`is_deleted = true`) and pets already missing (`missing_status != null`).
- Each item shows pet avatar + name.
- After picking a pet, the rest of the form is the **upgraded Report Missing form** — see `screen_9` Section 8.
- Data source: `FamilyPets query (BB)` for the active family; the client filters out already-missing pets.

**No reportable pet:**
- If the user has **no active family**, or the active family has no eligible pet, tapping Report shows a prompt: *"You don't have a pet to report yet."* with a link to set/create an active family (Profile Settings → Family Pages, `screen_5`).

> The Report button / banner is shown to **all logged-in users** (the screen itself requires login). A user with no active family simply hits this prompt.

---

### 2. Map Preview

A compact map showing the last-seen location of each missing report in the selected city.

| Element | Description |
|---------|-------------|
| Pins | One pin per report, placed at `lastSeen.lat` / `lastSeen.lng` |
| `[Open map]` button | Top-right — opens a **full-screen map** view of the same pins (clustered when dense) |
| "N missing nearby" chip | Bottom-left — count of reports currently plotted |

**Centring:**
- GPS granted → centre on the user's real position; distance-based UI is meaningful.
- GPS unavailable → centre on the selected city's centre (`Cities (CA)` lat/lng).

**Pin interaction:**
- Tap a pin → mini card (pet avatar + name + family + last-seen) → tap card → **Lost Pet Detail** (`screen_19`).

**Full-screen map (`[Open map]`):** same pin set, pan/zoom, clustering; tapping a pin/cluster behaves the same. (Detailed map-screen interactions can be split into a follow-up; data is identical to this screen.)

**Map pin data:** reuse `LostPets (CB)` requesting the full set for the city (high `limit`, the list already returns `lastSeen.lat/lng`). If report volume per city grows large, switch the map to a lightweight pins-only query — flagged in Open Items.

---

### 3. Filter Chips

Horizontal scrollable chip row. Two independent dimensions; **distance** and **species** combine (e.g. `5km` + `Cats`).

| Chip | Dimension | Effect |
|------|-----------|--------|
| **All** | — | Clears both dimensions (default state) |
| **Within 1km** | Distance | `maxDistanceKm: 1` (mutually exclusive with `5km`) |
| **5km** | Distance | `maxDistanceKm: 5` (mutually exclusive with `Within 1km`) |
| **Cats** | Species | `species: [CAT]` (mutually exclusive with Dogs/Other) |
| **Dogs** | Species | `species: [DOG]` (mutually exclusive with Cats/Other) |
| **Other** | Species | `species: [OTHER]` — server sentinel for any species not CAT/DOG (mutually exclusive with Cats/Dogs) |

- **Distance chips require real GPS.** Without GPS (`distanceKm` is null because `originLat/Lng` weren't supplied), `Within 1km` and `5km` are **disabled** (greyed). Species chips and `All` remain available.
- Selecting a chip re-runs `LostPets (CB)` with the combined filter; resets pagination.
- `All` resets to default (no filters).

---

### 4. Lost Pets List

Full, paginated list of missing reports in the selected city (after filters).

**Row** — identical to the Lost Pet preview item in `screen_17` → Section 5:

| Field | Display |
|-------|---------|
| `pet.avatarUrl` | Pet avatar with **"Missing"** badge overlaid bottom-left |
| `pet.name` `(family.name)` | `"Măng (Minh's Family)"` |
| `pet.species` · `pet.breed` | `"Pony · buckskin"` (species only if breed is null) |
| `lastSeen.cityShortName`, `countryCode` | Muted: `"Đà Lạt, VN"` |

- **Sorting:** default `reportedAt` desc (newest first). When a **distance chip** is active, sort by `distanceKm` asc (nearest first).
- **Pagination:** infinite scroll via `LostPets (CB)` `cursor`.
- **Tap a row** → **Lost Pet Detail** (`screen_19`) for that `reportId`.
- **Empty state** (filters match nothing): *"No lost pets match your filters."* (the banner + map stay visible).

---

## API Endpoints Required

> All calls go to `POST /graphql`. **Auth:** Required.
> **No new query for this screen** — everything is reused:
> - `LostPets (CB)` + `Cities (CA)` from `screen_17` — list, map pins, filters, pagination.
> - `FamilyPets (BB)` from `screen_9` — the **active family's** pets for the report pet selector (includes private; client filters out already-missing pets).
> - `ReportMissing (BE)` from `screen_9` — submit the report (upgraded form).

---

## User Flow Diagrams

### Enter Lost Pets

```
More tab → "Lost Pets" icon  OR  "View All →"
  └─> Lost Pets screen (selected city passed in)
        ├─> LostPets (CB) { cityCode, countryCode, limit, [filter] }  → list
        └─> map pins from the same city data
              └─ GPS granted   → distance chips enabled, distanceKm populated
              └─ GPS unavailable → distance chips disabled
```

### Report from banner

```
User taps [Report]
  └─> active family?
        ├─ none → "no reportable pet" prompt → set/create active family
        └─ yes → FamilyPets (BB) { active familyId }, filter out already-missing
              ├─ no eligible pet → "no reportable pet" prompt
              └─ has pets → open Report Missing form (screen_9 §8) in pet-selector mode
                    └─> pick pet (incl. private) → fill the form → Submit
                          └─> ReportMissing (BE) → success → report appears in Lost Pets
```

### Open a report

```
User taps a list row  OR  a map pin's card
  └─> Lost Pet Detail (screen_19) { reportId }
```

---

## Edge Cases & Notes

| Case | Expected Behaviour |
|------|--------------------|
| Not logged in | More tab (and this screen) redirect to Login |
| Selected city has 0 reports | List empty state; map shows "0 missing nearby"; banner still shown |
| GPS denied | Distance chips (`1km`/`5km`) disabled; map centres on city centre |
| GPS granted | Distance chips enabled; `distanceKm` populated; distance sort available |
| No active family / active family has no eligible pet | Report → "no reportable pet" prompt to set/create an active family |
| Pet already missing | Excluded from the report selector (can't double-report) |
| Private pet | Appears in selector with 🔒; selectable; once reported, surfaces publicly in Lost Pets |
| Filters match nothing | "No lost pets match your filters."; banner + map remain |
| Distance + species chips | Combine (e.g. `5km` + `Cats`); within each dimension chips are mutually exclusive |
| Many reports in a city | Map clusters pins; list paginates; consider pins-only query (Open Items) |
| Pet marked **found** | Dropped from the list and the map (`LostPets (CB)` returns active reports only); its detail stays reachable via shared link → found state (`screen_19`) |

---

## Decisions Log

| # | Question | Decision |
|---|----------|----------|
| 1 | Report entry from Lost Pets | Banner → opens the shared Report Missing form inline (not a redirect to My Pets) |
| 2 | Pet selector scope | Pets of the user's **active family** only (switch active family to report another family's pet) |
| 3 | Private pets in report | Included — reporting ignores pet privacy; shown with 🔒 but selectable |
| 4 | Already-missing / deleted pets | Excluded from the selector |
| 5 | No reportable pet | Show prompt linking to set/create an active family (Profile Settings → Family Pages) |
| 6 | Filter dimensions | Distance (All/1km/5km) + Species (Cats/Dogs/Other) combine; mutually exclusive within each. `Other` → `species: [OTHER]` sentinel |
| 7 | Distance chips without GPS | Disabled |
| 8 | List sort | `reportedAt` desc by default; `distanceKm` asc when a distance chip is active |
| 9 | City scope | Inherited from More tab's selected city |

---

## Open Items (next steps)

- **Full-screen map** (`[Open map]`) — detailed pan/zoom/cluster interactions; optional pins-only query if volume grows.

> Done: Lost Pet Detail is specified in `screen_19`.
