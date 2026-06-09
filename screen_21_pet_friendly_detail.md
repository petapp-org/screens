# Screen 21: Pet Friendly Place Detail

## Overview

Detail view of a **single pet-friendly place** — its info, location, and **user ratings + reviews**.

Navigated to from:
- Pet Friendly section row on **More** (`screen_17`),
- Pet Friendly list row or map pin on the **Pet Friendly** screen (`screen_20`).

Requires login (it lives under the More tab). Reading reviews + viewing info needs login like the rest of More; writing a review also requires login (always true here).

This screen is also where the place's **rating data comes from**: `avgRating` / `reviewCount` shown on the rows in `screen_17` / `screen_20` are aggregated from the reviews submitted here.

---

## Place Data Model (admin-entered)

Places are **created/edited by admins** (no AI generation, no external provider import). This is the canonical field set; the rows (`screen_17` §6) and this screen read from it.

| Group | Field | Req | Notes |
|-------|-------|-----|-------|
| **Basics** | `name` | ✅ | Display name |
| | `category` | ✅ | enum: `CAFE` / `RESTAURANT` / `HOTEL` / `PARK` / `BEACH` / `OTHER` → drives category filter chips |
| | `photos[]` | ✅ | ≥ 1; first photo = thumbnail/cover; rest shown in the detail carousel |
| | `description` | — | Longer free text, shown on this detail screen |
| **Location** | `city` | ✅ | Picked from supported Cities (`CA`) → derives `cityShortName` / `cityCode` / `country` / `countryCode` |
| | `address` | — | Free text |
| | `lat` / `lng` | ✅ | Map pin → distance + map marker |
| **Pet** | `suitableFor` | ✅ | multi-select `CAT` / `DOG` / `OTHER` → display **Cat / Dog / Pet friendly**; drives the suitability chips |
| | `tagText` | — | **≤ 18 chars** free text (AA) — short pet features after the suitability label |
| | `highlightText` | — | **≤ 20 chars** free text (BB) — one "good to know" note after city-country |
| **Status** | `isPublished` | ✅ | Hidden from all listings when `false` |
| **Computed** | `avgRating`, `reviewCount` | — | Aggregated from user reviews (not admin-entered) |

**Admin form hints (placeholder/tooltip shown before typing):**
- `tagText` — *"Tối đa 18 ký tự. Vài đặc điểm pet-friendly cách nhau bằng ` · `. VD: `Carriers · <10kg`, `Patio · dogs OK`, `Off-leash 5–8am`."*
- `highlightText` — *"Tối đa 20 ký tự. Một điểm nổi bật để khách biết. VD: `play with 12 cats`, `large open area`, `200kđ pet fee`."*

> Both fields have a live character counter (`N/18`, `N/20`) in the admin form, and the app renders them **single-line + `…`** so they never wrap on mobile. `tagText` is the tighter of the two because its prefix (`suitable`, up to "Pet friendly") is longer than the city-country prefix.

---

## UI Layout

```
[Header]
  Left: Back button
  Center: place name (static)

[Photo carousel]                                 ← place.photos, swipeable, tap → zoom
  [ ◀   photo   ▶ ]   1/4

[Title block]
  Lava Cat Coffee                          ★ 4.8 · 126 reviews
  Café · Cat

[INFO]
  📍 12 Nguyễn Huệ, HCMC, VN                          3.4km
  [ map with pin ]            [ Directions ]
  open until 22:00                                    ← highlightText
  Cozy cat café with 12 resident cats…               ← description

[REVIEWS]
  ★ 4.8 · 126 reviews
  [ Write a review ]   ← or [ Edit your review ]
  ──────────────────────────────────────────────
  [avatar] Mai Anh        ★★★★★            2d
  "Tụi mèo siêu cute, nhân viên thân thiện 😍"
  ──────────────────────────────────────────────
  [avatar] Quang          ★★★★☆            1mo
  "Cà phê ổn, hơi đông cuối tuần."
  [ Load more ]
```

---

## Components

### 1. Photo Carousel

- Renders `photos[]`, swipeable, `N/Total` indicator.
- Tap a photo → fullscreen zoom (lightbox); tap outside / × to exit.
- Single photo → static image.

---

### 2. Title Block

| Element | Source |
|---------|--------|
| `name` | place name (large) |
| `★ avgRating · N reviews` | computed; tap → scroll to Reviews section |
| `category · suitable` | e.g. `"Café · Cat"`; `suitable` from `suitableFor` (Cat/Dog/Pet friendly) |

---

### 3. Info

| Element | Display |
|---------|---------|
| Address | `📍 {address}, {cityShortName}, {countryCode}` |
| `distanceKm` | Right of address; `< 10` → 1 decimal, `≥ 10` → integer |
| Map | Pin at `lat`/`lng`; **[Directions]** opens the device map app to the pin |
| `highlightText` | The "good to know" note (e.g. `"open until 22:00"`) |
| `description` | Full free text (no truncation) — omitted if empty |

---

### 4. Reviews

**Summary:** `★ avgRating · N reviews` (aggregated from **all** reviews — see rules).

**List** (`PlaceReviews (CF)`, paginated, newest first):

| Field | Display |
|-------|---------|
| `author.avatarUrl` + `displayName` | Reviewer; tap name → User Posts (`screen_12`) |
| `rating` | 1–5 stars |
| `comment` | Review text |
| `createdAt` | Relative time (same rules as post cards) |

- A reviewer with multiple reviews over time shows **multiple rows** (each is a point-in-time review).
- The viewer's **own latest** review shows an **Edit** affordance; the viewer's **older** reviews are read-only.

**Write / Edit review CTA:**

| Condition | Button | Action |
|-----------|--------|--------|
| No review yet, **or** the viewer's latest review is **≥ 1 month** old | **"Write a review"** | Opens the review form → `SubmitPlaceReview (CG)` (creates a **new** review) |
| The viewer's latest review is **< 1 month** old | **"Edit your review"** | Opens the form pre-filled with the latest review → `UpdatePlaceReview (CH)` (edits the **latest**) |

**Review form** (bottom sheet):
```
Rate Lava Cat Coffee
  ★ ★ ★ ★ ★        ← tap to set 1–5 (required)
  [ Comment textarea — required ]
  [ Submit ]
```
- Both **rating** and **comment** are required (a rating must carry a comment — per the decision that ratings always have text).
- Optimistic insert/update; on error revert + toast.

---

## Rating & Review Rules

| Rule | Detail |
|------|--------|
| Multiple reviews per user/place | Allowed — a user can review the same place again over time |
| Spacing | A new review is allowed only if the user's **latest** review on this place is **≥ 1 month** old (or they have none) |
| Editing | Only the user's **latest** review is editable; older ones are locked history |
| `avgRating` | Average of **all** reviews on the place (each review is a point-in-time data point); `reviewCount` = total reviews |
| Auth | Writing/editing requires login (always true on this screen) |

---

## API Endpoints Required

> All calls go to `POST /graphql`. **Auth:** Required.
> Reuses `PetFriendlyPlaces (CD)` (`screen_17`) for listings. New endpoints below.

---

### CE. Query: `PetFriendlyPlace`

Fetch a single place for the detail screen, including the viewer's review state.

**Operation:**
```graphql
query PetFriendlyPlace($placeId: ID!, $originLat: Float, $originLng: Float) {
  petFriendlyPlace(placeId: $placeId, originLat: $originLat, originLng: $originLng) {
    id
    name
    category
    photos
    description
    address
    cityShortName
    cityCode
    country
    countryCode
    lat
    lng
    distanceKm
    suitableFor
    tagText
    highlightText
    avgRating
    reviewCount
    viewerReview {
      id
      rating
      comment
      createdAt
    }
    canAddReview
  }
}
```

**Response `200 OK`:**
```json
{
  "data": {
    "petFriendlyPlace": {
      "id": "place_001",
      "name": "Lava Cat Coffee",
      "category": "CAFE",
      "photos": [
        "https://cdn.petapp.com/places/place_001/1.jpg",
        "https://cdn.petapp.com/places/place_001/2.jpg"
      ],
      "description": "Cozy cat café with 12 resident cats…",
      "address": "12 Nguyễn Huệ",
      "cityShortName": "HCMC",
      "cityCode": "HCM",
      "country": "Việt Nam",
      "countryCode": "VN",
      "lat": 10.7731,
      "lng": 106.7042,
      "distanceKm": 3.4,
      "suitableFor": ["CAT"],
      "tagText": "Carriers OK",
      "highlightText": "open until 22:00",
      "avgRating": 4.8,
      "reviewCount": 126,
      "viewerReview": {
        "id": "rev_555",
        "rating": 5,
        "comment": "Tụi mèo siêu cute 😍",
        "createdAt": "2026-05-01T09:00:00Z"
      },
      "canAddReview": true
    }
  }
}
```

**Notes:**
- `viewerReview` is the viewer's **latest** review on this place (`null` if none). It is the only one they can edit.
- `canAddReview` = `true` when there is no `viewerReview` **or** its `createdAt` is ≥ 1 month ago. Drives the CTA ("Write" vs "Edit").
- `avgRating` / `reviewCount` aggregate all reviews.

**Errors:**

| Status | Code | Scenario |
|--------|------|----------|
| `404` | `PLACE_NOT_FOUND` | Place does not exist or is unpublished |

---

### CF. Query: `PlaceReviews`

Paginated reviews for a place, newest first.

**Operation:**
```graphql
query PlaceReviews($placeId: ID!, $cursor: String, $limit: Int) {
  placeReviews(placeId: $placeId, cursor: $cursor, limit: $limit) {
    items {
      id
      author { id displayName avatarUrl }
      rating
      comment
      createdAt
      isOwn
      isEditable
    }
    nextCursor
    hasMore
    totalCount
  }
}
```

**Notes:**
- `isOwn` → authored by the viewer. `isEditable` → `true` only for the viewer's **latest** review (others, including older own reviews, are read-only).
- Sorted `createdAt` desc.

**Response `200 OK`:**
```json
{
  "data": {
    "placeReviews": {
      "items": [
        {
          "id": "rev_555",
          "author": { "id": "user_001", "displayName": "Mai Anh", "avatarUrl": "https://cdn.petapp.com/users/user_001/avatar.jpg" },
          "rating": 5,
          "comment": "Tụi mèo siêu cute, nhân viên thân thiện 😍",
          "createdAt": "2026-06-07T09:00:00Z",
          "isOwn": true,
          "isEditable": true
        }
      ],
      "nextCursor": "eyJpZCI6InJldl81NTUifQ==",
      "hasMore": true,
      "totalCount": 126
    }
  }
}
```

**Errors:**

| Status | Code | Scenario |
|--------|------|----------|
| `404` | `PLACE_NOT_FOUND` | Place does not exist or is unpublished |

---

### CG. Mutation: `SubmitPlaceReview`

Create a **new** review (rating + comment).

**Server validates:** the caller's latest review on this place is ≥ 1 month old (or none). Else → `REVIEW_TOO_SOON`.

**Operation:**
```graphql
mutation SubmitPlaceReview($input: PlaceReviewInput!) {
  submitPlaceReview(input: $input) {
    id
    rating
    comment
    createdAt
  }
}
```

**Variables:**
```json
{ "input": { "placeId": "place_001", "rating": 5, "comment": "Tụi mèo siêu cute 😍" } }
```

**Response `200 OK`:**
```json
{
  "data": {
    "submitPlaceReview": {
      "id": "rev_556",
      "rating": 5,
      "comment": "Tụi mèo siêu cute 😍",
      "createdAt": "2026-06-09T01:00:00Z"
    }
  }
}
```

**Errors:**

| Status | Code | Scenario |
|--------|------|----------|
| `401` | `UNAUTHENTICATED` | Not logged in |
| `404` | `PLACE_NOT_FOUND` | Place does not exist or is unpublished |
| `409` | `REVIEW_TOO_SOON` | Latest review is < 1 month old — must edit it instead (returns `nextEligibleAt`) |
| `422` | `EMPTY_COMMENT` | Comment is empty |
| `422` | `INVALID_RATING` | Rating not in 1–5 |

---

### CH. Mutation: `UpdatePlaceReview`

Edit the caller's **latest** review on a place.

**Operation:**
```graphql
mutation UpdatePlaceReview($input: UpdatePlaceReviewInput!) {
  updatePlaceReview(input: $input) {
    id
    rating
    comment
    createdAt
  }
}
```

**Variables:**
```json
{ "input": { "reviewId": "rev_556", "rating": 4, "comment": "Cập nhật: vẫn ổn 👍" } }
```

**Response `200 OK`:** the updated review.

**Errors:**

| Status | Code | Scenario |
|--------|------|----------|
| `401` | `UNAUTHENTICATED` | Not logged in |
| `403` | `NOT_OWN_REVIEW` | Review was not authored by the caller |
| `409` | `NOT_LATEST_REVIEW` | Only the caller's latest review is editable |
| `422` | `EMPTY_COMMENT` | Comment is empty |
| `422` | `INVALID_RATING` | Rating not in 1–5 |

---

## User Flow Diagrams

### Open place

```
Tap a Pet Friendly row / map pin
  └─> PetFriendlyPlace (CE) { placeId, originLat?, originLng? }
        └─> render info + map
              └─> PlaceReviews (CF) { placeId, limit } → review list
                    └─ canAddReview ? "Write a review" : "Edit your review"
```

### Write / edit review

```
Tap [Write a review] / [Edit your review]
  └─> review form (★ 1–5 + comment, both required)
        ├─ Write → SubmitPlaceReview (CG)
        │      └─ REVIEW_TOO_SOON → switch to edit-latest, toast "You reviewed recently — edit instead"
        └─ Edit  → UpdatePlaceReview (CH)  (latest only)
              └─> optimistic insert/update; recompute avgRating on refetch
```

---

## Edge Cases & Notes

| Case | Expected Behaviour |
|------|--------------------|
| No reviews yet | Reviews section shows "Be the first to review"; CTA = "Write a review" |
| Viewer's latest review < 1 month | CTA = "Edit your review"; SubmitPlaceReview would return `REVIEW_TOO_SOON` |
| Viewer's latest review ≥ 1 month | CTA = "Write a review" (new); previous reviews become locked history |
| Viewer edits an older own review | Not allowed — only the latest is editable (`isEditable=false`) |
| Empty comment on submit | Blocked client-side; server returns `EMPTY_COMMENT` |
| Place unpublished / removed | `PLACE_NOT_FOUND` → "This place is no longer available" |
| Single photo | Carousel shows one static image |
| Tap reviewer name | User Posts (`screen_12`) |
| GPS unavailable | `distanceKm` computed from city centre |

---

## Decisions Log

| # | Question | Decision |
|---|----------|----------|
| 1 | Place source | Admin-entered (no AI, no external provider) |
| 2 | AA / BB | Two free-text fields `tagText` (≤18) / `highlightText` (≤20) with admin tooltips + counters; single-line `…` on overflow |
| 3 | suitableFor display | `CAT`→Cat, `DOG`→Dog, `OTHER`→Pet friendly; multiple joined by ` · ` |
| 4 | Rating source | User reviews (not admin); `avgRating` = average of **all** reviews |
| 5 | Reviews per user | Multiple over time, each **≥ 1 month** apart |
| 6 | Editing | Only the user's **latest** review is editable |
| 7 | Rating requires comment | Yes — rating + comment both required |
| 8 | Auth | Requires login (under More tab) |

---

## Open Items (next steps)

- **Share / web-viewable place page** — not requested yet; could mirror `screen_19` if needed later.
- **Opening hours (structured)** — currently folded into `highlightText` free text; structured hours (live open/closed) is a possible later enhancement.
- **Admin CMS** for place create/edit — out of scope of these client specs.
