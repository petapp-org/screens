# Screen 20: Pet Friendly

## Overview

Full list of **pet-friendly places in the selected city**, with a map preview and filter chips, sorted by distance from the user. This is the "View All" destination of the Pet Friendly section on the **More** tab (`screen_17`).

Navigated to from:
- More tab → **Pet Friendly** category icon, or
- More tab → Pet Friendly section → **"View All →"**.

Both pass the **selected city** from the More tab's location bar. Like the section, the list is **city-scoped** and **distance-sorted** (a place outside the city is not shown; the mockup's "Hồ Tràm · 120km" is demo data only).

Requires login. Keeps the bottom navigation (More tab stays active) — it's a sub-screen of More.

---

## UI Layout

```
[Header]
  Left: Back button
  Center: "Pet Friendly"

[Filter chips — horizontal scroll]
  [All]  [Cat-friendly]  [Cafés]  [Restaurants]  [Hotels]  …

[Map preview]
  ┌──────────────────────────────────────────────┐
  │  (map with pins at each place)                │
  │                                  [Open map]   │
  │  ┌──────────────────────┐                     │
  │  │ 📍 12 places near you │                     │
  │  └──────────────────────┘                     │
  └──────────────────────────────────────────────┘

[List — pet-friendly rows, infinite scroll]
  ┌────────────────────────────────────────────────────────┐
  │ [photo]   Lava Cat Coffee  ★4.8                3.4km    │
  │           Cat · Carriers OK                             │
  │           HCMC, VN · open until 22:00                   │
  ├────────────────────────────────────────────────────────┤
  │ [photo]   Ailu Cat Café  ★4.7                  4.2km    │
  │           Cat · Resident cats                           │
  │           HCMC, VN · play with 12 cats                  │
  └────────────────────────────────────────────────────────┘

[Bottom Navigation]
  My Pets | Explore | Shops | Services | More (active)
```

---

## Components

### 1. Filter Chips

Horizontal scrollable row. **Single-select** — exactly one chip active at a time (default **All**). Tapping a chip re-runs `PetFriendlyPlaces (CD)` with the matching filter and resets pagination.

| Chip | Filter sent | Dimension |
|------|-------------|-----------|
| **All** | *(none)* | — (default) |
| **Cat-friendly** | `{ suitableFor: "CAT" }` | suitability |
| **Dog-friendly** | `{ suitableFor: "DOG" }` | suitability |
| **Cafés** | `{ category: "CAFE" }` | category |
| **Restaurants** | `{ category: "RESTAURANT" }` | category |
| **Hotels** | `{ category: "HOTEL" }` | category |
| **Parks** | `{ category: "PARK" }` | category |
| **Beaches** | `{ category: "BEACH" }` | category |

> Chip set is driven by the place taxonomy (`category` + `suitableFor`). Single-select keeps it simple this phase; combining dimensions can come later.

---

### 2. Map Preview

A compact map showing each place in the selected city (after the active filter).

| Element | Description |
|---------|-------------|
| Pins | One pin per place, at `lat` / `lng` |
| `[Open map]` button | Top-right — opens the shared **Map View** (`screen_28`) of the same pins (clustered when dense) |
| "N places near you" chip | Bottom-left — count of places currently plotted |

**Centring:** GPS granted → centre on the user; GPS unavailable → centre on the selected city's centre (`Cities (CA)` lat/lng).

**Pin interaction:** tap a pin → mini card (thumbnail + name + ★rating + distance) → tap card → **Pet Friendly Place Detail** (`screen_21`).

**Full-screen map (`[Open map]`):** opens the shared **Map View** (`screen_28`) with `category: PET_FRIENDLY`, the current city + filter, and the same pin set (themed pins + synced card carousel). Data identical to this screen.

---

### 3. Places List

Full, paginated list, sorted by **distance asc**.

**Row** — the canonical **Pet Friendly item** defined in `screen_17` → Section 6:

```
[photo]   {name}  ★{ratingAvg}                 {distanceKm}km
          {suitable} · {tagText}
          {cityShortName}, {countryCode} · {highlightText}
```

- `suitable`: `CAT`→"Cat", `DOG`→"Dog", `OTHER`→"Pet friendly"; multiple joined by ` · `.
- `distanceKm`: `< 10` → 1 decimal (`3.4km`); `≥ 10` → integer (`120km`).
- `tagText` ≤ 18, `highlightText` ≤ 20; single line, `…` on overflow.
- **Pagination:** infinite scroll via `PetFriendlyPlaces (CD)` `cursor`.
- **Tap a row** → **Pet Friendly Place Detail** (`screen_21`).
- **Empty state** (filter matches nothing): *"No pet-friendly places match this filter."* (map + chips stay visible).

---

## API Endpoints Required

> All calls go to `POST /graphql`. **Auth:** Required.
> **No new query for this screen** — reuses `PetFriendlyPlaces (CD)` + `Cities (CA)` from `screen_17` (list + map pins + filter + pagination). The place detail and reviews live in `screen_21`.

---

## User Flow Diagrams

### Enter Pet Friendly

```
More tab → "Pet Friendly" icon  OR  "View All →"
  └─> Pet Friendly screen (selected city passed in)
        └─> PetFriendlyPlaces (CD) { cityCode, countryCode, originLat?, originLng?, [filter] }
              ├─ items → list (distance asc) + map pins
              └─ empty → empty state
```

### Filter

```
Tap a chip
  └─> PetFriendlyPlaces (CD) with that filter (single-select) → reset pagination → re-render list + map
  └─ "All" → no filter
```

### Open a place

```
Tap a list row  OR  a map pin's card
  └─> Pet Friendly Place Detail (screen_21) { placeId }
```

---

## Edge Cases & Notes

| Case | Expected Behaviour |
|------|--------------------|
| Not logged in | More tab (and this screen) redirect to Login |
| Selected city has 0 places | List empty state; map shows "0 places near you" |
| GPS denied | Distance computed from city centre (still shown); map centres on city |
| GPS granted | Distance from user; map centres on user |
| Filter matches nothing | "No pet-friendly places match this filter."; map + chips remain |
| Distance ≥ 10km | Integer format (`120km`); `< 10km` → 1 decimal (`3.4km`) |
| Place unpublished | Excluded from results (`isPublished = false`) |
| Many places in a city | Map clusters; list paginates; optional pins-only query (Open Items) |

---

## Decisions Log

| # | Question | Decision |
|---|----------|----------|
| 1 | Scope | City-scoped (selected city), distance-sorted; outside-city places not shown (Hồ Tràm 120km = demo) |
| 2 | Distance origin | User GPS if available, else selected city centre |
| 3 | Distance format | `< 10` → 1 decimal; `≥ 10` → integer |
| 4 | Filter chips | Single-select; suitability (`Cat/Dog-friendly`) + category (`Cafés/Restaurants/Hotels/…`); `All` resets |
| 5 | Row layout | Canonical Pet Friendly item (`screen_17` §6): name + ★rating + distance / suitable · tagText / city-country · highlightText |

---

## Open Items (next steps)

- **Full-screen map** (`[Open map]`) — done: shared **Map View** (`screen_28`). Optional pins-only query if volume grows.
- **Combined filters** (suitability + category together) — possible later enhancement.
