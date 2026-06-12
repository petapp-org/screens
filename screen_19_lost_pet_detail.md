# Screen 19: Lost Pet Detail

## Overview

Full detail view of a **single missing-pet report**. Built to be **shareable and web-viewable** — anyone (including unauthenticated / web visitors) can open it via a shared link and read everything. Login is only required to **message the family** ("I saw …").

Navigated to from:
- Lost Pets section row on **More** (`screen_17`),
- Lost Pets list row or map pin on the **Lost Pets** screen (`screen_18`),
- a shared link / deep link (web or app).

> **Once the pet is marked found** (`screen_9` → Mark as Found, `status = FOUND`), the report is removed from **all in-app entry points** — it no longer appears in the More Lost Pets section, the Lost Pets list, or the map. From that point the **only way to open this screen is a previously-shared link / deep link**, and it renders the **found state** (see Section 2 / Edge Cases). The report is never hard-deleted, so old links keep resolving.

---

## UI Layout

```
[Header]
  Left: Back button
  Center: pet name (static)

[Header photo carousel]                          ← all report photos, cover (photos[0]) first
  [ ◀   photo   ▶ ]   1/3
  · swipeable; N/Total indicator top-right
  · tap → fullscreen zoom; tap outside / × → exit

[Status + identity block]
  [📍 MISSING]                            Yesterday, 5:30 PM   ← report time
  Măng                                                         ← pet name (large)
  [family avatar]  Minh's Family
  Reported missing from you                                    ← family members only

[LAST SEEN]
  Location   Đà Lạt, VN
  When       Yesterday, May 9 · 5:30 PM
  [ map with pin at lat/lng ]

[DESCRIPTION]
  Buckskin Vietnamese pony, dark mane and tail, friendly with
  people. Was wearing green halter rope. Responds to name "Măng".

[HOW YOU CAN HELP]
  1  If you spot Măng, do not chase. Stay calm and call to gain trust.
  2  Take a photo from a distance and share location.
  3  Message Minh's Family directly through PetApp.

[Fixed bottom action bar]
  [ Share ]            [ ✓ I saw Măng ]
```

> No separate "More photos" section — all photos live in the **header carousel** (decision, merged with the cover). Since reporting requires **≥ 1 photo** (`screen_9` §8), the header always has at least one image; the carousel only scrolls when there are 2+.

---

## Components

### 1. Header Photo Carousel

- Renders `photos[]` from the report, **cover (`photos[0]`) first**, in order.
- Swipeable; `N/Total` indicator in the top-right when there are 2+ photos.
- **Tap a photo → fullscreen zoom** (lightbox): pinch/zoom, swipe between photos; **tap outside / ×** to exit.
- Single photo → static image, no swipe.

---

### 2. Status + Identity Block

| Element | Display |
|---------|---------|
| Missing badge | `📍 MISSING` pill (badge text is **"MISSING"** — no "Report") |
| Report time | `reportedAt`, shown to the right of the badge as an absolute-ish string (e.g. `"Yesterday, 5:30 PM"`, `"May 9, 5:30 PM"`) |
| `pet.name` | Large heading |
| `family.avatarUrl` + `family.name` | Family that owns the pet; **tap family name → Family Posts** (`screen_3`) / My Pets if own family |

---

### 3. "Reported missing from …" line

Shown **only to members of the pet's family** (`viewerIsFamilyMember = true`). Hidden entirely for everyone else (non-members, unauthenticated).

| Viewer | Text |
|--------|------|
| The reporter themselves | `"Reported missing from you"` |
| Another family member | `"Reported missing from {reporter.displayName}"` |
| Non-member / not logged in | *(line hidden)* |

Source: `reportedBy` on the report — returned by the API **only** to family members; `null` otherwise.

---

### 4. Last Seen

| Field | Display |
|-------|---------|
| Location | `"{cityShortName}, {countryCode}"` (e.g. `"Đà Lạt, VN"`) — from `lastSeen.cityShortName` / `countryCode` (consistent with the More tab + Lost Pets rows) |
| When | `lastSeen.at` as a full datetime (e.g. `"Yesterday, May 9 · 5:30 PM"`) — this is **when the pet was last seen**, distinct from the report time in Section 2 |
| Map | A map with a single pin at `lastSeen.lat` / `lastSeen.lng`; tappable to open in the device map app / fullscreen |

---

### 5. Description

- Renders the report's `description` (required field — always present).
- Plain text, full (no truncation).

---

### 6. How You Can Help

**Fixed, client-side text** (not from the API) — 3 numbered tips, with the pet name and family name interpolated:

```
1  If you spot {pet.name}, do not chase. Stay calm and call to gain trust.
2  Take a photo from a distance and share location.
3  Message {family.name} directly through PetApp.
```

---

### 7. Action Bar (fixed bottom)

| Action | Behaviour | Visibility |
|--------|-----------|------------|
| **Share** | Copy the report's canonical web URL (`shareUrl`) to clipboard → toast *"Link copied"*. (On platforms with a native share sheet, may invoke it instead.) The link opens this same detail on web. | Always |
| **I saw {pet.name}** | Message the pet's **family**. Tap → if not logged in → **redirect to Login**; else open a thread toward the family via `StartThread (BI)` (see `screen_10`) — sender selection ("Send as…": self or active family) applies; the composer is **pre-filled** with *"I think I saw {pet.name}!"*. Reuses an existing thread if one exists. | **Hidden for members of the pet's family** (you can't message your own family — `screen_10` rule). Shown to everyone else; tap while logged out → Login. |

- When "I saw …" is hidden (family member), the **Share** button spans the full width.

---

## API Endpoints Required

> All calls go to `POST /graphql`.
> Messaging reuses `StartThread (BI)` from `screen_10`. New query below.

---

### CC. Query: `GetLostPetReport`

> ⚠️ **GAP petapp-be#888:** `getLostPetReport(id)` exists but returns a flat legacy `LostPetReportGQL` (petName/species/breed as strings, `area`, single `photoUrl`) — not the structured shape here (nested pet/family/reportedBy, geo `lastSeen`, `photos[]`). Kept as-is, pending backend enrichment.

Fetch a single missing-pet report for the detail screen.

**Auth:** Optional — the report is public/web-viewable. When a valid token is present and the caller is a family member, `reportedBy` and `viewerIsFamilyMember` are populated.

**Operation:**
```graphql
query GetLostPetReport($id: ID!) {
  getLostPetReport(id: $id) {
    reportId
    status
    reportedAt
    reportedBy {
      id
      displayName
      avatarUrl
    }
    viewerIsFamilyMember
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
    description
    photos
    shareUrl
  }
}
```

**Variables:**
```json
{ "id": "missing_report_001" }
```

**Response `200 OK`:**
```json
{
  "data": {
    "getLostPetReport": {
      "reportId": "missing_report_001",
      "status": "MISSING",
      "reportedAt": "2026-06-08T03:00:00Z",
      "reportedBy": {
        "id": "user_001",
        "displayName": "Minh Tuan",
        "avatarUrl": "https://cdn.petapp.com/users/user_001/avatar.jpg"
      },
      "viewerIsFamilyMember": true,
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
      "description": "Buckskin Vietnamese pony, dark mane and tail, friendly with people. Was wearing green halter rope. Responds to name \"Măng\".",
      "photos": [
        "https://cdn.petapp.com/media/miss_cover.jpg",
        "https://cdn.petapp.com/media/miss_2.jpg",
        "https://cdn.petapp.com/media/miss_3.jpg"
      ],
      "shareUrl": "https://petapp.com/lost/missing_report_001"
    }
  }
}
```

**Notes:**
- `status`: `MISSING` | `FOUND`. A `FOUND` report still resolves (deep links don't 404) but renders the found state (see Edge Cases).
- `reportedBy` is returned **only** when the caller is a family member; `null` otherwise. `viewerIsFamilyMember` is `false` for unauthenticated callers.
- `photos` is ordered, `photos[0]` = cover; always ≥ 1 item.
- `shareUrl` is the canonical web URL used by the Share action.

**Errors:**

| Status | Code | Scenario |
|--------|------|----------|
| `404` | `REPORT_NOT_FOUND` | Report does not exist |

> **"I saw …" messaging:** uses `StartThread (BI)` (`screen_10`) — `{ receiverType: FAMILY, receiverId: family.id, senderType: USER | FAMILY }`, optional pre-filled body. No new endpoint. ⏳ GAP — `StartThread (BI)` chưa có ở backend (messaging chưa build, petapp-be#831, epic #403).

---

## User Flow Diagrams

### Open detail

```
Tap a Lost Pet row / map pin / shared link
  └─> GetLostPetReport query (CC) { id }
        ├─ REPORT_NOT_FOUND → "This report no longer exists" state
        ├─ status=FOUND → render found state
        └─ status=MISSING → render full detail
              └─ viewerIsFamilyMember → show "Reported missing from {you/member}"; hide "I saw"
              └─ else → hide that line; show "I saw {pet}"
```

### Share

```
Tap [Share]
  └─> copy shareUrl to clipboard (or native share sheet) → toast "Link copied"
```

### I saw {pet}

```
Tap [I saw {pet}]
  ├─ not logged in → redirect to Login (return back here after)
  └─ logged in (non-member)
        └─> "Send as…" (self / active family) → StartThread (BI) { receiver: family }
              prefilled "I think I saw {pet}!" → open thread (screen_10 Thread View)
```

---

## Edge Cases & Notes

| Case | Expected Behaviour |
|------|--------------------|
| Unauthenticated / web visitor | Reads everything; "Reported missing from…" line hidden; tapping "I saw" → Login |
| Viewer is a family member | Show "Reported missing from {you/member}"; **hide** "I saw" (Share goes full-width) |
| Report `status = FOUND` | Reachable **only via a shared link / deep link** (no in-app entry points list found reports). Show a found state (e.g. *"🎉 {pet} has been found"*); photos/description still visible; "I saw" hidden/disabled; Share still works |
| Single photo | Header shows one static image, no carousel swipe |
| Multiple photos | Header carousel with `N/Total`; tap → fullscreen zoom + swipe |
| Tap photo | Fullscreen lightbox; tap outside / × to close |
| Tap last-seen map | Open device map / fullscreen at the pin |
| Tap family name | Family Posts (`screen_3`), or My Pets if it's the viewer's own family |
| Report deleted / resolved while open | On refresh → not-found or found state |
| Deep link, report exists | Loads directly via CC; Back returns to previous screen, or Explore/More if no history |

---

## Decisions Log

| # | Question | Decision |
|---|----------|----------|
| 1 | Visibility | Public / **web-viewable** (optional auth); deep-linkable via `shareUrl` |
| 2 | Photos layout | Single **header carousel**, cover (`photos[0]`) first; no separate "more photos" section |
| 3 | Badge text | `MISSING` only (drop the "Report" word from the mockup) |
| 4 | Two timestamps | Top badge row = `reportedAt`; "Last seen → When" = `lastSeen.at` |
| 5 | "Reported missing from …" | Shown to **family members only**; "you" if viewer is the reporter, else the reporter's name |
| 6 | How you can help | **Fixed client-side text**, pet/family names interpolated; not from API |
| 7 | Share | Copy canonical `shareUrl` to clipboard (native share sheet where available) |
| 8 | "I saw {pet}" | `StartThread (BI)` to the family with a pre-filled message; **hidden for family members**; login required |
| 9 | Found reports | Removed from all in-app entry points (More section, Lost Pets list, map); reachable **only via shared/deep link**; never hard-deleted so links keep resolving; render the found state instead of the active layout |

---

## Open Items (next steps)

- **Full-screen map** — shared **Map View** (`screen_28`), opened via `[Open map]` on the Lost Pets list (`screen_18`); not specific to this screen.
- Other More categories: **Rescue** — future phase (Pet Friendly `screen_20`/`21` and Events `screen_23`/`24` are done).
