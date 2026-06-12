# Screen 24: Event Detail

## Overview

Detail view of a **single event** — its photos, time, full address, and description, plus an **Interested** action. A simple **read-only display** screen: everything shown is **admin-entered** (no AI, no charity/family hosting, no reviews).

Navigated to from:
- Events section row on **More** (`screen_17`),
- Events list row on the **Events** screen (`screen_23`).

Requires login (it lives under the More tab). The only interactive element is **Interested** (toggle + optional add-to-calendar).

> **No share** this phase (decision). The screen shows the event info, the **full address** (which the rows only abbreviate to `venueName · city`), and a **map** with a pin at the venue (consistent with all other More screens).

---

## Event Data Model (admin-entered)

Events are **created/edited by admins** (no AI generation, no external import, not tied to any family/charity). This is the canonical field set; the rows (`screen_17` §7 / `screen_23`) and this screen read from it.

| Group | Field | Req | Notes |
|-------|-------|-----|-------|
| **Basics** | `title` | ✅ | Display name |
| | `photos[]` | ✅ | ≥ 1; first photo = thumbnail/cover; rest shown in the detail carousel |
| | `description` | — | Longer free text, shown in the About section |
| **Time** | `startAt` | ✅ | ISO datetime — drives date-chip, time row, countdown, sort, time-window filters |
| | `endAt` | ✅ | ISO datetime — drives countdown (`Ends in …`) and the "ended → hidden" rule |
| | `price` | — | Free text (e.g. `"200kđ"`); empty → treated as Free |
| | `isFree` | ✅ | `true` → render `"Free"` + power the **Free** filter chip on `screen_23` |
| **Location** | `venueName` | ✅ | e.g. `"Lava Cat Coffee"` |
| | `address` | ✅ | **Full address** (e.g. `"12 Nguyễn Huệ, Bến Nghé, Quận 1, HCMC"`) — shown only on this detail screen |
| | `city` | ✅ | Picked from supported Cities (`CA`) → derives `cityShortName` / `cityCode` / `country` / `countryCode` |
| | `lat` / `lng` | ✅ | Map pin (detail) + distance computation |
| **Status** | `isPublished` | ✅ | Hidden from all listings when `false` |
| **Computed** | `distanceKm` | — | From the caller's origin (GPS or city centre); display-only |
| | `interestedCount` | — | Number of users who marked Interested |
| | `viewerInterested` | — | Whether the current viewer has marked Interested |

---

## UI Layout

```
[Header]
  Left: Back button
  Center: event title (static)

[Photo carousel]                                 ← photos[], swipeable, tap → zoom
  [ ◀   photo   ▶ ]   1/3

[Title block]
  Cat Adoption Day                          [ ⏳ Starts in 2d ]   ← countdown pill
  🗓️ Sat, Jun 15 · 9:00–12:00
  💵 Free

[LOCATION]
  📍 Lava Cat Coffee
     12 Nguyễn Huệ, Bến Nghé, Quận 1, HCMC          ← full address (admin)
  ┌──────────────────────────────────┐
  │     (map, pin tại lat/lng)        │   3.0km        ← tap → open device map
  └──────────────────────────────────┘

[ABOUT]
  Ngày hội nhận nuôi mèo do PetApp tổ chức.
  Mang theo CMND nếu muốn nhận bé về…               ← description (full text)

[Fixed bottom bar]
  [ ♡  Interested · 48 ]                            ← full-width; count on the button
```

---

## Components

### 1. Photo Carousel

- Renders `photos[]`, swipeable, `N/Total` indicator.
- Tap a photo → fullscreen zoom (lightbox); tap outside / × to exit.
- Single photo → static image, no swipe.

---

### 2. Title Block

| Element | Display |
|---------|---------|
| `title` | Event name (large) |
| Countdown pill | Top-right — `Starts in {N}d` / `Starts in {N}h` / `Ends in {N}h` / `Happening now` (see **Countdown rule**, `screen_17` §7) |
| Date & time | `🗓️ {weekday}, {Month} {day} · {startTime}–{endTime}` from `startAt` / `endAt` |
| Price | `💵 {price}`; `isFree` → `"Free"` |

---

### 3. Location

| Element | Display |
|---------|---------|
| `venueName` | `📍 {venueName}` (bold) |
| `address` | **Full address** on its own line (the rows only show `venueName · city`; the detail is where the complete address appears) |
| Map | A map with a pin at `lat` / `lng`; **tap → open the device map app** at the venue |
| `distanceKm` | Shown by the map; `< 10` → 1 decimal, `≥ 10` → integer; **display-only** |

---

### 4. About

- Renders the event `description` (free text, full — no truncation).
- Omitted entirely if `description` is empty.

---

### 5. Interested (fixed bottom bar)

A single full-width button combining the action and the count.

```
[ ♡  Interested · 48 ]      ← not yet interested (outline heart)
[ ♥  Interested · 49 ]      ← interested (filled heart)
```

| State | Button | Count |
|-------|--------|-------|
| Not interested | outline heart `♡` | `interestedCount` |
| Interested | filled heart `♥` | `interestedCount` (already +1 for the viewer) |

**Interaction:**

```
Tap [ ♡ Interested · 48 ]
  │
  ├─ off → on:  SetEventInterest (CK) { eventId, interested: true }   (optimistic)
  │        button → [ ♥ Interested · 49 ]
  │        └─> Add-to-calendar prompt:
  │            ┌─────────────────────────────────────┐
  │            │  Added to your interested list 🐾    │
  │            │  Add this event to your calendar?   │
  │            │     [ Not now ]      [ Add 🗓️ ]      │
  │            └─────────────────────────────────────┘
  │              ├─ Add    → deep-link to the device calendar app,
  │              │           prefilled { title, startAt, endAt, venueName + address }
  │              └─ Not now → dismiss; interest stays ON
  │
  └─ on → off:  SetEventInterest (CK) { eventId, interested: false }  (optimistic)
           button → [ ♡ Interested · 48 ]
           (no prompt; any calendar entry already added is left untouched)
```

- The **Add-to-calendar prompt appears only on the first toggle to ON** — never on un-toggling, never repeatedly.
- The calendar hand-off is a **client-side deep link** (no backend) — PetApp does not track whether the user actually saved it.
- On API error → revert the optimistic toggle + count, show a toast.
- The same toggle (without the count) is available as the quick `[ ♡ Interested ]` button on `screen_23` rows.

---

## API Endpoints Required

> All calls go to `POST /graphql`. **Auth:** Required.
> Reuses `Events (CI)` (`screen_17`) for listings. New endpoints below.

---

### CJ. Query: `Event`

> ⏳ GAP petapp-be#428, epic #401 — `event` query chưa có ở backend; events domain chưa build.

Fetch a single event for the detail screen, including the viewer's interest state.

**Operation:**
```graphql
query Event($eventId: ID!, $originLat: Float, $originLng: Float) {
  event(eventId: $eventId, originLat: $originLat, originLng: $originLng) {
    id
    title
    photos
    description
    startAt
    endAt
    price
    isFree
    venueName
    address
    cityShortName
    cityCode
    country
    countryCode
    lat
    lng
    distanceKm
    interestedCount
    viewerInterested
  }
}
```

**Variables:**
```json
{ "eventId": "event_001", "originLat": 10.78, "originLng": 106.70 }
```

**Response `200 OK`:**
```json
{
  "data": {
    "event": {
      "id": "event_001",
      "title": "Cat Adoption Day",
      "photos": [
        "https://cdn.petapp.com/events/event_001/1.jpg",
        "https://cdn.petapp.com/events/event_001/2.jpg"
      ],
      "description": "Ngày hội nhận nuôi mèo do PetApp tổ chức. Mang theo CMND nếu muốn nhận bé về…",
      "startAt": "2026-06-15T02:00:00Z",
      "endAt": "2026-06-15T05:00:00Z",
      "price": "Free",
      "isFree": true,
      "venueName": "Lava Cat Coffee",
      "address": "12 Nguyễn Huệ, Bến Nghé, Quận 1, HCMC",
      "cityShortName": "HCMC",
      "cityCode": "HCM",
      "country": "Việt Nam",
      "countryCode": "VN",
      "lat": 10.7731,
      "lng": 106.7042,
      "distanceKm": 3.0,
      "interestedCount": 48,
      "viewerInterested": false
    }
  }
}
```

**Notes:**
- `distanceKm` from `originLat`/`originLng` (user GPS); when absent, the server uses the city centre. Display-only.
- `viewerInterested` drives the heart state; `interestedCount` is the number on the button.
- A `FOUND`-style "ended" event still resolves but renders the ended/past state (see Edge Cases).

**Errors:**

| Status | Code | Scenario |
|--------|------|----------|
| `404` | `EVENT_NOT_FOUND` | Event does not exist or is unpublished |

---

### CK. Mutation: `SetEventInterest`

> ⏳ GAP petapp-be#428, epic #401 — `setEventInterest` mutation và `SetEventInterestInput` chưa có ở backend; events domain chưa build.

Set or clear the caller's Interested state for an event.

**Auth:** Required

**Operation:**
```graphql
mutation SetEventInterest($input: SetEventInterestInput!) {
  setEventInterest(input: $input) {
    eventId
    viewerInterested
    interestedCount
  }
}
```

**Variables:**
```json
{ "input": { "eventId": "event_001", "interested": true } }
```

**Response `200 OK`:**
```json
{
  "data": {
    "setEventInterest": {
      "eventId": "event_001",
      "viewerInterested": true,
      "interestedCount": 49
    }
  }
}
```

**Notes:**
- Idempotent — setting `interested: true` when already interested is a no-op (returns current state); same for `false`.
- The **add-to-calendar** step is purely client-side (a calendar deep link) and is **not** part of this mutation.

**Errors:**

| Status | Code | Scenario |
|--------|------|----------|
| `401` | `UNAUTHENTICATED` | Not logged in |
| `404` | `EVENT_NOT_FOUND` | Event does not exist or is unpublished |

---

## User Flow Diagrams

### Open event

```
Tap an Event row (screen_17 section / screen_23 list)
  └─> Event (CJ) { eventId, originLat?, originLng? }
        └─> render carousel + title + countdown + location (venue + full address) + about
              └─> bottom bar: viewerInterested ? [ ♥ Interested · N ] : [ ♡ Interested · N ]
```

### Mark Interested + calendar

```
Tap [ ♡ Interested · N ]
  ├─ off → on → SetEventInterest (CK) { interested: true }  (optimistic, N+1)
  │        └─> "Add this event to your calendar?"
  │              ├─ Add    → device calendar deep link (prefilled)
  │              └─ Not now → dismiss
  └─ on → off → SetEventInterest (CK) { interested: false } (optimistic, N−1; no prompt)
```

---

## Edge Cases & Notes

| Case | Expected Behaviour |
|------|--------------------|
| Not logged in | Redirect to Login (under the More tab) |
| Event unpublished / removed | `EVENT_NOT_FOUND` → "This event is no longer available" |
| Event already ended (deep link to a past event) | Render a read-only **ended** state (e.g. *"This event has ended"*); countdown hidden; Interested disabled |
| `description` empty | About section omitted |
| Single photo | Carousel shows one static image |
| `price` empty | Render `"Free"` |
| Toggle Interested, API fails | Revert optimistic count + heart; toast |
| Add-to-calendar prompt | Shown **only** on first toggle to ON; choosing "Not now" keeps interest ON |
| Un-toggle Interested | No prompt; calendar entry already added is left untouched (client can't revoke it) |
| GPS unavailable | `distanceKm` computed from city centre |

---

## Decisions Log

| # | Question | Decision |
|---|----------|----------|
| 1 | Event source | Admin-entered (no AI, no external import, **not** charity/family-hosted) |
| 2 | Screen purpose | Read-only display; the only interaction is Interested |
| 3 | Map | **Map with a pin** at the venue (consistent with all other More screens); tap → open device map. No separate Directions button (tapping the map covers it) |
| 4 | Share | **Removed** this phase |
| 5 | Full address | Shown only on this detail screen; rows abbreviate to `venueName · city` |
| 6 | Interested | Single action with inline count; toggle via `SetEventInterest (CK)` |
| 7 | Add-to-calendar | Prompted **only** on first toggle to ON; client-side calendar deep link (no backend tracking); not re-prompted; un-toggling doesn't touch the calendar |
| 8 | Countdown | `Starts in …` before start, `Ends in …` / `Happening now` while ongoing, hidden once ended (shared rule, `screen_17` §7) |
| 9 | Ended events | Resolve via deep link but render a read-only "ended" state; dropped from all listings |

---

## Open Items (next steps)

- **Ticketing / RSVP "Going"** — only a single **Interested** signal this phase; richer RSVP or paid ticketing could come later.
- **Share / web-viewable event page** — removed this phase; could mirror `screen_19` if needed later.
- **Full-screen map** — the detail shows a single-pin map; the shared **Map View** (`screen_28`) is opened via `[Open map]` on the Events list (`screen_23`).
- **Admin CMS** for event create/edit — out of scope of these client specs.
