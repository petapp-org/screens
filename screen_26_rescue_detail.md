# Screen 26: Rescue Detail

## Overview

Full detail view of a **single rescue listing** — a pet a **charity family** posted for adoption. Built to be **shareable and web-viewable**: anyone (including unauthenticated / web visitors) can open it via a shared link and read everything, to help spread the word. Login is required only to **act** (Inquire to Adopt / Donate).

This screen also hosts the **Create Rescue Form** (Section 8) — the form charity members use to post a new listing, shared by the Rescue screen banner (`screen_25`) and Manage Rescues (`screen_27`).

Navigated to from:
- Rescue section row on **More** (`screen_17`),
- Rescue list row or map pin on the **Rescue** screen (`screen_25`),
- a Manage Rescues row (`screen_27`),
- a shared link / deep link (web or app).

> **Once the listing is marked adopted** (`status = ADOPTED`, see `screen_27`), it is removed from **all in-app entry points** — it no longer appears in the More Rescue section, the Rescue list, or the map. From that point the **only way to open this screen is a previously-shared link / deep link**, and it renders the **adopted state**. The listing is never hard-deleted, so old links keep resolving.

---

## Rescue Listing Data Model

A rescue listing is a **standalone entity** created by a charity family — **not** a named family pet (no health tabs, no AI, no posts). The rows (`screen_17` §8 / `screen_25` / `screen_27`) and this screen read from it.

| Group | Field | Req | Notes |
|-------|-------|-----|-------|
| **Basics** | `name` | — | Pet's name; blank → render `"Chưa đặt tên"` |
| | `photos[]` | ✅ | ≥ 1; `photos[0]` = cover; rest in the detail carousel; uploaded only, **no AI scan** |
| | `species` | ✅ | `CAT` / `DOG` / `OTHER` → drives the species filter |
| | `breed` | — | Free text |
| | `ageText` | — | Free text estimate, e.g. `"~4 months"` |
| | `gender` | — | `MALE` / `FEMALE` / `UNKNOWN` |
| | `goodToKnow` | — | **Free text** flair (≤ 60 chars) — adoption-relevant notes; admin hint suggests `·`-separated traits (see Section 8) |
| | `story` | ✅ | Longer free text — background, temperament, adoption notes |
| **Location** | `city` | ✅ | Defaults to the charity family's city; editable. Picked from supported Cities (`CA`) → derives `cityShortName`/`cityCode`/`country`/`countryCode` |
| | `lat` / `lng` | ✅ | Map pin → distance; defaults to charity location, draggable |
| **Owner** | `charity` | ✅ | The posting charity family (`type = charity`); rendered with CHARITY badge |
| **Status** | `status` | ✅ | `OPEN` / `ADOPTED` — only `OPEN` appears in public listings |
| **Computed** | `distanceKm` | — | From caller origin (GPS or city centre); display-only |
| | `inquiriesCount` | — | Distinct users who tapped Inquire to Adopt (see `InquireRescue (CR)`) |
| | `viewerIsCharityMember` | — | True when the viewer belongs to the posting charity |
| | `shareUrl` | — | Canonical web URL for sharing |

---

## UI Layout

```
[Header]
  Left: Back button
  Center: pet name (static)

[Photo carousel]                                 ← photos[], cover first, swipeable, tap → zoom
  [ ◀   photo   ▶ ]   1/4

[Identity block]
  Miu
  Cat · Domestic shorthair · ~4 months · ♀
  Vaccinated · Neutered · Litter-trained        ← goodToKnow (free text)

[STORY]
  Miu được cứu từ một khu chợ, rất quấn người…

[RESCUE BY]
  [avatar]  Paws Rescue Saigon  🏷CHARITY        ← tap → Family Posts (screen_3)
  📍 Quận 3, HCMC
  ┌──────────────────────────────────┐
  │     (map, pin tại lat/lng)        │   2.3km        ← tap → open device map
  └──────────────────────────────────┘

[Fixed bottom action bar]   ← varies by viewer (see Section 7)
  [ Donate ]                 [ ✓ Inquire to Adopt ]
```

> The location is the charity's **city label + a map pin** at `lat`/`lng` (consistent with all other More screens). There is no street address field for rescues — the pin marks the pickup/charity location; tapping the map opens the device map app.

---

## Components

### 1. Photo Carousel

- Renders `photos[]`, cover (`photos[0]`) first; swipeable; `N/Total` indicator when 2+.
- Tap a photo → fullscreen zoom (lightbox); tap outside / × to exit.
- Single photo → static image. (Posting requires ≥ 1 photo, so there's always at least one.)

---

### 2. Identity Block

| Element | Display |
|---------|---------|
| `name` | Large heading; `"Chưa đặt tên"` if blank |
| species · breed · age · gender | `"Cat · Domestic shorthair · ~4 months · ♀"` — ` · `-joined, skip null parts |
| `goodToKnow` | Muted line of adoption-relevant traits (free text); omitted if empty |

---

### 3. Story

- Renders `story` (required field — always present). Plain text, full (no truncation).

---

### 4. Rescue By

| Element | Display |
|---------|---------|
| `charity.avatarUrl` + `charity.name` | The posting charity family, with **CHARITY** badge; **tap name → Family Posts** (`screen_3`) |
| `cityShortName` | `📍 {cityShortName}` (e.g. `"Quận 3, HCMC"` style city label) |
| Map | A map with a pin at `lat` / `lng`; **tap → open the device map app**. `distanceKm` shown by the map |

---

### 5. Adopted State

When `status = ADOPTED` (reached only via a shared/deep link — no in-app entry lists adopted listings):
- Show a celebratory banner, e.g. *"🏡 Miu has found a home"*.
- Photos / story / charity still visible.
- **Inquire to Adopt** is hidden/disabled; **Donate** still works; the listing stays web-viewable.

---

### 6. Inquiries (count)

- `inquiriesCount` = number of distinct users who tapped **Inquire to Adopt** on this listing.
- Shown to the posting charity's members (and surfaced in **Manage Rescues**, `screen_27`). Not shown to outside viewers on the public detail (kept clean) — it's a signal for the charity.

---

### 7. Action Bar (fixed bottom)

The bar adapts to the viewer:

| Viewer | Buttons |
|--------|---------|
| **Not logged in** (shared link / web) | `[ Donate ]` + `[ Inquire to Adopt ]` — tapping **either** → **redirect to Login** (return here after) |
| **Logged in, outside viewer** | `[ Donate ]` + `[ Inquire to Adopt ]` |
| **Charity member, this charity is active** | `[ Edit ]` + `[ Mark Adopted ]` (manage mode — see `screen_27`) |
| **Charity member, this charity NOT active** | Read-only (no action buttons) — must switch active family to the charity to manage |

**Inquire to Adopt** (`InquireRescue (CR)`):
- Records the viewer's inquiry (idempotent — one per user per listing → feeds `inquiriesCount`) **and** opens/returns a message thread to the charity family, pre-filled *"Mình muốn tìm hiểu nhận nuôi {name} 🐾"*.
- Sender selection ("Send as…": self or active family) applies, same as any family-addressed thread (`screen_10`).
- Reuses an existing thread if one exists (no duplicate; re-tapping doesn't double-count).
- **Hidden** for members of the posting charity (you can't inquire about your own listing — mirrors the Lost Pet "I saw" rule).

**Donate**:
- Opens the family's **Donate screen** for the posting charity (currently a "Coming Soon" page, same as the Donate button elsewhere — `screen_3` / `screen_5`).
- Tapping while logged out → **redirect to Login**.

**Mark Adopted** / **Edit** (charity-active viewer): see `screen_27` (`CloseRescue (CP)` / Edit via `UpdateRescue (CS)`).

---

### 8. Create Rescue Form

A **full-screen form** (`← Post a Rescue`). Opened from the Rescue screen banner (`screen_25`) or Manage Rescues (`screen_27`). Used by **charity family members** while that charity is their **active family**. The same form, pre-filled, is the **Edit** surface.

```
[Header]  ← Post a Rescue

PHOTOS *                                      (at least 1)
  [ + ]  [ photo① ★cover ]  [ photo② ]  →  scroll
  · first photo = cover; tap a photo → "Set as cover"; reorderable; 1–10; uploaded only, no AI scan

NAME            [ Miu ]                        ← optional; blank → "Chưa đặt tên"
SPECIES *       ( Cat / Dog / Other )
BREED           [ Domestic shorthair ]        ← optional
AGE (estimate)  [ ~4 months ]                 ← optional free text
GENDER          ( ♀ / ♂ / Unknown )           ← optional

GOOD TO KNOW    [ Vaccinated · Neutered · Litter-trained ]
  hint: "Tối đa 60 ký tự. Vài điểm cần biết khi nhận nuôi, cách nhau bằng ` · `.
         VD: `Vaccinated · Neutered`, `Good with kids`, `Needs quiet home`."

STORY *
  [ multiline — hoàn cảnh, tính cách, lưu ý khi nhận nuôi… ]

LOCATION *      [ city — default theo charity, đổi được ]
                [ map — drag pin 📍 ] → lat/lng (default = charity location)

[ Submit ]   ← full-width; disabled until ALL required fields set
```

| Field | Required | Notes |
|-------|----------|-------|
| `photos` | **Yes** | ≥ 1, up to 10; ordered, `photos[0]` = cover; reorderable; no AI scan |
| `name` | No | Blank → `"Chưa đặt tên"` |
| `species` | **Yes** | `CAT` / `DOG` / `OTHER` |
| `breed` | No | Free text |
| `ageText` | No | Free text estimate |
| `gender` | No | `MALE` / `FEMALE` / `UNKNOWN` |
| `goodToKnow` | No | **Free text ≤ 60**, single-line with live counter + the hint above; rendered single-line + `…` |
| `story` | **Yes** | Free text — background / temperament / adoption notes; non-empty |
| `location` | **Yes** | `city` defaults to the charity's city (editable) + draggable pin → `lat`/`lng` (default = charity location) |

> **Submit** enabled only when required fields are set: ≥ 1 photo, species, a non-empty story, and a location.

**Upload:** each photo via `RequestMediaUpload (BV)` `{ purpose: "RESCUE_PHOTO" }` (`screen_4`) → use the returned `publicUrl`; no AI scan.

**On submit → `CreateRescue (CO)`** (or `UpdateRescue (CS)` in edit mode):
- Listing is created with `status = OPEN`, `charity` = the active charity family, `createdAt` = now.
- Appears in the Rescue section / list / map for its city, and in Manage Rescues (`screen_27`).

---

## API Endpoints Required

> All calls go to `POST /graphql`.
> Reuses `Rescues (CL)` (`screen_17`) for listings, `RequestMediaUpload (BV)` (`screen_4`) for photos. New endpoints below. Messaging for Inquire builds on `StartThread (BI)` (`screen_10`).

---

### CM. Query: `Rescue`

Fetch a single rescue listing for the detail screen.

**Auth:** Optional — the listing is public/web-viewable. When a valid token is present and the caller belongs to the posting charity, `viewerIsCharityMember` is `true` and `inquiriesCount` is populated.

**Operation:**
```graphql
query Rescue($rescueId: ID!, $originLat: Float, $originLng: Float) {
  rescue(rescueId: $rescueId, originLat: $originLat, originLng: $originLng) {
    id
    status            # OPEN | ADOPTED
    name
    photos
    species
    breed
    ageText
    gender
    goodToKnow
    story
    charity { id name avatarUrl }
    cityShortName
    cityCode
    country
    countryCode
    lat
    lng
    distanceKm
    inquiriesCount
    viewerIsCharityMember
    viewerHasInquired
    shareUrl
    createdAt
  }
}
```

**Response `200 OK`:**
```json
{
  "data": {
    "rescue": {
      "id": "rescue_001",
      "status": "OPEN",
      "name": "Miu",
      "photos": [
        "https://cdn.petapp.com/rescues/rescue_001/1.jpg",
        "https://cdn.petapp.com/rescues/rescue_001/2.jpg"
      ],
      "species": "Cat",
      "breed": "Domestic shorthair",
      "ageText": "~4 months",
      "gender": "FEMALE",
      "goodToKnow": "Vaccinated · Neutered · Litter-trained",
      "story": "Miu được cứu từ một khu chợ, rất quấn người…",
      "charity": {
        "id": "fam_paws",
        "name": "Paws Rescue Saigon",
        "avatarUrl": "https://cdn.petapp.com/families/fam_paws/avatar.jpg"
      },
      "cityShortName": "HCMC",
      "cityCode": "HCM",
      "country": "Việt Nam",
      "countryCode": "VN",
      "lat": 10.7820,
      "lng": 106.6960,
      "distanceKm": 2.3,
      "inquiriesCount": 12,
      "viewerIsCharityMember": false,
      "viewerHasInquired": false,
      "shareUrl": "https://petapp.com/rescue/rescue_001",
      "createdAt": "2026-06-07T09:00:00Z"
    }
  }
}
```

**Notes:**
- `status = ADOPTED` still resolves (deep links don't 404) but renders the adopted state.
- `inquiriesCount` / `viewerIsCharityMember` are meaningful for the charity; `viewerHasInquired` drives the inquired affordance for outside viewers.

**Errors:**

| Status | Code | Scenario |
|--------|------|----------|
| `404` | `RESCUE_NOT_FOUND` | Listing does not exist |

---

### CO. Mutation: `CreateRescue`

Create a new rescue listing. **Auth:** Required — caller must be a member of a **charity** family **and that charity is their active family**.

**Operation:**
```graphql
mutation CreateRescue($input: RescueInput!) {
  createRescue(input: $input) {
    id
    status
  }
}
```

**`RescueInput`:**
```json
{
  "name": "Miu",
  "photos": ["https://cdn.petapp.com/media/r1.jpg", "https://cdn.petapp.com/media/r2.jpg"],
  "species": "CAT",
  "breed": "Domestic shorthair",
  "ageText": "~4 months",
  "gender": "FEMALE",
  "goodToKnow": "Vaccinated · Neutered · Litter-trained",
  "story": "Miu được cứu từ một khu chợ…",
  "location": { "city": "Ho Chi Minh City", "cityCode": "HCM", "country": "Việt Nam", "countryCode": "VN", "lat": 10.7820, "lng": 106.6960 }
}
```

**Side effects:** listing created `status = OPEN`, owned by the active charity family; appears in city listings + Manage Rescues.

**Errors:**

| Status | Code | Scenario |
|--------|------|----------|
| `401` | `UNAUTHENTICATED` | Not logged in |
| `403` | `NOT_CHARITY_FAMILY` | Active family is not a charity (or caller not a member) |
| `422` | `MISSING_PHOTOS` | No photo provided (≥ 1 required) |
| `422` | `MISSING_SPECIES` | Species not provided |
| `422` | `MISSING_STORY` | Story empty |
| `422` | `MISSING_LOCATION` | Location (lat/lng) not provided |

---

### CR. Mutation: `InquireRescue`

Register the caller's interest to adopt **and** create/return the message thread to the charity. **Auth:** Required.

> **Triggers notification:** on the caller's **first** inquiry for this listing, fires a `RESCUE_INQUIRY` notification to the posting charity's members (see `screen_22`). Idempotent — re-inquiring does not re-notify.

**Operation:**
```graphql
mutation InquireRescue($input: InquireRescueInput!) {
  inquireRescue(input: $input) {
    rescueId
    thread { id }          # thread to the charity (existing or newly created)
    inquiriesCount
    viewerHasInquired
  }
}
```

**`InquireRescueInput`:**
```json
{ "rescueId": "rescue_001", "senderType": "USER" }   // senderType: USER | FAMILY (Send as…)
```

**Behaviour:**
- Upserts a `RescueInquiry { rescueId, userId }` row — **idempotent** (one per user per listing); re-inquiring does not increase `inquiriesCount`.
- Creates or reuses a thread to the posting charity (`StartThread (BI)` semantics), to be opened in the **Messages** Thread View (`screen_10`) and pre-filled client-side with *"Mình muốn tìm hiểu nhận nuôi {name} 🐾"*.
- Returns the updated `inquiriesCount` and `thread.id` to navigate to.

**Errors:**

| Status | Code | Scenario |
|--------|------|----------|
| `401` | `UNAUTHENTICATED` | Not logged in |
| `403` | `OWN_LISTING` | Caller belongs to the posting charity (can't inquire about own listing) |
| `404` | `RESCUE_NOT_FOUND` | Listing does not exist |
| `409` | `RESCUE_CLOSED` | Listing already adopted/closed |

---

### Charity management mutations (defined in `screen_27`)

`CloseRescue (CP)` marks a listing `ADOPTED`; `ReopenRescue (CQ)` reverses it; `UpdateRescue (CS)` edits a listing via the Create Rescue form in edit mode. All require the caller to be a member of the posting charity **with that charity active**. Full definitions live in `screen_27`.

> **Inquire messaging** lands in **Messages** (`screen_10`) → `{ receiverType: FAMILY, receiverId: charity.id, senderType: USER | FAMILY }`, optional pre-filled body. No new messaging endpoint.

---

## User Flow Diagrams

### Open detail

```
Tap a Rescue row / map pin / shared link
  └─> Rescue query (CM) { rescueId }
        ├─ RESCUE_NOT_FOUND → "This listing no longer exists"
        ├─ status=ADOPTED → render adopted state ("🏡 found a home")
        └─ status=OPEN → render full detail
              ├─ viewerIsCharityMember + charity active → [Edit] [Mark Adopted]
              ├─ viewerIsCharityMember + not active     → read-only
              └─ else → [Donate] [Inquire to Adopt]
```

### Inquire to adopt

```
Tap [Inquire to Adopt]
  ├─ not logged in → redirect to Login (return here after)
  └─ logged in (outside viewer)
        └─> "Send as…" (self / active family) → InquireRescue (CR) { rescueId, senderType }
              └─> record inquiry (idempotent) + open thread (screen_10), prefilled
```

### Post a rescue (charity)

```
[Post a Rescue] (screen_25 banner / screen_27) → Create Rescue Form
  └─> upload photos (RequestMediaUpload, RESCUE_PHOTO)
        └─> CreateRescue (CO) → listing OPEN → visible in listings + Manage Rescues
```

---

## Edge Cases & Notes

| Case | Expected Behaviour |
|------|--------------------|
| Unauthenticated / web visitor | Reads everything; tapping Inquire or Donate → Login |
| Viewer is charity member (active) | Inquire/Donate hidden → show Edit + Mark Adopted |
| Viewer is charity member (not active) | Read-only; switch active family to manage |
| `status = ADOPTED` | Reachable only via shared/deep link; adopted banner; Inquire hidden; Donate works; Share resolves |
| No name | Render `"Chưa đặt tên"` |
| No breed / age | Skip those parts on the identity line |
| `goodToKnow` empty | Omit that line |
| Single photo | Static image, no carousel swipe |
| Re-tap Inquire | Idempotent — reuses thread, no extra inquiry counted |
| Inquire on a just-closed listing | `RESCUE_CLOSED` → toast "This pet has been adopted" |
| Tap charity name | Family Posts (`screen_3`) |
| GPS unavailable | `distanceKm` from city centre (display only; not shown on this screen) |

---

## Decisions Log

| # | Question | Decision |
|---|----------|----------|
| 1 | Entity | Standalone **rescue listing** (not a named family pet); no health/AI/posts |
| 2 | Who can post | **Charity families only** (`type = charity`), while that charity is the active family |
| 3 | Visibility | Public / web-viewable (optional auth); deep-linkable via `shareUrl` |
| 4 | Acting requires login | Inquire / Donate → redirect to Login when logged out |
| 5 | Inquire | `InquireRescue (CR)` — records an idempotent inquiry (feeds `inquiriesCount`) **and** opens a thread to the charity (`screen_10`), pre-filled; hidden for own-charity members |
| 6 | Donate | Reuses the charity's Donate screen (currently "Coming Soon") |
| 7 | Map | **Map with a pin** at `lat`/`lng` (consistent with all other More screens); tap → open device map. No street-address field (city label + pin only); no separate Directions button |
| 8 | goodToKnow | **Free text ≤ 60** with admin hint + counter (not a fixed checklist); single-line `…` |
| 9 | Adopted | `status = ADOPTED` removes it from all in-app entry points; reachable only via shared/deep link; never hard-deleted; renders adopted state |
| 10 | Location default | Defaults to the charity family's city + location; editable (city + draggable pin) |

---

## Open Items (next steps)

- **Full-screen map** — the detail shows a single-pin map; the shared **Map View** (`screen_28`) is opened via `[Open map]` on the Rescue list (`screen_25`).
- **Adoption application / structured form** — only free-text Inquire (message) this phase.
- **Admin moderation** of charity-posted listings — out of scope of these client specs.
