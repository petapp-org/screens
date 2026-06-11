# Screen 27: Manage Rescues

## Overview

A **charity family's** management view for the rescue listings it has posted — split into **Open** and **Adopted**, with quick actions to edit a listing, mark it adopted, or reopen it. This is where a charity closes a rescue once a pet finds a home.

**Access:** only members of a **charity family** (`familyType = CHARITY`), and only while **that charity is their active family**. A charity member whose active family is something else must switch active family first (mirrors the post/close permission rule). Non-charity users never reach this screen.

Navigated to from:
- **My Pets** (`screen_8`) → **Rescues** row (shown only when the active family is a charity),
- **Rescue** screen (`screen_25`) — charity members can jump here from the post banner area.

Requires login.

---

## UI Layout

```
[Header]
  Left: Back button
  Center: "Your Rescues"
  Right: [ + Post ]

[Tabs]
  [ Open · 5 ]      [ Adopted · 23 ]      ← counts per status

──────── OPEN tab ────────
  ┌──────────────────────────────────────────────────┐
  │ [photo]  Miu · Cat · ~4mo · ♀                     │
  │          Posted 3d ago · 12 inquiries             │
  │          [ Edit ]              [ Mark Adopted ]    │
  ├──────────────────────────────────────────────────┤
  │ [photo]  Bông · Dog · ~2mo · ♂                    │
  │          Posted 1w ago · 3 inquiries              │
  │          [ Edit ]              [ Mark Adopted ]    │
  └──────────────────────────────────────────────────┘
  (Empty: "No open rescues. Tap + Post to add one.")

──────── ADOPTED tab ────────
  ┌──────────────────────────────────────────────────┐
  │ [photo]  Lucky · Dog · ~1yr · ♂        🏡 Adopted │
  │          Adopted 2w ago                           │
  │          [ Reopen ]                               │
  └──────────────────────────────────────────────────┘
  (Empty: "No adopted rescues yet.")

[Bottom Navigation]
  My Pets (active) | Explore | Shops | Services | More
```

---

## Components

### 1. Header `[ + Post ]`

- Opens the **Create Rescue Form** (`screen_26` → Section 8) for the active charity family.
- Same form / endpoint (`CreateRescue (CO)`) as the Rescue screen banner.

---

### 2. Tabs (Open / Adopted)

- **Open** — listings with `status = OPEN` (publicly visible).
- **Adopted** — listings with `status = ADOPTED` (closed; removed from public listings but kept here as history).
- Each tab label shows a count. Switching tabs re-queries `MyRescues (CN)` with the matching status.

---

### 3. Listing Row

**Open row:**

| Element | Display |
|---------|---------|
| Thumbnail | `photos[0]` |
| Identity | `{name} · {species} · {ageText} · {gender}` (skip null parts) |
| Meta | `"Posted {relativeTime} ago · {inquiriesCount} inquiries"` |
| `[ Edit ]` | Opens the Create Rescue Form pre-filled (`UpdateRescue (CS)`) |
| `[ Mark Adopted ]` | Confirm → `CloseRescue (CP)` → moves to the Adopted tab |

**Adopted row:**

| Element | Display |
|---------|---------|
| Thumbnail + identity | same, with a muted **🏡 Adopted** badge |
| Meta | `"Adopted {relativeTime} ago"` |
| `[ Reopen ]` | Confirm → `ReopenRescue (CQ)` → back to Open + visible publicly again |

- **Tap a row** (outside the buttons) → **Rescue Detail** (`screen_26`) in manage context (Edit / Mark Adopted available there too).
- `inquiriesCount` lets the charity prioritise pets people are asking about.

---

## Actions & Permissions

All actions require the caller to be a member of the posting charity **with that charity active**:

| Action | Endpoint | Effect |
|--------|----------|--------|
| Post | `CreateRescue (CO)` (`screen_26`) | New `OPEN` listing |
| Edit | `UpdateRescue (CS)` | Edit fields (Create form in edit mode) |
| Mark Adopted | `CloseRescue (CP)` | `status → ADOPTED`; drops from public listings; share link → adopted state |
| Reopen | `ReopenRescue (CQ)` | `status → OPEN`; reappears in public listings |

---

## API Endpoints Required

> All calls go to `POST /graphql`. **Auth:** Required (charity member, charity active). Reuses `CreateRescue (CO)` (`screen_26`).

---

### CN. Query: `MyRescues`

> ⚠️ **GAP petapp-be#888:** backend has no `myRescues` query (charity's own listings). Kept as-is, pending backend.

List the **active charity family's** own rescue listings, filtered by status.

**Operation:**
```graphql
query MyRescues($status: RescueStatus!, $cursor: String, $limit: Int) {
  myRescues(status: $status, cursor: $cursor, limit: $limit) {
    items {
      id
      name
      species
      ageText
      gender
      thumbnailUrl
      status
      inquiriesCount
      createdAt
      adoptedAt
    }
    counts { open adopted }
    nextCursor
    hasMore
  }
}
```

**Variables:** `{ "status": "OPEN", "limit": 20 }`

**Notes:**
- Scoped server-side to the caller's **active charity family**; no `familyId` argument needed.
- `status`: `OPEN` | `ADOPTED`. `counts { open, adopted }` powers the tab labels.
- `adoptedAt` is set on Adopted rows (null while Open).

**Errors:**

| Code | Scenario |
|------|----------|
| `UNAUTHENTICATED` | Not logged in |
| `NOT_CHARITY_FAMILY` | Active family is not a charity (or caller not a member) |

---

### CP. Mutation: `CloseRescue`

Mark a listing as adopted. **Auth:** charity member, charity active.

```graphql
mutation CloseRescue($rescueId: ID!) {
  closeRescue(rescueId: $rescueId) { id status adoptedAt }
}
```

**Side effects:** `status = ADOPTED`; removed from the Rescue section/list/map (`Rescues (CL)` returns OPEN only); the **Rescue Detail** share link keeps resolving and renders the adopted state (`screen_26`); listing is never hard-deleted.

**Errors:**

| Status | Code | Scenario |
|--------|------|----------|
| `401` | `UNAUTHENTICATED` | Not logged in |
| `403` | `NOT_LISTING_OWNER` | Caller's active charity does not own this listing |
| `404` | `RESCUE_NOT_FOUND` | Listing does not exist |
| `409` | `ALREADY_ADOPTED` | Listing is already adopted |

---

### CQ. Mutation: `ReopenRescue`

Reopen an adopted listing. **Auth:** charity member, charity active.

```graphql
mutation ReopenRescue($rescueId: ID!) {
  reopenRescue(rescueId: $rescueId) { id status }
}
```

**Side effects:** `status = OPEN`; reappears in public listings.

**Errors:**

| Status | Code | Scenario |
|--------|------|----------|
| `401` | `UNAUTHENTICATED` | Not logged in |
| `403` | `NOT_LISTING_OWNER` | Caller's active charity does not own this listing |
| `404` | `RESCUE_NOT_FOUND` | Listing does not exist |
| `409` | `ALREADY_OPEN` | Listing is already open |

---

### CS. Mutation: `UpdateRescue`

Edit a listing (Create Rescue form in edit mode). **Auth:** charity member, charity active.

```graphql
mutation UpdateRescue($rescueId: ID!, $input: RescueInput!) {
  updateRescue(rescueId: $rescueId, input: $input) { id }
}
```

- `RescueInput` is the same shape as `CreateRescue (CO)` (`screen_26`).

**Errors:**

| Status | Code | Scenario |
|--------|------|----------|
| `401` | `UNAUTHENTICATED` | Not logged in |
| `403` | `NOT_LISTING_OWNER` | Caller's active charity does not own this listing |
| `404` | `RESCUE_NOT_FOUND` | Listing does not exist |
| `422` | `MISSING_PHOTOS` / `MISSING_SPECIES` / `MISSING_STORY` / `MISSING_LOCATION` | Required field cleared |

---

## User Flow Diagrams

### Open Manage Rescues

```
My Pets (active family = charity) → "Rescues" row
  └─> Manage Rescues
        └─> MyRescues (CN) { status: OPEN }  → Open tab + counts
              switch tab → MyRescues (CN) { status: ADOPTED }
```

### Mark adopted / reopen

```
[ Mark Adopted ] → confirm → CloseRescue (CP)  → row moves to Adopted; drops from public listings
[ Reopen ]       → confirm → ReopenRescue (CQ) → row moves to Open; visible publicly again
```

### Post / edit

```
[ + Post ]  → Create Rescue Form (screen_26 §8) → CreateRescue (CO)
[ Edit ]    → Create Rescue Form pre-filled       → UpdateRescue (CS)
```

---

## Edge Cases & Notes

| Case | Expected Behaviour |
|------|--------------------|
| Active family not a charity | Screen not reachable (no Rescues row in My Pets); direct nav → `NOT_CHARITY_FAMILY` |
| Charity member, charity not active | Must switch active family to the charity to manage |
| Open tab empty | "No open rescues. Tap + Post to add one." |
| Adopted tab empty | "No adopted rescues yet." |
| Mark Adopted | Confirm dialog; on success row moves to Adopted; public listings drop it |
| Reopen | Confirm dialog; on success row returns to Open |
| Edit clears a required field | Submit disabled / server `MISSING_*` error |
| Listing already adopted (double action) | `ALREADY_ADOPTED` / `ALREADY_OPEN` guard |
| `inquiriesCount` | Helps prioritise; tap row → detail to read the inquiry threads (via Messages) |

---

## Decisions Log

| # | Question | Decision |
|---|----------|----------|
| 1 | Access | Charity family members only, **while that charity is active**; entry via My Pets "Rescues" row |
| 2 | Structure | Two tabs — **Open** / **Adopted** — with counts |
| 3 | Close | **Mark Adopted** (`CloseRescue CP`) → `ADOPTED`; drops from public listings, share link still resolves; never hard-deleted |
| 4 | Reopen | **Reopen** (`ReopenRescue CQ`) → back to `OPEN` |
| 5 | Edit | Reuses the Create Rescue form in edit mode (`UpdateRescue CS`) |
| 6 | Inquiries | Show `inquiriesCount` on Open rows so charities can prioritise (from `RescueInquiry`, `screen_26`) |
| 7 | Own-listing scope | All mutations server-scoped to the caller's active charity; `NOT_LISTING_OWNER` otherwise |

---

## Open Items (next steps)

- **Per-listing inquiry list** — drill into who inquired (currently the count only; threads live in Messages `screen_10`).
- **Bulk actions** — mark multiple adopted at once (not this phase).
