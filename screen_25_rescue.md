# Screen 25: Rescue

## Overview

Full list of **open rescue listings in the selected city** — pets that **charity families** have posted for adoption — with a map preview and a species filter. This is the "View All" destination of the Rescue section on the **More** tab (`screen_17`).

Navigated to from:
- More tab → **Rescue** category icon, or
- More tab → Rescue section → **"View All →"**.

Both pass the **selected city** from the More tab's location bar. The list is **city-scoped** and shows only **open** listings (`status = OPEN`); adopted/closed listings drop out (like Lost Pets `found`).

> **Mirror of Lost Pets.** Lost Pets = a pet looking for its *old* home; Rescue = a pet looking for a *new* home. The two screens share the city scope, map, and detail/share patterns.

Requires login. Keeps the bottom navigation (More tab stays active) — it's a sub-screen of More.

---

## UI Layout

```
[Header]
  Left: Back button
  Center: "Rescue"

[Banner — "Have a pet that needs a home?"]   ← only when active family is a charity
  Have a pet that needs a home?
  Post it so people nearby can adopt.            [ Post a Rescue ]
                                          Manage your rescues →

[Filter + sort row]
  [All]  [Cats]  [Dogs]  [Other]                 Sort: [ Nearest ▼ ]

[Map preview]
  ┌──────────────────────────────────────────────┐
  │  (map with pins at each listing's location)   │
  │                                  [Open map]   │
  │  ┌──────────────────────┐                     │
  │  │ 🐾 9 rescues near you │                     │
  │  └──────────────────────┘                     │
  └──────────────────────────────────────────────┘

[List — rescue rows, infinite scroll]
  ┌────────────────────────────────────────────────────────┐
  │ [photo]   Miu                                  2.3km    │
  │           Cat · Domestic shorthair · ~4mo · ♀          │
  │           Paws Rescue Saigon  🏷CHARITY                 │
  ├────────────────────────────────────────────────────────┤
  │ [photo]   Lucky                                3.1km    │
  │           Dog · Mixed · ~1yr · ♂                       │
  │           HCMC Animal Rescue  🏷CHARITY                 │
  └────────────────────────────────────────────────────────┘

[Bottom Navigation]
  My Pets | Explore | Shops | Services | More (active)
```

---

## Components

### 1. "Have a pet that needs a home?" Banner

```
Have a pet that needs a home?
Post it so people nearby can adopt.            [ Post a Rescue ]
```

- Shown **only when the user's active family is a charity** (`family.type = charity`). Non-charity active family (or no active family) → banner hidden.
- **[Post a Rescue]** → opens the **Create Rescue form** (`screen_26` → Section "Create Rescue Form").
- **"Manage your rescues →"** secondary link → opens **Manage Rescues** (`screen_27`) — the charity's Open/Adopted listings. (Same destination as the My Pets "Rescues" row, `screen_8`.)
- A charity member whose charity is **not** currently active won't see the banner here — they post/manage from **Manage Rescues** (`screen_27`) after switching their active family to the charity. (Mirrors the Lost Pets active-family rule.)

---

### 2. Filter + Sort Row

**Species filter chips** — single-select, default **All**:

| Chip | Filter sent |
|------|-------------|
| **All** | *(none)* |
| **Cats** | `{ species: "CAT" }` |
| **Dogs** | `{ species: "DOG" }` |
| **Other** | `{ species: "OTHER" }` — any species not CAT/DOG |

**Sort control** — toggle between:

| Sort | Effect |
|------|--------|
| **Nearest** (default) | `sort: NEAREST` → distance asc |
| **Newest** | `sort: NEWEST` → `createdAt` desc |

Changing a chip or the sort re-runs `Rescues (CL)` and resets pagination.

---

### 3. Map Preview

A compact map showing each open listing in the selected city (after the active filter).

| Element | Description |
|---------|-------------|
| Pins | One pin per listing, at `lat` / `lng` |
| `[Open map]` button | Top-right — opens a **full-screen map** of the same pins (clustered when dense) |
| "N rescues near you" chip | Bottom-left — count of listings currently plotted |

**Centring:** GPS granted → centre on the user; GPS unavailable → centre on the selected city's centre (`Cities (CA)` lat/lng).

**Pin interaction:** tap a pin → mini card (photo + name + species + distance) → tap card → **Rescue Detail** (`screen_26`).

**Full-screen map (`[Open map]`):** same pin set, pan/zoom/cluster; data identical to this screen. (Detailed map interactions = shared follow-up with `screen_18` / `screen_20` maps — see Open Items.)

---

### 4. Rescue List

Full, paginated list of open listings in the selected city (after the active filter), sorted by the chosen sort.

**Row** — the canonical **Rescue item** defined in `screen_17` → Section 8:

```
[photo]   {name}                                 {distanceKm}km
          {species} · {breed} · {ageText} · {gender}
          {charityName}  🏷CHARITY
```

| Element | Source | Notes |
|---------|--------|-------|
| Thumbnail | `thumbnailUrl` (`photos[0]`) | cover photo |
| `name` | `name` | bold; `"Chưa đặt tên"` if blank |
| `distanceKm` | computed per request | `< 10` → 1 decimal, `≥ 10` → integer |
| species · breed · age · gender | `species`, `breed`, `ageText`, `gender` | ` · `-joined; **skip any null** (breed/age optional); gender `♀`/`♂`/Unknown |
| `charityName` + CHARITY badge | `charity.name` | the posting charity family; badge always shown |

- **Pagination:** infinite scroll via `Rescues (CL)` `cursor`.
- **Tap a row** → **Rescue Detail** (`screen_26`).
- **Empty state** (filter matches nothing): *"No rescues match this filter."* (banner + map + chips stay visible).

---

## API Endpoints Required

> All calls go to `POST /graphql`. **Auth:** Required.
> **No new query for the list** — reuses `Rescues (CL)` + `Cities (CA)` from `screen_17` (list + map pins + filter + sort + pagination). The detail, create form, and management live in `screen_26` / `screen_27`.

---

## User Flow Diagrams

### Enter Rescue

```
More tab → "Rescue" icon  OR  "View All →"
  └─> Rescue screen (selected city passed in)
        └─> Rescues (CL) { cityCode, countryCode, sort, originLat?, originLng?, [filter] }
              ├─ items → list + map pins
              └─ empty → empty state
        └─> active family is charity? → show "Post a Rescue" banner
```

### Filter / sort

```
Tap a species chip / change sort
  └─> Rescues (CL) with that filter+sort → reset pagination → re-render list + map
```

### Post a rescue (charity)

```
Tap [Post a Rescue]   (banner — charity active family only)
  └─> Create Rescue form (screen_26) → CreateRescue (CO) → listing appears (OPEN)
```

### Open a listing

```
Tap a list row  OR  a map pin's card
  └─> Rescue Detail (screen_26) { rescueId }
```

---

## Edge Cases & Notes

| Case | Expected Behaviour |
|------|--------------------|
| Not logged in | More tab (and this screen) redirect to Login |
| Active family not a charity | "Post a Rescue" banner hidden; browsing still works |
| Selected city has 0 open listings | List empty state; map shows "0 rescues near you"; banner still shown for charity |
| Listing marked adopted while open | On refresh it drops out (`status = OPEN` only) |
| GPS denied | Distance from city centre (still shown); map centres on city; sort `Nearest` still works off city-centre distance |
| GPS granted | Distance from user; map centres on user |
| Filter matches nothing | "No rescues match this filter."; banner + map + chips remain |
| Distance ≥ 10km | Integer format (`120km`); `< 10km` → 1 decimal (`2.3km`) |
| Listing missing breed / age | Skip that part on line 2 |
| Many listings in a city | Map clusters; list paginates; optional pins-only query (Open Items) |

---

## Decisions Log

| # | Question | Decision |
|---|----------|----------|
| 1 | Concept | Rescue = open adoption listings posted by **charity families**; mirror of Lost Pets (new home vs old home) |
| 2 | Scope | City-scoped (selected city); **open** listings only (`status = OPEN`) |
| 3 | Sort | Toggle **Nearest** (default) / **Newest** |
| 4 | Filter chips | Single-select species (`Cats` / `Dogs` / `Other`); `All` resets |
| 5 | Map | **Kept** (like Lost Pets / Pet Friendly) — rescues have a location you visit |
| 6 | Posting entry | Banner shown only when active family is a charity → Create Rescue form (`screen_26`); also from Manage Rescues (`screen_27`) |
| 7 | Row layout | Canonical Rescue item (`screen_17` §8): photo + name + distance / species·breed·age·gender / charity · CHARITY |

---

## Open Items (next steps)

- **Full-screen map** (`[Open map]`) — shared pan/zoom/cluster behaviour with `screen_18` / `screen_20`; optional pins-only query if volume grows.
- **Combined filters** (species + age/size) — possible later enhancement.
