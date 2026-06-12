# Screen 28: Map View (shared full-screen map)

## Overview

A **single shared full-screen map** for all four More categories. It is the `[Open map]` destination from the Lost Pets (`screen_18`), Pet Friendly (`screen_20`), Events (`screen_23`), and Rescue (`screen_25`) screens.

The screen is **parameterised by category** — it doesn't mix categories. Whatever opened it determines the data query, the filter chips, the pin style, the mini-card, and the detail screen a card opens.

```
params: { category, cityCode, countryCode, filter, sort?, originLat?, originLng? }
```

| `category` | Pins from | Filter chips (mirror source) | Mini-card | Card tap → detail |
|------------|-----------|------------------------------|-----------|-------------------|
| `LOST_PETS` | `LostPets (CB)` | All / 1km / 5km / Cats / Dogs / Other (`screen_18`) | Lost Pet card | `screen_19` |
| `PET_FRIENDLY` | `PetFriendlyPlaces (CD)` | All / Cat / Dog / Cafés / … (`screen_20`) | Pet Friendly card | `screen_21` |
| `EVENTS` | `Events (CI)` | All / This week / This weekend / Free (`screen_23`) | Event card | `screen_24` |
| `RESCUE` | `Rescues (CL)` | All / Cats / Dogs / Other + Nearest/Newest (`screen_25`) | Rescue card | `screen_26` |

City-scoped like the list it came from — it loads the **same city set** under the current filter (no "search this area" re-query this phase; decision). Requires login. Keeps the More tab's context (Back returns to the source list, preserving filter).

---

## UI Layout

```
[Header]
  ← Map View

[Filter chips — overlay, top]                  ← same chips as the source list screen
  [All] [Cats] [Dogs] [Other]

[Map — full bleed]
                          🖤                    ⊕   ← recenter (GPS) button, top-right
              ❤️(selected)        🖤
                   🖤      🔵12 (cluster)
        (pins themed per category — see Pins)

[Card carousel — bottom, horizontal scroll, next card peeking]
  ┌──────────────────────┐ ┌────────────────────
  │ [img] Saigon Pet…    │ │ [img] Miu's Famil…
  │ ★4.9 · HCMC · Open   │ │ ...
  └──────────────────────┘ └────────────────────
```

---

## Components

### 1. Header

Back button + title **"Map View"**. Back → returns to the source list screen with its filter intact.

---

### 2. Filter Chips (overlay)

- The **same chip set** as the source list screen (per the table above), rendered as a floating row over the top of the map.
- **Single shared filter state** with the source list: changing a chip here re-queries the pins **and** is applied to the list when the user goes Back (and vice-versa) — no need to re-set the filter.
- Re-running the query refreshes pins + cards; resets the selected card to the first.

> Rescue also carries its **Nearest / Newest** sort control here (same as `screen_25`).

---

### 3. Map

Full-bleed map showing every item in the city under the active filter.

**Pins — themed per category** (so the map reads at a glance):

| Category | Pin |
|----------|-----|
| Lost Pets | the **pet's avatar** inside the pin (with a small "Missing" tint) |
| Pet Friendly | a **category glyph** (`☕` cafe · `🍽` restaurant · `🏨` hotel · `🌳` park · `🏖` beach · `📍` other) |
| Events | a **date badge** (`15` / `JUN`) or `🗓️` |
| Rescue | the **pet's photo** (`photos[0]`) inside the pin |

- **Selected pin** → brand colour (red) + enlarged + raised; all others neutral/dark (matches the mock).
- **Clustering:** when pins are dense, collapse into a numbered cluster (`🔵 12`); tap → zoom in to split.
- **Recenter button (`⊕`)** top-right → recenter on the user's GPS (disabled/greyed when GPS unavailable → stays on city centre).

**Centring on open:** GPS granted → user position; else the selected city's centre (`Cities (CA)` lat/lng).

---

### 4. Card Carousel (bottom) + pin sync

A horizontally-scrollable row of compact cards, the next one **peeking** at the edge (as in the mock). The carousel and the map are **two-way synced** (Airbnb/Google-Maps pattern):

```
Swipe carousel → next card          ↔  matching pin turns red + map pans to it
Tap a pin                           ↔  carousel scrolls to that card + pin turns red
Tap a card (body)                   →  open that item's Detail screen (per category)
```

**Mini-card per category** (compact form of the canonical row):

| Category | Card content |
|----------|--------------|
| Lost Pets | avatar · `{name} ({family})` · `{species}·{breed}` · **Missing** |
| Pet Friendly | photo · `{name}` · `★{ratingAvg}` · `{distanceKm}km` · open/closed (`highlightText`) |
| Events | date-chip · `{title}` · `{weekday}·{start}–{end}` · `{price}` |
| Rescue | photo · `{name}` · `{species}·{ageText}` · `{charity}` 🏷 |

- Only the data already returned by the category's list query is needed (all four return `lat`/`lng` + the row fields).

---

## API Endpoints Required

> All calls go to `POST /graphql`. **Auth:** Required.
> **No new query** — reuses the source category's list query (`LostPets CB` / `PetFriendlyPlaces CD` / `Events CI` / `Rescues CL`) with the **same filter** and a **high `first`** (or paged) to fetch all pins for the city. Each already returns `lat`/`lng` and the fields the mini-card needs.

> **Pins-only optimisation (future):** if a city ever returns a very large pin set, a lightweight `…Pins` query returning just `{ id, lat, lng, thumbnail }` could back the map, loading the full card on selection. Flagged in Open Items; not needed this phase.

---

## User Flow Diagrams

### Open the map

```
[Open map] on screen_18 / 20 / 23 / 25
  └─> Map View { category, cityCode, countryCode, filter, sort? }
        └─> reuse the category's list query (CB/CD/CI/CL) with current filter, high first
              ├─ items → themed pins + card carousel (first card selected)
              └─ empty → "No {category} in this area" overlay
```

### Pin ↔ card

```
Swipe carousel → select card N        → pin N → red, map pans to pin N
Tap pin N                              → carousel scrolls to card N, pin N → red
Tap card body                          → category Detail (19 / 21 / 24 / 26)
Tap cluster                            → zoom in to split
Tap ⊕ recenter                         → recenter on user GPS
```

### Filter

```
Tap a chip / change sort (Rescue)
  └─> re-query (CB/CD/CI/CL) with new filter → refresh pins + cards (reset selection)
        └─> shared with the source list (applied on Back)
```

### Back

```
← Back → source list screen, filter preserved
```

---

## Edge Cases & Notes

| Case | Expected Behaviour |
|------|--------------------|
| Not logged in | Inherited from the source More screen → redirect to Login |
| GPS denied | Map centres on city centre; recenter `⊕` greyed/disabled |
| GPS granted | Centre on user; recenter works |
| 0 items under filter | "No {category} in this area" overlay; empty carousel |
| 1 item | Single pin auto-selected; one card |
| Dense pins | Cluster with count; tap to zoom/split |
| Filter changed on map | Pins + cards refresh; selection resets to first; applied to source list on Back |
| Item ends/closes while open (event ended / pet found / rescue adopted) | Drops on next refresh (same `…active/open only` rules as the lists) |
| Category has its own sort (Rescue) | Nearest/Newest control shown; reorders the carousel, not the pins |
| Pin avatar/photo missing | Fall back to a generic category pin |

---

## Decisions Log

| # | Question | Decision |
|---|----------|----------|
| 1 | One screen or four | **One shared, parameterised** Map View for all four categories (`category` param drives query/chips/pins/card/detail) |
| 2 | Pin style | **Themed per category** — pet avatar (Lost Pets / Rescue), category glyph (Pet Friendly), date badge (Events); selected pin red + enlarged |
| 3 | Pin ↔ card | **Two-way synced** carousel (swipe card pans to pin; tap pin scrolls to card); tap card → detail |
| 4 | "Search this area" | **Not this phase** — loads the same city set as the list under the current filter (no pan re-query) |
| 5 | Filter state | **Shared** with the source list (two-way); chips mirror the source screen |
| 6 | Data | **Reuse** the category list query (`CB/CD/CI/CL`) with a high `first`; no new endpoint |
| 7 | Map/List toggle | None — Back returns to the list |
| 8 | Clustering + recenter | Cluster dense pins; `⊕` recenter to GPS (city centre fallback) |

---

## Open Items (next steps)

- **Pins-only lightweight query** — optimisation if a city's pin volume grows large (load full card on selection).
- **"Search this area" (pan re-query)** — deferred; could be added if users want cross-city/area browsing on the map.
