# Screen 23: Events

## Overview

Full list of **upcoming events in the selected city**, grouped by time window and sorted soonest-first. This is the "View All" destination of the Events section on the **More** tab (`screen_17`).

Navigated to from:
- More tab → **Events** category icon, or
- More tab → Events section → **"View All →"**.

Both pass the **selected city** from the More tab's location bar. The list is **city-scoped** and **time-sorted** (`startAt` asc); only **upcoming/ongoing** events (`endAt >= now`) are shown. A **map preview** shows each event's venue (consistent with the other More screens). Distance is **display-only** — events are still **sorted by time**, never by distance (unlike Pet Friendly); the map and distance are just orientation aids.

All events are **admin-entered** (no AI, no charity/family hosting — see `screen_24` field model).

Requires login. Keeps the bottom navigation (More tab stays active) — it's a sub-screen of More.

---

## UI Layout

```
[Header]
  Left: Back button
  Center: "Events"

[Filter chips — horizontal scroll]
  [All]  [This week]  [This weekend]  [Free]

[Map preview]
  ┌──────────────────────────────────────────────┐
  │  (map with pins at each event's venue)        │
  │                                  [Open map]   │
  │  ┌──────────────────────┐                     │
  │  │ 🗓️ 8 events near you  │                     │
  │  └──────────────────────┘                     │
  └──────────────────────────────────────────────┘

[List — grouped by time window, soonest first, infinite scroll]
  ─── This weekend ───
  ┌────────────────────────────────────────────────────────┐
  │ ┌─────┐  Cat Adoption Day                      2.1km    │
  │ │ JUN │  Sat · 9:00–12:00                               │
  │ │ 15  │  Lava Cat Coffee · HCMC                         │
  │ └─────┘  Free · ⏳ Starts in 2d        [ ♡ Interested ] │
  └────────────────────────────────────────────────────────┘
  ─── Next week ───
  ┌────────────────────────────────────────────────────────┐
  │ ┌─────┐  Dog Run Meetup                        5.4km    │
  │ │ JUN │  Tue · 17:30–19:00                              │
  │ │ 18  │  Tao Đàn Park · HCMC                            │
  │ └─────┘  Free · ⏳ Starts in 5d        [ ♡ Interested ] │
  ├────────────────────────────────────────────────────────┤
  │ ┌─────┐  Pet Health Workshop                   3.0km    │
  │ │ JUN │  Sat · 14:00–16:00                              │
  │ │ 22  │  PetCare Clinic Q1 · HCMC                       │
  │ └─────┘  200kđ · ⏳ Starts in 9d       [ ♥ Interested ] │
  └────────────────────────────────────────────────────────┘

[Bottom Navigation]
  My Pets | Explore | Shops | Services | More (active)
```

---

## Components

### 1. Filter Chips

Horizontal scrollable row. **Single-select** — exactly one chip active at a time (default **All**). Tapping a chip re-runs `Events (CI)` with the matching filter and resets pagination.

| Chip | Filter sent | Dimension |
|------|-------------|-----------|
| **All** | *(none)* | — (default) |
| **This week** | `{ timeWindow: "THIS_WEEK" }` | time |
| **This weekend** | `{ timeWindow: "THIS_WEEKEND" }` | time |
| **Free** | `{ isFree: true }` | price |

> **No category chips** (Adoption / Meetup / Workshop …) this phase — decision: keep filters to time + price only. Single-select keeps it simple; combining dimensions can come later.

---

### 2. Map Preview

A compact map showing each event's venue in the selected city (after the active filter).

| Element | Description |
|---------|-------------|
| Pins | One pin per event, at `lat` / `lng` (from `Events (CI)`) |
| `[Open map]` button | Top-right — opens the shared **Map View** (`screen_28`) of the same pins (clustered when dense) |
| "N events near you" chip | Bottom-left — count of events currently plotted |

**Centring:** GPS granted → centre on the user; GPS unavailable → centre on the selected city's centre (`Cities (CA)` lat/lng).

**Pin interaction:** tap a pin → mini card (date-chip + title + time + distance) → tap card → **Event Detail** (`screen_24`).

**Full-screen map (`[Open map]`):** opens the shared **Map View** (`screen_28`) with `category: EVENTS`, the current city + filter, and the same pin set (themed pins + synced card carousel). Data identical to this screen.

> The map is an orientation aid only — the list stays **time-sorted** (`startAt` asc), never reordered by distance.

---

### 3. Events List

Full, paginated list of upcoming events in the selected city (after the active filter), **grouped by time window** with sticky-ish group headers.

**Grouping:** rows are bucketed by `startAt` relative to now — e.g. **Today**, **Tomorrow**, **This weekend**, **Next week**, then **{Month}** for anything further out. Within and across groups the order is `startAt` asc (soonest first). Group headers are derived client-side from `startAt`.

**Row** — the canonical **Event item** defined in `screen_17` → Section 7:

```
[date-chip]   {title}                            {distanceKm}km
              {weekday} · {startTime}–{endTime}
              {venueName} · {cityShortName}
              {price} · {countdown}          [ ♡ Interested ]
```

| Element | Source | Notes |
|---------|--------|-------|
| date-chip | `startAt` | Calendar-style chip: short month + day (e.g. `JUN 15`) |
| `title` | `title` | Event name (bold, line 1) |
| `distanceKm` | computed per request | top-right; `< 10` → 1 decimal (`2.1km`), `≥ 10` → integer (`120km`); **display-only** |
| weekday · time | `startAt`, `endAt` | `"Sat · 9:00–12:00"` |
| `venueName` · `cityShortName` | event location | `"Lava Cat Coffee · HCMC"` (short venue + city; full address only on `screen_24`) |
| `price` | `price` / `isFree` | `isFree` → `"Free"`; else the admin `price` text |
| `countdown` | `startAt` / `endAt` | See **Countdown rule** (`screen_17` §7) — `Starts in …` / `Ends in …` / `Happening now` |

**Quick Interested toggle (`[ ♡ Interested ]` on the row):**
- Initial heart state comes from `viewerInterested` on each `Events (CI)` item (filled `♥` when true, outline `♡` when false).
- Tap → toggles the viewer's interest via `SetEventInterest (CK)` (optimistic; `interestedCount` updates).
- Off → On: fills the heart (`♥`) and shows the **Add-to-calendar prompt** (same as `screen_24`); see `screen_24` → Interested interaction.
- On → Off: un-fills; **no prompt**, no change to any calendar entry already added.
- The row does **not** show the numeric count (kept compact); the full count lives on `screen_24`.

- **Pagination:** infinite scroll via `Events (CI)` `cursor`.
- **Tap a row** (anywhere except the Interested button) → **Event Detail** (`screen_24`).
- **Empty state** (filter matches nothing): *"No events match this filter."* (map + chips stay visible).

---

## API Endpoints Required

> All calls go to `POST /graphql`. **Auth:** Required.
> **No new query for the list** — reuses `Events (CI)` + `Cities (CA)` from `screen_17` (list + filter + pagination). The Interested toggle reuses `SetEventInterest (CK)` from `screen_24`. The event detail lives in `screen_24`.

---

## User Flow Diagrams

### Enter Events

```
More tab → "Events" icon  OR  "View All →"
  └─> Events screen (selected city passed in)
        └─> Events (CI) { cityCode, countryCode, originLat?, originLng?, [filter] }
              ├─ items → grouped list (startAt asc) + map pins
              └─ empty → empty state
```

### Filter

```
Tap a chip
  └─> Events (CI) with that filter (single-select) → reset pagination → re-render grouped list
  └─ "All" → no filter
```

### Toggle Interested (row)

```
Tap [ ♡ Interested ] on a row
  ├─ off → on → SetEventInterest (CK) { eventId, interested: true } (optimistic)
  │            └─> Add-to-calendar prompt (see screen_24)
  └─ on → off → SetEventInterest (CK) { eventId, interested: false } (no prompt)
```

### Open an event

```
Tap an event row (outside the Interested button)  OR  a map pin's card
  └─> Event Detail (screen_24) { eventId }
```

---

## Edge Cases & Notes

| Case | Expected Behaviour |
|------|--------------------|
| Not logged in | More tab (and this screen) redirect to Login |
| Selected city has 0 upcoming events | List empty state |
| Event ends while list is open | On refresh it drops out (`endAt >= now` only) |
| GPS denied | Distance from city centre (still shown); map centres on the city centre |
| GPS granted | Distance from user (display-only); map centres on the user; ordering unchanged (always `startAt` asc) |
| Filter matches nothing | "No events match this filter."; chips remain |
| Distance ≥ 10km | Integer format (`120km`); `< 10km` → 1 decimal (`2.1km`) |
| Event unpublished | Excluded from results (`isPublished = false`) |
| Ongoing event | Countdown shows `Ends in {N}h` / `Happening now`; still listed until `endAt` |

---

## Decisions Log

| # | Question | Decision |
|---|----------|----------|
| 1 | Scope | City-scoped (selected city); upcoming/ongoing only (`endAt >= now`) |
| 2 | Sort | `startAt` asc (soonest first); **not** distance-sorted |
| 3 | Grouping | List grouped by time window (Today / This weekend / Next week / {Month}); headers derived client-side from `startAt` |
| 4 | Filter chips | Single-select; time (`This week` / `This weekend`) + price (`Free`); **no category chips** this phase; `All` resets |
| 5 | Map | **Map preview kept** (consistent with all other More screens) — pins at each venue; orientation aid only, list stays time-sorted (not distance-sorted) |
| 6 | Distance | Display-only on rows; never a sort key |
| 7 | Row Interested | Quick toggle on the row (heart only, no count); first-time On triggers the Add-to-calendar prompt (shared with `screen_24`) |
| 8 | Row layout | Canonical Event item (`screen_17` §7): date-chip + title + distance / time / venue·city / price · countdown |

---

## Open Items (next steps)

- **Category filter / structured event types** — possible later enhancement (deliberately omitted this phase).
- **Calendar / agenda view toggle** — list-only this phase.
- **Full-screen map** (`[Open map]`) — done: shared **Map View** (`screen_28`).
