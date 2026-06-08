# Screen 17: More

## Overview

The **"More" tab** — last tab in the bottom navigation. A location-aware hub for community/discovery features grouped into categories: **Lost Pets**, **Pet Friendly**, **Events**, **Rescue**.

This phase ships **Lost Pets only**; the other three categories are placeholders (see Edge Cases).

Requires login. If not logged in → tapping the More tab redirects to Login (same rule as My Pets / Shops / Services — see `screen_1_home_explore.md` → Bottom Navigation auth rules).

The whole screen is scoped to a **selected city** shown in the location bar at the top. On first open the app attempts to auto-detect the nearest city; the user can change it anytime.

---

## UI Layout

```
[Header]
  Title: "More"
  Right actions: [Search icon] | [Messages icon] | [Profile avatar]
  (identical header to My Pets — see screen_8; Messages red dot = combined unread)

[Location bar]
  [📍 icon]  HCMC, VN                                  [Change]

[Categories row — 4 icon buttons]
  [⚠️]          [📍]            [🗓️]        [♡]
  Lost Pets    Pet Friendly    Events      Rescue

[LOST PETS section]
  "LOST PETS"                                    [View All →]
  ┌────────────────────────────────────────────────────────┐
  │ [avatar]   Măng  (Minh's Family)                        │
  │ [Missing]  Pony · buckskin                              │
  │            Đà Lạt, VN                                   │
  ├────────────────────────────────────────────────────────┤
  │ [avatar]   Pudding  (Pudding's Family)                  │
  │ [Missing]  Cat · grey tabby                             │
  │            HCMC, VN                                     │
  ├────────────────────────────────────────────────────────┤
  │ [avatar]   Bibi  (Bibi's Family)                        │
  │ [Missing]  Dog · Pomeranian                             │
  │            HCMC, VN                                     │
  └────────────────────────────────────────────────────────┘
  (max 3 rows; "View All →" only shown when total > 0)

[PET FRIENDLY section]   ← placeholder this phase (see Edge Cases)
[EVENTS section]         ← placeholder this phase
[RESCUE section]         ← placeholder this phase

[Bottom Navigation]
  My Pets | Explore | Shops | Services | More (active)
```

---

## Components

### 1. Header

Identical to the My Pets header (`screen_8`): Search icon · Messages icon (red dot when any unread — combined Chats + Notifications, opens `screen_10`) · Profile avatar. No back button (it's a root tab).

---

### 2. Location Bar

Single row pinned below the header.

| Element | Description |
|---------|-------------|
| Pin icon + label | Current **selected city**, formatted `"{cityCode}, {countryCode}"` (e.g. `"HCMC, VN"`) |
| Change button | Tap → opens **Choose Your City** bottom sheet |

**Auto-detection (on app open / first visit to More):**

```
App requests location permission (OS prompt)
  ├─ Granted + position obtained
  │     └─> Compute nearest city: client-side haversine over the Cities list (CA)
  │           ├─ Nearest is AVAILABLE   → set as selected city
  │           └─ Nearest is COMING_SOON → set as selected city, but content is LOCKED
  │                                        (see "Coming-soon locked state" below)
  ├─ Denied / unavailable / timeout
  │     └─> Default to HCMC, VN  (the only AVAILABLE city this phase)
  └─ Permission already decided previously
        └─> Use last selected city from local storage (no re-prompt)
```

- Permission is requested **once**; the OS remembers the decision. We never re-prompt within the app — user changes city manually via **Change**.
- Selected city is stored **client-side** (localStorage / device storage). It is **not** the same as the user's active family or any post location — it's a discovery preference for the More tab only.
- Nearest-city computation runs over **all** cities (including `COMING_SOON`); the genuinely nearest one wins even if it isn't live yet (decision: show-but-lock).

**Coming-soon locked state** (selected city has `status = COMING_SOON`):
- Location bar still shows the detected city (e.g. `"HN, VN"`).
- All category sections (Lost Pets, etc.) are replaced by a single locked message: *"PetApp is coming soon to {City}. Switch to a live city to explore."* + a **[Change city]** button.
- No data is fetched for locked cities.

---

### 3. Choose Your City (bottom sheet)

Opened by tapping **Change**.

```
[drag handle]
Choose Your City                                    [× close]
──────────────────────────────────────────────────────────
🇻🇳  Ho Chi Minh City                                    ✓
     Vietnam
──────────────────────────────────────────────────────────
🇻🇳  Hà Nội                                      [COMING SOON]
     Vietnam
──────────────────────────────────────────────────────────
🇸🇬  Singapore                                   [COMING SOON]
     Singapore
... (scrollable list)
```

| Element | Description |
|---------|-------------|
| Flag emoji | `flagEmoji` from the city record |
| City name | Full display name (e.g. `"Ho Chi Minh City"`) |
| Country | Sub-label (e.g. `"Vietnam"`) |
| Trailing | `✓` checkmark on the **currently selected** city; `COMING SOON` pill on cities with `status = COMING_SOON` |

**Selection rules:**
- `AVAILABLE` rows → tappable. Tap → set as selected city, persist, close sheet, refresh sections.
- `COMING_SOON` rows → **disabled** (greyed pill, no tap action). User cannot manually switch to a coming-soon city.
- The currently selected city always shows the `✓` checkmark, even if it was auto-detected as a `COMING_SOON` city.

**Data source:** `Cities query (CA)`. The same list powers both the picker and the client-side nearest detection — fetched once and cached.

---

### 4. Categories Row

4 fixed icon buttons. Only **Lost Pets** is functional this phase.

| Button | Icon | Tap action (this phase) |
|--------|------|-------------------------|
| Lost Pets | ⚠️ triangle | Navigate to **Lost Pets screen** (full list + map) — same destination as the section's "View All →" |
| Pet Friendly | 📍 | "Coming Soon" placeholder screen |
| Events | 🗓️ | "Coming Soon" placeholder screen |
| Rescue | ♡ | "Coming Soon" placeholder screen |

---

### 5. Lost Pets Section

Shows missing pets **in the selected city** (decision: scoped to the selected city — not nearest-distance across cities).

- Section header: **"LOST PETS"** + **"View All →"** link on the right.
- **Max 3 rows** (preview). "View All →" navigates to the **Lost Pets screen** (full list + map + filters).
- Sorted by `reportedAt` desc (most recently reported first) within the city.

**Each row (Lost Pet preview item):**

| Field | Display |
|-------|---------|
| `pet.avatarUrl` | Pet avatar (circular), with a **"Missing"** badge overlaid bottom-left |
| `pet.name` | Pet name (bold) |
| `family.name` | Family name in parentheses next to the name: `"Măng (Minh's Family)"` |
| `pet.species` · `pet.breed` | Second line: `"Pony · buckskin"` — show species only if breed is null |
| `lastSeen.cityCode` · `countryCode` | Third line, muted: `"Đà Lạt, VN"` |

- **Tap a row** → **Lost Pet Detail screen** for that report.
- **Empty state** (no missing pets in the selected city): show a compact empty message e.g. *"No lost pets reported in {City} right now"* and **hide the "View All →" link**.

> **Reporting a missing pet is NOT done from the More hub section.** There is no Report button in this section. Reports are filed either from the pet's own page (`screen_9` Pet Detail → **Report Missing button**, located below the category tabs) or from the **Lost Pets screen** banner (`screen_18`), which opens the same Report Missing form with a pet selector.

---

### 6. Other Category Sections (placeholder this phase)

`Pet Friendly`, `Events`, `Rescue` sections are **out of scope** for this phase. The mockup shows a Pet Friendly section for layout reference only. For now:
- Either hide these sections entirely, OR render the section header with a "Coming soon" inline state.
- No API, no data binding. Final behaviour defined in a later phase.

---

## API Endpoints Required

> All calls go to `POST /graphql`. **Auth:** Required (the More tab requires login).

---

### CA. Query: `Cities`

Returns the list of supported cities with coordinates and availability — powers the **Choose Your City** picker and the **client-side nearest-city** detection. Fetched once and cached.

**Auth:** Optional (static reference data).

**Operation:**
```graphql
query Cities {
  cities {
    city
    cityCode
    country
    countryCode
    flagEmoji
    lat
    lng
    status
  }
}
```

**Response `200 OK`:**
```json
{
  "data": {
    "cities": [
      {
        "city": "Ho Chi Minh City",
        "cityCode": "HCM",
        "country": "Vietnam",
        "countryCode": "VN",
        "flagEmoji": "🇻🇳",
        "lat": 10.7769,
        "lng": 106.7009,
        "status": "AVAILABLE"
      },
      {
        "city": "Hà Nội",
        "cityCode": "HN",
        "country": "Vietnam",
        "countryCode": "VN",
        "flagEmoji": "🇻🇳",
        "lat": 21.0278,
        "lng": 105.8342,
        "status": "COMING_SOON"
      },
      {
        "city": "Singapore",
        "cityCode": "SG",
        "country": "Singapore",
        "countryCode": "SG",
        "flagEmoji": "🇸🇬",
        "lat": 1.3521,
        "lng": 103.8198,
        "status": "COMING_SOON"
      }
    ]
  }
}
```

**Notes:**
- `status`: `AVAILABLE` | `COMING_SOON`.
- `lat` / `lng` = city centre, used by the client to pick the nearest city via haversine distance. No separate "nearest city" endpoint is needed (the list is small and cached).
- Ordering is server-defined (AVAILABLE first, then a curated coming-soon order as in the mockup).

---

### CB. Query: `LostPets`

Returns missing-pet reports for a city. Used by both the **More → Lost Pets section** (`limit: 3`) and the full **Lost Pets screen** (paginated, with filters). Filter args beyond `cityCode` are consumed by the full screen — documented here for reuse; the More section passes only `cityCode` + `limit`.

**Auth:** Required.

**Operation:**
```graphql
query LostPets(
  $cityCode: String!
  $countryCode: String!
  $filter: LostPetsFilter
  $cursor: String
  $limit: Int
) {
  lostPets(
    cityCode: $cityCode
    countryCode: $countryCode
    filter: $filter
    cursor: $cursor
    limit: $limit
  ) {
    items {
      reportId
      pet {
        id
        name
        species
        breed
        avatarUrl
      }
      family {
        id
        name
        avatarUrl
      }
      lastSeen {
        city
        cityCode
        country
        countryCode
        lat
        lng
        at
      }
      reportedAt
      distanceKm
    }
    nextCursor
    hasMore
    totalCount
  }
}
```

**`LostPetsFilter` (used by the full Lost Pets screen, not the More section):**
```json
{
  "species": ["CAT", "DOG"],
  "maxDistanceKm": 5,
  "originLat": 10.78,
  "originLng": 106.70
}
```

**Variables (More section — preview):**
```json
{ "cityCode": "HCM", "countryCode": "VN", "limit": 3 }
```

**Response `200 OK`:**
```json
{
  "data": {
    "lostPets": {
      "items": [
        {
          "reportId": "miss_001",
          "pet": {
            "id": "pet_201",
            "name": "Măng",
            "species": "Pony",
            "breed": "buckskin",
            "avatarUrl": "https://cdn.petapp.com/pets/pet_201/avatar.jpg"
          },
          "family": {
            "id": "fam_minh",
            "name": "Minh's Family",
            "avatarUrl": "https://cdn.petapp.com/families/fam_minh/avatar.jpg"
          },
          "lastSeen": {
            "city": "Đà Lạt",
            "cityCode": "DL",
            "country": "Việt Nam",
            "countryCode": "VN",
            "lat": 11.9404,
            "lng": 108.4583,
            "at": "2026-06-07T17:30:00Z"
          },
          "reportedAt": "2026-06-07T17:35:00Z",
          "distanceKm": null
        }
      ],
      "nextCursor": "eyJpZCI6Im1pc3NfMDAxIn0=",
      "hasMore": true,
      "totalCount": 12
    }
  }
}
```

**Notes:**
- `cityCode` + `countryCode` scope results to the selected city (decision #4).
- `distanceKm` is populated only when `originLat`/`originLng` are supplied (user's real GPS) — it powers the `Within 1km / 5km` chips on the full screen. `null` when GPS is unavailable → those chips are disabled on the full screen.
- `lastSeen.at` = when the pet was last seen (user-entered). `reportedAt` = when the report was filed. They can differ.
- Server only returns **active** missing reports (pet `missing_status != null` and not yet marked found).

**Errors:**

| Code | Scenario |
|------|----------|
| `UNAUTHENTICATED` | Caller is not logged in |
| `CITY_NOT_FOUND` | Unknown `cityCode` / `countryCode` |
| `CITY_NOT_AVAILABLE` | City has `status = COMING_SOON` (client should render the locked state instead of calling this) |

---

## User Flow Diagrams

### Open More tab

```
User taps More tab
  ├─ [not logged in] → redirect to Login
  └─ [logged in]
        └─> Resolve selected city:
              ├─ stored city exists → use it
              └─ no stored city → request location permission
                    ├─ granted → Cities (CA) → nearest by haversine
                    │      ├─ AVAILABLE   → set + fetch sections
                    │      └─ COMING_SOON → set + show LOCKED state
                    └─ denied/fail → default HCMC, VN → fetch sections
                          └─> LostPets (CB) { cityCode, countryCode, limit: 3 }
                                ├─ items → render up to 3 rows + "View All →"
                                └─ empty → empty message, hide "View All →"
```

### Change city

```
User taps Change → Choose Your City sheet (Cities CA, cached)
  ├─ tap AVAILABLE city → persist + close → refetch all sections for new city
  └─ tap COMING_SOON city → no-op (disabled)
```

### Enter Lost Pets

```
User taps "Lost Pets" category icon  OR  "View All →" in the section
  └─> Lost Pets screen (full list + map + filters), scoped to selected city

User taps a Lost Pet row (section)
  └─> Lost Pet Detail screen for that reportId
```

---

## Edge Cases & Notes

| Case | Expected Behaviour |
|------|--------------------|
| Not logged in | More tab redirects to Login |
| Location permission denied / unavailable | Default to HCMC, VN |
| Nearest detected city is `COMING_SOON` | Show that city in the bar but render locked state ("coming soon"); no data fetched |
| User manually opens picker, taps a `COMING_SOON` city | Disabled — no action |
| Selected city has 0 missing pets | Empty message; hide "View All →" |
| Selected city has 1–3 missing pets | Show all; "View All →" still shown (full screen may add map/filters) |
| Selected city has > 3 missing pets | Show first 3; "View All →" → full screen |
| Pet has no breed | Show species only on the 2nd line |
| GPS available | `distanceKm` populated → distance chips usable on full screen |
| GPS unavailable | `distanceKm` null → `Within 1km/5km` chips disabled on full screen |
| Pet Friendly / Events / Rescue | Out of scope this phase — placeholder / hidden |
| City list fetch fails | Fall back to HCMC default; allow retry on Change |

---

## Decisions Log

| # | Question | Decision |
|---|----------|----------|
| 1 | City reference data shape | Each city carries `lat`/`lng` (centre) + `status` (`AVAILABLE`/`COMING_SOON`); nearest computed client-side via haversine |
| 2 | Location permission denied | Default to HCMC, VN |
| 3 | Nearest city is `COMING_SOON` | Show the detected city but lock content ("coming soon"); prompt to Change |
| 4 | Manual selection of `COMING_SOON` city | Disabled in picker |
| 5 | Lost Pets section scope | Scoped to the **selected city** (not cross-city nearest); mockup showing Đà Lạt under HCMC is demo data only |
| 6 | Distance chips (`1km/5km`) | Filter **within** the selected city; require real GPS (`distanceKm`); disabled without GPS |
| 7 | Reporting entry point | Not in the More hub section — reports filed from `screen_9` (Pet Detail → Report Missing button, below the category tabs) or the Lost Pets screen banner (`screen_18`) |
| 8 | Pet Friendly / Events / Rescue | Out of scope this phase |

---

## Open Items (next steps)

- **Lost Pets screen** (full list + map + filter chips + "Have a missing pet?" banner) — to be specified (`screen_18`).
- **Lost Pet Detail screen** (header photo carousel, last-seen map, description, how-you-can-help, Share / "I saw {pet}") — to be specified (`screen_19`).
- **Report Missing form upgrade** in `screen_9` — add `lastSeenAt`, map pin `lat`/`lng`, `description`, cover photo selection; extend `ReportMissing (BE)` + `missing_status`.
