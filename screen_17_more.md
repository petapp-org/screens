# Screen 17: More

## Overview

The **"More" tab** — last tab in the bottom navigation. A location-aware hub for community/discovery features grouped into categories: **Lost Pets**, **Pet Friendly**, **Events**, **Rescue**.

This phase ships **all four** categories: **Lost Pets**, **Pet Friendly**, **Events**, and **Rescue**.

Requires login. If not logged in → tapping the More tab redirects to Login (same rule as My Pets / Shops / Services — see `screen_1_home_explore.md` → Bottom Navigation auth rules).

The whole screen is scoped to a **selected city** shown in the location bar at the top. On first open the app attempts to auto-detect the nearest city; the user can change it anytime.

---

## UI Layout

```
[Header]
  Title: "More"
  Right actions: [Search icon] | [Messages icon ✉] | [Notifications icon 🔔] | [Profile avatar]
  (identical header to My Pets — see screen_8; ✉ dot = unread chats, 🔔 dot = unread activity)

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

[PET FRIENDLY section]
  "PET FRIENDLY"                                 [View All →]
  ┌────────────────────────────────────────────────────────┐
  │ [photo]   Lava Cat Coffee  ★4.8                3.4km    │
  │           Cat · Carriers OK                             │
  │           HCMC, VN · open until 22:00                   │
  ├────────────────────────────────────────────────────────┤
  │ [photo]   Ailu Cat Café  ★4.7                  4.2km    │
  │           Cat · Resident cats                           │
  │           HCMC, VN · play with 12 cats                  │
  └────────────────────────────────────────────────────────┘
  (max 3 rows — nearest in the selected city; "View All →" only when ≥ 1 item)

[EVENTS section]
  "EVENTS"                                        [View All →]
  ┌────────────────────────────────────────────────────────┐
  │ ┌─────┐  Cat Adoption Day                      2.1km    │
  │ │ JUN │  Sat · 9:00–12:00                               │
  │ │ 15  │  Lava Cat Coffee · HCMC                         │
  │ └─────┘  Free · ⏳ Starts in 2d                         │
  ├────────────────────────────────────────────────────────┤
  │ ┌─────┐  Dog Run Meetup                        5.4km    │
  │ │ JUN │  Tue · 17:30–19:00                              │
  │ │ 18  │  Tao Đàn Park · HCMC                            │
  │ └─────┘  Free · ⏳ Starts in 5d                         │
  └────────────────────────────────────────────────────────┘
  (max 3 — soonest upcoming in the selected city; "View All →" only when ≥ 1 item)

[RESCUE section]
  "RESCUE"                                        [View All →]
  ┌────────────────────────────────────────────────────────┐
  │ [photo]   Miu                                  2.3km    │
  │           Cat · Domestic shorthair · ~4mo · ♀          │
  │           Paws Rescue Saigon  🏷CHARITY                 │
  ├────────────────────────────────────────────────────────┤
  │ [photo]   Lucky                                3.1km    │
  │           Dog · Mixed · ~1yr · ♂                       │
  │           HCMC Animal Rescue  🏷CHARITY                 │
  └────────────────────────────────────────────────────────┘
  (max 3 — nearest open listings in the selected city; "View All →" only when ≥ 1 item)

[Bottom Navigation]
  My Pets | Explore | Shops | Services | More (active)
```

---

## Components

### 1. Header

Identical to the My Pets header (`screen_8`): Search icon · Messages icon `✉` (red dot when any unread chat, opens `screen_10`) · Notifications icon `🔔` (red dot when any unread activity, opens `screen_22`) · Profile avatar. No back button (it's a root tab).

---

### 2. Location Bar

Single row pinned below the header.

| Element | Description |
|---------|-------------|
| Pin icon + label | Current **selected city**, formatted `"{cityShortName}, {countryCode}"` (e.g. `"HCMC, VN"`). `cityShortName` is the curated short label (e.g. `HCMC`, `HN`, `Đà Lạt`) — distinct from `code` (`HCM`) and the full name (`nameVi` / `nameEn`) shown in the picker. |
| Change button | Tap → opens **Choose Your City** bottom sheet |

**Auto-detection (on app open / first visit to More):**

```
App requests location permission (OS prompt)
  ├─ Granted + position obtained
  │     └─> Compute nearest city: client-side haversine over the Cities list (CA)
  │           ├─ Nearest is ACTIVE      → set as selected city
  │           └─ Nearest is COMING_SOON → set as selected city, but content is LOCKED
  │                                        (see "Coming-soon locked state" below)
  ├─ Denied / unavailable / timeout
  │     └─> Default to HCMC, VN  (the only ACTIVE city this phase)
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
| City name | Full display name — `nameVi` in Vietnamese locale, `nameEn` in English locale (e.g. `"Hồ Chí Minh"` / `"Ho Chi Minh City"`) |
| Country | Sub-label derived from `countryCode` (e.g. `"VN"` → `"Vietnam"`) |
| Trailing | `✓` checkmark on the **currently selected** city; `COMING SOON` pill on cities with `status = COMING_SOON` |

**Selection rules:**
- `ACTIVE` rows → tappable. Tap → set as selected city, persist, close sheet, refresh sections.
- `COMING_SOON` rows → **disabled** (greyed pill, no tap action). User cannot manually switch to a coming-soon city.
- The currently selected city always shows the `✓` checkmark, even if it was auto-detected as a `COMING_SOON` city.

**Data source:** `Cities query (CA)`. The same list powers both the picker and the client-side nearest detection — fetched once and cached.

---

### 4. Categories Row

4 fixed icon buttons — **all functional** this phase.

| Button | Icon | Tap action |
|--------|------|------------|
| Lost Pets | ⚠️ triangle | Navigate to **Lost Pets screen** (`screen_18`, full list + map) — same destination as the section's "View All →" |
| Pet Friendly | 📍 | Navigate to **Pet Friendly screen** (`screen_20`) — same as the section's "View All →" |
| Events | 🗓️ | Navigate to **Events screen** (`screen_23`) — same as the section's "View All →" |
| Rescue | ♡ | Navigate to **Rescue screen** (`screen_25`) — same as the section's "View All →" |

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
| `lastSeen.cityShortName` · `countryCode` | Third line, muted: `"Đà Lạt, VN"` (short label + country code) |

- **Tap a row** → **Lost Pet Detail screen** for that report.
- **Empty state** (no missing pets in the selected city): show a compact empty message e.g. *"No lost pets reported in {City} right now"* and **hide the "View All →" link**.

> **Reporting a missing pet is NOT done from the More hub section.** There is no Report button in this section. Reports are filed either from the pet's own page (`screen_9` Pet Detail → **Report Missing button**, located below the category tabs) or from the **Lost Pets screen** banner (`screen_18`), which opens the same Report Missing form with a pet selector.

---

### 6. Pet Friendly Section

Shows pet-friendly places **in the selected city**, the **3 nearest** to the user.

- Section header: **"PET FRIENDLY"** + **"View All →"** link → **Pet Friendly screen** (`screen_20`).
- **Max 3 rows** (preview), sorted by **distance asc** (nearest first) from the user's origin (GPS if available, else the selected city's centre).
- **Empty state** (no places in the selected city): hide the section (or show a compact "No pet-friendly places yet"); **hide "View All →"**.

**Each row (Pet Friendly item)** — canonical layout, reused on `screen_20` and `screen_21`:

```
[photo]   {name}  ★{ratingAvg}                 {distanceKm}km
          {suitable} · {tagText}
          {cityShortName}, {countryCode} · {highlightText}
```

| Element | Source | Notes |
|---------|--------|-------|
| Thumbnail | `thumbnailUrl` | first photo of the place |
| `name` + `★ratingAvg` | `name`, `ratingAvg` | rating sits next to the name (line 1) |
| `distanceKm` | computed per request | top-right; `< 10` → 1 decimal (`3.4km`), `≥ 10` → integer (`120km`) |
| `suitable` | `suitableFor[]` | `CAT`→"Cat", `DOG`→"Dog", `OTHER`→"Pet friendly"; multiple joined by ` · ` |
| `tagText` | admin free text, **≤ 18 chars** | line-2 flair after suitable; single line, `…` if overflow |
| `cityShortName`, `countryCode` | place location | e.g. `"HCMC, VN"` |
| `highlightText` | admin free text, **≤ 20 chars** | line-3 flair after city-country; single line, `…` if overflow |

- **Tap a row** → **Pet Friendly Place Detail** (`screen_21`).
- Data source: `PetFriendlyPlaces query (CD)` `{ cityCode, countryCode, originLat?, originLng?, first: 3 }`.

---

### 7. Events Section

Shows **upcoming events in the selected city**, the **3 soonest** to start.

- Section header: **"EVENTS"** + **"View All →"** link → **Events screen** (`screen_23`).
- **Max 3 rows** (preview), only events that have **not ended** (`endAt >= now`), sorted by **`startAt` asc** (soonest first).
- **Empty state** (no upcoming events in the selected city): hide the section (or show a compact "No upcoming events"); **hide "View All →"**.

**Each row (Event item)** — canonical layout, reused on `screen_23`:

```
[date-chip]   {title}                            {distanceKm}km
              {weekday} · {startTime}–{endTime}
              {venueName} · {cityShortName}
              {price} · {countdown}
```

| Element | Source | Notes |
|---------|--------|-------|
| date-chip | `startAt` | Calendar-style chip: short month + day (e.g. `JUN 15`) |
| `title` | `title` | Event name (bold, line 1) |
| `distanceKm` | computed per request | top-right; `< 10` → 1 decimal (`2.1km`), `≥ 10` → integer (`120km`) |
| weekday · time | `startAt`, `endAt` | Line 2: `"Sat · 9:00–12:00"` |
| `venueName` · `cityShortName` | event location | Line 3: `"Lava Cat Coffee · HCMC"` (short venue + city; full address only on detail) |
| `price` | `price` | Line 4; empty/zero → `"Free"` |
| `countdown` | `startAt` / `endAt` | Line 4 after price — see **Countdown rule** below |

**Countdown rule** (shared by the section, `screen_23`, and `screen_24`):

| State | Condition | Label |
|-------|-----------|-------|
| Upcoming | `now < startAt` | `Starts in {N}d` (≥ 24h) / `Starts in {N}h` (< 24h) |
| Ongoing | `startAt ≤ now ≤ endAt` | `Ends in {N}h` / `Happening now` |
| Ended | `now > endAt` | *(not shown — filtered out of all listings)* |

- **Tap a row** → **Event Detail** (`screen_24`).
- Data source: `Events query (CI)` `{ cityCode, countryCode, originLat?, originLng?, first: 3 }`.

---

### 8. Rescue Section

Shows **open rescue listings in the selected city** — pets that **charity families** have posted for adoption, the **3 nearest** to the user.

- Section header: **"RESCUE"** + **"View All →"** link → **Rescue screen** (`screen_25`).
- **Max 3 rows** (preview), only **open** listings (`status = OPEN`), sorted by **distance asc** (nearest first).
- **Empty state** (no open listings in the selected city): hide the section (or show a compact "No rescues nearby yet"); **hide "View All →"**.

**Each row (Rescue item)** — canonical layout, reused on `screen_25` / `screen_27`:

```
[photo]   {name}                                 {distanceKm}km
          {species} · {breed} · {ageText} · {gender}
          {charityName}  🏷CHARITY
```

| Element | Source | Notes |
|---------|--------|-------|
| Thumbnail | `photos[0]` | cover photo of the listing |
| `name` | `name` | listing name (bold); `"Chưa đặt tên"` if blank |
| `distanceKm` | computed per request | top-right; `< 10` → 1 decimal, `≥ 10` → integer |
| species · breed · age · gender | `species`, `breed`, `ageText`, `gender` | line 2, ` · `-joined; skip any null part (breed/age optional) |
| `charityName` + CHARITY badge | posting `family.name` | line 3 — the rescue org (charity family); badge always shown (only charity families can post) |

- **Tap a row** → **Rescue Detail** (`screen_26`).
- Data source: `Rescues query (CL)` `{ cityCode, countryCode, originLat?, originLng?, sort: NEAREST, first: 3 }`.

> **Posting a rescue is NOT done from the More hub section** — there is no Post button here. Charity families post from the **Rescue screen** banner (`screen_25`) or from **Manage Rescues** (`screen_27`).

---

## API Endpoints Required

> All calls go to `POST /graphql`. **Auth:** Required (the More tab requires login).

---

### CA. Query: `Cities`

Returns the list of supported cities with coordinates and availability — powers the **Choose Your City** picker and the **client-side nearest-city** detection. Fetched once and cached.

**Shipped:** `cities: [City!]!` landed in petapp-be#852 / petapp-be#965.

**Auth:** Optional (static reference data).

**Operation:**
```graphql
query Cities {
  cities {
    code
    nameVi
    nameEn
    cityShortName
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
        "code": "HCM",
        "nameVi": "Hồ Chí Minh",
        "nameEn": "Ho Chi Minh City",
        "cityShortName": "HCMC",
        "countryCode": "VN",
        "flagEmoji": "🇻🇳",
        "lat": 10.7769,
        "lng": 106.7009,
        "status": "ACTIVE"
      },
      {
        "code": "HN",
        "nameVi": "Hà Nội",
        "nameEn": "Hanoi",
        "cityShortName": "HN",
        "countryCode": "VN",
        "flagEmoji": "🇻🇳",
        "lat": 21.0278,
        "lng": 105.8342,
        "status": "COMING_SOON"
      },
      {
        "code": "SG",
        "nameVi": "Singapore",
        "nameEn": "Singapore",
        "cityShortName": "Singapore",
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
- `status`: `ACTIVE` | `COMING_SOON` | `INACTIVE` (enum `CityStatus`).
- UI displays `nameVi` or `nameEn` depending on locale; `cityShortName` is the curated short label for compact display (e.g. `HCMC`, `HN`).
- `lat` / `lng` = city centre, used by the client to pick the nearest city via haversine distance. No separate "nearest city" endpoint is needed (the list is small and cached).
- Ordering is server-defined (`ACTIVE` first, then a curated coming-soon order as in the mockup).

---

### CB. Query: `LostPets`

> ⚠️ **GAP petapp-be#888:** backend has no browse-by-city lost-pets query (only `searchLostPets(q!, lat/lng)` text-search + `listLostPetReports(type, cursor)`). Kept as-is, pending backend `lostPetsByCity(... first, after)`.

Returns missing-pet reports for a city. Used by both the **More → Lost Pets section** (`first: 3`) and the full **Lost Pets screen** (paginated, with filters). Filter args beyond `cityCode` are consumed by the full screen — documented here for reuse; the More section passes only `cityCode` + `first`.

**Auth:** Required.

**Operation:**
```graphql
query LostPets(
  $cityCode: String!
  $countryCode: String!
  $filter: LostPetsFilter
  $after: String
  $first: Int! = 20
) {
  lostPets(
    cityCode: $cityCode
    countryCode: $countryCode
    filter: $filter
    after: $after
    first: $first
  ) {
    reportsCount
    edges {
      cursor
      node {
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
          cityShortName
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
    }
    pageInfo {
      hasNextPage
      endCursor
    }
  }
}
```

> **Note:** `reportsCount` is a sibling field on `lostPets` (total number of matching reports), alongside `edges`/`pageInfo` but not inside the Relay connection — per ADR-0023.

**`LostPetsFilter` (used by the full Lost Pets screen, not the More section):**
```json
{
  "species": ["CAT", "DOG", "OTHER"],
  "maxDistanceKm": 5,
  "originLat": 10.78,
  "originLng": 106.70
}
```

- `species` values: `CAT` | `DOG` | `OTHER`. `OTHER` is a server-side sentinel meaning "any species that is not CAT or DOG" — it powers the **Other** filter chip on the full screen.

**Variables (More section — preview):**
```json
{ "cityCode": "HCM", "countryCode": "VN", "first": 3 }
```

**Response `200 OK`:**
```json
{
  "data": {
    "lostPets": {
      "reportsCount": 12,
      "edges": [
        {
          "cursor": "eyJpZCI6Im1pc3NfMDAxIn0=",
          "node": {
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
              "cityShortName": "Đà Lạt",
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
        }
      ],
      "pageInfo": {
        "hasNextPage": true,
        "endCursor": "eyJpZCI6Im1pc3NfMDAxIn0="
      }
    }
  }
}
```

**Notes:**
- `cityCode` + `countryCode` scope results to the selected city (decision #4).
- `distanceKm` is populated only when `originLat`/`originLng` are supplied (user's real GPS) — it powers the `Within 1km / 5km` chips on the full screen. `null` when GPS is unavailable → those chips are disabled on the full screen.
- `lastSeen.at` = when the pet was last seen (user-entered). `reportedAt` = when the report was filed. They can differ.
- Server only returns **active** missing reports (pet `missingStatus != null` and not yet marked found).

**Errors:**

| Code | Scenario |
|------|----------|
| `UNAUTHENTICATED` | Caller is not logged in |
| `CITY_NOT_FOUND` | Unknown `cityCode` / `countryCode` |
| `CITY_NOT_AVAILABLE` | City has `status = COMING_SOON` (client should render the locked state instead of calling this) |

---

### CD. Query: `PetFriendlyPlaces`

> ⚠️ **Partial GAP petapp-be#888:** backend `places(category, lat, lng, radiusM, sort, offset, limit)` is geo+offset, not cityCode+cursor. Kept as-is, pending backend cityCode/Relay support.

Returns pet-friendly places for a city, sorted by distance from the user. Used by both the **More → Pet Friendly section** (`first: 3`) and the full **Pet Friendly screen** (`screen_20`, paginated + filtered).

**Auth:** Required.

**Operation:**
```graphql
query PetFriendlyPlaces(
  $cityCode: String!
  $countryCode: String!
  $filter: PetFriendlyFilter
  $originLat: Float
  $originLng: Float
  $after: String
  $first: Int! = 20
) {
  petFriendlyPlaces(
    cityCode: $cityCode
    countryCode: $countryCode
    filter: $filter
    originLat: $originLat
    originLng: $originLng
    after: $after
    first: $first
  ) {
    placesCount
    edges {
      cursor
      node {
        id
        name
        category
        thumbnailUrl
        suitableFor
        tagText
        highlightText
        ratingAvg
        reviewCount
        cityShortName
        countryCode
        lat
        lng
        distanceKm
      }
    }
    pageInfo {
      hasNextPage
      endCursor
    }
  }
}
```

> **Note:** `placesCount` is a sibling field on the query result (total number of matching places), not inside the connection — per ADR-0023.

**`PetFriendlyFilter`** (single active chip on the full screen; omit for "All"):
```json
{ "suitableFor": "CAT" }      // Cat-friendly / Dog-friendly chips
// or
{ "category": "CAFE" }        // Cafés / Restaurants / Hotels / … chips
```

**Variables (More section — preview):**
```json
{ "cityCode": "HCM", "countryCode": "VN", "originLat": 10.78, "originLng": 106.70, "first": 3 }
```

**Response `200 OK`:**
```json
{
  "data": {
    "petFriendlyPlaces": {
      "placesCount": 12,
      "edges": [
        {
          "cursor": "eyJpZCI6InBsYWNlXzAwMSJ9",
          "node": {
            "id": "place_001",
            "name": "Lava Cat Coffee",
            "category": "CAFE",
            "thumbnailUrl": "https://cdn.petapp.com/places/place_001/1.jpg",
            "suitableFor": ["CAT"],
            "tagText": "Carriers OK",
            "highlightText": "open until 22:00",
            "ratingAvg": 4.8,
            "reviewCount": 126,
            "cityShortName": "HCMC",
            "countryCode": "VN",
            "lat": 10.7731,
            "lng": 106.7042,
            "distanceKm": 3.4
          }
        }
      ],
      "pageInfo": {
        "hasNextPage": true,
        "endCursor": "eyJpZCI6InBsYWNlXzAwMSJ9"
      }
    }
  }
}
```

**Notes:**
- City-scoped (`cityCode` + `countryCode`); sorted by `distanceKm` asc.
- `distanceKm` from `originLat`/`originLng` (user GPS). When GPS is absent, the server uses the **city centre** as origin (distance still returned, relative to centre).
- Distance display rule (client): `< 10` → 1 decimal (`3.4km`); `≥ 10` → integer (`120km`).
- `ratingAvg` / `reviewCount` are aggregated from user reviews (see `screen_21`); not admin-entered.
- `tagText` ≤ 18, `highlightText` ≤ 20 chars (admin-entered, see `screen_21` field model); render single-line + `…`.
- Only `isPublished = true` places are returned.

**Errors:**

| Code | Scenario |
|------|----------|
| `UNAUTHENTICATED` | Caller is not logged in |
| `CITY_NOT_FOUND` | Unknown `cityCode` / `countryCode` |
| `CITY_NOT_AVAILABLE` | City has `status = COMING_SOON` (render locked state instead) |

---

### CI. Query: `Events`

> ⚠️ **GAP petapp-be#888:** the Events domain does not exist on backend (no `Event` type, no query). Kept as-is, pending backend.

Returns **upcoming** events for a city, sorted soonest-first. Used by both the **More → Events section** (`first: 3`) and the full **Events screen** (`screen_23`, paginated + time/price filters). All events are **admin-entered** (no AI, no charity/family hosting).

**Auth:** Required.

**Operation:**
```graphql
query Events(
  $cityCode: String!
  $countryCode: String!
  $filter: EventsFilter
  $originLat: Float
  $originLng: Float
  $after: String
  $first: Int! = 20
) {
  events(
    cityCode: $cityCode
    countryCode: $countryCode
    filter: $filter
    originLat: $originLat
    originLng: $originLng
    after: $after
    first: $first
  ) {
    eventsCount
    edges {
      cursor
      node {
        id
        title
        thumbnailUrl
        startAt
        endAt
        price
        isFree
        venueName
        cityShortName
        countryCode
        lat
        lng
        distanceKm
        interestedCount
        viewerInterested
      }
    }
    pageInfo {
      hasNextPage
      endCursor
    }
  }
}
```

> **Note:** `eventsCount` is a sibling field on the query result (total number of matching events), not inside the connection — per ADR-0023.

**`EventsFilter`** (used by the full Events screen; single active chip — omit for "All"):
```json
{ "timeWindow": "THIS_WEEK" }   // THIS_WEEK | THIS_WEEKEND
// or
{ "isFree": true }              // Free chip
```

**Variables (More section — preview):**
```json
{ "cityCode": "HCM", "countryCode": "VN", "originLat": 10.78, "originLng": 106.70, "first": 3 }
```

**Response `200 OK`:**
```json
{
  "data": {
    "events": {
      "eventsCount": 8,
      "edges": [
        {
          "cursor": "eyJpZCI6ImV2ZW50XzAwMSJ9",
          "node": {
            "id": "event_001",
            "title": "Cat Adoption Day",
            "thumbnailUrl": "https://cdn.petapp.com/events/event_001/1.jpg",
            "startAt": "2026-06-15T02:00:00Z",
            "endAt": "2026-06-15T05:00:00Z",
            "price": "Free",
            "isFree": true,
            "venueName": "Lava Cat Coffee",
            "cityShortName": "HCMC",
            "countryCode": "VN",
            "lat": 10.7731,
            "lng": 106.7042,
            "distanceKm": 2.1,
            "interestedCount": 48,
            "viewerInterested": false
          }
        }
      ],
      "pageInfo": {
        "hasNextPage": true,
        "endCursor": "eyJpZCI6ImV2ZW50XzAwMSJ9"
      }
    }
  }
}
```

**Notes:**
- City-scoped (`cityCode` + `countryCode`); returns only **upcoming/ongoing** events (`endAt >= now`), sorted by `startAt` asc.
- `distanceKm` from `originLat`/`originLng` (user GPS); when GPS is absent, the server uses the **city centre** as origin. Distance is **display-only** (rows show it) — events are **not** sorted by distance.
- `price` is admin free text shown verbatim; `isFree = true` → render `"Free"` and power the **Free** filter chip.
- `interestedCount` = number of users who marked Interested (see `screen_24` → `SetEventInterest (CK)`).
- `viewerInterested` = whether the current viewer has marked Interested — drives the filled/outline heart on the `screen_23` row quick-toggle. (The More section row doesn't render the toggle, so it ignores this field.)
- Only `isPublished = true` events are returned.

**Errors:**

| Code | Scenario |
|------|----------|
| `UNAUTHENTICATED` | Caller is not logged in |
| `CITY_NOT_FOUND` | Unknown `cityCode` / `countryCode` |
| `CITY_NOT_AVAILABLE` | City has `status = COMING_SOON` (render locked state instead) |

---

### CL. Query: `Rescues`

> ⚠️ **GAP petapp-be#888:** backend has no browse-by-city rescue query (only `searchRescueCases(q!, lat/lng)`). Kept as-is, pending backend `rescuesByCity(... first, after)`.

Returns **open** rescue listings for a city (pets posted for adoption by **charity families**). Used by both the **More → Rescue section** (`first: 3`) and the full **Rescue screen** (`screen_25`, paginated + species filter + sort). All listings are created by charity families (`family.familyType = CHARITY`).

**Auth:** Required.

**Operation:**
```graphql
query Rescues(
  $cityCode: String!
  $countryCode: String!
  $filter: RescuesFilter
  $sort: RescuesSort        # NEAREST | NEWEST  (default NEAREST)
  $originLat: Float
  $originLng: Float
  $after: String
  $first: Int! = 20
) {
  rescues(
    cityCode: $cityCode
    countryCode: $countryCode
    filter: $filter
    sort: $sort
    originLat: $originLat
    originLng: $originLng
    after: $after
    first: $first
  ) {
    rescuesCount
    edges {
      cursor
      node {
        id
        name
        species
        breed
        ageText
        gender
        thumbnailUrl
        charity { id name avatarUrl }
        cityShortName
        countryCode
        lat
        lng
        distanceKm
        createdAt
      }
    }
    pageInfo {
      hasNextPage
      endCursor
    }
  }
}
```

> **Note:** `rescuesCount` is a sibling field on the query result (total number of matching rescue listings), not inside the connection — per ADR-0023.

**`RescuesFilter`** (species chip on the full screen; omit for "All"):
```json
{ "species": "CAT" }     // CAT | DOG | OTHER  (OTHER = any species not CAT/DOG)
```

**Variables (More section — preview):**
```json
{ "cityCode": "HCM", "countryCode": "VN", "sort": "NEAREST", "originLat": 10.78, "originLng": 106.70, "first": 3 }
```

**Response `200 OK`:**
```json
{
  "data": {
    "rescues": {
      "rescuesCount": 9,
      "edges": [
        {
          "cursor": "eyJpZCI6InJlc2N1ZV8wMDEifQ==",
          "node": {
            "id": "rescue_001",
            "name": "Miu",
            "species": "Cat",
            "breed": "Domestic shorthair",
            "ageText": "~4 months",
            "gender": "FEMALE",
            "thumbnailUrl": "https://cdn.petapp.com/rescues/rescue_001/1.jpg",
            "charity": {
              "id": "fam_paws",
              "name": "Paws Rescue Saigon",
              "avatarUrl": "https://cdn.petapp.com/families/fam_paws/avatar.jpg"
            },
            "cityShortName": "HCMC",
            "countryCode": "VN",
            "lat": 10.7820,
            "lng": 106.6960,
            "distanceKm": 2.3,
            "createdAt": "2026-06-07T09:00:00Z"
          }
        }
      ],
      "pageInfo": {
        "hasNextPage": true,
        "endCursor": "eyJpZCI6InJlc2N1ZV8wMDEifQ=="
      }
    }
  }
}
```

**Notes:**
- City-scoped (`cityCode` + `countryCode`); returns only `status = OPEN` listings (adopted/closed are excluded — like Lost Pets `found`).
- `sort`: `NEAREST` (distance asc) or `NEWEST` (`createdAt` desc). The More section always passes `NEAREST, first: 3`.
- `distanceKm` from `originLat`/`originLng` (user GPS); absent → server uses city centre.
- `breed` / `ageText` are optional (admin/charity entered) — skip null parts in the row's line 2.
- `charity` is always a `family.familyType = CHARITY` (only charity families can post) → CHARITY badge always shown.

**Errors:**

| Code | Scenario |
|------|----------|
| `UNAUTHENTICATED` | Caller is not logged in |
| `CITY_NOT_FOUND` | Unknown `cityCode` / `countryCode` |
| `CITY_NOT_AVAILABLE` | City has `status = COMING_SOON` (render locked state instead) |

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
                    │      ├─ ACTIVE       → set + fetch sections
                    │      └─ COMING_SOON → set + show LOCKED state
                    └─ denied/fail → default HCMC, VN → fetch sections
                          └─> fetch all 4 sections in parallel (each first: 3):
                                ├─ LostPets (CB)          { cityCode, countryCode }            (sorted reportedAt desc)
                                ├─ PetFriendlyPlaces (CD) { cityCode, countryCode, originLat?, originLng? }  (nearest)
                                ├─ Events (CI)            { cityCode, countryCode, originLat?, originLng? }  (soonest)
                                └─ Rescues (CL)           { cityCode, countryCode, originLat?, originLng?, sort: NEAREST }
                                   └─ each: items → up to 3 rows + "View All →"; empty → hide section/link
```

### Change city

```
User taps Change → Choose Your City sheet (Cities CA, cached)
  ├─ tap ACTIVE city → persist + close → refetch all sections for new city
  └─ tap COMING_SOON city → no-op (disabled)
```

### Enter Lost Pets

```
User taps "Lost Pets" category icon  OR  "View All →" in the section
  └─> Lost Pets screen (full list + map + filters), scoped to selected city

User taps a Lost Pet row (section)
  └─> Lost Pet Detail screen for that reportId
```

### Enter Events

```
User taps "Events" category icon  OR  "View All →" in the section
  └─> Events screen (screen_23), scoped to selected city → Events (CI)

User taps an Event row (section)
  └─> Event Detail (screen_24) for that eventId
```

### Enter Rescue

```
User taps "Rescue" category icon  OR  "View All →" in the section
  └─> Rescue screen (screen_25), scoped to selected city → Rescues (CL)

User taps a Rescue row (section)
  └─> Rescue Detail (screen_26) for that rescueId
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
| Pet Friendly section: 0 places in city | Hide section + "View All →" (or compact empty message) |
| Pet Friendly section: 1–3 places | Show all; "View All →" still shown |
| Events section: 0 upcoming events in city | Hide section + "View All →" (or compact empty message) |
| Events section: 1–3 upcoming events | Show all; "View All →" still shown |
| Event has ended (`endAt < now`) | Dropped from the section/list/detail listings (only upcoming/ongoing returned) |
| Rescue section: 0 open listings in city | Hide section + "View All →" (or compact empty message) |
| Rescue section: 1–3 open listings | Show all; "View All →" still shown |
| Rescue marked adopted (`status != OPEN`) | Dropped from the section/list (only open listings returned) |
| City list fetch fails | Fall back to HCMC default; allow retry on Change |

---

## Decisions Log

| # | Question | Decision |
|---|----------|----------|
| 1 | City reference data shape | Each city carries `lat`/`lng` (centre) + `status` (`ACTIVE`/`COMING_SOON`/`INACTIVE`); nearest computed client-side via haversine |
| 2 | Location permission denied | Default to HCMC, VN |
| 3 | Nearest city is `COMING_SOON` | Show the detected city but lock content ("coming soon"); prompt to Change |
| 4 | Manual selection of `COMING_SOON` city | Disabled in picker |
| 5 | Lost Pets section scope | Scoped to the **selected city** (not cross-city nearest); mockup showing Đà Lạt under HCMC is demo data only |
| 6 | Distance chips (`1km/5km`) | Filter **within** the selected city; require real GPS (`distanceKm`); disabled without GPS |
| 7 | Reporting entry point | Not in the More hub section — reports filed from `screen_9` (Pet Detail → Report Missing button, below the category tabs) or the Lost Pets screen banner (`screen_18`) |
| 8 | Pet Friendly | Active this phase — section (max 3 nearest in city) + full screen (`screen_20`) + place detail with reviews (`screen_21`). |
| 9 | City display label | `cityShortName` (curated short label, e.g. `HCMC` / `Đà Lạt` / `HN`) used in the location bar + lost-pet rows + Lost Pet Detail; `code` is the city key for args/logic; `nameVi`/`nameEn` are locale-aware full names shown in the picker |
| 10 | "Other" species filter | `LostPetsFilter.species` accepts an `OTHER` sentinel = any species not `CAT`/`DOG` |
| 11 | Events | Active this phase — section (max 3 soonest in city) + full screen (`screen_23`) + event detail (`screen_24`). **Admin-entered**, not charity/family-hosted. Sorted by `startAt` asc (upcoming only); distance is display-only, not a sort key. |
| 12 | Event countdown | `now < startAt` → `Starts in {N}d/{N}h`; `startAt ≤ now ≤ endAt` → `Ends in {N}h` / `Happening now`; ended events are filtered out |
| 13 | Rescue | Active this phase — section (max 3 nearest open in city) + full screen (`screen_25`) + detail/create (`screen_26`) + charity management (`screen_27`). Posted by **charity families** only; standalone listings (not named pets); closed when adopted (drops from listings, share link still resolves). All four More categories now ship. |

---

## Open Items (next steps)

- **Pins-only lightweight query** — optimisation for the shared Map View (`screen_28`) if a city's pin volume grows large.

> Done — all four categories ship, plus the shared full-screen map: Lost Pets (`screen_18`/`19`) + Report Missing upgrade (`screen_9`); Pet Friendly (`screen_20`/`21`); Events (`screen_23`/`24`); Rescue (`screen_25`/`26`/`27`); **Map View** (`screen_28`).
